# 优化查询（解决N+1问题）

您是否注意到某些GraphQL查询需要执行数百个数据库查询，这些查询通常包含重复的数据，让我们来看看为什么以及如何修复它。

## 查询解析

想象一下，如果您有一个简单的查询，例如：

```graphql
query { todos { users { name } } }
```

实现`User`的resolver代码如下：

```rust
struct User {
    id: u64,
}

#[Object]
impl User {
    async fn name(&self, ctx: &Context<'_>) -> Result<String> {
        let pool = ctx.data_unchecked::<Pool<Postgres>>();
        let (name,): (String,) = sqlx::query_as("SELECT name FROM user WHERE id = $1")
            .bind(self.id)
            .fetch_one(pool)
            .await?;
        Ok(name)
    }
}
```

执行查询将调用`Todos`的resolver，该resolver执行`SELECT * FROM todo`并返回N个`Todo`对象。然后对每个`Todo`对象同时调用`User`的
resolver执行`SELECT name FROM user where id = $1`。

例如：

```sql
SELECT id, todo, user_id FROM todo
SELECT name FROM user WHERE id = $1
SELECT name FROM user WHERE id = $1
SELECT name FROM user WHERE id = $1
SELECT name FROM user WHERE id = $1
SELECT name FROM user WHERE id = $1
SELECT name FROM user WHERE id = $1
SELECT name FROM user WHERE id = $1
SELECT name FROM user WHERE id = $1
SELECT name FROM user WHERE id = $1
SELECT name FROM user WHERE id = $1
SELECT name FROM user WHERE id = $1
SELECT name FROM user WHERE id = $1
SELECT name FROM user WHERE id = $1
SELECT name FROM user WHERE id = $1
SELECT name FROM user WHERE id = $1
SELECT name FROM user WHERE id = $1
```

执行了多次`SELECT name FROM user WHERE id = $1`，并且，大多数`Todo`对象都属于同一个用户，我们需要优化这些代码！

## Dataloader

我们需要对查询分组，并且排除重复的查询。`Dataloader`就能完成这个工作，[facebook](https://github.com/facebook/dataloader) 给出了一个请求范围的批处理和缓存解决方案。

下面是使用`DataLoader`来优化查询请求的例子：

```rust
use async_graphql::*;
use async_graphql::dataloader::*;
use itertools::Itertools;

struct UserNameLoader {
    pool: sqlx::Pool<Postgres>,
}

#[async_trait::async_trait]
impl Loader for UserNameLoader {
    type Key = u64;
    type Value = String;
    type Error = sqlx::Error;
    
    async fn load(&self, keys: HashSet<Self::Key>) -> Result<HashMap<Self::Key, Self::Value>, Self::Error> {
        let pool = ctx.data_unchecked::<Pool<Postgres>>();
        let query = format!("SELECT name FROM user WHERE id IN ({})", keys.iter().join(","));
        Ok(sqlx::query_as(query)
            .fetch(&self.pool)
            .map_ok(|name: String| name)
            .try_collect().await?)
    }
}

struct User {
    id: u64,
}

#[Object]
impl User {
    async fn name(&self, ctx: &Context<'_>) -> Result<String> {
        let loader = ctx.data_unchecked::<DataLoader<UserNameLoader>>();
        let name: Option<String> = loader.load_one(self.id).await?;
        name.ok_or_else(|| "Not found".into())
    }
}
```

最终只需要两个查询语句，就查询出了我们想要的结果！

```sql
SELECT id, todo, user_id FROM todo
SELECT name FROM user WHERE id IN (1, 2, 3, 4)
```