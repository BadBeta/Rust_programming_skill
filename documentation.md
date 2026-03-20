# Documentation in Rust

Rustdoc conventions, doc comments, doc tests, intra-doc links, feature-gated docs, docs.rs publishing, and crate-level documentation architecture — patterns sourced from serde, axum, anyhow, and tokio.

## Rules for Documentation (LLM)

1. **ALWAYS write `///` doc comments for all public items** — rustdoc generates API docs from these; undocumented public items are a bug in library crates
2. **ALWAYS include `# Examples` sections in doc comments for non-trivial public functions** — doc examples are tested by `cargo test`, serving as both documentation and regression tests
3. **ALWAYS use `//!` for module-level and crate-level documentation** — place in `lib.rs` or at the top of module files; this is the first thing users see in generated docs
4. **ALWAYS document error conditions with `# Errors`** — list which error variants can be returned and under what conditions; callers need this to handle errors correctly
5. **ALWAYS document panic conditions with `# Panics`** — if a function can panic, document exactly when; this is a contract with callers
6. **ALWAYS document unsafe with `# Safety`** — every `unsafe fn` and `unsafe` block must document the invariants the caller must uphold
7. **ALWAYS use intra-doc links** (`[`Type`]`, `[`Type::method`]`) instead of URLs — they are checked by rustdoc, survive refactoring, and link correctly across crate boundaries
8. **ALWAYS add `#[doc(hidden)]` to public items that are implementation details** — proc macro internals, re-exports for macro use, and unstable APIs should be hidden from generated docs
9. **PREFER `#[doc = include_str!("docs/topic.md")]` for long documentation** — keeps source files readable while maintaining comprehensive docs (pattern from axum)
10. **ALWAYS configure `[package.metadata.docs.rs]` for docs.rs builds** — specify features, targets, and rustdoc args so docs.rs renders your crate correctly

### Common Mistakes (BAD/GOOD)

**Missing doc sections:**
```rust
// BAD: no indication of failure modes
/// Connects to the database.
pub fn connect(url: &str) -> Result<Connection, DbError> { /* ... */ }

// GOOD: complete contract
/// Connects to the database at the given URL.
///
/// # Errors
///
/// Returns [`DbError::InvalidUrl`] if the URL cannot be parsed.
/// Returns [`DbError::ConnectionFailed`] if the server is unreachable.
///
/// # Examples
///
/// ```no_run
/// let conn = mylib::connect("postgres://localhost/mydb")?;
/// # Ok::<(), mylib::DbError>(())
/// ```
pub fn connect(url: &str) -> Result<Connection, DbError> { /* ... */ }
```

**Hardcoded URLs instead of intra-doc links:**
```rust
// BAD: URL breaks when items are renamed or moved
/// See [Config](https://docs.rs/mycrate/0.1.0/mycrate/struct.Config.html)

// GOOD: intra-doc link — checked by rustdoc, survives refactoring
/// See [`Config`] for available options.
/// Uses [`Config::default()`] when no file is provided.
```

**Doc tests that hide the point:**
```rust
// BAD: boilerplate obscures the example
/// ```
/// use std::collections::HashMap;
/// use mycrate::config::{Config, ConfigBuilder, Environment};
/// let mut env = HashMap::new();
/// env.insert("PORT".to_string(), "8080".to_string());
/// let config = ConfigBuilder::new().env(env).build().unwrap();
/// assert_eq!(config.port, 8080);
/// ```

// GOOD: use # to hide setup, show only the point
/// ```
/// # use mycrate::config::ConfigBuilder;
/// let config = ConfigBuilder::new()
///     .port(8080)
///     .host("localhost")
///     .build()?;
/// assert_eq!(config.port, 8080);
/// # Ok::<(), Box<dyn std::error::Error>>(())
/// ```
```

## Doc Comment Syntax

### Item Documentation (`///`)

```rust
/// Short summary line — appears in module listing and search results.
///
/// Longer description with **markdown** formatting. This paragraph explains
/// the purpose, behavior, and design rationale.
///
/// # Examples
///
/// ```
/// use mycrate::Widget;
///
/// let widget = Widget::new("example");
/// assert_eq!(widget.name(), "example");
/// ```
///
/// # Errors
///
/// Returns [`WidgetError::InvalidName`] if `name` is empty.
///
/// # Panics
///
/// Panics if the global allocator is exhausted.
///
/// # Safety
///
/// (Only for `unsafe fn`) The caller must ensure that `ptr` is valid
/// and aligned to `align_of::<T>()`.
pub fn new(name: &str) -> Result<Widget, WidgetError> {
    // ...
}
```

### Module/Crate Documentation (`//!`)

```rust
//! # My Crate
//!
//! `mycrate` provides utilities for processing widgets.
//!
//! ## Features
//!
//! - Fast widget parsing with zero-copy deserialization
//! - Async processing via tokio
//! - Comprehensive error types
//!
//! ## Quick Start
//!
//! ```
//! use mycrate::Widget;
//!
//! let widget = Widget::parse("data")?;
//! println!("{}", widget.summary());
//! # Ok::<(), mycrate::Error>(())
//! ```
//!
//! ## Feature Flags
//!
//! | Feature | Description | Default |
//! |---------|-------------|---------|
//! | `json`  | JSON serialization support | Yes |
//! | `async` | Async processing with tokio | No |
//! | `cli`   | Command-line interface | No |

#![doc(html_root_url = "https://docs.rs/mycrate/0.1.0")]
#![cfg_attr(docsrs, feature(doc_cfg))]
```

### Doc Comment Sections — Standard Order

| Section | When to Include | Content |
|---------|----------------|---------|
| Summary line | Always | One-line description (shown in listings) |
| Description | When non-obvious | Behavior, design rationale, trade-offs |
| `# Examples` | Public items | Working code tested by `cargo test` |
| `# Errors` | Returns `Result` | Which error variants and when |
| `# Panics` | Can panic | Exact conditions that trigger panic |
| `# Safety` | `unsafe fn` | Invariants the caller must uphold |
| `# Arguments` | Complex params | Only when parameter names aren't self-documenting |

## Doc Tests

Doc tests are compiled and run by `cargo test`. They serve as both examples and regression tests.

### Doc Test Attributes

```rust
/// Always runs (default):
/// ```
/// assert_eq!(2 + 2, 4);
/// ```
///
/// Compiles but doesn't run (network, filesystem, etc.):
/// ```no_run
/// let response = reqwest::blocking::get("https://example.com")?;
/// # Ok::<(), reqwest::Error>(())
/// ```
///
/// Expected to fail compilation (negative test):
/// ```compile_fail
/// let x: i32 = "not a number";
/// ```
///
/// Expected to panic:
/// ```should_panic
/// panic!("this is expected");
/// ```
///
/// Skipped entirely (pseudocode, other languages):
/// ```ignore
/// This is not Rust code, just illustration.
/// ```
///
/// Not even highlighted as Rust:
/// ```text
/// This is plain text output.
/// ```
pub fn example() {}
```

### Hiding Setup Lines with `#`

Lines prefixed with `# ` are compiled but hidden from rendered docs:

```rust
/// Parse a configuration file.
///
/// ```
/// # use std::collections::HashMap;
/// # fn main() -> Result<(), Box<dyn std::error::Error>> {
/// let config = mycrate::parse_config("host = 'localhost'")?;
/// assert_eq!(config.host, "localhost");
/// # Ok(())
/// # }
/// ```
pub fn parse_config(input: &str) -> Result<Config, Error> {
    // ...
}
```

**Common hidden patterns:**

```rust
/// ```
/// # use mycrate::Error;
/// # fn main() -> Result<(), Error> {
/// // ... visible example code ...
/// # Ok(())
/// # }
/// ```
```

### Testing Doc Tests

```bash
cargo test --doc                    # Run only doc tests
cargo test --doc -- parse_config    # Run doc tests for specific item
cargo test                          # Runs unit, integration, AND doc tests
```

## Intra-Doc Links

Rustdoc resolves links to items within your crate and dependencies. These are checked at build time — broken links produce warnings (errors with `#![deny(rustdoc::broken_intra_doc_links)]`).

### Link Syntax

```rust
/// Returns a [`Widget`] configured with the given [`Config`].
///
/// Uses [`Config::default()`] if no config is provided.
/// See the [`builder`](crate::builder) module for advanced configuration.
///
/// For error handling, see [`WidgetError`] and the
/// [`error`](crate::error) module.
///
/// This is equivalent to calling [`Widget::builder()`] followed by
/// [`WidgetBuilder::build()`].
pub fn create(config: Option<Config>) -> Result<Widget, WidgetError> {
    // ...
}
```

### Link Target Syntax

| Syntax | Links To |
|--------|----------|
| `` [`Type`] `` | Struct, enum, trait, or type alias |
| `` [`Type::method`] `` | Method or associated function |
| `` [`Type::CONST`] `` | Associated constant |
| `` [`module`] `` | Module |
| `` [`crate::path::Type`] `` | Absolute path within crate |
| `` [`trait@Send`] `` | Disambiguate: trait named `Send` |
| `` [`struct@Config`] `` | Disambiguate: struct named `Config` |
| `` [`mod@util`] `` | Disambiguate: module named `util` |
| `` [`value@TIMEOUT`] `` | Disambiguate: constant named `TIMEOUT` |
| `` [`macro@vec`] `` | Disambiguate: macro named `vec` |
| `` [display text](`Type`) `` | Custom display text |

### Enforcing Link Correctness

```rust
// In lib.rs — make broken links a compile error
#![deny(rustdoc::broken_intra_doc_links)]
#![deny(rustdoc::private_intra_doc_links)]

// Also warn on missing docs for public items
#![warn(missing_docs)]
```

### Cross-Crate Links

```rust
/// Wraps a [`std::io::Error`] with additional context.
///
/// Compatible with [`serde::Serialize`] when the `serde` feature is enabled.
///
/// See [`tokio::spawn`] for running this asynchronously.
pub struct MyError {
    // ...
}
```

## Doc Attributes

### Visibility Control

```rust
// Show re-exported item inline in this crate's docs
#[doc(inline)]
pub use self::extract::Json;

// Link to the original crate's docs instead of duplicating
#[doc(no_inline)]
pub use http::StatusCode;

// Hide from generated docs entirely
// Common for: proc macro internals, unstable APIs, backward-compat re-exports
#[doc(hidden)]
pub mod __private {
    // Used by proc macros but not part of public API
}

// Add search aliases (user types "connection" and finds "Pool")
#[doc(alias = "connection")]
#[doc(alias = "database")]
pub struct Pool { /* ... */ }
```

### Feature-Gated Documentation

Show which features are required for an item on docs.rs:

```rust
// In lib.rs — enable doc_cfg on docs.rs builds only
#![cfg_attr(docsrs, feature(doc_cfg))]

// Mark items with their required feature
#[cfg(feature = "json")]
#[cfg_attr(docsrs, doc(cfg(feature = "json")))]
pub mod json {
    //! JSON serialization support.
    //!
    //! Requires the `json` feature flag.
}

// Re-exports with feature gates (pattern from axum)
#[cfg(feature = "multipart")]
#[cfg_attr(docsrs, doc(cfg(feature = "multipart")))]
pub use self::extract::Multipart;
```

This renders as a badge on docs.rs: `Available on crate feature json only.`

### Including External Documentation

```rust
// Embed markdown files as documentation (pattern from axum)
#[doc = include_str!("docs/routing.md")]
pub mod routing;

// Crate-level docs from README
#![doc = include_str!("../README.md")]

// Conditional inclusion
#[cfg_attr(feature = "unstable", doc = include_str!("docs/unstable.md"))]
pub mod experimental;
```

### Conditional Compilation for Docs

```rust
// Items that only exist in docs (for illustration)
#[cfg(doc)]
pub mod examples {
    //! Example types used in documentation only.
    //! Not compiled into the actual library.
}

// Different doc content for different targets
#[cfg_attr(unix, doc = "On Unix, uses file descriptors.")]
#[cfg_attr(windows, doc = "On Windows, uses HANDLEs.")]
pub struct Handle;
```

## Crate-Level Documentation Architecture

### Production Crate Pattern (from serde, axum, anyhow)

```rust
// lib.rs

//! # MyCrate
//!
//! Short tagline describing the crate.
//!
//! ## Overview
//!
//! Longer description of what the crate does and why.
//!
//! ## Quick Start
//!
//! ```
//! use mycrate::Client;
//!
//! let client = Client::new();
//! let result = client.process("data")?;
//! # Ok::<(), mycrate::Error>(())
//! ```
//!
//! ## Feature Flags
//!
//! | Feature | Default | Description |
//! |---------|---------|-------------|
//! | `serde` | No | Enable Serialize/Deserialize derives |
//! | `async` | Yes | Async support via tokio |
//!
//! ## Modules
//!
//! - [`client`] — HTTP client and connection management
//! - [`error`] — Error types and Result alias
//! - [`config`] — Configuration and builder patterns

#![doc(html_root_url = "https://docs.rs/mycrate/0.1.0")]
#![cfg_attr(docsrs, feature(doc_cfg))]
#![deny(rustdoc::broken_intra_doc_links)]
#![warn(missing_docs)]

pub mod client;
pub mod config;
pub mod error;

// Re-exports for convenience — show inline in docs
#[doc(inline)]
pub use client::Client;
#[doc(inline)]
pub use config::Config;
#[doc(inline)]
pub use error::{Error, Result};
```

### Module Documentation Pattern

```rust
// src/client.rs

//! HTTP client for connecting to the API.
//!
//! # Examples
//!
//! ```no_run
//! use mycrate::Client;
//!
//! # #[tokio::main]
//! # async fn main() -> mycrate::Result<()> {
//! let client = Client::builder()
//!     .timeout(std::time::Duration::from_secs(30))
//!     .build()?;
//!
//! let response = client.get("/users").await?;
//! # Ok(())
//! # }
//! ```

use crate::config::Config;
use crate::error::Result;
```

## Running and Configuring Rustdoc

### Generating Documentation

```bash
cargo doc                           # Build docs for your crate + dependencies
cargo doc --open                    # Build and open in browser
cargo doc --no-deps                 # Only your crate (faster)
cargo doc --document-private-items  # Include private items (for internal docs)
cargo doc --all-features            # Enable all feature flags

# Check for documentation warnings/errors
RUSTDOCFLAGS="--deny warnings" cargo doc --no-deps

# Generate JSON output (unstable, for tooling)
cargo +nightly rustdoc -- --output-format json
```

### docs.rs Configuration

Configure how docs.rs builds your crate's documentation:

```toml
# Cargo.toml

[package.metadata.docs.rs]
# Enable all features for docs.rs build
all-features = true

# Or specify features explicitly
features = ["json", "async", "unstable"]

# Build for specific targets
targets = ["x86_64-unknown-linux-gnu"]

# Pass flags to rustdoc
rustdoc-args = ["--cfg", "docsrs"]

# Set cargo args
cargo-args = ["-Zunstable-options", "-Zrustdoc-scrape-examples"]
```

### README as Crate Docs

Reuse your README.md as crate-level documentation:

```rust
// lib.rs — embed the README
#![doc = include_str!("../README.md")]
```

To keep doc tests in the README working:

```toml
# Cargo.toml
[lib]
doctest = true  # default, but be explicit
```

```markdown
<!-- README.md — doc tests work here too -->
# My Crate

```rust
use mycrate::Widget;
let w = Widget::new("test");
assert!(w.is_valid());
```
```

## Lints and Quality

### Rustdoc Lints

```rust
// Recommended for library crates
#![deny(rustdoc::broken_intra_doc_links)]   // Broken [`links`]
#![deny(rustdoc::private_intra_doc_links)]  // Links to private items
#![warn(rustdoc::missing_crate_level_docs)] // Missing //! in lib.rs
#![warn(missing_docs)]                       // Missing /// on public items

// In Cargo.toml (alternative)
// [lints.rust]
// missing_docs = "warn"
//
// [lints.rustdoc]
// broken_intra_doc_links = "deny"
```

### Clippy Documentation Lints

```bash
# Check documentation quality
cargo clippy -- -W clippy::missing_docs_in_private_items
cargo clippy -- -W clippy::missing_errors_doc     # Missing # Errors
cargo clippy -- -W clippy::missing_panics_doc     # Missing # Panics
cargo clippy -- -W clippy::missing_safety_doc     # Missing # Safety
```

## Badge Patterns

### Crate Documentation Badges

```markdown
<!-- In README.md -->
[![Crates.io](https://img.shields.io/crates/v/mycrate.svg)](https://crates.io/crates/mycrate)
[![Documentation](https://docs.rs/mycrate/badge.svg)](https://docs.rs/mycrate)
[![CI](https://github.com/user/mycrate/actions/workflows/ci.yml/badge.svg)](https://github.com/user/mycrate/actions)
[![License](https://img.shields.io/crates/l/mycrate.svg)](LICENSE)
```

### In Doc Comments (crate-level)

```rust
//! [![crates.io](https://img.shields.io/crates/v/mycrate.svg)](https://crates.io/crates/mycrate)
//! [![docs.rs](https://docs.rs/mycrate/badge.svg)](https://docs.rs/mycrate)
//!
//! # MyCrate
//!
//! Short description.
```

## Documenting Traits

```rust
/// A data source that can be queried asynchronously.
///
/// # Implementor Notes
///
/// Implementations must be [`Send`] and [`Sync`] for use with async runtimes.
/// The [`fetch`](DataSource::fetch) method should be idempotent — calling it
/// multiple times with the same `id` must return the same result.
///
/// # Examples
///
/// Implementing for a PostgreSQL backend:
///
/// ```
/// # use mycrate::DataSource;
/// struct PgSource { /* ... */ }
///
/// impl DataSource for PgSource {
///     type Error = sqlx::Error;
///
///     async fn fetch(&self, id: u64) -> Result<Vec<u8>, Self::Error> {
///         // ...
///         # todo!()
///     }
/// }
/// ```
pub trait DataSource: Send + Sync {
    /// The error type returned by this data source.
    type Error: std::error::Error;

    /// Fetches data for the given `id`.
    ///
    /// # Errors
    ///
    /// Returns [`Self::Error`] if the fetch fails due to network
    /// or database issues.
    async fn fetch(&self, id: u64) -> Result<Vec<u8>, Self::Error>;
}
```

## Documenting Macros

```rust
/// Creates a [`HashMap`] from key-value pairs.
///
/// # Examples
///
/// ```
/// use mycrate::hashmap;
///
/// let map = hashmap! {
///     "key1" => 1,
///     "key2" => 2,
/// };
/// assert_eq!(map["key1"], 1);
/// ```
///
/// Empty map:
///
/// ```
/// # use mycrate::hashmap;
/// let empty: std::collections::HashMap<String, i32> = hashmap! {};
/// assert!(empty.is_empty());
/// ```
#[macro_export]
macro_rules! hashmap {
    // ...
}
```

## Documenting Unsafe Code

```rust
/// Converts a byte slice to a string without checking for valid UTF-8.
///
/// # Safety
///
/// The caller must ensure that `bytes` contains valid UTF-8 data.
/// Passing invalid UTF-8 is undefined behavior.
///
/// # Examples
///
/// ```
/// # unsafe {
/// let bytes = b"hello";
/// let s = unsafe { mycrate::str_from_bytes_unchecked(bytes) };
/// assert_eq!(s, "hello");
/// # }
/// ```
pub unsafe fn str_from_bytes_unchecked(bytes: &[u8]) -> &str {
    std::str::from_utf8_unchecked(bytes)
}
```

## Documentation Checklist

| Item | Check |
|------|-------|
| `//!` crate-level docs in `lib.rs` | Overview, quick start, feature table |
| `///` on all public items | Summary + relevant sections |
| `# Examples` with tested code | At least one runnable example per public fn |
| `# Errors` on Result-returning fns | List error variants and conditions |
| `# Panics` on panicking fns | Document exact panic conditions |
| `# Safety` on unsafe fns | Document caller's obligations |
| Intra-doc links for all type references | `[`Type`]` not URLs |
| `#[deny(rustdoc::broken_intra_doc_links)]` | In lib.rs |
| `#[warn(missing_docs)]` | In lib.rs for libraries |
| `[package.metadata.docs.rs]` | In Cargo.toml |
| `#[doc(hidden)]` on internal public items | Macro internals, compat shims |
| Feature-gated items documented | `#[cfg_attr(docsrs, doc(cfg(...)))]` |

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: doc attribute basics (`#[doc(inline)]`, `#[doc(hidden)]`), module visibility
- **[testing.md](testing.md)** — Doc tests as part of the test suite, `cargo test --doc`
- **[macros.md](macros.md)** — Documenting proc macros and derive macros
- **[deployment.md](deployment.md)** — CI/CD integration for doc builds, publishing to docs.rs
- **[architecture.md](architecture.md)** — Crate organization, module structure, public API surface
