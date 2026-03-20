# Database Integration & Caching in Rust

SQLx, Diesel, MongoDB, migrations, SQL injection prevention, in-memory and Redis caching, cache invalidation strategies, and chat/messaging schema patterns.

### Section Index

| Section | Content |
|---------|---------|
| [Rules for Database Code (LLM)](#rules-for-database-code-llm) | 8 rules, BAD/GOOD pairs for SQL injection, transactions, testing |
| [Database Integration](#database-integration) | SQLx async, connection pools, LazyLock, repository pattern |
| [Diesel ORM Patterns](#diesel-orm-patterns) | Schema, CRUD, associations, joins |
| [Schema Migrations](#schema-migrations) | sqlx migrate, Diesel migrations, embedded migrations |
| [SQL Injection Prevention](#sql-injection-prevention) | Parameterized queries, QueryBuilder for dynamic SQL |
| [Trait-Based Async Transactions](#trait-based-async-transactions-with-descriptors) | Transaction descriptors, trait-bounded executors |
| [Database Testing](#database-testing-with-sqlxtest) | `#[sqlx::test]`, isolated per-test databases |
| [SeaORM](#seaorm--activerecord-style-orm) | ActiveRecord-style ORM patterns |
| [MongoDB Integration](#mongodb-integration) | BSON, collections, CRUD, aggregation |
| [Caching Patterns](#caching-patterns) | In-memory (moka), Redis distributed cache, cached repository |
| [Cache Invalidation Strategies](#cache-invalidation-strategies) | TTL, event-driven, write-through, versioned keys |
| [Chat & Messaging Schema](#chat--messaging-schema-patterns) | Rooms, members, messages, cursor-based history, DMs, unread counts |

## Rules for Database Code (LLM)

1. **ALWAYS use parameterized queries** — never interpolate user input into SQL strings; `sqlx::query!("SELECT * FROM users WHERE id = $1", id)` prevents SQL injection
2. **ALWAYS set `DATABASE_URL` for compile-time checked queries** — `sqlx::query!` and `sqlx::query_as!` verify SQL at compile time against the database schema; without `DATABASE_URL` they fail to compile
3. **NEVER use `sqlx::query("...")` with string formatting** — `format!("SELECT * FROM {} WHERE ...", table)` bypasses parameterization; use the macro forms or `query_builder` for dynamic queries
4. **ALWAYS use connection pools (`PgPool`) instead of single connections** — pools handle connection reuse, timeouts, and recovery; set `max_connections` based on expected concurrency
5. **ALWAYS wrap multi-step mutations in transactions** — `pool.begin()` + `tx.commit()` ensures atomicity; without transactions, partial failures leave inconsistent state
6. **ALWAYS use `QueryBuilder` for dynamic queries** — never build SQL strings with `format!`; `QueryBuilder::push_bind()` prevents injection and respects database parameter limits (PostgreSQL: 65535, SQLite: 32766, MSSQL: 2100)
7. **ALWAYS use `#[sqlx::test]` for database tests** — it creates isolated per-test databases via SHA-512 hash naming, runs migrations, and cleans up automatically; never share test databases between tests
8. **PREFER `query_scalar!` for single-value queries** — `query_scalar!("SELECT count(*) FROM users")` returns the value directly without wrapping in a struct

### Common Mistakes (BAD/GOOD)

**String-formatted dynamic queries:**
```rust
// BAD: SQL injection via format! — never do this
async fn search(pool: &PgPool, table: &str, term: &str) -> Result<Vec<Row>, sqlx::Error> {
    let sql = format!("SELECT * FROM {} WHERE name LIKE '%{}%'", table, term);
    sqlx::query(&sql).fetch_all(pool).await
}

// GOOD: QueryBuilder with push_bind for safe dynamic queries
async fn search(pool: &PgPool, filters: &[(&str, &str)]) -> Result<Vec<UserRecord>, sqlx::Error> {
    let mut qb: QueryBuilder<Postgres> = QueryBuilder::new("SELECT * FROM users WHERE 1=1");
    for (col, val) in filters {
        qb.push(format!(" AND {} = ", col)); // column names from code, not user input
        qb.push_bind(val);                   // values safely bound
    }
    qb.build_query_as::<UserRecord>().fetch_all(pool).await
}
```

**Missing connection pool limits:**
```rust
// BAD: unbounded pool exhausts database connections
let pool = PgPool::connect(url).await?;

// GOOD: explicit limits matching your deployment
let pool = PgPoolOptions::new()
    .max_connections(10)
    .min_connections(2)
    .acquire_timeout(Duration::from_secs(3))
    .idle_timeout(Duration::from_secs(600))
    .connect(url).await?;
```

**Forgetting to commit transactions:**
```rust
// BAD: transaction auto-rolls back on drop — changes silently lost
async fn update(pool: &PgPool) -> Result<(), sqlx::Error> {
    let mut tx = pool.begin().await?;
    sqlx::query!("UPDATE items SET status = 'done' WHERE id = 1").execute(&mut *tx).await?;
    Ok(()) // tx dropped without commit!
}

// GOOD: explicit commit
async fn update(pool: &PgPool) -> Result<(), sqlx::Error> {
    let mut tx = pool.begin().await?;
    sqlx::query!("UPDATE items SET status = 'done' WHERE id = 1").execute(&mut *tx).await?;
    tx.commit().await?;
    Ok(())
}
```

## Database Integration

### SQLx Async Patterns

```rust
use sqlx::{PgPool, postgres::PgPoolOptions, FromRow};
use uuid::Uuid;

#[derive(Debug, FromRow)]
struct UserRecord {
    id: Uuid,
    username: String,
    email: String,
    created_at: chrono::DateTime<chrono::Utc>,
}

// Connection pool setup
async fn create_pool(database_url: &str) -> Result<PgPool, sqlx::Error> {
    PgPoolOptions::new()
        .max_connections(10)
        .acquire_timeout(Duration::from_secs(3))
        .connect(database_url)
        .await
}
```

### Global Connection Pool with LazyLock

For global access to the database pool without passing it through every function:

```rust
use sqlx::postgres::{PgPool, PgPoolOptions};
use std::sync::LazyLock;

/// Global database pool initialized on first access
pub static DB_POOL: LazyLock<PgPool> = LazyLock::new(|| {
    let connection_string = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    let max_connections = std::env::var("DB_MAX_CONNECTIONS")
        .unwrap_or_else(|_| "5".to_string())
        .parse::<u32>()
        .expect("DB_MAX_CONNECTIONS must be a valid number");

    PgPoolOptions::new()
        .max_connections(max_connections)
        .connect_lazy(&connection_string)
        .expect("Failed to create database pool")
});

// Usage anywhere in the codebase
async fn get_all_items() -> Result<Vec<Item>, sqlx::Error> {
    sqlx::query_as::<_, Item>("SELECT * FROM items")
        .fetch_all(&*DB_POOL)
        .await
}

// Compile-time checked queries with query_as!
async fn find_user_by_id(pool: &PgPool, id: Uuid) -> Result<Option<UserRecord>, sqlx::Error> {
    sqlx::query_as!(
        UserRecord,
        r#"
        SELECT id, username, email, created_at
        FROM users
        WHERE id = $1
        "#,
        id
    )
    .fetch_optional(pool)
    .await
}

// Transactions
async fn transfer_funds(
    pool: &PgPool,
    from_id: Uuid,
    to_id: Uuid,
    amount: i64,
) -> Result<(), sqlx::Error> {
    let mut tx = pool.begin().await?;

    sqlx::query!(
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
        amount, from_id
    )
    .execute(&mut *tx)
    .await?;

    sqlx::query!(
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        amount, to_id
    )
    .execute(&mut *tx)
    .await?;

    tx.commit().await?;
    Ok(())
}

// Streaming large result sets
use futures::stream::StreamExt;

async fn process_all_users(pool: &PgPool) -> Result<(), sqlx::Error> {
    let mut stream = sqlx::query_as::<_, UserRecord>(
        "SELECT id, username, email, created_at FROM users"
    )
    .fetch(pool);

    while let Some(user) = stream.next().await {
        let user = user?;
        println!("Processing user: {}", user.username);
    }
    Ok(())
}
```

### Repository Implementation with SQLx

Uses `#[async_trait]` here because repository traits are often used as `dyn UserRepository` for dependency injection. If you only use generics (`impl UserRepository`), prefer native async traits (Rust 1.75+) — see [domain-patterns.md](domain-patterns.md) for the native pattern.

```rust
use async_trait::async_trait;

pub struct SqlxUserRepository {
    pool: PgPool,
}

impl SqlxUserRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
}

#[async_trait]
impl UserRepository for SqlxUserRepository {
    async fn find_by_id(&self, id: UserId) -> Result<User, RepositoryError> {
        let record = sqlx::query_as!(
            UserRecord,
            "SELECT id, username, email, created_at FROM users WHERE id = $1",
            id.0 as i64
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(|e| RepositoryError::DatabaseError(Box::new(e)))?
        .ok_or(RepositoryError::NotFound)?;

        Ok(record.into())  // Convert UserRecord to User
    }

    async fn save(&self, user: &User) -> Result<(), RepositoryError> {
        sqlx::query!(
            r#"
            INSERT INTO users (id, username, email, created_at)
            VALUES ($1, $2, $3, $4)
            ON CONFLICT (id) DO UPDATE SET
                username = EXCLUDED.username,
                email = EXCLUDED.email
            "#,
            user.id.0 as i64,
            user.username,
            user.email,
            chrono::Utc::now()
        )
        .execute(&self.pool)
        .await
        .map_err(|e| RepositoryError::DatabaseError(Box::new(e)))?;

        Ok(())
    }
}
```

### Diesel ORM Patterns

Diesel provides compile-time checked queries with an ORM approach (vs SQLx's raw SQL).

```rust
// Cargo.toml dependencies
// diesel = { version = "2.1", features = ["postgres", "uuid", "chrono"] }
// diesel-async = { version = "0.4", features = ["postgres", "deadpool"] }

// schema.rs - generated by `diesel print-schema`
diesel::table! {
    users (id) {
        id -> Uuid,
        username -> Varchar,
        email -> Varchar,
        created_at -> Timestamp,
    }
}

// models.rs
use diesel::prelude::*;
use uuid::Uuid;

#[derive(Debug, Clone, Queryable, Selectable)]
#[diesel(table_name = crate::schema::users)]
pub struct UserRecord {
    pub id: Uuid,
    pub username: String,
    pub email: String,
    pub created_at: chrono::NaiveDateTime,
}

#[derive(Debug, Insertable)]
#[diesel(table_name = crate::schema::users)]
pub struct NewUserRecord<'a> {
    pub id: Uuid,
    pub username: &'a str,
    pub email: &'a str,
}
```

**Async Repository with diesel-async:**

```rust
use diesel::prelude::*;
use diesel_async::{AsyncPgConnection, RunQueryDsl};
use deadpool_diesel::postgres::Pool;

pub struct DieselUserRepository {
    pool: Pool,
}

impl DieselUserRepository {
    pub fn new(pool: Pool) -> Self {
        Self { pool }
    }
}

#[async_trait]
impl UserRepository for DieselUserRepository {
    async fn find_by_id(&self, user_id: UserId) -> Result<User, RepositoryError> {
        use crate::schema::users::dsl::*;

        let mut conn = self.pool.get().await
            .map_err(|e| RepositoryError::DatabaseError(Box::new(e)))?;

        let record = users
            .filter(id.eq(user_id.0))
            .select(UserRecord::as_select())
            .first(&mut conn)
            .await
            .optional()
            .map_err(|e| RepositoryError::DatabaseError(Box::new(e)))?
            .ok_or(RepositoryError::NotFound)?;

        Ok(record.into())
    }

    async fn save(&self, user: &User) -> Result<(), RepositoryError> {
        use crate::schema::users::dsl::*;

        let mut conn = self.pool.get().await
            .map_err(|e| RepositoryError::DatabaseError(Box::new(e)))?;

        let new_record = NewUserRecord {
            id: user.id.0,
            username: &user.username,
            email: &user.email,
        };

        diesel::insert_into(users)
            .values(&new_record)
            .on_conflict(id)
            .do_update()
            .set((
                username.eq(&user.username),
                email.eq(&user.email),
            ))
            .execute(&mut conn)
            .await
            .map_err(|e| RepositoryError::DatabaseError(Box::new(e)))?;

        Ok(())
    }

    async fn find_by_username(&self, name: &str) -> Result<Option<User>, RepositoryError> {
        use crate::schema::users::dsl::*;

        let mut conn = self.pool.get().await
            .map_err(|e| RepositoryError::DatabaseError(Box::new(e)))?;

        let record = users
            .filter(username.eq(name))
            .select(UserRecord::as_select())
            .first(&mut conn)
            .await
            .optional()
            .map_err(|e| RepositoryError::DatabaseError(Box::new(e)))?;

        Ok(record.map(User::from))
    }
}
```

**SQLx vs Diesel Comparison:**

| Aspect | SQLx | Diesel |
|--------|------|--------|
| Query style | Raw SQL strings | Type-safe DSL |
| Compile-time checks | Against live DB | Against schema.rs |
| Flexibility | Full SQL power | ORM abstractions |
| Learning curve | Lower (just SQL) | Higher (Diesel DSL) |
| Migrations | sqlx-cli | diesel_cli |
| Best for | Complex queries, raw perf | CRUD operations, type safety |

### Schema Migrations

Database schema evolution is critical for long-lived applications. Both sqlx and Diesel provide migration tools.

**SQLx Migrations:**

```
# Directory structure
migrations/
├── 20240101000000_create_users.sql
├── 20240102000000_add_posts.sql
└── 20240103000000_add_email_verified.sql

# Commands
sqlx migrate add create_users       # Create new migration
sqlx migrate run                    # Apply pending migrations
sqlx migrate revert                 # Rollback last migration
```

```sql
-- migrations/20240101000000_create_users.sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(255) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- migrations/20240102000000_add_posts.sql
CREATE TABLE IF NOT EXISTS posts (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

**Diesel Migrations:**

```
# Commands
diesel setup                        # Initialize database
diesel migration generate create_users  # Create migration files
diesel migration run                # Apply migrations
diesel migration revert             # Rollback
diesel print-schema                 # Generate schema.rs
```

```sql
-- up.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR NOT NULL,
    email VARCHAR NOT NULL
);

-- down.sql
DROP TABLE users;
```

**Zero-Downtime Migration Patterns:**

| Pattern | Description | Use Case |
|---------|-------------|----------|
| Expand/Contract | Add new structure, migrate data, remove old | Column renames, type changes |
| Blue-Green | Migrate DB ahead, switch traffic | Major schema changes |
| Feature Flags | Dual-write to old and new | Gradual rollout |

**Migration Best Practices:**

- **Atomicity**: Each migration should be a single atomic change
- **Reversibility**: Always provide down/rollback migrations
- **Idempotency**: Use `IF NOT EXISTS`, `IF EXISTS` clauses
- **Testing**: Test migrations in staging before production
- **Performance**: Schedule large table alterations during off-peak hours

### SQL Injection Prevention

SQL injection is when malicious SQL is inserted into query parameters. **Always use parameterized queries with bind**.

```rust
// BAD - SQL injection vulnerable
async fn find_user_bad(pool: &PgPool, username: &str) -> Result<User, sqlx::Error> {
    // NEVER do this - user input directly in query string
    let query = format!("SELECT * FROM users WHERE username = '{}'", username);
    sqlx::query_as::<_, User>(&query).fetch_one(pool).await
}
// Attack: username = "'; DROP TABLE users; --"

// GOOD - Parameterized query with bind
async fn find_user_good(pool: &PgPool, username: &str) -> Result<User, sqlx::Error> {
    sqlx::query_as::<_, User>(
        "SELECT * FROM users WHERE username = $1"
    )
    .bind(username)  // Sanitized by SQLX
    .fetch_one(pool)
    .await
}

// GOOD - Using query_as! macro (compile-time checked)
async fn find_user_macro(pool: &PgPool, username: &str) -> Result<User, sqlx::Error> {
    sqlx::query_as!(
        User,
        "SELECT * FROM users WHERE username = $1",
        username  // Automatically bound and sanitized
    )
    .fetch_one(pool)
    .await
}
```

**Key Rules:**
1. Never concatenate user input into SQL strings
2. Always use `$1`, `$2`, etc. placeholders with `.bind()`
3. SQLX handles escaping and type checking automatically
4. The `query!` and `query_as!` macros verify queries at compile time

### QueryBuilder for Safe Dynamic Queries

When queries have variable structure (dynamic WHERE clauses, IN lists, bulk inserts), use `QueryBuilder` instead of string formatting:

```rust
use sqlx::{QueryBuilder, Postgres};

// Dynamic WHERE clause with multiple optional filters
async fn search_users(
    pool: &PgPool,
    name: Option<&str>,
    email: Option<&str>,
    status: Option<&str>,
) -> Result<Vec<UserRecord>, sqlx::Error> {
    let mut qb: QueryBuilder<Postgres> = QueryBuilder::new(
        "SELECT id, username, email, created_at FROM users WHERE 1=1"
    );

    if let Some(name) = name {
        qb.push(" AND username ILIKE ");
        qb.push_bind(format!("%{name}%"));
    }
    if let Some(email) = email {
        qb.push(" AND email = ");
        qb.push_bind(email);
    }
    if let Some(status) = status {
        qb.push(" AND status = ");
        qb.push_bind(status);
    }

    qb.push(" ORDER BY created_at DESC LIMIT 100");
    qb.build_query_as::<UserRecord>().fetch_all(pool).await
}

// IN clause with separated()
async fn find_users_by_ids(
    pool: &PgPool,
    ids: &[Uuid],
) -> Result<Vec<UserRecord>, sqlx::Error> {
    let mut qb: QueryBuilder<Postgres> = QueryBuilder::new(
        "SELECT * FROM users WHERE id IN ("
    );

    let mut sep = qb.separated(", ");
    for id in ids {
        sep.push_bind(id);
    }
    sep.push_unseparated(")");

    qb.build_query_as::<UserRecord>().fetch_all(pool).await
}

// Bulk INSERT with push_values()
async fn insert_users_bulk(
    pool: &PgPool,
    users: &[NewUser],
) -> Result<(), sqlx::Error> {
    // Respect parameter limits: PostgreSQL allows 65535 bind params
    // With 3 columns per row, max ~21845 rows per batch
    const BATCH_SIZE: usize = 5000;

    for chunk in users.chunks(BATCH_SIZE) {
        let mut qb: QueryBuilder<Postgres> = QueryBuilder::new(
            "INSERT INTO users (username, email, status) "
        );
        qb.push_values(chunk, |mut sep, user| {
            sep.push_bind(&user.username);
            sep.push_bind(&user.email);
            sep.push_bind(&user.status);
        });
        qb.build().execute(pool).await?;
    }
    Ok(())
}
```

**Database parameter limits:**

| Database   | Max bind parameters |
|-----------|-------------------|
| PostgreSQL | 65,535            |
| MySQL      | 65,535            |
| SQLite     | 32,766            |
| MSSQL      | 2,100             |

### query_scalar! for Single Values

When you need a single value (count, sum, exists), use `query_scalar!` to avoid wrapping in a struct:

```rust
// Count users
let count: i64 = sqlx::query_scalar!("SELECT count(*) FROM users")
    .fetch_one(pool).await?;

// Check existence
let exists: bool = sqlx::query_scalar!(
    "SELECT EXISTS(SELECT 1 FROM users WHERE email = $1)",
    email
).fetch_one(pool).await?;

// Get single column
let usernames: Vec<String> = sqlx::query_scalar!(
    "SELECT username FROM users WHERE status = $1",
    "active"
).fetch_all(pool).await?;
```

### Embedded Migrations with sqlx::migrate!

Embed SQL migrations directly into the Rust binary for self-contained deployment:

```rust
use sqlx::PgPool;

/// Run all pending migrations embedded in the binary
pub async fn run_migrations(pool: &PgPool) {
    // Embeds migrations from ./migrations directory at compile time
    let mut migrator = sqlx::migrate!("./migrations");

    // Skip migrations that exist in DB but not in code (useful for rollbacks)
    migrator.ignore_missing = true;

    let result = migrator.run(pool).await;
    println!("Migration result: {:?}", result);
}

// Call before server starts
#[tokio::main]
async fn main() -> std::io::Result<()> {
    run_migrations(&*DB_POOL).await;

    HttpServer::new(|| {
        App::new().configure(api_routes)
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

**Migration File Structure:**

```
migrations/
├── 20240101120000_create_items.sql
├── 20240102120000_add_user_id.sql
└── 20240103120000_add_indexes.sql
```

```sql
-- migrations/20240101120000_create_items.sql
CREATE TABLE IF NOT EXISTS items (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- migrations/20240102120000_add_user_id.sql
ALTER TABLE items ADD COLUMN user_id INTEGER REFERENCES users(id);
```

**Benefits of Embedded Migrations:**
- Single binary deployment (migrations included)
- No separate migration step in deployment
- Version-controlled with code
- Compile-time embedding ensures migrations exist

### Trait-Based Async Transactions with Descriptors

Use descriptor structs and traits with `impl Future` for swappable storage backends:

```rust
use std::future::Future;

/// Marker struct for PostgreSQL storage
pub struct SqlxPostGresDescriptor;

/// Marker struct for JSON file storage
pub struct JsonFileDescriptor;

/// Trait for creating items - implementations vary by storage backend
pub trait CreateOne {
    fn create_one(item: NewItem) ->
        impl Future<Output = Result<Item, NanoServiceError>> + Send;
}

#[cfg(feature = "sqlx-postgres")]
impl CreateOne for SqlxPostGresDescriptor {
    fn create_one(item: NewItem) ->
        impl Future<Output = Result<Item, NanoServiceError>> + Send
    {
        sqlx_postgres_create_one(item)
    }
}

#[cfg(feature = "json-file")]
impl CreateOne for JsonFileDescriptor {
    fn create_one(item: NewItem) ->
        impl Future<Output = Result<Item, NanoServiceError>> + Send
    {
        json_file_create_one(item)
    }
}

// Implementation for PostgreSQL
#[cfg(feature = "sqlx-postgres")]
async fn sqlx_postgres_create_one(item: NewItem) -> Result<Item, NanoServiceError> {
    let created = sqlx::query_as::<_, Item>(
        "INSERT INTO items (title, status) VALUES ($1, $2) RETURNING *"
    )
    .bind(&item.title)
    .bind(&item.status.to_string())
    .fetch_one(&*SQLX_POSTGRES_POOL)
    .await
    .map_err(|e| NanoServiceError::new(e.to_string(), NanoServiceErrorStatus::Unknown))?;

    Ok(created)
}

// Generic handler uses trait bounds
pub async fn create<T: CreateOne + GetAll>(
    body: Json<NewItem>,
) -> Result<HttpResponse, NanoServiceError> {
    let _ = T::create_one(body.into_inner()).await?;
    let items = T::get_all().await?;
    Ok(HttpResponse::Created().json(items))
}

// Mount with specific descriptor type
app.route("/create", post().to(create::<SqlxPostGresDescriptor>))
```

**Cargo.toml Feature Gates:**

```toml
[features]
default = ["json-file"]
json-file = ["serde_json"]
sqlx-postgres = ["sqlx"]

[dependencies]
sqlx = { version = "0.8", features = ["postgres", "runtime-tokio"], optional = true }
serde_json = { version = "1.0", optional = true }
```

**Benefits:**
- Compile-time selection of storage backend
- No runtime overhead from dynamic dispatch
- Easy testing with mock/file-based storage
- Production deployment with real database

### Database Testing with #[sqlx::test]

The `#[sqlx::test]` macro creates an isolated database per test function, runs migrations, applies fixtures, and cleans up automatically:

```rust
// Cargo.toml: sqlx = { version = "0.8", features = ["runtime-tokio", "postgres"] }
// Requires DATABASE_URL with CREATE/DROP DATABASE privileges

#[sqlx::test]
async fn test_user_creation(pool: PgPool) {
    // pool connects to a fresh database named _sqlx_test_{sha512_hash}
    // Migrations from ./migrations are applied automatically

    let user = sqlx::query_as!(
        UserRecord,
        "INSERT INTO users (username, email) VALUES ($1, $2) RETURNING *",
        "alice", "alice@example.com"
    )
    .fetch_one(&pool)
    .await
    .unwrap();

    assert_eq!(user.username, "alice");
    // Database is dropped automatically after test
}

// With SQL fixtures — applied after migrations
#[sqlx::test(fixtures("users", "posts"))]
async fn test_user_posts(pool: PgPool) {
    // fixtures/users.sql and fixtures/posts.sql are executed before test
    let count: i64 = sqlx::query_scalar!("SELECT count(*) FROM posts")
        .fetch_one(&pool).await.unwrap();
    assert!(count > 0);
}

// With custom pool options
#[sqlx::test]
async fn test_with_connection(pool: PgPool) {
    let mut conn = pool.acquire().await.unwrap();
    sqlx::query!("SELECT 1 as one").fetch_one(&mut *conn).await.unwrap();
}
```

**Fixture files** go in `fixtures/` directory relative to the test file:

```sql
-- fixtures/users.sql
INSERT INTO users (id, username, email)
VALUES
    ('550e8400-e29b-41d4-a716-446655440001', 'alice', 'alice@test.com'),
    ('550e8400-e29b-41d4-a716-446655440002', 'bob', 'bob@test.com');
```

**Key behaviors:**
- Each test gets a unique database — tests run in parallel safely
- Failed tests preserve the database for debugging
- A global semaphore prevents exhausting database connections
- Pool is closed with a 10-second timeout after test completion

### SeaORM — ActiveRecord-Style ORM

For applications preferring an ORM over raw SQL, [SeaORM](https://www.sea-ql.org/SeaORM/) provides ActiveModel patterns with async support:

```rust
// Entity definition — usually generated by `sea-orm-cli generate entity`
use sea_orm::entity::prelude::*;

#[derive(Clone, Debug, PartialEq, DeriveEntityModel)]
#[sea_orm(table_name = "users")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i32,
    pub username: String,
    pub email: String,
    pub created_at: DateTimeUtc,
}

#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {
    #[sea_orm(has_many = "super::post::Entity")]
    Posts,
}

impl Related<super::post::Entity> for Entity {
    fn to() -> RelationDef {
        Relation::Posts.def()
    }
}

impl ActiveModelBehavior for ActiveModel {}
```

**CRUD operations:**

```rust
use sea_orm::{ActiveModelTrait, EntityTrait, QueryFilter, ColumnTrait, Set, DatabaseConnection};

// Create
let user = user::ActiveModel {
    username: Set("alice".to_string()),
    email: Set("alice@example.com".to_string()),
    ..Default::default()
};
let result = user.insert(&db).await?;

// Read with filtering
let users = user::Entity::find()
    .filter(user::Column::Email.contains("@example.com"))
    .all(&db).await?;

// Update
let mut user: user::ActiveModel = user::Entity::find_by_id(1)
    .one(&db).await?.unwrap().into();
user.email = Set("new@example.com".to_string());
user.update(&db).await?;

// Eager loading (N+1 prevention)
let users_with_posts = user::Entity::find()
    .find_with_related(post::Entity)
    .all(&db).await?;
```

**SQLx vs Diesel vs SeaORM:**

| Aspect | SQLx | Diesel | SeaORM |
|--------|------|--------|--------|
| Query style | Raw SQL | Type-safe DSL | ActiveRecord |
| Compile-time checks | Against live DB | Against schema.rs | Runtime |
| Async support | Native | Via diesel-async | Native |
| Learning curve | Low (just SQL) | Medium | Medium |
| Flexibility | Full SQL power | ORM abstractions | ORM + raw SQL |
| Code generation | None | diesel print-schema | sea-orm-cli |
| Best for | Performance, complex SQL | Type safety, CRUD | Rapid development |

### MongoDB Integration

For document-oriented data, MongoDB with the `mongodb` crate provides async operations.

```rust
use mongodb::{Client, Collection, bson::{doc, oid::ObjectId}};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct User {
    #[serde(rename = "_id", skip_serializing_if = "Option::is_none")]
    pub id: Option<ObjectId>,
    pub username: String,
    pub email: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

pub struct MongoUserRepository {
    collection: Collection<User>,
}

impl MongoUserRepository {
    pub async fn new(uri: &str, db: &str, coll: &str) -> Result<Self, mongodb::error::Error> {
        let client = Client::with_uri_str(uri).await?;
        let collection = client.database(db).collection(coll);
        Ok(Self { collection })
    }
}

#[async_trait]
impl UserRepository for MongoUserRepository {
    async fn find_by_id(&self, id: &str) -> Result<Option<User>, RepositoryError> {
        let oid = ObjectId::parse_str(id)
            .map_err(|e| RepositoryError::InvalidData)?;

        self.collection
            .find_one(doc! { "_id": oid }, None)
            .await
            .map_err(|e| RepositoryError::DatabaseError(Box::new(e)))
    }

    async fn save(&self, user: &User) -> Result<(), RepositoryError> {
        if user.id.is_some() {
            // Update existing
            let filter = doc! { "_id": user.id.unwrap() };
            self.collection
                .replace_one(filter, user, None)
                .await
                .map_err(|e| RepositoryError::DatabaseError(Box::new(e)))?;
        } else {
            // Insert new
            self.collection
                .insert_one(user, None)
                .await
                .map_err(|e| RepositoryError::DatabaseError(Box::new(e)))?;
        }
        Ok(())
    }

    async fn find_by_email(&self, email: &str) -> Result<Option<User>, RepositoryError> {
        self.collection
            .find_one(doc! { "email": email }, None)
            .await
            .map_err(|e| RepositoryError::DatabaseError(Box::new(e)))
    }
}
```

**MongoDB Query Patterns:**

```rust
// Find with filters
let filter = doc! {
    "status": "active",
    "age": { "$gte": 18 }
};
let users: Vec<User> = collection.find(filter, None).await?.try_collect().await?;

// Aggregation pipeline
let pipeline = vec![
    doc! { "$match": { "status": "active" } },
    doc! { "$group": { "_id": "$department", "count": { "$sum": 1 } } },
    doc! { "$sort": { "count": -1 } },
];
let results = collection.aggregate(pipeline, None).await?;

// Update with operators
collection.update_one(
    doc! { "_id": user_id },
    doc! { "$set": { "email": new_email }, "$inc": { "login_count": 1 } },
    None,
).await?;
```

## Caching Patterns

### In-Memory Caching

For single-instance applications, in-memory caching provides the fastest access.

**Basic Cache with TTL:**

```rust
use std::collections::HashMap;
use std::time::{Duration, Instant};

#[derive(Clone)]
struct CacheEntry<T> {
    data: T,
    expires_at: Instant,
}

pub struct InMemoryCache<K, V> {
    store: HashMap<K, CacheEntry<V>>,
    default_ttl: Duration,
}

impl<K: Eq + std::hash::Hash + Clone, V: Clone> InMemoryCache<K, V> {
    pub fn new(default_ttl: Duration) -> Self {
        Self {
            store: HashMap::new(),
            default_ttl,
        }
    }

    pub fn get(&mut self, key: &K) -> Option<V> {
        if let Some(entry) = self.store.get(key) {
            if Instant::now() < entry.expires_at {
                return Some(entry.data.clone());
            }
            // Expired - remove it
            self.store.remove(key);
        }
        None
    }

    pub fn set(&mut self, key: K, value: V) {
        self.set_with_ttl(key, value, self.default_ttl);
    }

    pub fn set_with_ttl(&mut self, key: K, value: V, ttl: Duration) {
        self.store.insert(key, CacheEntry {
            data: value,
            expires_at: Instant::now() + ttl,
        });
    }

    pub fn remove(&mut self, key: &K) -> Option<V> {
        self.store.remove(key).map(|e| e.data)
    }
}
```

**Thread-Safe Cache with Mutex:**

```rust
use std::sync::{Arc, Mutex};

pub struct ThreadSafeCache<K, V> {
    inner: Arc<Mutex<InMemoryCache<K, V>>>,
}

impl<K: Eq + std::hash::Hash + Clone + Send, V: Clone + Send> ThreadSafeCache<K, V> {
    pub fn new(default_ttl: Duration) -> Self {
        Self {
            inner: Arc::new(Mutex::new(InMemoryCache::new(default_ttl))),
        }
    }

    pub fn get(&self, key: &K) -> Option<V> {
        self.inner.lock().unwrap().get(key)
    }

    pub fn set(&self, key: K, value: V) {
        self.inner.lock().unwrap().set(key, value);
    }
}

impl<K, V> Clone for ThreadSafeCache<K, V> {
    fn clone(&self) -> Self {
        Self { inner: Arc::clone(&self.inner) }
    }
}
```

**Production-Grade with moka Crate:**

```rust
use moka::future::Cache;
use std::time::Duration;

// Configure cache with capacity and TTL
let cache: Cache<String, User> = Cache::builder()
    .max_capacity(10_000)
    .time_to_live(Duration::from_secs(300))      // Expire 5 min after insertion
    .time_to_idle(Duration::from_secs(60))       // Expire 1 min after last access
    .build();

// Get or compute pattern
let user = cache.get_with(user_id.clone(), async {
    fetch_user_from_db(&user_id).await.unwrap()
}).await;

// Manual operations
cache.insert(key, value).await;
cache.invalidate(&key).await;
cache.invalidate_all();
```

**Eviction Policies:**

| Policy | Description | Use Case |
|--------|-------------|----------|
| LRU | Evict least recently used | General purpose, access patterns matter |
| LFU | Evict least frequently used | Hot data stays, cold data evicted |
| FIFO | Evict oldest first | Time-sensitive data |
| TTL | Evict after time expires | Data freshness requirements |

### Redis Distributed Cache

For multi-instance applications, Redis provides shared caching across all instances.

**Cache Service Abstraction:**

```rust
use async_trait::async_trait;
use serde::{de::DeserializeOwned, Serialize};
use std::time::Duration;

#[derive(Debug, thiserror::Error)]
pub enum CacheError {
    #[error("Key not found")]
    NotFound,
    #[error("Connection error: {0}")]
    Connection(String),
    #[error("Serialization error: {0}")]
    Serialization(String),
}

#[async_trait]
pub trait CacheService: Send + Sync {
    async fn get<T: DeserializeOwned + Send>(&self, key: &str) -> Result<T, CacheError>;
    async fn set<T: Serialize + Send + Sync>(&self, key: &str, value: &T, ttl: Option<Duration>) -> Result<(), CacheError>;
    async fn delete(&self, key: &str) -> Result<(), CacheError>;
    async fn exists(&self, key: &str) -> Result<bool, CacheError>;
}
```

**Redis Implementation:**

```rust
use redis::{AsyncCommands, Client};

pub struct RedisCacheService {
    client: Client,
}

impl RedisCacheService {
    pub fn new(url: &str) -> Result<Self, CacheError> {
        let client = Client::open(url)
            .map_err(|e| CacheError::Connection(e.to_string()))?;
        Ok(Self { client })
    }

    async fn get_conn(&self) -> Result<redis::aio::MultiplexedConnection, CacheError> {
        self.client
            .get_multiplexed_async_connection()
            .await
            .map_err(|e| CacheError::Connection(e.to_string()))
    }
}

#[async_trait]
impl CacheService for RedisCacheService {
    async fn get<T: DeserializeOwned + Send>(&self, key: &str) -> Result<T, CacheError> {
        let mut conn = self.get_conn().await?;
        let bytes: Option<Vec<u8>> = conn.get(key).await
            .map_err(|e| CacheError::Connection(e.to_string()))?;

        match bytes {
            Some(b) => serde_json::from_slice(&b)
                .map_err(|e| CacheError::Serialization(e.to_string())),
            None => Err(CacheError::NotFound),
        }
    }

    async fn set<T: Serialize + Send + Sync>(
        &self,
        key: &str,
        value: &T,
        ttl: Option<Duration>,
    ) -> Result<(), CacheError> {
        let mut conn = self.get_conn().await?;
        let bytes = serde_json::to_vec(value)
            .map_err(|e| CacheError::Serialization(e.to_string()))?;

        match ttl {
            Some(d) => conn.set_ex(key, bytes, d.as_secs() as u64).await,
            None => conn.set(key, bytes).await,
        }
        .map_err(|e| CacheError::Connection(e.to_string()))
    }

    async fn delete(&self, key: &str) -> Result<(), CacheError> {
        let mut conn = self.get_conn().await?;
        conn.del(key).await
            .map_err(|e| CacheError::Connection(e.to_string()))
    }

    async fn exists(&self, key: &str) -> Result<bool, CacheError> {
        let mut conn = self.get_conn().await?;
        conn.exists(key).await
            .map_err(|e| CacheError::Connection(e.to_string()))
    }
}
```

### Cached Repository Pattern

Compose caching with repositories using the decorator pattern:

```rust
pub struct CachedUserRepository<R: UserRepository, C: CacheService> {
    repository: R,
    cache: C,
    ttl: Duration,
}

impl<R: UserRepository, C: CacheService> CachedUserRepository<R, C> {
    pub fn new(repository: R, cache: C, ttl: Duration) -> Self {
        Self { repository, cache, ttl }
    }

    fn cache_key(id: &str) -> String {
        format!("user:{}", id)
    }
}

#[async_trait]
impl<R: UserRepository + Send + Sync, C: CacheService> UserRepository
    for CachedUserRepository<R, C>
{
    async fn find_by_id(&self, id: &str) -> Result<Option<User>, RepositoryError> {
        let key = Self::cache_key(id);

        // 1. Try cache first
        match self.cache.get::<User>(&key).await {
            Ok(user) => {
                tracing::debug!("Cache hit for user {}", id);
                return Ok(Some(user));
            }
            Err(CacheError::NotFound) => {
                tracing::debug!("Cache miss for user {}", id);
            }
            Err(e) => {
                tracing::warn!("Cache error for user {}: {}", id, e);
                // Continue to repository on cache errors
            }
        }

        // 2. Fetch from repository
        let user = self.repository.find_by_id(id).await?;

        // 3. Populate cache
        if let Some(ref u) = user {
            if let Err(e) = self.cache.set(&key, u, Some(self.ttl)).await {
                tracing::warn!("Failed to cache user {}: {}", id, e);
            }
        }

        Ok(user)
    }

    async fn save(&self, user: &User) -> Result<(), RepositoryError> {
        // 1. Save to repository
        self.repository.save(user).await?;

        // 2. Invalidate cache
        if let Some(id) = &user.id {
            let key = Self::cache_key(&id.to_string());
            if let Err(e) = self.cache.delete(&key).await {
                tracing::warn!("Failed to invalidate cache: {}", e);
            }
        }

        Ok(())
    }
}
```

### Cache Invalidation Strategies

| Strategy | How It Works | Pros | Cons |
|----------|--------------|------|------|
| **Write-Through** | Write to DB and cache simultaneously | Strong consistency | Higher write latency |
| **Write-Around** | Write to DB only, cache on read | Lower write latency | First read is slow |
| **Write-Back** | Write to cache, async persist to DB | Lowest latency | Risk of data loss |

**Write-Through Example:**

```rust
async fn update_user(&self, user: &User) -> Result<(), Error> {
    // Write to both simultaneously
    self.repository.save(user).await?;
    let key = format!("user:{}", user.id);
    self.cache.set(&key, user, Some(self.ttl)).await?;
    Ok(())
}
```

**Write-Around with TTL:**

```rust
async fn update_user(&self, user: &User) -> Result<(), Error> {
    // Write to DB only
    self.repository.save(user).await?;
    // Invalidate cache - next read will repopulate
    let key = format!("user:{}", user.id);
    self.cache.delete(&key).await.ok();
    Ok(())
}
```

**Cache-Aside Pattern (Read-Through):**

```rust
async fn get_user(&self, id: &str) -> Result<User, Error> {
    let key = format!("user:{}", id);

    // Check cache
    if let Ok(user) = self.cache.get::<User>(&key).await {
        return Ok(user);
    }

    // Load from DB
    let user = self.repository.find_by_id(id).await?
        .ok_or(Error::NotFound)?;

    // Populate cache
    self.cache.set(&key, &user, Some(Duration::from_secs(300))).await.ok();

    Ok(user)
}
```

## Chat & Messaging Schema Patterns

Schema design for persistent chat with room management. See [web-apis.md](web-apis.md#multi-room-chat-with-persistence) for the WebSocket layer that uses these queries.

### Chat Schema (PostgreSQL)

```sql
-- Rooms (chat channels, DMs, group chats)
CREATE TABLE rooms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255),
    room_type VARCHAR(20) NOT NULL DEFAULT 'group',  -- 'direct', 'group', 'channel'
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID REFERENCES users(id)
);

-- Room membership — who can see/post in which rooms
CREATE TABLE room_members (
    room_id UUID NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL DEFAULT 'member',  -- 'owner', 'admin', 'member'
    joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_read_at TIMESTAMPTZ,  -- For unread message counting
    PRIMARY KEY (room_id, user_id)
);

-- Messages — append-heavy, optimized for cursor-based pagination
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    room_id UUID NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
    sender_id UUID NOT NULL REFERENCES users(id),
    content TEXT NOT NULL,
    sent_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    edited_at TIMESTAMPTZ
);

-- Critical indexes for chat query patterns
CREATE INDEX idx_messages_room_sent ON messages (room_id, sent_at DESC);  -- History loading
CREATE INDEX idx_room_members_user ON room_members (user_id);             -- "My rooms" query
```

**Why `sent_at DESC` index?** Chat history loads newest-first. The composite index `(room_id, sent_at DESC)` allows PostgreSQL to satisfy `WHERE room_id = $1 ORDER BY sent_at DESC LIMIT 50` with an index-only scan — no sort step.

### Cursor-Based Chat History

```rust
#[derive(Debug, sqlx::FromRow, Serialize)]
pub struct ChatMessage {
    pub id: Uuid,
    pub room_id: Uuid,
    pub sender_id: Uuid,
    pub content: String,
    pub sent_at: chrono::DateTime<chrono::Utc>,
    pub edited_at: Option<chrono::DateTime<chrono::Utc>>,
}

/// Load messages before a cursor, newest first.
/// Returns messages in reverse chronological order — client reverses for display.
pub async fn load_room_history(
    pool: &PgPool,
    room_id: Uuid,
    before: Option<chrono::DateTime<chrono::Utc>>,
    limit: i64,
) -> Result<Vec<ChatMessage>, sqlx::Error> {
    let before = before.unwrap_or(chrono::Utc::now());
    sqlx::query_as!(
        ChatMessage,
        r#"SELECT id, room_id, sender_id, content, sent_at, edited_at
           FROM messages
           WHERE room_id = $1 AND sent_at < $2
           ORDER BY sent_at DESC
           LIMIT $3"#,
        room_id, before, limit.min(100)  // Cap at 100 to prevent abuse
    )
    .fetch_all(pool)
    .await
}

/// Count unread messages for a user in a room
pub async fn unread_count(
    pool: &PgPool,
    room_id: Uuid,
    user_id: Uuid,
) -> Result<i64, sqlx::Error> {
    sqlx::query_scalar!(
        r#"SELECT COUNT(*) as "count!" FROM messages m
           JOIN room_members rm ON rm.room_id = m.room_id AND rm.user_id = $2
           WHERE m.room_id = $1 AND m.sent_at > COALESCE(rm.last_read_at, '1970-01-01')"#,
        room_id, user_id
    )
    .fetch_one(pool)
    .await
}

/// Update last-read cursor when user reads messages
pub async fn mark_read(
    pool: &PgPool,
    room_id: Uuid,
    user_id: Uuid,
) -> Result<(), sqlx::Error> {
    sqlx::query!(
        "UPDATE room_members SET last_read_at = NOW() WHERE room_id = $1 AND user_id = $2",
        room_id, user_id
    )
    .execute(pool)
    .await?;
    Ok(())
}
```

### Direct Messages (1:1 Chat)

For DMs, create a room with `room_type = 'direct'` and exactly two members. Find existing DM rooms before creating duplicates:

```rust
/// Find or create a direct message room between two users
pub async fn find_or_create_dm(
    pool: &PgPool,
    user_a: Uuid,
    user_b: Uuid,
) -> Result<Uuid, sqlx::Error> {
    // Check for existing DM room with both users
    let existing = sqlx::query_scalar!(
        r#"SELECT r.id as "id!" FROM rooms r
           JOIN room_members rm1 ON rm1.room_id = r.id AND rm1.user_id = $1
           JOIN room_members rm2 ON rm2.room_id = r.id AND rm2.user_id = $2
           WHERE r.room_type = 'direct'
           LIMIT 1"#,
        user_a, user_b
    )
    .fetch_optional(pool)
    .await?;

    if let Some(room_id) = existing {
        return Ok(room_id);
    }

    // Create new DM room in a transaction
    let mut tx = pool.begin().await?;
    let room_id = Uuid::new_v4();
    sqlx::query!(
        "INSERT INTO rooms (id, room_type) VALUES ($1, 'direct')",
        room_id
    ).execute(&mut *tx).await?;

    for user_id in [user_a, user_b] {
        sqlx::query!(
            "INSERT INTO room_members (room_id, user_id, role) VALUES ($1, $2, 'member')",
            room_id, user_id
        ).execute(&mut *tx).await?;
    }

    tx.commit().await?;
    Ok(room_id)
}
```

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: error handling with `?`, serde for query result mapping, async basics
- **[web-apis.md](web-apis.md)** — SQLx integration with Axum handlers, connection pool state management, WebSocket chat patterns
- **[serde-serialization.md](serde-serialization.md)** — `FromRow` derive, custom deserialization for database types
- **[error-handling.md](error-handling.md)** — Database error translation across layers, `thiserror` for repository errors
- **[services.md](services.md)** — Redis caching patterns, job queues, distributed data
- **[testing.md](testing.md)** — Database test fixtures, transaction rollback isolation
