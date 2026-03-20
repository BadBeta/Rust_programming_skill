# Error Handling in Rust

Comprehensive patterns for error handling including Result/Option, custom error types, thiserror/anyhow, color-eyre/miette, error propagation, recovery strategies, and multi-layer error translation.

## Rules for Error Handling (LLM)

1. **ALWAYS use `thiserror` for library error types** — libraries must expose typed errors that callers can pattern match on; never use `anyhow::Error` in a library's public API
2. **ALWAYS use `anyhow` or `color-eyre` for application error handling** — applications benefit from context chaining (`.context()`) and type erasure; don't define custom error enums in `main.rs`
3. **ALWAYS attach context to errors with `.context()` or `.wrap_err()`** — bare `?` loses the call site; `fs::read(path).context("reading config")?` tells you *what* failed, not just *how*
4. **NEVER use `.unwrap()` in production code paths** — use `.expect("reason")` for invariants, `?` for propagation, or `.unwrap_or_default()` for fallbacks
5. **ALWAYS define a crate-level `Result` type alias** — `pub type Result<T, E = Error> = core::result::Result<T, E>;` simplifies every function signature in the crate (pattern used by anyhow, serde, reqwest, axum)
6. **ALWAYS implement `From` for errors that cross layer boundaries** — `#[from]` for 1:1 wrapping, manual `From` impl when you need to transform or classify the error
7. **NEVER expose infrastructure error types in domain layers** — domain errors must be pure; translate `sqlx::Error` to `RepositoryError` at the infrastructure boundary
8. **PREFER `color-eyre` over `anyhow` for user-facing CLI tools** — colorized output, SpanTrace integration, `.suggestion()` and `.section()` for actionable diagnostics

### Common Mistakes (BAD/GOOD)

**String errors instead of typed errors:**
```rust
// BAD: callers can't match on error type
fn parse(s: &str) -> Result<Config, String> {
    toml::from_str(s).map_err(|e| e.to_string())
}

// GOOD: typed error with From conversion
#[derive(Debug, thiserror::Error)]
enum ConfigError {
    #[error("parse error")]
    Parse(#[from] toml::de::Error),
}
fn parse(s: &str) -> Result<Config, ConfigError> {
    Ok(toml::from_str(s)?)
}
```

**Bare ? without context:**
```rust
// BAD: error says "No such file" — which file?
fn load() -> anyhow::Result<Config> {
    let s = std::fs::read_to_string("config.toml")?;
    Ok(toml::from_str(&s)?)
}

// GOOD: context says what operation failed
fn load() -> anyhow::Result<Config> {
    let s = std::fs::read_to_string("config.toml")
        .context("failed to read config.toml")?;
    let config = toml::from_str(&s)
        .context("failed to parse config.toml")?;
    Ok(config)
}
```

**Leaking internal errors to users:**
```rust
// BAD: exposes database internals in HTTP response
Err(e) => HttpResponse::InternalServerError().body(e.to_string())
// Output: "error returned from database: relation "users" does not exist"

// GOOD: log internally, return generic message to user
Err(e) => {
    tracing::error!(error = %e, "request failed");
    HttpResponse::InternalServerError().json(json!({"error": "internal error"}))
}
```

## Error Handling Philosophy

### When to Use What

```rust
// Result<T, E> — Recoverable errors, expected failure cases
fn parse_config(path: &str) -> Result<Config, ConfigError> {
    let content = std::fs::read_to_string(path)?;
    toml::from_str(&content).map_err(ConfigError::Parse)
}

// Option<T> — Absence of value (not an error)
fn find_user(id: u64) -> Option<User> {
    users.get(&id).cloned()
}

// panic! — Unrecoverable errors, programming bugs
fn get_index(slice: &[i32], index: usize) -> i32 {
    slice[index]  // Panics on out-of-bounds (caller's bug)
}

// assert! — Invariants that must hold
fn calculate(divisor: i32) -> i32 {
    assert!(divisor != 0, "divisor must be non-zero");
    100 / divisor
}
```

### Decision Guide

| Situation | Use |
|-----------|-----|
| Invalid function arguments (caller's responsibility) | `panic!` |
| File not found, network timeout | `Result` |
| Index out of bounds in internal code | `panic!` |
| User input validation failure | `Result` |
| Unrecoverable state corruption | `panic!` |
| Parse error on external data | `Result` |
| Violated invariant/contract | `panic!` or `debug_assert!` |
| Value may or may not exist | `Option` |

## The ? Operator

### Basic Propagation

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username() -> Result<String, io::Error> {
    let mut file = File::open("username.txt")?;  // Returns early on Err
    let mut username = String::new();
    file.read_to_string(&mut username)?;
    Ok(username)
}

// Equivalent without ?
fn read_username_verbose() -> Result<String, io::Error> {
    let mut file = match File::open("username.txt") {
        Ok(f) => f,
        Err(e) => return Err(e),
    };
    let mut username = String::new();
    match file.read_to_string(&mut username) {
        Ok(_) => Ok(username),
        Err(e) => Err(e),
    }
}
```

### ? with Option

```rust
fn get_nested_value(data: &HashMap<String, HashMap<String, i32>>) -> Option<i32> {
    let inner = data.get("outer")?;
    let value = inner.get("inner")?;
    Some(*value)
}

// Convert Option to Result
fn get_required_value(map: &HashMap<String, String>, key: &str) -> Result<&String, Error> {
    map.get(key).ok_or_else(|| Error::MissingKey(key.to_string()))
}
```

### Chaining with ?

```rust
fn process_data(path: &str) -> Result<ProcessedData, Error> {
    let content = std::fs::read_to_string(path)?;
    let parsed: RawData = serde_json::from_str(&content)?;
    let validated = validate(parsed)?;
    let processed = transform(validated)?;
    Ok(processed)
}
```

## Custom Error Types

### Manual Implementation

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
pub enum ConfigError {
    IoError(std::io::Error),
    ParseError(String),
    MissingField(String),
    InvalidValue { field: String, value: String },
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::IoError(e) => write!(f, "IO error: {}", e),
            ConfigError::ParseError(msg) => write!(f, "Parse error: {}", msg),
            ConfigError::MissingField(field) => write!(f, "Missing field: {}", field),
            ConfigError::InvalidValue { field, value } => {
                write!(f, "Invalid value '{}' for field '{}'", value, field)
            }
        }
    }
}

impl Error for ConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            ConfigError::IoError(e) => Some(e),
            _ => None,
        }
    }
}

impl From<std::io::Error> for ConfigError {
    fn from(err: std::io::Error) -> Self {
        ConfigError::IoError(err)
    }
}
```

### Using thiserror (Libraries)

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum DatabaseError {
    #[error("Connection failed: {0}")]
    ConnectionFailed(String),

    #[error("Query failed: {query}")]
    QueryFailed { query: String, #[source] cause: sqlx::Error },

    #[error("Record not found: {entity} with id {id}")]
    NotFound { entity: &'static str, id: String },

    #[error("Constraint violation: {0}")]
    ConstraintViolation(String),

    // Automatic From impl with #[from]
    #[error("IO error")]
    Io(#[from] std::io::Error),

    #[error("Serialization error")]
    Serialization(#[from] serde_json::Error),

    // Transparent: delegates Display and source() to inner
    #[error(transparent)]
    Other(#[from] anyhow::Error),
}

fn query_user(id: &str) -> Result<User, DatabaseError> {
    let file = std::fs::read_to_string("users.json")?;  // Auto-converts io::Error
    let users: Vec<User> = serde_json::from_str(&file)?;  // Auto-converts serde error

    users.into_iter()
        .find(|u| u.id == id)
        .ok_or_else(|| DatabaseError::NotFound {
            entity: "User",
            id: id.to_string(),
        })
}
```

### Using anyhow (Applications)

```rust
use anyhow::{Context, Result, bail, ensure};

// Result is anyhow::Result<T> = Result<T, anyhow::Error>
fn load_config(path: &str) -> Result<Config> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("Failed to read config from {}", path))?;

    let config: Config = toml::from_str(&content)
        .context("Failed to parse config file")?;

    ensure!(config.port > 0, "Port must be positive, got {}", config.port);

    if config.name.is_empty() {
        bail!("Config name cannot be empty");
    }

    Ok(config)
}

// Main can return Result
fn main() -> Result<()> {
    let config = load_config("config.toml")?;
    run_server(config)?;
    Ok(())
}

// Attach context to any error
fn process_file(path: &str) -> Result<()> {
    let data = read_file(path)
        .with_context(|| format!("Failed to process {}", path))?;

    validate(&data).context("Validation failed")?;
    Ok(())
}

// anyhow! macro for one-off errors (no enum variant needed)
fn validate_port(port: u16) -> Result<()> {
    if port == 0 {
        return Err(anyhow::anyhow!("port must be non-zero"));
    }
    if port < 1024 {
        return Err(anyhow::anyhow!("port {} requires root privileges", port));
    }
    Ok(())
}
```

### root_cause — Accessing the Deepest Error

```rust
fn diagnose(err: &anyhow::Error) {
    // root_cause() returns the deepest error in the chain
    let root = err.root_cause();
    tracing::error!("Root cause: {root}");

    // Useful for retry decisions: retry on transient, fail on permanent
    if let Some(io_err) = root.downcast_ref::<std::io::Error>() {
        match io_err.kind() {
            std::io::ErrorKind::ConnectionRefused => { /* retry */ }
            std::io::ErrorKind::PermissionDenied => { /* fail fast */ }
            _ => {}
        }
    }
}
```

### thiserror vs anyhow

| Aspect | thiserror | anyhow |
|--------|-----------|--------|
| Use case | Libraries | Applications |
| Error type | Custom enum | `anyhow::Error` |
| Type information | Preserved | Erased |
| Matching errors | Pattern match | Downcast |
| Compile-time checking | Full | Limited |
| Context | Manual | `.context()` |

```rust
// Library: thiserror for typed errors users can match
#[derive(Debug, thiserror::Error)]
pub enum MyLibError {
    #[error("not found")]
    NotFound,
    #[error("permission denied")]
    PermissionDenied,
}

// Application: anyhow for convenience
fn main() -> anyhow::Result<()> {
    my_lib::do_thing().context("failed to do thing")?;
    Ok(())
}
```

### Crate-Level Result Alias

Every production library defines a crate-level Result alias (anyhow, serde_json, reqwest, axum):

```rust
// In your crate's lib.rs — simplifies all function signatures
pub type Result<T, E = Error> = core::result::Result<T, E>;

// Users write:
fn load(path: &str) -> mylib::Result<Config> { /* ... */ }
// Instead of:
fn load(path: &str) -> Result<Config, mylib::Error> { /* ... */ }
```

### Downcasting anyhow::Error

When using `anyhow::Error` (type-erased), recover the original typed error with `downcast_ref`:

```rust
use anyhow::Result;

fn handle_error(err: &anyhow::Error) {
    // Try to recover the original typed error
    if let Some(db_err) = err.downcast_ref::<DatabaseError>() {
        match db_err {
            DatabaseError::NotFound { entity, id } => {
                tracing::warn!("{entity} {id} not found — returning 404");
            }
            DatabaseError::ConnectionFailed(_) => {
                tracing::error!("DB connection lost — triggering circuit breaker");
            }
            _ => {}
        }
    } else if let Some(io_err) = err.downcast_ref::<std::io::Error>() {
        tracing::error!("IO error: {io_err}");
    }
}

// In tests — verify the underlying error type
#[test]
fn returns_not_found_error() {
    let result = find_user("nonexistent");
    let err = result.unwrap_err();
    assert!(err.downcast_ref::<DatabaseError>()
        .is_some_and(|e| matches!(e, DatabaseError::NotFound { .. })));
}
```

### Walking the Error Chain

`anyhow::Error::chain()` iterates through the full source chain:

```rust
fn log_full_error(err: &anyhow::Error) {
    // Print: "failed to load config" → "IO error" → "file not found"
    for (i, cause) in err.chain().enumerate() {
        if i == 0 {
            tracing::error!("Error: {cause}");
        } else {
            tracing::error!("  Caused by: {cause}");
        }
    }
}
```

### #[error(transparent)] — When and Why

`#[error(transparent)]` delegates both `Display` and `source()` to the inner error, making the wrapper invisible in error chains:

```rust
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    // VISIBLE in chain: displays "Database error: ..." then shows cause
    #[error("Database error: {0}")]
    Database(#[from] DatabaseError),

    // TRANSPARENT: displays the inner error directly, as if unwrapped
    // Use when you want to erase the variant but preserve the full chain
    #[error(transparent)]
    Other(#[from] anyhow::Error),

    // TRANSPARENT with Box<dyn Error>: catch-all for unknown errors
    #[error(transparent)]
    Unknown(Box<dyn std::error::Error + Send + Sync>),
}
// Use transparent when:
// - Wrapping anyhow::Error or Box<dyn Error> as a catch-all
// - The wrapper variant name adds no information
// Don't use transparent when:
// - The variant name provides useful categorization (Database, Network, etc.)
```

### #[from] vs Manual From — When to Choose

```rust
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    // Use #[from] when: 1:1 mapping, no transformation needed
    #[error("IO error")]
    Io(#[from] std::io::Error),

    // Use manual From when: you need to transform or add context
    #[error("Database error: {0}")]
    Database(String),
}

impl From<sqlx::Error> for AppError {
    fn from(err: sqlx::Error) -> Self {
        // Transform: extract meaningful info, don't just wrap
        match err {
            sqlx::Error::RowNotFound => AppError::Database("record not found".into()),
            other => AppError::Database(other.to_string()),
        }
    }
}
// Rule: if From::from does anything other than wrap, implement it manually
```

## color-eyre — Enhanced Error Reporting

color-eyre is a fork of anyhow that provides colorized, structured error reports with span traces. Ideal for applications and CLI tools where human-readable error output matters.

### Setup

```rust
// Cargo.toml
// [dependencies]
// color-eyre = "0.6"
// tracing = "0.1"
// tracing-subscriber = { version = "0.3", features = ["env-filter"] }

use color_eyre::eyre::{Result, WrapErr, bail, ensure};

fn main() -> Result<()> {
    // Install color-eyre's panic and error hooks — call once at startup
    color_eyre::install()?;

    // Optional: integrate with tracing for SpanTrace capture
    tracing_subscriber::fmt()
        .with_env_filter("info")
        .init();

    run()
}
```

### Usage — Drop-in anyhow Replacement

```rust
use color_eyre::eyre::{Result, WrapErr};

fn load_config(path: &str) -> Result<Config> {
    let content = std::fs::read_to_string(path)
        .wrap_err_with(|| format!("Failed to read config from {path}"))?;

    let config: Config = toml::from_str(&content)
        .wrap_err("Failed to parse config")?;

    Ok(config)
}

// Error output includes:
// - Colored error chain (each .wrap_err adds a layer)
// - SpanTrace (if tracing is configured)
// - Backtrace (if RUST_BACKTRACE=1)
```

### SpanTrace Integration

```rust
use color_eyre::eyre::{Result, WrapErr};
use tracing::instrument;

#[instrument(skip(db))]
async fn process_order(db: &PgPool, order_id: i64) -> Result<()> {
    let order = fetch_order(db, order_id).await
        .wrap_err("Failed to fetch order")?;

    validate_order(&order)
        .wrap_err("Order validation failed")?;

    charge_payment(&order).await
        .wrap_err("Payment processing failed")?;

    Ok(())
}

// Error output automatically includes the tracing span context:
//   0: process_order{order_id=42}
//     at src/orders.rs:15
// This tells you WHICH order failed without manual context strings
```

### Custom Sections and Suggestions

```rust
use color_eyre::{eyre::eyre, Section, SectionExt};

fn validate_config(config: &Config) -> Result<()> {
    if config.workers == 0 {
        return Err(eyre!("Invalid worker count: 0"))
            .suggestion("Set workers to at least 1 in config.toml")
            .note("Worker count determines parallelism level");
    }

    if config.port < 1024 && !running_as_root() {
        return Err(eyre!("Cannot bind to port {}", config.port))
            .suggestion("Use a port >= 1024 or run as root")
            .section(format!("Requested port: {}", config.port).header("Details:"));
    }

    Ok(())
}
```

### anyhow vs color-eyre

| Aspect | anyhow | color-eyre |
|--------|--------|------------|
| Error type | `anyhow::Error` | `eyre::Report` (compatible) |
| Context method | `.context()` | `.wrap_err()` (preferred) |
| Colorized output | No | Yes |
| SpanTrace | No | Yes (with tracing) |
| Suggestions | No | `.suggestion()` |
| Custom sections | No | `.section()`, `.note()` |
| Performance | Slightly faster | Negligible overhead |
| Best for | Libraries/servers | CLI tools, applications |

## miette — Diagnostic Error Reporting

miette provides rich diagnostic output with source code snippets, labels, and help text. Best for compilers, linters, config parsers — anywhere source location matters.

### Setup

```rust
// Cargo.toml
// [dependencies]
// miette = { version = "7", features = ["fancy"] }
// thiserror = "2"

use miette::{Diagnostic, SourceSpan, NamedSource, Result};
use thiserror::Error;
```

### Diagnostic Errors

```rust
#[derive(Debug, Error, Diagnostic)]
pub enum ConfigDiagnostic {
    #[error("Unknown field `{name}`")]
    #[diagnostic(
        code(config::unknown_field),
        help("Valid fields are: host, port, workers, log_level")
    )]
    UnknownField {
        name: String,
        #[source_code]
        src: NamedSource<String>,
        #[label("this field is not recognized")]
        span: SourceSpan,
    },

    #[error("Invalid port number")]
    #[diagnostic(
        code(config::invalid_port),
        severity(Error),
        help("Port must be between 1 and 65535")
    )]
    InvalidPort {
        #[source_code]
        src: NamedSource<String>,
        #[label("port value out of range")]
        span: SourceSpan,
    },

    #[error("Duplicate key `{key}`")]
    #[diagnostic(code(config::duplicate_key))]
    DuplicateKey {
        key: String,
        #[source_code]
        src: NamedSource<String>,
        #[label("first defined here")]
        first: SourceSpan,
        #[label("redefined here")]
        second: SourceSpan,
    },
}

fn validate_config(source: &str, filename: &str) -> Result<Config, ConfigDiagnostic> {
    // When an error occurs, include source location
    if let Some(pos) = find_unknown_field(source) {
        return Err(ConfigDiagnostic::UnknownField {
            name: "databse".to_string(),
            src: NamedSource::new(filename, source.to_string()),
            span: (pos, 7).into(),  // (offset, length)
        });
    }
    // ... parse and return config
}

// Output (with "fancy" feature):
//   × Unknown field `databse`
//    ╭─[config.toml:3:1]
//  3 │ databse = "postgres://..."
//    ·         ─── this field is not recognized
//    ╰────
//   help: Valid fields are: host, port, workers, log_level
```

### Multiple Labels and Related Errors

```rust
#[derive(Debug, Error, Diagnostic)]
#[error("Type mismatch")]
#[diagnostic(code(check::type_mismatch))]
struct TypeMismatch {
    #[source_code]
    src: NamedSource<String>,
    #[label("expected {expected} here")]
    expected_span: SourceSpan,
    #[label("but got {actual}")]
    actual_span: SourceSpan,
    expected: String,
    actual: String,
    #[related]
    related: Vec<TypeHint>,
}

#[derive(Debug, Error, Diagnostic)]
#[error("note: {message}")]
#[diagnostic(severity(Advice))]
struct TypeHint {
    message: String,
    #[source_code]
    src: NamedSource<String>,
    #[label("{message}")]
    span: SourceSpan,
}
```

### Using miette with main()

```rust
fn main() -> miette::Result<()> {
    // Install miette's panic hook for pretty panics too
    miette::set_hook(Box::new(|_| {
        Box::new(miette::MietteHandlerOpts::new()
            .terminal_links(true)
            .unicode(true)
            .context_lines(2)
            .build())
    }))?;

    let config = load_and_validate("config.toml")?;
    run(config)
}
```

### When to Use Which

| Tool | Best For | Key Feature |
|------|----------|-------------|
| **thiserror** | Library error types | Typed enums, `#[from]` |
| **anyhow** | Application error handling | `.context()`, type erasure |
| **color-eyre** | CLI tools, user-facing apps | Colorized output, SpanTrace |
| **miette** | Compilers, linters, config parsers | Source snippets, labels |

## Error Conversion Patterns

### From Trait Conversions

```rust
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("Database error: {0}")]
    Database(#[from] DatabaseError),

    #[error("Network error: {0}")]
    Network(#[from] reqwest::Error),

    #[error("Configuration error: {0}")]
    Config(#[from] ConfigError),
}

// Now ? automatically converts
fn fetch_user_data(id: &str) -> Result<UserData, AppError> {
    let config = load_config()?;           // ConfigError -> AppError
    let user = db.find_user(id)?;          // DatabaseError -> AppError
    let profile = api.fetch_profile(id)?;  // reqwest::Error -> AppError
    Ok(UserData { user, profile })
}
```

### Map Error for Custom Conversion

```rust
fn parse_port(s: &str) -> Result<u16, ConfigError> {
    s.parse::<u16>()
        .map_err(|_| ConfigError::InvalidValue {
            field: "port".to_string(),
            value: s.to_string(),
        })
}
```

### Error Wrapping for Context

```rust
#[derive(Debug, thiserror::Error)]
pub enum ServiceError {
    #[error("Failed to process order {order_id}: {source}")]
    OrderProcessing {
        order_id: String,
        #[source]
        source: Box<dyn std::error::Error + Send + Sync>,
    },
}

fn process_order(id: &str) -> Result<(), ServiceError> {
    do_processing(id).map_err(|e| ServiceError::OrderProcessing {
        order_id: id.to_string(),
        source: Box::new(e),
    })
}
```

## Error Recovery Strategies

### Fallback Values

```rust
// unwrap_or: constant fallback
let port: u16 = env::var("PORT")
    .ok()
    .and_then(|s| s.parse().ok())
    .unwrap_or(8080);

// unwrap_or_else: computed fallback
let config = load_config()
    .unwrap_or_else(|e| {
        eprintln!("Using default config: {}", e);
        Config::default()
    });

// unwrap_or_default: Default trait
let name: String = get_name().unwrap_or_default();
```

### Retry Logic

```rust
use std::time::Duration;

fn with_retry<T, E, F>(mut f: F, max_attempts: u32) -> Result<T, E>
where
    F: FnMut() -> Result<T, E>,
{
    let mut attempts = 0;
    loop {
        match f() {
            Ok(value) => return Ok(value),
            Err(e) if attempts < max_attempts => {
                attempts += 1;
                std::thread::sleep(Duration::from_millis(100 * attempts as u64));
            }
            Err(e) => return Err(e),
        }
    }
}

let result = with_retry(|| fetch_data(), 3)?;
```

### Async Retry with Exponential Backoff

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn with_backoff<T, E, F, Fut>(mut f: F, max_attempts: u32) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<T, E>>,
{
    let mut attempts = 0;
    loop {
        match f().await {
            Ok(value) => return Ok(value),
            Err(e) if attempts < max_attempts => {
                attempts += 1;
                let delay = Duration::from_millis(100 * 2u64.pow(attempts));
                sleep(delay).await;
            }
            Err(e) => return Err(e),
        }
    }
}
```

### Graceful Degradation

```rust
async fn get_user_with_avatar(id: &str) -> Result<UserProfile, Error> {
    let user = db.get_user(id).await?;  // Required — propagate error

    // Optional enhancement — degrade gracefully
    let avatar = match image_service.get_avatar(id).await {
        Ok(url) => Some(url),
        Err(e) => {
            tracing::warn!("Failed to load avatar for {}: {}", id, e);
            None
        }
    };

    Ok(UserProfile { user, avatar })
}
```

## Collecting Multiple Errors

### Partition Results

```rust
fn process_all(items: Vec<Item>) -> (Vec<Output>, Vec<Error>) {
    let (successes, failures): (Vec<_>, Vec<_>) = items
        .into_iter()
        .map(process_item)
        .partition(Result::is_ok);

    let outputs: Vec<Output> = successes.into_iter().map(Result::unwrap).collect();
    let errors: Vec<Error> = failures.into_iter().map(Result::unwrap_err).collect();

    (outputs, errors)
}
```

### Collect All Errors (Validation)

```rust
fn validate_user(user: &UserInput) -> Result<(), Vec<ValidationError>> {
    let mut errors = Vec::new();

    if user.name.is_empty() {
        errors.push(ValidationError::EmptyName);
    }
    if !user.email.contains('@') {
        errors.push(ValidationError::InvalidEmail);
    }
    if user.age > 150 {
        errors.push(ValidationError::InvalidAge);
    }

    if errors.is_empty() { Ok(()) } else { Err(errors) }
}
```

### First Error vs All Errors

```rust
// Short-circuit at first error
fn process_items(items: Vec<Item>) -> Result<Vec<Output>, Error> {
    items.into_iter()
        .map(process_item)
        .collect()  // Stops at first Err
}

// Collect all errors
fn process_items_all(items: Vec<Item>) -> Result<Vec<Output>, Vec<Error>> {
    let mut outputs = Vec::new();
    let mut errors = Vec::new();

    for item in items {
        match process_item(item) {
            Ok(output) => outputs.push(output),
            Err(e) => errors.push(e),
        }
    }

    if errors.is_empty() { Ok(outputs) } else { Err(errors) }
}
```

## Multi-Layer Error Translation

In layered architectures, errors propagate across boundaries. Each layer should have its own error types.

### Layer-Specific Error Types

```rust
use thiserror::Error;

// --- Domain Layer (innermost) ---
// Pure business errors, no infrastructure concerns
#[derive(Debug, Error)]
pub enum DomainError {
    #[error("User not found: {user_id}")]
    UserNotFound { user_id: u32 },

    #[error("Insufficient balance: required {required}, available {available}")]
    InsufficientBalance { required: u64, available: u64 },

    #[error("Order validation failed: {reason}")]
    ValidationFailed { reason: String },

    #[error("Business rule violated: {rule}")]
    BusinessRuleViolation { rule: String },
}

// --- Application Layer ---
// Orchestrates use cases, translates infrastructure errors
#[derive(Debug, Error)]
pub enum ApplicationError {
    #[error("Domain error: {0}")]
    Domain(#[from] DomainError),

    #[error("Repository operation failed: {operation}")]
    RepositoryFailure {
        operation: String,
        #[source]
        source: Box<dyn std::error::Error + Send + Sync>,
    },

    #[error("External service unavailable: {service}")]
    ServiceUnavailable { service: String },

    #[error("Authorization failed")]
    Unauthorized,
}

// --- Infrastructure Layer (outermost) ---
#[derive(Debug, Error)]
pub enum InfrastructureError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("HTTP error: {0}")]
    Http(#[from] reqwest::Error),

    #[error("Serialization error: {0}")]
    Serialization(#[from] serde_json::Error),

    #[error("Connection pool exhausted")]
    PoolExhausted,
}
```

### Error Translation Across Layers

```rust
// Infrastructure -> Application translation
impl From<InfrastructureError> for ApplicationError {
    fn from(err: InfrastructureError) -> Self {
        match err {
            InfrastructureError::Database(e) => ApplicationError::RepositoryFailure {
                operation: "database query".to_string(),
                source: Box::new(e),
            },
            InfrastructureError::Http(e) if e.is_connect() => {
                ApplicationError::ServiceUnavailable {
                    service: "external API".to_string(),
                }
            }
            InfrastructureError::Http(e) => ApplicationError::RepositoryFailure {
                operation: "HTTP request".to_string(),
                source: Box::new(e),
            },
            InfrastructureError::Serialization(e) => ApplicationError::RepositoryFailure {
                operation: "data serialization".to_string(),
                source: Box::new(e),
            },
            InfrastructureError::PoolExhausted => ApplicationError::ServiceUnavailable {
                service: "database pool".to_string(),
            },
        }
    }
}

// Repository implementation translates errors
pub struct PostgresUserRepository { pool: sqlx::PgPool }

impl UserRepository for PostgresUserRepository {
    async fn find_by_id(&self, id: u32) -> Result<Option<User>, ApplicationError> {
        let result = sqlx::query_as::<_, UserRow>("SELECT * FROM users WHERE id = $1")
            .bind(id as i32)
            .fetch_optional(&self.pool)
            .await
            .map_err(InfrastructureError::Database)?;
        // InfrastructureError automatically converts to ApplicationError via From
        Ok(result.map(User::from))
    }
}
```

### Complete Error Flow Example

```rust
// Use case in application layer
pub struct TransferFundsUseCase<R: AccountRepository> {
    repository: R,
}

impl<R: AccountRepository> TransferFundsUseCase<R> {
    pub async fn execute(
        &self, from_id: u32, to_id: u32, amount: u64,
    ) -> Result<(), ApplicationError> {
        let mut from = self.repository.find_by_id(from_id).await?
            .ok_or(DomainError::UserNotFound { user_id: from_id })?;

        let mut to = self.repository.find_by_id(to_id).await?
            .ok_or(DomainError::UserNotFound { user_id: to_id })?;

        from.withdraw(amount).map_err(|_| DomainError::InsufficientBalance {
            required: amount, available: from.balance,
        })?;

        to.deposit(amount);
        self.repository.save(&from).await?;
        self.repository.save(&to).await?;
        Ok(())
    }
}

// Web handler translates to HTTP response
async fn transfer_handler(
    State(use_case): State<Arc<TransferFundsUseCase<impl AccountRepository>>>,
    Json(req): Json<TransferRequest>,
) -> Result<Json<serde_json::Value>, ApiError> {
    match use_case.execute(req.from_id, req.to_id, req.amount).await {
        Ok(()) => Ok(Json(serde_json::json!({"status": "success"}))),
        Err(ApplicationError::Domain(DomainError::UserNotFound { user_id })) => {
            Err(ApiError::not_found(format!("User {user_id} not found")))
        }
        Err(ApplicationError::Domain(DomainError::InsufficientBalance { required, available })) => {
            Err(ApiError::bad_request(format!("Need {required}, have {available}")))
        }
        Err(ApplicationError::Unauthorized) => Err(ApiError::unauthorized()),
        Err(e) => {
            tracing::error!(error = %e, "Internal error");
            Err(ApiError::internal())
        }
    }
}
```

### Error Mapping Macros

A utility macro for consistent error mapping across layers:

```rust
/// Maps errors to a unified error type with status codes
#[macro_export]
macro_rules! safe_eject {
    // Basic: map error to status
    ($e:expr, $err_status:expr) => {
        $e.map_err(|x| ServiceError {
            message: x.to_string(),
            status: $err_status,
        })
    };
    // With context: add prefix to error message
    ($e:expr, $err_status:expr, $context:expr) => {
        $e.map_err(|x| ServiceError {
            message: format!("{}: {}", $context, x),
            status: $err_status,
        })
    };
}

// Usage
fn parse_config(data: &str) -> Result<Config, ServiceError> {
    let parsed = safe_eject!(
        serde_json::from_str::<Config>(data),
        ErrorStatus::BadRequest
    )?;

    let validated = safe_eject!(
        validate_config(&parsed),
        ErrorStatus::BadRequest,
        "Config validation failed"
    )?;

    Ok(validated)
}

fn read_items(path: &str) -> Result<Vec<Item>, ServiceError> {
    let data = safe_eject!(
        std::fs::read_to_string(path),
        ErrorStatus::InternalError,
        "Failed to read data file"
    )?;

    safe_eject!(
        serde_json::from_str(&data),
        ErrorStatus::InternalError,
        "Failed to parse data file"
    )
}
```

### Web Framework Error Integration

```rust
// --- Actix Web: ResponseError ---
use actix_web::{HttpResponse, http::StatusCode as ActixStatus, error::ResponseError};

#[derive(Debug, Clone, Copy)]
pub enum ErrorStatus {
    NotFound,
    BadRequest,
    Unauthorized,
    Forbidden,
    Conflict,
    InternalError,
}

#[derive(Debug, thiserror::Error)]
#[error("{message}")]
pub struct ServiceError {
    pub message: String,
    pub status: ErrorStatus,
}

impl ResponseError for ServiceError {
    fn status_code(&self) -> ActixStatus {
        match self.status {
            ErrorStatus::NotFound => ActixStatus::NOT_FOUND,
            ErrorStatus::BadRequest => ActixStatus::BAD_REQUEST,
            ErrorStatus::Unauthorized => ActixStatus::UNAUTHORIZED,
            ErrorStatus::Forbidden => ActixStatus::FORBIDDEN,
            ErrorStatus::Conflict => ActixStatus::CONFLICT,
            ErrorStatus::InternalError => ActixStatus::INTERNAL_SERVER_ERROR,
        }
    }

    fn error_response(&self) -> HttpResponse {
        HttpResponse::build(self.status_code())
            .json(serde_json::json!({
                "error": self.message,
                "status": self.status_code().as_u16()
            }))
    }
}

// Actix handlers can return Result<T, ServiceError>
async fn get_user(id: web::Path<u64>) -> Result<HttpResponse, ServiceError> {
    let user = find_user(*id)
        .ok_or(ServiceError {
            message: format!("User {} not found", id),
            status: ErrorStatus::NotFound,
        })?;
    Ok(HttpResponse::Ok().json(user))
}
```

```rust
// --- Axum: IntoResponse ---
use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};

#[derive(Debug)]
pub struct ApiError {
    status: StatusCode,
    message: String,
}

impl ApiError {
    pub fn not_found(msg: impl Into<String>) -> Self {
        Self { status: StatusCode::NOT_FOUND, message: msg.into() }
    }
    pub fn bad_request(msg: impl Into<String>) -> Self {
        Self { status: StatusCode::BAD_REQUEST, message: msg.into() }
    }
    pub fn unauthorized() -> Self {
        Self { status: StatusCode::UNAUTHORIZED, message: "Unauthorized".into() }
    }
    pub fn internal() -> Self {
        Self { status: StatusCode::INTERNAL_SERVER_ERROR, message: "Internal error".into() }
    }
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let body = Json(serde_json::json!({
            "error": self.message,
            "status": self.status.as_u16()
        }));
        (self.status, body).into_response()
    }
}
```

### Error Translation Best Practices

| Practice | Description |
|----------|-------------|
| **Domain errors are pure** | No infrastructure types in domain layer |
| **Translate at boundaries** | Convert errors when crossing layer boundaries |
| **Preserve source chain** | Use `#[source]` for debugging |
| **Log at entry points** | Log full chains at HTTP handlers, CLI entry |
| **Map to user-facing** | Convert to HTTP status codes at outermost layer |
| **Don't leak internals** | Never expose database details to users |

## Testing Error Conditions

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn returns_error_on_invalid_input() {
        let result = parse_config("");
        assert!(result.is_err());
    }

    #[test]
    fn returns_specific_error_variant() {
        let result = parse_config("");
        assert!(matches!(result, Err(ConfigError::ParseError(_))));
    }

    #[test]
    fn error_message_contains_context() {
        let result = parse_config("invalid");
        let err = result.unwrap_err();
        assert!(err.to_string().contains("parse"));
    }

    #[test]
    #[should_panic(expected = "must be non-zero")]
    fn panics_on_zero_divisor() {
        calculate(0);
    }

    #[test]
    fn error_source_is_preserved() {
        let err = ConfigError::IoError(
            std::io::Error::new(std::io::ErrorKind::NotFound, "file not found")
        );
        assert!(err.source().is_some());
    }

    // Testing with color-eyre
    #[test]
    fn error_chain_has_context() {
        let err = load_config("/nonexistent")
            .unwrap_err();
        let chain: Vec<_> = err.chain().map(|e| e.to_string()).collect();
        assert!(chain.iter().any(|msg| msg.contains("config")));
    }
}
```

## Numeric Error Codes for Protocol Boundaries

When errors cross language boundaries (FFI, network protocols, binary APIs), string-based errors are fragile. Use `#[repr(u32)]` enums with numeric discriminants for machine-readable error codes that serialize compactly and are stable across versions.

```rust
use num_enum::{TryFromPrimitive, IntoPrimitive};
use derive_more::Display;

/// Error codes organized by category prefix (hex ranges)
/// 0x0000 = general, 0x1000 = local track, 0x2000 = remote track, 0x3000 = mixer
#[derive(Debug, Clone, Copy, PartialEq, Eq, TryFromPrimitive, IntoPrimitive, Display)]
#[repr(u32)]
pub enum EndpointError {
    #[display("endpoint not in room")]
    NotInRoom = 0x0001,
    #[display("local track has no source")]
    LocalTrackNoSource = 0x1001,
    #[display("invalid local track priority")]
    LocalTrackInvalidPriority = 0x1002,
    #[display("remote track stopped")]
    RemoteTrackStopped = 0x2001,
    #[display("audio mixer wrong mode")]
    AudioMixerWrongMode = 0x3001,
    #[display("endpoint shutting down")]
    Destroying = 0x4001,
}

// Convert to/from u32 for wire protocols
let code: u32 = EndpointError::NotInRoom.into();       // 0x0001
let err = EndpointError::try_from(0x2001u32);           // Ok(RemoteTrackStopped)
let unknown = EndpointError::try_from(0xFFFFu32);       // Err(TryFromPrimitiveError)
```

**When to use numeric codes vs thiserror strings:**

| Error crosses... | Use | Why |
|-----------------|-----|-----|
| Crate boundaries (Rust → Rust) | `thiserror` enums | Type-safe, pattern matching |
| Language boundaries (Rust → C/JS) | `#[repr(u32)]` codes | Stable ABI, no string allocation |
| Network/protocol boundaries | `#[repr(u32)]` codes | Compact, version-independent |
| User-facing output | `Display` impl on either | Human-readable messages |

The pattern used by atm0s-media-server: `num_enum` for safe `u32 ↔ enum` conversion, `derive_more::Display` for human-readable messages. Category prefixes (0x1000, 0x2000) enable range-based error classification without matching every variant.

## Crate Comparison Summary

```
thiserror  → Library error types (typed enums, pattern matching)
anyhow     → Application errors (context chaining, type erasure)
color-eyre → User-facing apps (colorized output, SpanTrace, suggestions)
miette     → Source-aware diagnostics (code snippets, labels, help text)
```

Choose based on who sees the error:
- **Other code** (libraries) → `thiserror`
- **Developers** (logs, servers) → `anyhow`
- **Users** (CLI tools) → `color-eyre`
- **Users + source context** (compilers, linters) → `miette`

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: `Result`/`Option`, `?` operator, error design essentials
- **[architecture.md](architecture.md)** — Multi-layer error translation, domain vs infrastructure errors
- **[web-apis.md](web-apis.md)** — HTTP error mapping, `IntoResponse` for error types
- **[language-patterns.md](language-patterns.md)** — `?` operator chains with `.context()`, error propagation patterns
- **[testing.md](testing.md)** — Testing error paths, `assert!(matches!(result, Err(...)))` patterns
