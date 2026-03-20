# Testing in Rust

Unit tests, integration tests, mocking, async testing, property-based testing, snapshot testing, fuzz testing, database testing, E2E patterns, and microservice testing strategies.

## Rules for Testing (LLM)

1. **ALWAYS test error paths, not just happy paths** — every `Result::Err` variant and every error branch must have at least one test; LLMs consistently under-test failure cases
2. **NEVER mock what you don't own** — mock trait boundaries you defined, not third-party crate internals; if you need to mock `reqwest`, wrap it in your own `HttpClient` trait first
3. **ALWAYS use `#[cfg(test)]` for test modules and `#[cfg(test)]` for test-only dependencies** — test code must never compile into release binaries
4. **ALWAYS test with `assert!(matches!(result, Err(MyError::Specific(_))))` for error variants** — not just `assert!(result.is_err())` which passes for any error type
5. **NEVER use `unwrap()` or `expect()` in tests without intent** — if the test should fail on `Err`, use `?` with `-> Result<(), Box<dyn Error>>` return type for clear error messages; use `unwrap()` only when panicking IS the assertion
6. **ALWAYS isolate database tests with transactions that roll back** — never leave test data in shared databases; use `sqlx::test` or manual `BEGIN`/`ROLLBACK` wrappers
7. **PREFER property-based tests for data transformation functions** — `proptest` catches edge cases (empty strings, unicode, overflow) that hand-written tests miss
8. **ALWAYS run `cargo +nightly miri test` on unsafe code** — MIRI detects undefined behavior, use-after-free, and alignment violations that normal tests cannot catch

### Common Mistakes (BAD/GOOD)

**Testing compiler guarantees:**
```rust
// BAD: the compiler already prevents this — the test adds nothing
#[test]
fn test_user_has_name_field() {
    let u = User { name: "Alice".into(), age: 30 };
    assert_eq!(u.name, "Alice");  // This tests struct field access, not behavior
}

// GOOD: test domain logic the compiler can't verify
#[test]
fn test_user_display_name_falls_back_to_email() {
    let user = User { name: None, email: "alice@example.com".into() };
    assert_eq!(user.display_name(), "alice@example.com");
}
```

**Testing mock wiring instead of behavior:**
```rust
// BAD: only verifies mock was called, not that the system works correctly
mock.expect_send_email().times(1).returning(|_| Ok(()));
service.register(&user);
// What if register() called send_email but with wrong content?

// GOOD: assert on the result and observable side effects
let result = service.register(&user);
assert!(result.is_ok());
assert!(mock.last_email().unwrap().contains("Welcome"));
```

**Brittle internal-detail tests:**
```rust
// BAD: breaks when internal algorithm changes
assert_eq!(cache.eviction_count(), 3);
assert_eq!(cache.internal_buckets(), 16);

// GOOD: test the observable behavior
cache.insert("a", 1);
cache.insert("b", 2);
assert_eq!(cache.get("a"), Some(&1));
```

### Section Index

| Section | Topics |
|---------|--------|
| [Test-Driven Development](#test-driven-development-tdd) | Red-green-refactor cycle, Rust-specific TDD, type-driven design |
| [Test Organization](#test-organization) | Module structure, integration tests, doc tests, test utilities |
| [Assert Patterns](#assert-patterns) | assert macros, custom matchers, error variant matching |
| [Creating Mock Implementations](#creating-mock-implementations) | mockall, manual mocks, trait-based mocking |
| [Unit Test Isolation](#unit-test-isolation) | Dependency injection for tests, test doubles |
| [Testing Async Code](#testing-async-code) | #[tokio::test], timeouts, async mock expectations |
| [Integration Testing](#integration-testing-with-real-dependencies) | Database tests, HTTP tests, sqlx::test, testcontainers |
| [E2E Testing](#end-to-end-e2e-testing) | Full-stack tests, reqwest-based API tests |
| [Microservice E2E Testing](#microservice-end-to-end-testing) | Multi-service test orchestration, docker-compose |
| [Snapshot Testing](#snapshot-testing-insta) | insta crate, review workflow, JSON/YAML snapshots |
| [Property-Based Testing](#property-based-testing-proptest) | proptest strategies, shrinking, regex generators |
| [Fuzz Testing](#fuzz-testing-cargo-fuzz) | cargo-fuzz, arbitrary, corpus management, coverage |
| [Test Configuration](#test-configuration) | CI settings, feature flags, test filtering |
| [Test Design Best Practices](#test-design-best-practices) | What to test, test naming, coverage |

## Test-Driven Development (TDD)

### The Rust TDD Cycle

TDD in Rust follows **Red → Green → Refactor**, but the compiler adds a unique dimension. Type checking, borrow checking, and exhaustive match verification act as a **free verification layer** that catches entire bug classes other languages rely on tests for. This means Rust TDD focuses tests on behavior and domain logic, not type correctness.

```
┌─────────────────────────────────────────────────────────────┐
│  1. DESIGN    — Define types, traits, error enums (contract)│
│  2. RED       — Write a failing test (may not compile yet)  │
│  3. GREEN     — Write minimum code to pass                  │
│  4. REFACTOR  — Improve design while tests stay green       │
│  5. REPEAT    — Next behavior                               │
└─────────────────────────────────────────────────────────────┘
```

**Tooling for tight feedback loops:**

```bash
# Auto-run tests on save — the core TDD workflow
cargo install cargo-watch
cargo watch -x test                        # All tests
cargo watch -x 'test -- test_order'        # Filter by name
cargo watch -x 'test -- --nocapture'       # Show println output

# Run only tests in the module you're working on
cargo watch -x 'test --lib order::tests'
```

### Trait-First Design — Tests Drive the API

In Rust TDD, you typically **define the trait (contract) before the implementation**. The trait is your specification — tests verify behavior through the trait, and implementations can be swapped.

```rust
// ── Step 1: DESIGN — Define domain types and the trait contract ──

#[derive(Debug, Clone)]
pub struct Money {
    cents: i64,
    currency: Currency,
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum Currency { USD, EUR, GBP }

#[derive(Debug, thiserror::Error)]
pub enum PricingError {
    #[error("no price rule for product {0}")]
    NoPriceRule(String),
    #[error("quantity {0} exceeds maximum order size")]
    QuantityExceeded(u32),
    #[error("currency mismatch: expected {expected:?}, got {got:?}")]
    CurrencyMismatch { expected: Currency, got: Currency },
}

/// The contract — all pricing strategies implement this
pub trait PricingStrategy {
    fn calculate_price(
        &self,
        product_id: &str,
        quantity: u32,
    ) -> Result<Money, PricingError>;
}
```

```rust
// ── Step 2: RED — Write tests before implementation ──

#[cfg(test)]
mod tests {
    use super::*;

    // Test 1: Basic price calculation
    #[test]
    fn calculates_unit_price_times_quantity() {
        let pricer = StandardPricer::new(vec![
            PriceRule::new("BOLT-M8", 150, Currency::USD),  // $1.50 each
        ]);
        let total = pricer.calculate_price("BOLT-M8", 10).unwrap();
        assert_eq!(total.cents, 1500);  // 10 × $1.50
    }

    // Test 2: Error case — unknown product
    #[test]
    fn unknown_product_returns_error() {
        let pricer = StandardPricer::new(vec![]);
        let result = pricer.calculate_price("NONEXISTENT", 1);
        assert!(matches!(result, Err(PricingError::NoPriceRule(ref id)) if id == "NONEXISTENT"));
    }

    // Test 3: Error case — quantity limit
    #[test]
    fn quantity_exceeding_max_returns_error() {
        let pricer = StandardPricer::new(vec![
            PriceRule::new("BOLT-M8", 150, Currency::USD),
        ]);
        let result = pricer.calculate_price("BOLT-M8", 10_001);
        assert!(matches!(result, Err(PricingError::QuantityExceeded(10_001))));
    }
}
// These tests won't compile yet — StandardPricer and PriceRule don't exist.
// That's the RED state. The compiler error IS the failing test.
```

```rust
// ── Step 3: GREEN — Minimum implementation to pass ──

pub struct PriceRule {
    product_id: String,
    unit_price_cents: i64,
    currency: Currency,
}

impl PriceRule {
    pub fn new(product_id: &str, unit_price_cents: i64, currency: Currency) -> Self {
        Self {
            product_id: product_id.to_string(),
            unit_price_cents,
            currency,
        }
    }
}

pub struct StandardPricer {
    rules: Vec<PriceRule>,
    max_quantity: u32,
}

impl StandardPricer {
    pub fn new(rules: Vec<PriceRule>) -> Self {
        Self { rules, max_quantity: 10_000 }
    }
}

impl PricingStrategy for StandardPricer {
    fn calculate_price(
        &self,
        product_id: &str,
        quantity: u32,
    ) -> Result<Money, PricingError> {
        if quantity > self.max_quantity {
            return Err(PricingError::QuantityExceeded(quantity));
        }

        let rule = self.rules.iter()
            .find(|r| r.product_id == product_id)
            .ok_or_else(|| PricingError::NoPriceRule(product_id.to_string()))?;

        Ok(Money {
            cents: rule.unit_price_cents * quantity as i64,
            currency: rule.currency,
        })
    }
}
// All 3 tests now pass — GREEN state.
```

```rust
// ── Step 4: REFACTOR — Add volume discount, tests still pass ──

#[test]
fn applies_volume_discount_above_threshold() {
    let pricer = StandardPricer::new(vec![
        PriceRule::new("BOLT-M8", 150, Currency::USD),
    ]).with_volume_discount(100, 0.10);  // 10% off above 100 units

    let total = pricer.calculate_price("BOLT-M8", 200).unwrap();
    // 200 × $1.50 = $300.00, minus 10% = $270.00
    assert_eq!(total.cents, 27_000);
}
// RED again — with_volume_discount doesn't exist.
// Implement it, go GREEN, then REFACTOR.
```

### Growing a Module Test-First

TDD builds modules incrementally. Each cycle adds one behavior. Here's how a `UserService` grows test-first using trait boundaries for testability:

```rust
// Cycle 1: "A user can be created"
// ─────────────────────────────────────────

// Define the repository trait (test seam)
#[async_trait::async_trait]
pub trait UserRepo: Send + Sync {
    async fn find_by_email(&self, email: &str) -> Result<Option<User>, RepoError>;
    async fn save(&self, user: &User) -> Result<(), RepoError>;
}

// RED: test that doesn't compile yet
#[tokio::test]
async fn creates_user_successfully() {
    let repo = MockUserRepo::new();  // Doesn't exist
    let svc = UserService::new(Box::new(repo));
    let user = svc.create("alice@example.com", "Alice").await.unwrap();
    assert_eq!(user.email, "alice@example.com");
}

// GREEN: implement MockUserRepo, UserService::new, UserService::create
// with minimum code to pass.
```

```rust
// Cycle 2: "Duplicate email is rejected"
// ─────────────────────────────────────────

#[tokio::test]
async fn rejects_duplicate_email() {
    let mut repo = MockUserRepo::new();
    repo.seed(User { email: "alice@example.com".into(), name: "Alice".into() });

    let svc = UserService::new(Box::new(repo));
    let result = svc.create("alice@example.com", "Bob").await;

    assert!(matches!(result, Err(ServiceError::EmailAlreadyExists(_))));
}

// GREEN: add the duplicate check in create().
// The first test still passes — no regressions.
```

```rust
// Cycle 3: "Notification is sent after creation"
// ─────────────────────────────────────────

// Add a second trait boundary for notifications
#[async_trait::async_trait]
pub trait Notifier: Send + Sync {
    async fn send(&self, to: &str, message: &str) -> Result<(), NotifyError>;
}

#[tokio::test]
async fn sends_welcome_notification() {
    let repo = MockUserRepo::new();
    let notifier = MockNotifier::new();  // New mock

    let svc = UserService::new(Box::new(repo), Box::new(notifier.clone()));
    svc.create("bob@example.com", "Bob").await.unwrap();

    assert_eq!(notifier.sent_count(), 1);
    assert!(notifier.last_message().contains("Welcome"));
}

// GREEN: update UserService to accept a Notifier, call it in create().
// Previous tests need updating to pass a MockNotifier — this is normal.
// The compiler tells you exactly which tests need the new parameter.
```

### Async TDD

Async code follows the same TDD cycle, with a few Rust-specific considerations:

```rust
// Testing async with controlled time
#[tokio::test]
async fn retries_on_transient_failure() {
    let mut repo = MockUserRepo::new();
    // First call fails, second succeeds
    repo.expect_save()
        .times(1)
        .returning(|_| Err(RepoError::ConnectionLost));
    repo.expect_save()
        .times(1)
        .returning(|u| Ok(()));

    let svc = UserService::new(Box::new(repo), Box::new(MockNotifier::new()));
    let result = svc.create_with_retry("test@example.com", "Test", 3).await;

    assert!(result.is_ok());
}

// Testing timeouts
#[tokio::test]
async fn times_out_slow_operations() {
    tokio::time::pause();  // Control time in tests

    let mut repo = MockUserRepo::new();
    repo.expect_find_by_email()
        .returning(|_| {
            Box::pin(async {
                tokio::time::sleep(Duration::from_secs(30)).await;
                Ok(None)
            })
        });

    let svc = UserService::new(Box::new(repo), Box::new(MockNotifier::new()));
    let result = tokio::time::timeout(
        Duration::from_secs(5),
        svc.create("slow@example.com", "Slow"),
    ).await;

    assert!(result.is_err());  // Timed out
}
```

### When TDD Fits (and When It Doesn't) in Rust

**TDD works best for:**

| Domain | Why TDD Helps |
|--------|---------------|
| **Business logic / domain services** | Tests document rules, catch regressions |
| **Data transformations** | Clear input → output, easy to specify |
| **API handlers** | Request → Response behavior is testable through traits |
| **Error handling paths** | Forces you to think about every failure mode upfront |
| **State machines** | Each transition is a test case |
| **Parsers / validators** | Well-defined accept/reject criteria |

**TDD is less valuable for:**

| Domain | Why | Better Approach |
|--------|-----|-----------------|
| **Type-level correctness** | The compiler already tests this | Rely on type system |
| **UI / rendering** | Output is visual, not easily asserted | Snapshot tests (insta) |
| **Exploratory / prototype code** | Requirements unclear | Write code first, add tests when design stabilizes |
| **FFI bindings** | Behavior defined by C library | Integration tests against the real library |
| **Performance optimization** | Correctness already tested | Benchmarks (criterion) |

### TDD Anti-Patterns in Rust

**Testing compiler-enforced properties:**
```rust
// BAD: the compiler already prevents this — the test adds nothing
#[test]
fn string_is_not_i32() {
    let x: String = "hello".into();
    // Can't even write assert_ne!(x, 42) — won't compile
}

// BAD: testing that a private field is private
// If it compiles, the visibility rules are enforced.
```

**Over-mocking — testing the mocks, not the logic:**
```rust
// BAD: test only verifies mock wiring, not real behavior
#[test]
fn calls_repo_save() {
    let mut repo = MockUserRepo::new();
    repo.expect_save().times(1).returning(|_| Ok(()));  // Mock setup
    let svc = UserService::new(Box::new(repo));
    svc.create("a@b.com", "A").unwrap();
    // Only assertion is that save() was called — doesn't test WHAT was saved
}

// GOOD: assert on the result and side effects, not method calls
#[test]
fn creates_user_with_correct_data() {
    let repo = InMemoryUserRepo::new();  // Real in-memory implementation
    let svc = UserService::new(Box::new(repo.clone()));
    let user = svc.create("a@b.com", "Alice").unwrap();
    assert_eq!(user.email, "a@b.com");
    assert_eq!(user.name, "Alice");
    assert!(repo.contains("a@b.com"));  // Verify state, not method calls
}
```

**Testing implementation details instead of behavior:**
```rust
// BAD: breaks when internal algorithm changes
#[test]
fn uses_hashmap_internally() {
    let cache = Cache::new();
    cache.insert("key", "value");
    // Asserts something about internal HashMap — fragile
}

// GOOD: test the observable behavior
#[test]
fn retrieves_previously_inserted_value() {
    let cache = Cache::new();
    cache.insert("key", "value");
    assert_eq!(cache.get("key"), Some("value"));
}
```

### TDD with Property-Based Testing

Combine TDD with `proptest` to discover edge cases you wouldn't write by hand:

```rust
use proptest::prelude::*;

// Start with a specific TDD test
#[test]
fn parses_valid_email() {
    assert!(Email::parse("user@example.com").is_ok());
}

#[test]
fn rejects_email_without_at() {
    assert!(Email::parse("userexample.com").is_err());
}

// Then generalize with property-based tests
proptest! {
    #[test]
    fn parsed_email_roundtrips(local in "[a-zA-Z0-9.]+", domain in "[a-zA-Z]+\\.[a-zA-Z]{2,}") {
        let input = format!("{local}@{domain}");
        let email = Email::parse(&input).unwrap();
        prop_assert_eq!(email.to_string(), input);
    }

    #[test]
    fn rejects_strings_without_at_sign(s in "[^@]+") {
        prop_assert!(Email::parse(&s).is_err());
    }

    #[test]
    fn quantity_never_produces_negative_total(
        price in 1i64..100_000,
        quantity in 1u32..10_000,
    ) {
        let total = price * quantity as i64;
        prop_assert!(total > 0);
    }
}
```

### TDD Checklist

| Step | Action | Rust-Specific Notes |
|------|--------|---------------------|
| **Design** | Define types, traits, error enums | Traits are test seams — design for injectability |
| **Red** | Write a test that fails | In Rust, "fails" often means "doesn't compile" — that counts |
| **Green** | Minimum code to pass | Don't add fields, methods, or match arms you don't need yet |
| **Refactor** | Improve while green | Leverage `cargo clippy` suggestions during this phase |
| **Error paths** | Test every `Err` variant | Use `assert!(matches!(...))` for specific variants |
| **Edge cases** | Add `proptest` after hand-written tests | Catches unicode, empty, overflow, boundary issues |
| **Integration** | Write integration tests in `tests/` | Test the public API as users see it |

## Test Organization

### Unit Tests (Inline)

```rust
// src/calculator.rs
pub fn divide(a: f64, b: f64) -> Result<f64, &'static str> {
    if b == 0.0 { return Err("division by zero"); }
    Ok(a / b)
}

#[cfg(test)]  // Only compiled during `cargo test`
mod tests {
    use super::*;  // Import parent module

    #[test]
    fn divides_correctly() {
        assert_eq!(divide(10.0, 2.0).unwrap(), 5.0);
    }

    #[test]
    fn division_by_zero_returns_error() {
        assert!(matches!(divide(1.0, 0.0), Err("division by zero")));
    }

    #[test]
    #[should_panic(expected = "must be positive")]
    fn panics_on_negative() {
        sqrt(-1.0);
    }

    #[test]
    fn returns_specific_variant() {
        let result = parse("");
        assert!(matches!(result, Err(ParseError::Empty)));
    }

    #[test]
    #[ignore]  // Skip unless `cargo test -- --ignored`
    fn slow_integration_test() {
        // expensive operation
    }
}
```

### Integration Tests (tests/ directory)

```rust
// tests/integration.rs — each file in tests/ is a separate crate
use my_crate::{Config, Server};

#[test]
fn server_starts_with_valid_config() {
    let config = Config::new("localhost", 8080);
    let server = Server::new(config);
    assert!(server.is_ready());
}
```

### Test Helpers Module

```rust
// tests/common/mod.rs — shared test helpers
pub fn test_config() -> Config {
    Config { host: "localhost".into(), port: 0, db_url: test_db_url() }
}

pub fn test_db_url() -> String {
    std::env::var("TEST_DATABASE_URL")
        .unwrap_or_else(|_| "postgres://test:test@localhost:5433/test_db".into())
}

// tests/api_test.rs
mod common;

#[test]
fn test_with_shared_helpers() {
    let config = common::test_config();
    // ...
}
```

### Doc Tests

```rust
/// Adds two numbers together.
///
/// ```
/// assert_eq!(my_crate::add(2, 3), 5);
/// ```
///
/// Negative numbers work too:
/// ```
/// assert_eq!(my_crate::add(-1, 1), 0);
/// ```
pub fn add(a: i32, b: i32) -> i32 { a + b }
```

### File Structure

```
src/
├── lib.rs
├── domain/
│   ├── mod.rs
│   ├── user.rs
│   └── user_repository.rs
├── application/
│   ├── mod.rs
│   └── user_service.rs
└── infrastructure/
    ├── mod.rs
    └── postgres_user_repository.rs

tests/
├── common/
│   ├── mod.rs
│   └── mocks.rs          # Shared mock implementations
├── unit/
│   └── user_service_test.rs
└── integration/
    └── user_repository_test.rs
```

### Test Module Pattern

```rust
// In src/application/user_service.rs
pub struct UserService { /* ... */ }

impl UserService {
    pub fn create_user(&self, /* ... */) -> Result<User, ServiceError> {
        // Implementation
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    // Test helpers specific to this module
    fn create_test_user(username: &str) -> User {
        User {
            id: 0,
            username: username.to_string(),
            email: format!("{}@test.com", username),
        }
    }

    mod create_user {
        use super::*;

        #[test]
        fn succeeds_with_valid_input() {
            // ...
        }

        #[test]
        fn fails_when_username_exists() {
            // ...
        }

        #[test]
        fn sends_notification_on_success() {
            // ...
        }
    }

    mod get_user {
        use super::*;

        #[test]
        fn returns_user_when_found() {
            // ...
        }

        #[test]
        fn returns_none_when_not_found() {
            // ...
        }
    }
}
```

### Test Naming Conventions

```rust
#[test]
fn test_create_user_with_valid_data_returns_user() { }

#[test]
fn test_create_user_with_duplicate_username_returns_error() { }

#[test]
fn test_create_user_sends_welcome_notification() { }

// Or using nested modules:
mod create_user {
    #[test]
    fn succeeds_with_valid_data() { }

    #[test]
    fn fails_with_duplicate_username() { }

    #[test]
    fn sends_welcome_notification() { }
}
```

## Assert Patterns

```rust
// Basic assertions
assert!(condition);
assert_eq!(left, right);      // Shows both values on failure
assert_ne!(left, right);

// With messages
assert!(x > 0, "expected positive, got {x}");
assert_eq!(result, expected, "failed for input: {input:?}");

// Pattern matching
assert!(matches!(result, Ok(42)));
assert!(matches!(value, Some(x) if x > 10));
assert!(matches!(err, Err(Error::NotFound { .. })));

// Float comparison
assert!((result - expected).abs() < f64::EPSILON);

// Collection assertions
assert!(vec.contains(&42));
assert!(vec.is_empty());
assert_eq!(vec.len(), 3);

// Panic assertion with expected message
#[test]
#[should_panic(expected = "index out of bounds")]
fn panics_on_invalid_index() {
    let v: Vec<i32> = vec![];
    let _ = v[0];
}
```

### Compile-Time Trait Assertions

Verify types satisfy trait bounds at compile time — no runtime cost. Used extensively by axum to ensure handlers are `Send + Sync`:

```rust
// Define assertion helpers (zero-cost — optimized away)
fn assert_send<T: Send>() {}
fn assert_sync<T: Sync>() {}
fn assert_unpin<T: Unpin>() {}

#[test]
fn service_types_are_thread_safe() {
    // These fail at compile time if the bounds aren't met
    assert_send::<MyService>();
    assert_sync::<MyService>();
    assert_send::<AppState>();
    assert_sync::<AppState>();
}

#[test]
fn futures_are_send() {
    // Verify async functions produce Send futures (required for tokio::spawn)
    fn check_send<F: std::future::Future + Send>(_f: F) {}
    check_send(my_async_handler());
}
```

### Data-Driven (Table-Driven) Tests

Test many cases with a single function — scales better than one `#[test]` per case:

```rust
#[test]
fn parses_valid_inputs() {
    let cases = [
        ("42", 42),
        ("0", 0),
        ("-1", -1),
        ("2147483647", i32::MAX),
    ];
    for (input, expected) in cases {
        let result: i32 = input.parse().unwrap();
        assert_eq!(result, expected, "failed for input: {input:?}");
    }
}

// For error cases, test both the variant and context
fn test_parse_err<T: std::str::FromStr>(cases: &[(&str, &str)])
where
    T::Err: std::fmt::Display,
{
    for (input, expected_msg) in cases {
        let err = input.parse::<T>().unwrap_err();
        assert!(
            err.to_string().contains(expected_msg),
            "input {input:?}: expected error containing {expected_msg:?}, got: {err}",
        );
    }
}
```

### Serde Round-Trip Testing

For any type that implements `Serialize + Deserialize`:

```rust
fn assert_round_trip<T>(value: &T)
where
    T: serde::Serialize + for<'de> serde::Deserialize<'de> + PartialEq + std::fmt::Debug,
{
    let json = serde_json::to_string(value).expect("serialize");
    let back: T = serde_json::from_str(&json).expect("deserialize");
    assert_eq!(value, &back, "round-trip failed for: {json}");
}

// Paired encode/decode helpers (from serde_json's own test suite)
fn test_encode_ok<T: serde::Serialize + std::fmt::Debug>(cases: &[(T, &str)]) {
    for (value, expected) in cases {
        assert_eq!(
            serde_json::to_string(value).unwrap(),
            *expected,
            "encoding {:?}",
            value,
        );
    }
}
```

### Edge Case Testing Checklist

Systematically test boundary conditions (pattern from serde_json's test suite):

| Category | Cases to Test |
|----------|---------------|
| **Empty values** | `""`, `vec![]`, `HashMap::new()`, `None`, `0`, `false` |
| **Unicode** | Multi-byte chars (`"日本語"`), emoji (`"🦀"`), surrogate pairs, control chars |
| **Number extremes** | `i64::MIN`, `i64::MAX`, `u64::MAX`, `f64::INFINITY`, `f64::NAN`, `f64::MIN_POSITIVE` |
| **Size limits** | Empty string, 1-char string, very long string, empty collection, single-element, large collection |
| **Nesting** | Deeply nested structures (test recursion limits) |
| **Whitespace** | Leading/trailing spaces, tabs, newlines, mixed whitespace |
| **Special chars** | Null bytes, backslashes, quotes, path separators |

```rust
#[test]
fn handles_edge_cases() {
    let cases = [
        ("", MyType::Empty),
        ("  \t\n  ", MyType::Empty),  // whitespace-only
        ("日本語", MyType::Text("日本語".into())),
        ("\0", MyType::Text("\0".into())),  // null byte
    ];
    for (input, expected) in &cases {
        assert_eq!(MyType::parse(input), *expected, "input: {input:?}");
    }
}
```

### Error Message Assertions

Test that error messages include useful context (line, column, field name):

```rust
#[test]
fn error_includes_location() {
    let bad_json = r#"{"name": "alice", "age": "not_a_number"}"#;
    let err = serde_json::from_str::<User>(bad_json).unwrap_err();

    // Verify error message contains useful context
    let msg = err.to_string();
    assert!(msg.contains("line 1"), "missing line info: {msg}");
    assert!(msg.contains("column"), "missing column info: {msg}");
}

#[test]
fn custom_error_includes_field() {
    let result = validate_config(input);
    let err = result.unwrap_err();
    assert!(
        err.to_string().contains("port"),
        "error should mention the failing field: {err}",
    );
}
```

## Creating Mock Implementations

### Basic Mock Structure

Mocks implement traits to substitute real dependencies during testing:

```rust
use std::cell::RefCell;

// The trait to mock
pub trait Notifier {
    fn send(&self, message: &str) -> Result<(), NotificationError>;
}

#[derive(Debug, PartialEq)]
pub enum NotificationError {
    ConnectionFailed,
    MessageTooLong,
    Other(String),
}

// Mock implementation
#[derive(Debug, Default)]
pub struct MockNotifier {
    // Track what was sent (RefCell for interior mutability with &self)
    sent_messages: RefCell<Vec<String>>,
    // Configure return values
    results: RefCell<Vec<Result<(), NotificationError>>>,
}

impl MockNotifier {
    pub fn new() -> Self {
        Self::default()
    }

    // Configure expected results
    pub fn expect_result(&self, result: Result<(), NotificationError>) {
        self.results.borrow_mut().push(result);
    }

    // Configure multiple results (returned in LIFO order)
    pub fn expect_results(&self, results: Vec<Result<(), NotificationError>>) {
        *self.results.borrow_mut() = results;
    }

    // Assertion helpers
    pub fn assert_sent(&self, expected: &str) {
        assert!(
            self.sent_messages.borrow().contains(&expected.to_string()),
            "Expected message '{}' not found in {:?}",
            expected,
            self.sent_messages.borrow()
        );
    }

    pub fn assert_sent_count(&self, count: usize) {
        assert_eq!(
            self.sent_messages.borrow().len(),
            count,
            "Expected {} messages, got {}",
            count,
            self.sent_messages.borrow().len()
        );
    }

    pub fn assert_sent_in_order(&self, expected: &[&str]) {
        let actual: Vec<String> = self.sent_messages.borrow().clone();
        let expected: Vec<String> = expected.iter().map(|s| s.to_string()).collect();
        assert_eq!(actual, expected);
    }

    pub fn get_sent_messages(&self) -> Vec<String> {
        self.sent_messages.borrow().clone()
    }
}

impl Notifier for MockNotifier {
    fn send(&self, message: &str) -> Result<(), NotificationError> {
        // Always record the call
        self.sent_messages.borrow_mut().push(message.to_string());

        // Return configured result or default error
        self.results
            .borrow_mut()
            .pop()
            .unwrap_or(Err(NotificationError::Other("No result configured".into())))
    }
}
```

### Mock Repository Pattern

```rust
use std::cell::RefCell;
use std::collections::HashMap;

pub trait UserRepository {
    fn find_by_id(&self, id: u64) -> Option<User>;
    fn find_by_username(&self, username: &str) -> Option<User>;
    fn save(&self, user: User) -> Result<User, RepositoryError>;
    fn delete(&self, id: u64) -> Result<(), RepositoryError>;
}

#[derive(Debug, Clone, PartialEq)]
pub struct User {
    pub id: u64,
    pub username: String,
    pub email: String,
}

#[derive(Debug)]
pub enum RepositoryError {
    NotFound,
    SaveError(String),
}

#[derive(Debug, Default)]
pub struct MockUserRepository {
    // In-memory storage
    users_by_id: RefCell<HashMap<u64, User>>,
    users_by_username: RefCell<HashMap<String, u64>>,

    // Track operations for assertions
    saved_users: RefCell<Vec<User>>,
    deleted_ids: RefCell<Vec<u64>>,
    looked_up_ids: RefCell<Vec<u64>>,
    looked_up_usernames: RefCell<Vec<String>>,

    // Configure behavior
    save_results: RefCell<Vec<Result<User, RepositoryError>>>,
    next_id: RefCell<u64>,
}

impl MockUserRepository {
    pub fn new() -> Self {
        Self {
            next_id: RefCell::new(1),
            ..Default::default()
        }
    }

    // Pre-populate test data
    pub fn with_user(&self, user: User) {
        let id = user.id;
        let username = user.username.clone();
        self.users_by_id.borrow_mut().insert(id, user);
        self.users_by_username.borrow_mut().insert(username, id);
    }

    // Configure save behavior
    pub fn expect_save_result(&self, result: Result<User, RepositoryError>) {
        self.save_results.borrow_mut().push(result);
    }

    // Assertions
    pub fn assert_user_saved(&self, username: &str) {
        assert!(
            self.saved_users.borrow().iter().any(|u| u.username == username),
            "User '{}' was not saved",
            username
        );
    }

    pub fn assert_save_count(&self, count: usize) {
        assert_eq!(self.saved_users.borrow().len(), count);
    }

    pub fn assert_deleted(&self, id: u64) {
        assert!(self.deleted_ids.borrow().contains(&id));
    }

    pub fn assert_looked_up_username(&self, username: &str) {
        assert!(self.looked_up_usernames.borrow().contains(&username.to_string()));
    }

    pub fn assert_no_saves(&self) {
        assert!(self.saved_users.borrow().is_empty(), "Expected no saves");
    }
}

impl UserRepository for MockUserRepository {
    fn find_by_id(&self, id: u64) -> Option<User> {
        self.looked_up_ids.borrow_mut().push(id);
        self.users_by_id.borrow().get(&id).cloned()
    }

    fn find_by_username(&self, username: &str) -> Option<User> {
        self.looked_up_usernames.borrow_mut().push(username.to_string());
        self.users_by_username
            .borrow()
            .get(username)
            .and_then(|id| self.users_by_id.borrow().get(id).cloned())
    }

    fn save(&self, mut user: User) -> Result<User, RepositoryError> {
        // Assign ID if new
        if user.id == 0 {
            user.id = *self.next_id.borrow();
            *self.next_id.borrow_mut() += 1;
        }

        // Record the save
        self.saved_users.borrow_mut().push(user.clone());

        // Store in mock database
        self.users_by_id.borrow_mut().insert(user.id, user.clone());
        self.users_by_username
            .borrow_mut()
            .insert(user.username.clone(), user.id);

        // Return configured result or success
        self.save_results
            .borrow_mut()
            .pop()
            .unwrap_or(Ok(user))
    }

    fn delete(&self, id: u64) -> Result<(), RepositoryError> {
        self.deleted_ids.borrow_mut().push(id);

        if let Some(user) = self.users_by_id.borrow_mut().remove(&id) {
            self.users_by_username.borrow_mut().remove(&user.username);
            Ok(())
        } else {
            Err(RepositoryError::NotFound)
        }
    }
}
```

### Verification Hook Pattern

For precise control over mock behavior and assertions:

```rust
pub struct HookBasedMock {
    hook: Box<dyn Fn(&str, f64) -> Result<(), String>>,
}

impl HookBasedMock {
    pub fn new<F>(hook: F) -> Self
    where
        F: Fn(&str, f64) -> Result<(), String> + 'static,
    {
        Self { hook: Box::new(hook) }
    }

    pub fn with_success() -> Self {
        Self::new(|_, _| Ok(()))
    }

    pub fn with_error(error: &'static str) -> Self {
        Self::new(move |_, _| Err(error.to_string()))
    }

    pub fn with_assertions(expected_recipient: &'static str, expected_amount: f64) -> Self {
        Self::new(move |recipient, amount| {
            assert_eq!(recipient, expected_recipient);
            assert!((amount - expected_amount).abs() < 0.01);
            Ok(())
        })
    }
}

impl PaymentNotifier for HookBasedMock {
    fn notify(&self, recipient: &str, amount: f64) -> Result<(), String> {
        (self.hook)(recipient, amount)
    }
}

// Usage in tests
#[test]
fn test_payment_sends_correct_notification() {
    let mock = HookBasedMock::with_assertions("user@example.com", 99.99);
    let processor = PaymentProcessor::new(Box::new(mock));

    let result = processor.process("user@example.com", 99.99);
    assert!(result.is_ok());
}
```

### mockall Crate

```rust
use mockall::{automock, predicate::*};

#[automock]  // Generates MockUserRepository automatically
pub trait UserRepository: Send + Sync {
    fn find_by_id(&self, id: u64) -> Option<User>;
    fn save(&self, user: &User) -> Result<(), Error>;
}

#[test]
fn creates_and_saves_user() {
    let mut mock = MockUserRepository::new();

    // Set expectations
    mock.expect_find_by_id()
        .with(eq(42))
        .times(1)
        .returning(|_| Some(User { id: 42, name: "alice".into() }));

    mock.expect_save()
        .with(always())
        .times(1)
        .returning(|_| Ok(()));

    let service = UserService::new(Box::new(mock));
    let result = service.get_user(42);
    assert!(result.is_some());
}

// Sequence expectations
#[test]
fn calls_in_order() {
    let mut mock = MockUserRepository::new();
    let mut seq = mockall::Sequence::new();

    mock.expect_find_by_id()
        .times(1)
        .in_sequence(&mut seq)
        .returning(|_| None);

    mock.expect_save()
        .times(1)
        .in_sequence(&mut seq)
        .returning(|_| Ok(()));
}
```

## Unit Test Isolation

### Service Under Test

```rust
pub struct UserService {
    repository: Box<dyn UserRepository>,
    notifier: Box<dyn Notifier>,
}

impl UserService {
    pub fn new(
        repository: Box<dyn UserRepository>,
        notifier: Box<dyn Notifier>,
    ) -> Self {
        Self { repository, notifier }
    }

    pub fn create_user(&self, username: &str, email: &str) -> Result<User, ServiceError> {
        // Check if username exists
        if self.repository.find_by_username(username).is_some() {
            return Err(ServiceError::UserAlreadyExists(username.to_string()));
        }

        // Create and save user
        let user = User {
            id: 0,
            username: username.to_string(),
            email: email.to_string(),
        };
        let saved_user = self.repository.save(user)?;

        // Send notification
        let message = format!("Welcome, {}! Your account has been created.", username);
        self.notifier.send(&message)?;

        Ok(saved_user)
    }
}

#[derive(Debug)]
pub enum ServiceError {
    UserAlreadyExists(String),
    RepositoryError(RepositoryError),
    NotificationError(NotificationError),
}

impl From<RepositoryError> for ServiceError {
    fn from(err: RepositoryError) -> Self {
        ServiceError::RepositoryError(err)
    }
}

impl From<NotificationError> for ServiceError {
    fn from(err: NotificationError) -> Self {
        ServiceError::NotificationError(err)
    }
}
```

### Complete Test Suite

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn setup_mocks() -> (MockUserRepository, MockNotifier) {
        (MockUserRepository::new(), MockNotifier::new())
    }

    fn create_service(
        repo: MockUserRepository,
        notifier: MockNotifier,
    ) -> UserService {
        UserService::new(Box::new(repo), Box::new(notifier))
    }

    #[test]
    fn test_create_user_success() {
        // Arrange
        let (mock_repo, mock_notifier) = setup_mocks();
        mock_notifier.expect_result(Ok(()));

        let service = create_service(mock_repo, mock_notifier);

        // Act
        let result = service.create_user("alice", "alice@example.com");

        // Assert
        assert!(result.is_ok());
        let user = result.unwrap();
        assert_eq!(user.username, "alice");
        assert_eq!(user.email, "alice@example.com");
        assert!(user.id > 0);
    }

    #[test]
    fn test_create_user_username_already_exists() {
        // Arrange
        let (mock_repo, mock_notifier) = setup_mocks();

        // Pre-populate with existing user
        mock_repo.with_user(User {
            id: 1,
            username: "existing".to_string(),
            email: "existing@example.com".to_string(),
        });

        let service = create_service(mock_repo, mock_notifier);

        // Act
        let result = service.create_user("existing", "new@example.com");

        // Assert
        assert!(matches!(
            result,
            Err(ServiceError::UserAlreadyExists(name)) if name == "existing"
        ));
    }

    #[test]
    fn test_create_user_repository_failure() {
        // Arrange
        let (mock_repo, mock_notifier) = setup_mocks();
        mock_repo.expect_save_result(Err(RepositoryError::SaveError("DB error".into())));

        let service = create_service(mock_repo, mock_notifier);

        // Act
        let result = service.create_user("alice", "alice@example.com");

        // Assert
        assert!(matches!(result, Err(ServiceError::RepositoryError(_))));
    }

    #[test]
    fn test_create_user_notification_failure_after_save() {
        // Arrange
        let (mock_repo, mock_notifier) = setup_mocks();
        mock_notifier.expect_result(Err(NotificationError::ConnectionFailed));

        let service = create_service(mock_repo, mock_notifier);

        // Act
        let result = service.create_user("alice", "alice@example.com");

        // Assert
        assert!(matches!(result, Err(ServiceError::NotificationError(_))));
    }

    #[test]
    fn test_create_user_does_not_notify_on_duplicate() {
        // Arrange
        let mock_repo = MockUserRepository::new();
        let mock_notifier = MockNotifier::new();

        mock_repo.with_user(User {
            id: 1,
            username: "existing".to_string(),
            email: "existing@example.com".to_string(),
        });

        // Wrap in Rc to access after service use
        use std::rc::Rc;
        let notifier_rc = Rc::new(mock_notifier);

        let service = UserService::new(
            Box::new(mock_repo),
            Box::new(NotifierWrapper(Rc::clone(&notifier_rc))),
        );

        // Act
        let _ = service.create_user("existing", "new@example.com");

        // Assert - notification should NOT have been called
        notifier_rc.assert_sent_count(0);
    }
}

// Wrapper to use Rc<MockNotifier> as Box<dyn Notifier>
struct NotifierWrapper(Rc<MockNotifier>);

impl Notifier for NotifierWrapper {
    fn send(&self, message: &str) -> Result<(), NotificationError> {
        self.0.send(message)
    }
}
```

## Testing Async Code

### Async Mock Repository

```rust
use async_trait::async_trait;
use tokio::sync::Mutex;
use std::collections::HashMap;

#[async_trait]
pub trait AsyncUserRepository: Send + Sync {
    async fn find_by_id(&self, id: u64) -> Option<User>;
    async fn save(&self, user: User) -> Result<User, RepositoryError>;
}

pub struct AsyncMockUserRepository {
    users: Mutex<HashMap<u64, User>>,
    save_results: Mutex<Vec<Result<User, RepositoryError>>>,
}

impl AsyncMockUserRepository {
    pub fn new() -> Self {
        Self {
            users: Mutex::new(HashMap::new()),
            save_results: Mutex::new(Vec::new()),
        }
    }

    pub async fn with_user(&self, user: User) {
        self.users.lock().await.insert(user.id, user);
    }

    pub async fn expect_save_result(&self, result: Result<User, RepositoryError>) {
        self.save_results.lock().await.push(result);
    }
}

#[async_trait]
impl AsyncUserRepository for AsyncMockUserRepository {
    async fn find_by_id(&self, id: u64) -> Option<User> {
        self.users.lock().await.get(&id).cloned()
    }

    async fn save(&self, user: User) -> Result<User, RepositoryError> {
        let mut results = self.save_results.lock().await;
        if let Some(result) = results.pop() {
            return result;
        }

        let mut users = self.users.lock().await;
        users.insert(user.id, user.clone());
        Ok(user)
    }
}

#[cfg(test)]
mod async_tests {
    use super::*;

    #[tokio::test]
    async fn test_async_find_user() {
        let mock = AsyncMockUserRepository::new();
        mock.with_user(User {
            id: 1,
            username: "alice".to_string(),
            email: "alice@example.com".to_string(),
        }).await;

        let result = mock.find_by_id(1).await;
        assert!(result.is_some());
        assert_eq!(result.unwrap().username, "alice");
    }
}
```

### tokio::test

```rust
#[tokio::test]
async fn fetches_user() {
    let pool = setup_test_db().await;
    let repo = PostgresUserRepo::new(pool);

    let user = repo.find_by_id(1).await;
    assert!(user.is_some());
}

// Multi-threaded runtime (default is current_thread)
#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn concurrent_test() {
    // ...
}
```

### Time Control

```rust
use tokio::time::{self, Duration};

#[tokio::test]
async fn timeout_triggers() {
    time::pause();  // Freeze time — advances only when awaiting

    let start = time::Instant::now();
    time::advance(Duration::from_secs(60)).await;

    assert!(start.elapsed() >= Duration::from_secs(60));
    // No actual 60 seconds waited!
}

#[tokio::test]
async fn retry_with_backoff() {
    time::pause();

    let result = tokio::time::timeout(
        Duration::from_secs(5),
        some_async_operation(),
    ).await;

    assert!(result.is_err()); // Timed out
}
```

## Integration Testing with Real Dependencies

### Database Test Setup

```rust
use sqlx::{PgPool, postgres::PgPoolOptions};

async fn get_test_pool() -> PgPool {
    let database_url = std::env::var("TEST_DATABASE_URL")
        .expect("TEST_DATABASE_URL must be set");

    PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await
        .expect("Failed to connect to test database")
}

async fn setup_test_schema(pool: &PgPool) {
    sqlx::query(r#"
        DROP TABLE IF EXISTS users CASCADE;
        CREATE TABLE users (
            id SERIAL PRIMARY KEY,
            username VARCHAR(255) UNIQUE NOT NULL,
            email VARCHAR(255) NOT NULL,
            created_at TIMESTAMP DEFAULT NOW()
        );
    "#)
    .execute(pool)
    .await
    .expect("Failed to create test schema");
}

async fn cleanup_test_data(pool: &PgPool) {
    sqlx::query("DELETE FROM users")
        .execute(pool)
        .await
        .expect("Failed to cleanup test data");
}
```

### Transaction-Based Test Isolation

The gold standard for database test isolation — each test runs in a transaction that's rolled back:

```rust
#[cfg(test)]
mod integration_tests {
    use super::*;
    use sqlx::Acquire;

    #[tokio::test]
    async fn test_save_and_find_user_in_transaction() {
        let pool = get_test_pool().await;
        setup_test_schema(&pool).await;

        // Start transaction
        let mut tx = pool.begin().await.unwrap();

        // Create repository that uses the transaction
        let repo = PostgresUserRepository::new_with_executor(&mut *tx);

        // Test operations
        let user = User {
            id: 0,
            username: "test_user".to_string(),
            email: "test@example.com".to_string(),
        };

        let saved = repo.save(user).await.unwrap();
        assert!(saved.id > 0);

        let found = repo.find_by_id(saved.id).await;
        assert!(found.is_some());
        assert_eq!(found.unwrap().username, "test_user");

        // Rollback - data is NOT persisted
        tx.rollback().await.unwrap();

        // Verify cleanup worked
        let check_repo = PostgresUserRepository::new(&pool);
        let should_be_none = check_repo.find_by_username("test_user").await;
        assert!(should_be_none.is_none());
    }

    #[tokio::test]
    async fn test_duplicate_username_fails() {
        let pool = get_test_pool().await;
        setup_test_schema(&pool).await;

        let mut tx = pool.begin().await.unwrap();
        let repo = PostgresUserRepository::new_with_executor(&mut *tx);

        // Save first user
        let user1 = User {
            id: 0,
            username: "duplicate".to_string(),
            email: "first@example.com".to_string(),
        };
        repo.save(user1).await.unwrap();

        // Try to save user with same username
        let user2 = User {
            id: 0,
            username: "duplicate".to_string(),
            email: "second@example.com".to_string(),
        };
        let result = repo.save(user2).await;

        assert!(result.is_err());

        tx.rollback().await.unwrap();
    }
}
```

### Repository Implementation Supporting Transactions

```rust
use sqlx::{PgPool, PgExecutor, FromRow};

#[derive(FromRow)]
struct UserRow {
    id: i32,
    username: String,
    email: String,
}

impl From<UserRow> for User {
    fn from(row: UserRow) -> Self {
        User {
            id: row.id as u64,
            username: row.username,
            email: row.email,
        }
    }
}

pub struct PostgresUserRepository<'e, E: PgExecutor<'e>> {
    executor: E,
    _phantom: std::marker::PhantomData<&'e ()>,
}

impl<'e, E: PgExecutor<'e>> PostgresUserRepository<'e, E> {
    pub fn new_with_executor(executor: E) -> Self {
        Self {
            executor,
            _phantom: std::marker::PhantomData,
        }
    }
}

// Simpler version using just the pool
pub struct SimplePostgresUserRepository {
    pool: PgPool,
}

impl SimplePostgresUserRepository {
    pub fn new(pool: &PgPool) -> Self {
        Self { pool: pool.clone() }
    }

    pub async fn find_by_id(&self, id: u64) -> Option<User> {
        sqlx::query_as::<_, UserRow>(
            "SELECT id, username, email FROM users WHERE id = $1"
        )
        .bind(id as i32)
        .fetch_optional(&self.pool)
        .await
        .ok()
        .flatten()
        .map(User::from)
    }

    pub async fn save(&self, user: User) -> Result<User, RepositoryError> {
        let row = sqlx::query_as::<_, UserRow>(
            r#"
            INSERT INTO users (username, email)
            VALUES ($1, $2)
            RETURNING id, username, email
            "#
        )
        .bind(&user.username)
        .bind(&user.email)
        .fetch_one(&self.pool)
        .await
        .map_err(|e| RepositoryError::SaveError(e.to_string()))?;

        Ok(User::from(row))
    }

    pub async fn find_by_username(&self, username: &str) -> Option<User> {
        sqlx::query_as::<_, UserRow>(
            "SELECT id, username, email FROM users WHERE username = $1"
        )
        .bind(username)
        .fetch_optional(&self.pool)
        .await
        .ok()
        .flatten()
        .map(User::from)
    }
}
```

### Test Pool Setup with LazyLock

```rust
use sqlx::PgPool;
use std::sync::LazyLock;

static TEST_POOL: LazyLock<PgPool> = LazyLock::new(|| {
    tokio::runtime::Runtime::new().unwrap().block_on(async {
        let url = std::env::var("TEST_DATABASE_URL")
            .unwrap_or_else(|_| "postgres://test:test@localhost:5433/test_db".into());
        PgPool::connect(&url).await.expect("Failed to connect to test DB")
    })
});

async fn get_test_pool() -> &'static PgPool {
    &TEST_POOL
}
```

## End-to-End (E2E) Testing

### CLI Testing with assert_cmd

Test command-line applications by executing the binary and asserting on output:

```rust
// tests/cli_e2e.rs
use assert_cmd::Command;
use predicates::prelude::*;

#[test]
fn test_add_and_list_items() -> Result<(), Box<dyn std::error::Error>> {
    // Add an item
    let mut cmd = Command::cargo_bin("my_app")?;
    cmd.args(["add", "Buy groceries"])
        .assert()
        .success()
        .stdout(predicate::str::contains("Added: Buy groceries"));

    // List items
    let mut cmd = Command::cargo_bin("my_app")?;
    cmd.args(["list"])
        .assert()
        .success()
        .stdout(predicate::str::contains("1. Buy groceries"));

    Ok(())
}

#[test]
fn test_mark_item_as_done() -> Result<(), Box<dyn std::error::Error>> {
    // Add item
    Command::cargo_bin("my_app")?
        .args(["add", "Call mom"])
        .assert()
        .success();

    // Mark as done
    Command::cargo_bin("my_app")?
        .args(["done", "1"])
        .assert()
        .success()
        .stdout(predicate::str::contains("Marked as done"));

    // Verify in list
    Command::cargo_bin("my_app")?
        .args(["list"])
        .assert()
        .success()
        .stdout(predicate::str::contains("[x] Call mom"));

    Ok(())
}

#[test]
fn test_invalid_command_shows_help() -> Result<(), Box<dyn std::error::Error>> {
    Command::cargo_bin("my_app")?
        .args(["invalid"])
        .assert()
        .failure()
        .stderr(predicate::str::contains("Usage:"));

    Ok(())
}
```

### Web API E2E Testing with reqwest

Test HTTP APIs by making real requests to a running server:

```rust
// tests/api_e2e.rs
use reqwest::Client;
use serde::{Deserialize, Serialize};
use std::time::Duration;
use tokio::time::sleep;

#[derive(Serialize)]
struct CreatePostRequest {
    title: String,
    content: String,
}

#[derive(Deserialize, Debug)]
struct Post {
    id: String,
    title: String,
    content: String,
}

async fn wait_for_server(base_url: &str) {
    let client = Client::new();
    for _ in 0..30 {
        if client.get(&format!("{}/health", base_url)).send().await.is_ok() {
            return;
        }
        sleep(Duration::from_millis(100)).await;
    }
    panic!("Server did not start in time");
}

#[tokio::test]
async fn test_create_and_retrieve_post() {
    let base_url = "http://localhost:8080";
    wait_for_server(base_url).await;

    let client = Client::new();

    // Create a post
    let create_req = CreatePostRequest {
        title: "E2E Test Post".to_string(),
        content: "This is test content".to_string(),
    };

    let create_response = client
        .post(&format!("{}/posts", base_url))
        .json(&create_req)
        .send()
        .await
        .expect("Failed to create post");

    assert_eq!(create_response.status(), 201);
    let created_post: Post = create_response.json().await.unwrap();
    assert_eq!(created_post.title, "E2E Test Post");

    // Retrieve the post
    let get_response = client
        .get(&format!("{}/posts/{}", base_url, created_post.id))
        .send()
        .await
        .expect("Failed to get post");

    assert_eq!(get_response.status(), 200);
    let retrieved_post: Post = get_response.json().await.unwrap();
    assert_eq!(retrieved_post.id, created_post.id);
    assert_eq!(retrieved_post.title, "E2E Test Post");

    // Test not found
    let not_found = client
        .get(&format!("{}/posts/nonexistent-id", base_url))
        .send()
        .await
        .expect("Failed to send request");

    assert_eq!(not_found.status(), 404);
}
```

### Axum TestServer Pattern

```rust
use axum::{body::Body, http::{Request, StatusCode}};
use tower::ServiceExt;  // for .oneshot()

#[tokio::test]
async fn test_create_user_endpoint() {
    let app = create_app(test_state()).await;

    let response = app
        .oneshot(
            Request::builder()
                .method("POST")
                .uri("/api/users")
                .header("content-type", "application/json")
                .body(Body::from(r#"{"username":"alice","email":"a@b.com"}"#))
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::CREATED);

    let body = axum::body::to_bytes(response.into_body(), usize::MAX).await.unwrap();
    let user: serde_json::Value = serde_json::from_slice(&body).unwrap();
    assert_eq!(user["username"], "alice");
}
```

### Actix-web TestServer

For tighter integration without spawning a separate process:

```rust
use actix_web::{test, web, App, HttpResponse};
use serde_json::json;

async fn create_post(body: web::Json<CreatePostRequest>) -> HttpResponse {
    // Handler implementation
    HttpResponse::Created().json(json!({
        "id": "generated-id",
        "title": body.title,
        "content": body.content
    }))
}

#[actix_web::test]
async fn test_create_post_e2e() {
    let app = test::init_service(
        App::new()
            .route("/posts", web::post().to(create_post))
    ).await;

    let req = test::TestRequest::post()
        .uri("/posts")
        .set_json(json!({
            "title": "Test Title",
            "content": "Test Content"
        }))
        .to_request();

    let resp = test::call_service(&app, req).await;
    assert!(resp.status().is_success());

    let body: serde_json::Value = test::read_body_json(resp).await;
    assert_eq!(body["title"], "Test Title");
}
```

### Docker Compose Test Environment

```yaml
# docker-compose.test.yml
version: '3.8'

services:
  db:
    image: postgres:16-alpine
    ports:
      - "5433:5432"  # Different port to avoid conflicts
    environment:
      POSTGRES_DB: test_db
      POSTGRES_USER: test_user
      POSTGRES_PASSWORD: test_password
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test_user -d test_db"]
      interval: 5s
      timeout: 5s
      retries: 5

  mock_api:
    image: mockserver/mockserver:latest
    ports:
      - "1080:1080"
    environment:
      MOCKSERVER_INITIALIZATION_JSON_PATH: /config/expectations.json
    volumes:
      - ./mock_expectations.json:/config/expectations.json
```

## Microservice End-to-End Testing

### Atomic vs Workflow E2E Testing

End-to-end tests come in two distinct forms with different purposes:

**Atomic E2E Testing**:
- Wipes database state between each HTTP request
- Provides quick feedback loop during development
- Tests individual endpoints in realistic conditions
- Detects: circular dependencies between services, integration errors
- Cannot detect: broken workflows, missing endpoints in user journeys

**Workflow E2E Testing**:
- Maintains state across multiple requests
- Tests complete user journeys from start to finish
- Slower but more realistic simulation of production
- Detects: workflow breaks, missing steps, state management issues

```
                    UNIT TESTS                    ATOMIC E2E                  WORKFLOW E2E
                    ──────────                    ──────────                  ────────────
Speed:              Very Fast                     Fast                        Slower
Isolation:          Complete (mocks)              Per-request                 None
Detects:            Logic errors                  Integration errors          Workflow breaks
                    Edge cases                    Circular dependencies       Missing endpoints
Cannot Detect:      Integration issues            Workflow breaks             N/A
```

### Why E2E Tests Catch What Unit Tests Miss

Unit tests mock external dependencies, which means certain bugs slip through:

```
Service A ──request──> Service B ──request──> Service A (circular dependency!)

Unit Test Result: ✓ All pass (mocks return expected values)
E2E Test Result:  ✗ Timeout (infinite loop detected)
```

Similarly, workflow E2E tests catch missing steps:

```
User Journey: Create User → Login → Create Item → Delete Item

If "Approve Item" endpoint is missing but required:
- Unit tests: ✓ All pass (each endpoint works in isolation)
- Atomic E2E: ✓ All pass (we manually set up correct state)
- Workflow E2E: ✗ Fails (can't complete the journey)
```

### Database Teardown Patterns

For deterministic tests, wipe database state completely between runs:

```rust
// In a shared glue/utils module
pub static WIPE_DB: &str = "
    DO $$
    DECLARE
        r RECORD;
    BEGIN
        -- Disable referential integrity checks temporarily
        SET session_replication_role = replica;

        -- Drop all tables in public schema
        FOR r IN (
            SELECT tablename FROM pg_tables WHERE schemaname = 'public'
        ) LOOP
            EXECUTE 'DROP TABLE IF EXISTS ' || quote_ident(r.tablename) || ' CASCADE';
        END LOOP;

        -- Re-enable referential integrity
        SET session_replication_role = DEFAULT;
    END $$;
";

// Usage in tests
async fn setup() {
    sqlx::query(WIPE_DB)
        .execute(&*SQLX_POSTGRES_POOL)
        .await
        .unwrap();
    run_migrations().await;
}

#[tokio::test]
async fn test_delete_item() {
    setup().await;  // Clean slate for every test
    // ... test code
}
```

### Workflow E2E Test Example

Test complete user journeys across multiple services:

```rust
#[cfg(test)]
mod tests {
    use to_do_kernel::api::basic_actions::{create::create_item, delete::delete_item};
    use auth_kernel::api::{users::create::create_user, auth::login::login_user};

    async fn setup() {
        // Wipe BOTH databases for multi-service tests
        sqlx::query(WIPE_DB).execute(&*todo_pool).await.unwrap();
        sqlx::query(WIPE_DB).execute(&*auth_pool).await.unwrap();
        run_todo_migrations().await;
        run_auth_migrations().await;
    }

    #[tokio::test]
    async fn test_delete_item_workflow() {
        setup().await;

        // Step 1: Create user via auth service
        let user = CreateUser {
            email: "test@example.com".to_string(),
            password: "secure_password".to_string(),
        };
        create_user(user).await.unwrap();

        // Step 2: Login to get token
        let token = login_user(
            "test@example.com".to_string(),
            "secure_password".to_string()
        ).await.unwrap();
        let header_token = HeaderToken::decode(&token).unwrap();
        let header_token_for_delete = HeaderToken::decode(&token).unwrap();

        // Step 3: Create item via todo service
        let items = create_item(
            header_token,
            "code review".to_string(),
            TaskStatus::PENDING
        ).await.unwrap();
        assert_eq!(items.pending.len(), 1);
        assert_eq!(items.done.len(), 0);

        // Step 4: Delete item
        let items = delete_item(
            header_token_for_delete,
            "code review".to_string()
        ).await.unwrap();
        assert_eq!(items.pending.len(), 0);
        assert_eq!(items.done.len(), 0);
    }
}
```

### Feature-Gated Test Modules

Use Cargo features to conditionally compile tests based on available backends:

```rust
// Only compile these tests when sqlx-postgres feature is enabled
#[cfg(feature = "sqlx-postgres")]
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_postgres_specific_behavior() {
        // Test that only makes sense with real PostgreSQL
    }
}
```

```toml
# Cargo.toml for test workspace
[dependencies]
dal = { path = "../../dal", features = ["sqlx-postgres"] }

[features]
default = []
sqlx-postgres = []
```

Run feature-specific tests:
```bash
cargo test -p dal --features sqlx-postgres -- --test-threads=1
```

### Separate Test Workspaces

Organize different test types into separate Cargo workspaces:

```
project/
├── Cargo.toml                    # Workspace root
├── .env                          # Environment variables
├── docker-compose.yml
├── scripts/
│   └── run_tests.sh
├── logs/                         # Test output logs
├── glue/                         # Shared utilities
│   └── src/
│       ├── urls.rs               # Service URL helpers
│       ├── sql_commands.rs       # WIPE_DB and other SQL
│       └── token.rs              # JWT handling
└── nanoservices/
    ├── auth/
    │   ├── core/                 # Business logic (unit testable)
    │   ├── dal/                  # Data access (DAL tests)
    │   ├── kernel/               # HTTP client abstraction
    │   └── networking/
    │       └── actix_server/     # Server implementation
    └── to_do/
        ├── core/
        ├── dal/
        ├── kernel/
        └── networking/
            ├── actix_server/
            ├── atomic-http-tests/    # Atomic E2E tests
            │   ├── Cargo.toml
            │   └── src/
            └── http-workflow-tests/  # Workflow E2E tests
                ├── Cargo.toml
                └── src/
```

## Snapshot Testing (insta)

Snapshot testing captures output and compares against stored "golden" files. Excellent for testing complex data structures, serialization output, or any output where manual assertion is tedious.

### Basic Snapshot Testing

```rust
use insta::assert_snapshot;

#[test]
fn test_user_display() {
    let user = User {
        id: 1,
        name: "Alice".to_string(),
        email: "alice@example.com".to_string(),
        roles: vec!["admin".to_string(), "user".to_string()],
    };

    // First run creates snapshot file, subsequent runs compare
    assert_snapshot!(format!("{:#?}", user));
}

#[test]
fn test_api_response() {
    let response = api::get_users();
    let json = serde_json::to_string_pretty(&response).unwrap();

    // Named snapshot for clarity
    assert_snapshot!("users_response", json);
}
```

### JSON Snapshot Testing

```rust
use insta::assert_json_snapshot;

#[test]
fn test_serialization() {
    let config = Config {
        host: "localhost".to_string(),
        port: 8080,
        features: vec!["auth".to_string(), "logging".to_string()],
    };

    // Automatically serializes to JSON for comparison
    assert_json_snapshot!(config);
}

#[test]
fn test_api_payload() {
    let payload = build_request_payload();

    // Redact dynamic fields that change between runs
    assert_json_snapshot!(payload, {
        ".timestamp" => "[timestamp]",
        ".request_id" => "[uuid]",
    });
}
```

### YAML Snapshot Testing

```rust
use insta::assert_yaml_snapshot;

#[test]
fn test_config_output() {
    let config = generate_default_config();
    assert_yaml_snapshot!(config);
}
```

### Inline Snapshots

For small outputs, store snapshot inline in the test file:

```rust
use insta::assert_snapshot;

#[test]
fn test_greeting() {
    let greeting = format_greeting("World");

    // Snapshot stored inline (insta fills this in on first run)
    assert_snapshot!(greeting, @"Hello, World!");
}

#[test]
fn test_error_message() {
    let error = ValidationError::InvalidEmail("bad@".to_string());

    assert_snapshot!(error.to_string(), @"Invalid email format: bad@");
}
```

### Snapshot Workflow

```bash
# Run tests - new/changed snapshots are pending
cargo test

# Review pending snapshots interactively
cargo insta review

# Accept all pending snapshots
cargo insta accept

# Reject all pending snapshots
cargo insta reject

# Show pending snapshots
cargo insta pending-snapshots
```

### Snapshot Testing Best Practices

```rust
// Use descriptive snapshot names
assert_snapshot!("user_registration_email_body", email_body);

// Group related snapshots with module structure
mod user_notifications {
    #[test]
    fn welcome_email() {
        assert_snapshot!(generate_welcome_email());
    }

    #[test]
    fn password_reset_email() {
        assert_snapshot!(generate_password_reset_email());
    }
}

// Redact non-deterministic values
assert_json_snapshot!(response, {
    ".created_at" => "[timestamp]",
    ".id" => "[id]",
    ".**.updated_at" => "[timestamp]",  // Nested field pattern
});

// Sort collections for deterministic output
let mut users: Vec<_> = get_users();
users.sort_by_key(|u| u.id);
assert_json_snapshot!(users);
```

## Property-Based Testing (proptest)

Property-based testing generates random inputs to verify that properties hold for all valid inputs. Finds edge cases that manual test cases miss.

### Basic Property Testing

```rust
use proptest::prelude::*;

// Property: reversing twice returns original
proptest! {
    #[test]
    fn reverse_twice_is_identity(s in ".*") {
        let reversed: String = s.chars().rev().collect();
        let double_reversed: String = reversed.chars().rev().collect();
        prop_assert_eq!(s, double_reversed);
    }
}

// Property: sorting is idempotent
proptest! {
    #[test]
    fn sort_is_idempotent(mut vec in prop::collection::vec(any::<i32>(), 0..100)) {
        vec.sort();
        let once_sorted = vec.clone();
        vec.sort();
        prop_assert_eq!(vec, once_sorted);
    }
}
```

### Testing Mathematical Properties

```rust
use proptest::prelude::*;

// Property: addition is commutative
proptest! {
    #[test]
    fn addition_commutative(a: i32, b: i32) {
        prop_assert_eq!(a.wrapping_add(b), b.wrapping_add(a));
    }
}

// Property: addition is associative
proptest! {
    #[test]
    fn addition_associative(a: i32, b: i32, c: i32) {
        let left = a.wrapping_add(b).wrapping_add(c);
        let right = a.wrapping_add(b.wrapping_add(c));
        prop_assert_eq!(left, right);
    }
}

// Property: parsing and formatting are inverses
proptest! {
    #[test]
    fn parse_format_roundtrip(n: u64) {
        let formatted = n.to_string();
        let parsed: u64 = formatted.parse().unwrap();
        prop_assert_eq!(n, parsed);
    }
}
```

### Custom Strategies

```rust
use proptest::prelude::*;

// Generate valid email addresses
fn email_strategy() -> impl Strategy<Value = String> {
    (
        "[a-z]{1,10}",           // local part
        "[a-z]{1,10}",           // domain
        prop_oneof!["com", "org", "net", "io"],  // TLD
    ).prop_map(|(local, domain, tld)| {
        format!("{}@{}.{}", local, domain, tld)
    })
}

proptest! {
    #[test]
    fn email_validation_accepts_valid(email in email_strategy()) {
        prop_assert!(validate_email(&email).is_ok());
    }
}

// Generate valid user structs
fn user_strategy() -> impl Strategy<Value = User> {
    (
        1..10000u64,                    // id
        "[A-Za-z ]{1,50}",             // name
        email_strategy(),               // email
        prop::collection::vec("[a-z]+", 0..5),  // roles
    ).prop_map(|(id, name, email, roles)| {
        User { id, name, email, roles }
    })
}

proptest! {
    #[test]
    fn user_serialization_roundtrip(user in user_strategy()) {
        let json = serde_json::to_string(&user).unwrap();
        let deserialized: User = serde_json::from_str(&json).unwrap();
        prop_assert_eq!(user, deserialized);
    }
}
```

### Testing Invariants

```rust
use proptest::prelude::*;

// Invariant: BoundedQueue never exceeds capacity
proptest! {
    #[test]
    fn bounded_queue_respects_capacity(
        capacity in 1..100usize,
        operations in prop::collection::vec(
            prop_oneof![
                Just(Op::Push(42)),
                Just(Op::Pop),
            ],
            0..200
        )
    ) {
        let mut queue = BoundedQueue::new(capacity);

        for op in operations {
            match op {
                Op::Push(val) => { queue.push(val); }
                Op::Pop => { queue.pop(); }
            }
            // Invariant must hold after every operation
            prop_assert!(queue.len() <= capacity);
        }
    }
}

// Invariant: Binary search tree maintains ordering
proptest! {
    #[test]
    fn bst_maintains_ordering(values in prop::collection::vec(any::<i32>(), 0..100)) {
        let mut bst = BinarySearchTree::new();
        for v in values {
            bst.insert(v);
            prop_assert!(bst.is_valid_bst(), "BST invariant violated after inserting {}", v);
        }
    }
}
```

### Shrinking for Minimal Failing Cases

When a property fails, proptest automatically "shrinks" the input to find a minimal failing case:

```rust
proptest! {
    #[test]
    fn find_bug(v in prop::collection::vec(any::<i32>(), 1..100)) {
        // Bug: crashes on vectors with more than 50 elements where sum > 1000
        let sum: i32 = v.iter().sum();
        if v.len() > 50 && sum > 1000 {
            panic!("Bug triggered!");
        }
        // Proptest will shrink to minimal case: ~51 elements summing to ~1001
    }
}
```

### Configuring Test Runs

```rust
use proptest::prelude::*;

proptest! {
    // Run more cases for thorough testing
    #![proptest_config(ProptestConfig::with_cases(10000))]

    #[test]
    fn thoroughly_tested_property(x: i32) {
        prop_assert!(some_property(x));
    }
}

// Per-test configuration
proptest! {
    #![proptest_config(ProptestConfig {
        cases: 1000,
        max_shrink_iters: 100000,
        ..ProptestConfig::default()
    })]

    #[test]
    fn expensive_property(data in complex_strategy()) {
        prop_assert!(expensive_check(&data));
    }
}
```

### Combining with Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;

    // Traditional unit tests for specific cases
    #[test]
    fn parse_valid_input() {
        assert_eq!(parse("42"), Ok(42));
        assert_eq!(parse("-1"), Ok(-1));
        assert_eq!(parse("0"), Ok(0));
    }

    #[test]
    fn parse_invalid_input() {
        assert!(parse("abc").is_err());
        assert!(parse("").is_err());
    }

    // Property tests for general behavior
    proptest! {
        #[test]
        fn parse_never_panics(s in ".*") {
            let _ = parse(&s);  // Should not panic, even on invalid input
        }

        #[test]
        fn parse_valid_integers(n: i64) {
            let s = n.to_string();
            prop_assert_eq!(parse(&s), Ok(n));
        }
    }
}
```

## Fuzz Testing (cargo-fuzz)

Fuzz testing (fuzzing) automatically generates random or semi-random inputs to discover bugs that traditional testing misses. It excels at finding:
- Parsing vulnerabilities
- Memory safety issues (even in safe Rust via logic bugs)
- Panics from unexpected inputs
- Edge cases in complex algorithms
- Security vulnerabilities

### Setting Up cargo-fuzz

```bash
# Install cargo-fuzz (requires nightly Rust)
cargo install cargo-fuzz

# Switch to nightly for fuzzing
rustup default nightly
# Or use per-project: rustup override set nightly

# Initialize fuzzing for your project
cargo fuzz init

# Add a new fuzz target
cargo fuzz add my_target
```

This creates:
```
fuzz/
├── Cargo.toml
├── fuzz_targets/
│   └── my_target.rs
```

### Basic Fuzz Target

```rust
// fuzz/fuzz_targets/my_target.rs
#![no_main]

use libfuzzer_sys::fuzz_target;
use your_crate::parse_data;

fuzz_target!(|data: &[u8]| {
    // Call the function under test
    // Ignore the result - we're looking for panics/crashes
    let _ = parse_data(data);
});
```

### Running Fuzz Tests

```bash
# Run a fuzz target
cargo fuzz run my_target

# Run for specific duration
cargo fuzz run my_target -- -max_total_time=60

# Run with specific number of jobs
cargo fuzz run my_target -- -jobs=4

# List available targets
cargo fuzz list
```

### Structured Fuzzing with Arbitrary

For complex input types, use the `arbitrary` crate to generate structured data:

```toml
# Cargo.toml
[dependencies]
arbitrary = { version = "1", features = ["derive"] }

# fuzz/Cargo.toml
[dependencies]
libfuzzer-sys = "0.4"
arbitrary = { version = "1", features = ["derive"] }
your_crate = { path = ".." }
```

```rust
// src/lib.rs
use arbitrary::Arbitrary;

#[derive(Arbitrary, Debug, Clone)]
pub struct Config {
    pub server_address: String,
    pub port: u16,
    pub timeout_ms: u32,
    pub features: Vec<String>,
}

#[derive(Arbitrary, Debug)]
pub struct ParsedItem {
    pub length: u8,
    pub data: Vec<u8>,
}

#[derive(Arbitrary, Debug)]
pub struct CustomInput {
    pub magic_number: [u8; 2],
    pub items: Vec<ParsedItem>,
}

pub fn process_config(config: &Config) -> Result<(), String> {
    if config.port == 0 {
        return Err("Port cannot be zero".to_string());
    }
    if config.timeout_ms > 60000 {
        return Err("Timeout too large".to_string());
    }
    Ok(())
}

pub fn process_input(input: &CustomInput) -> Result<(), String> {
    if input.magic_number != [0xCA, 0xFE] {
        return Err("Invalid magic".to_string());
    }
    for item in &input.items {
        if item.data.len() != item.length as usize {
            return Err("Length mismatch".to_string());
        }
    }
    Ok(())
}
```

### Structured Fuzz Target

```rust
// fuzz/fuzz_targets/structured_fuzz.rs
#![no_main]

use libfuzzer_sys::fuzz_target;
use arbitrary::{Arbitrary, Unstructured};
use your_crate::{Config, process_config};

fuzz_target!(|data: &[u8]| {
    // Create Unstructured from raw bytes
    let mut u = Unstructured::new(data);

    // Generate structured input
    if let Ok(config) = Config::arbitrary(&mut u) {
        let _ = process_config(&config);
    }
});
```

### Custom Arbitrary Implementations

For types that need special generation logic:

```rust
use arbitrary::{Arbitrary, Result, Unstructured};

#[derive(Debug, Clone)]
pub struct ValidEmail(String);

impl<'a> Arbitrary<'a> for ValidEmail {
    fn arbitrary(u: &mut Unstructured<'a>) -> Result<Self> {
        let local: String = u.arbitrary()?;
        let domains = ["example.com", "test.org", "mail.io"];
        let domain_idx: usize = u.int_in_range(0..=2)?;

        // Ensure we generate something that looks like an email
        let email = format!(
            "{}@{}",
            local.chars().filter(|c| c.is_alphanumeric()).take(20).collect::<String>(),
            domains[domain_idx]
        );

        Ok(ValidEmail(email))
    }
}

#[derive(Debug, Clone)]
pub struct BoundedVec<T, const MIN: usize, const MAX: usize>(Vec<T>);

impl<'a, T: Arbitrary<'a>, const MIN: usize, const MAX: usize> Arbitrary<'a>
    for BoundedVec<T, MIN, MAX>
{
    fn arbitrary(u: &mut Unstructured<'a>) -> Result<Self> {
        let len: usize = u.int_in_range(MIN..=MAX)?;
        let mut vec = Vec::with_capacity(len);
        for _ in 0..len {
            vec.push(T::arbitrary(u)?);
        }
        Ok(BoundedVec(vec))
    }
}
```

### Stateful Fuzzing

```rust
// fuzz/fuzz_targets/stateful_fuzz.rs
#![no_main]
use libfuzzer_sys::fuzz_target;
use arbitrary::{Arbitrary, Unstructured};
use your_crate::StateMachine;

#[derive(Arbitrary, Debug)]
enum Action {
    Add(u32),
    Remove(u32),
    Clear,
    Process,
}

fuzz_target!(|data: &[u8]| {
    let mut u = Unstructured::new(data);
    let mut machine = StateMachine::new();

    // Execute sequence of random actions
    while let Ok(action) = Action::arbitrary(&mut u) {
        match action {
            Action::Add(v) => machine.add(v),
            Action::Remove(v) => machine.remove(v),
            Action::Clear => machine.clear(),
            Action::Process => { let _ = machine.process(); }
        }
    }
});
```

### Fuzzing a Parser

```rust
// src/lib.rs
pub fn parse_packet(data: &[u8]) -> Result<Packet, ParseError> {
    if data.len() < 4 {
        return Err(ParseError::TooShort);
    }

    let version = data[0];
    let length = u16::from_be_bytes([data[1], data[2]]) as usize;
    let checksum = data[3];

    if length > data.len() - 4 {
        return Err(ParseError::InvalidLength);
    }

    let payload = &data[4..4 + length];

    // Verify checksum
    let calculated: u8 = payload.iter().fold(0u8, |acc, &b| acc.wrapping_add(b));
    if calculated != checksum {
        return Err(ParseError::ChecksumMismatch);
    }

    Ok(Packet { version, payload: payload.to_vec() })
}

// fuzz/fuzz_targets/parse_packet.rs
#![no_main]
use libfuzzer_sys::fuzz_target;
use your_crate::parse_packet;

fuzz_target!(|data: &[u8]| {
    let _ = parse_packet(data);
});
```

### Fuzzing Network Protocol Handling

```rust
// fuzz/fuzz_targets/protocol_fuzz.rs
#![no_main]
use libfuzzer_sys::fuzz_target;
use arbitrary::{Arbitrary, Unstructured};
use your_crate::protocol::{Request, Response, handle_request};

#[derive(Arbitrary, Debug)]
struct FuzzedRequest {
    method: RequestMethod,
    path: String,
    headers: Vec<(String, String)>,
    body: Option<Vec<u8>>,
}

#[derive(Arbitrary, Debug)]
enum RequestMethod {
    Get,
    Post,
    Put,
    Delete,
}

fuzz_target!(|data: &[u8]| {
    let mut u = Unstructured::new(data);

    if let Ok(fuzzed) = FuzzedRequest::arbitrary(&mut u) {
        let request = Request::from_fuzzed(fuzzed);
        let _ = handle_request(request);
    }
});
```

### Seed Corpus and Dictionaries

Provide valid inputs to help the fuzzer explore more efficiently:

```bash
# Create corpus directory
mkdir -p fuzz/corpus/my_target

# Add seed files (binary or text depending on your input format)
echo -ne '\xCA\xFE\x03ABC' > fuzz/corpus/my_target/valid_input_1
echo '{"key": "value"}' > fuzz/corpus/my_target/valid_json_1
```

Specify tokens/keywords relevant to your input format:

```bash
# Create dictionary file
cat > fuzz/dict/my_target.dict << 'EOF'
# Magic bytes
"\xCA\xFE"

# Keywords
"server_address"
"port"
"timeout"

# Common values
"localhost"
"127.0.0.1"
"8080"
"443"
EOF
```

Run with dictionary:
```bash
cargo fuzz run my_target -- -dict=fuzz/dict/my_target.dict
```

### Sanitizers

```bash
# AddressSanitizer (ASan) — buffer overflows, use-after-free, memory leaks
# (default with cargo-fuzz)
cargo fuzz run my_target
RUSTFLAGS="-Z sanitizer=address" cargo fuzz run my_target

# UndefinedBehaviorSanitizer (UBSan)
RUSTFLAGS="-Z sanitizer=undefined" cargo fuzz run my_target

# MemorySanitizer (MSan) — uninitialized memory reads
RUSTFLAGS="-Z sanitizer=memory" cargo fuzz run my_target

# ThreadSanitizer (TSan) — data races in concurrent code
RUSTFLAGS="-Z sanitizer=thread" cargo fuzz run my_target
```

### Handling Crashes and Debugging

```bash
# List crashes
ls fuzz/artifacts/my_target/

# Reproduce a crash
cargo fuzz run my_target fuzz/artifacts/my_target/crash-abc123

# Get backtrace
RUST_BACKTRACE=1 cargo fuzz run my_target fuzz/artifacts/my_target/crash-abc123

# Minimize crashing input to minimal reproducer
cargo fuzz tmin my_target fuzz/artifacts/my_target/crash-abc123
```

### Convert Crashes to Regression Tests

```rust
#[cfg(test)]
mod regression_tests {
    use super::*;

    #[test]
    fn test_crash_abc123() {
        // Include the crashing bytes directly
        let crash_input = include_bytes!("../fuzz/artifacts/my_target/crash-abc123");
        // Should not panic after fix
        let _ = parse_data(crash_input);
    }

    #[test]
    fn test_crash_structured() {
        // For structured inputs, recreate the problematic structure
        let config = Config {
            server_address: String::new(),
            port: 0,  // Edge case that caused crash
            timeout_ms: u32::MAX,
            features: vec![],
        };
        let _ = process_config(&config);
    }
}
```

### Advanced Fuzzing Techniques

```bash
# Coverage-guided: view what code paths are explored
cargo fuzz coverage my_target
cargo cov -- show target/x86_64-unknown-linux-gnu/coverage/my_target \
    --format=html -o coverage_report

# Corpus management: remove redundant inputs
cargo fuzz cmin my_target

# Parallel fuzzing
cargo fuzz run my_target -- -jobs=8 -workers=8

# Fork mode (multiple processes)
cargo fuzz run my_target -- -fork=4

# Per-input timeout (seconds)
cargo fuzz run my_target -- -timeout=5

# Total fuzzing time
cargo fuzz run my_target -- -max_total_time=3600
```

### CI/CD Integration for Fuzz Tests

```yaml
name: Fuzz Testing

on:
  schedule:
    - cron: '0 0 * * *'  # Daily
  workflow_dispatch:

jobs:
  fuzz:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust nightly
        uses: dtolnay/rust-toolchain@nightly

      - name: Install cargo-fuzz
        run: cargo install cargo-fuzz

      - name: Run fuzz tests
        run: |
          for target in $(cargo fuzz list); do
            cargo fuzz run $target -- -max_total_time=300
          done

      - name: Upload crashes
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: fuzz-crashes
          path: fuzz/artifacts/
```

### Fuzz Testing Best Practices

1. **Start with byte slice targets** — Simple to set up, catches many bugs
2. **Graduate to structured fuzzing** — Use arbitrary for complex inputs
3. **Maintain seed corpus** — Include valid inputs and edge cases
4. **Use dictionaries** — Speed up exploration for text/protocol formats
5. **Run continuously** — Integrate into CI for ongoing coverage
6. **Convert crashes to tests** — Prevent regressions
7. **Enable sanitizers** — Catch memory issues early
8. **Minimize crashing inputs** — Easier to debug small reproducers

## Test Configuration

### Cargo.toml

```toml
[dev-dependencies]
tokio = { version = "1", features = ["test-util", "macros", "rt-multi-thread"] }
mockall = "0.13"
proptest = "1"
insta = { version = "1", features = ["json"] }
assert_cmd = "2"
predicates = "3"
tempfile = "3"

# Separate fuzz setup
# cargo fuzz init creates fuzz/Cargo.toml
```

### Running Tests

```bash
cargo test                          # All tests
cargo test test_name                # Filter by name
cargo test -- --ignored             # Run ignored tests
cargo test -- --test-threads=1      # Sequential execution
cargo test -- --nocapture           # Show println! output
cargo test --lib                    # Only unit tests
cargo test --test integration       # Only tests/integration.rs
cargo test --doc                    # Only doc tests
RUST_LOG=debug cargo test           # With logging
```

## Test Design Best Practices

| Practice | Description |
|----------|-------------|
| **One assert per behavior** | Each test verifies one specific behavior |
| **Arrange-Act-Assert** | Clear test structure |
| **Test public API** | Don't test private implementation details |
| **Use helpers** | Extract shared setup into helper functions |
| **Name descriptively** | `test_returns_error_on_empty_input` not `test1` |
| **Prefer trait mocks** | Design for testability with trait boundaries |
| **Transaction rollback** | Isolate database tests with transactions |
| **Avoid test interdependence** | Tests must pass in any order |
| **Sequential for DB tests** | Use `--test-threads=1` for shared database |
| **Feature-gate backends** | Conditionally compile tests needing specific services |

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: error handling patterns, trait design for testability, `#[test]` basics
- **[error-handling.md](error-handling.md)** — Error types to test against, thiserror enum variants for `assert!(matches!())`
- **[async-concurrency.md](async-concurrency.md)** — `#[tokio::test]`, async test patterns, timeout testing
- **[web-apis.md](web-apis.md)** — API integration testing, reqwest-based E2E tests
- **[database.md](database.md)** — SQLx test fixtures, transaction rollback isolation
- **[unsafe-ffi.md](unsafe-ffi.md)** — MIRI testing for unsafe code, FFI boundary testing
