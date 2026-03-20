# Rust Type System

Comprehensive type system reference: traits, generics, type state, GATs, const generics, Pin/Unpin, sealed traits, async traits, conversions (From/Into/TryFrom/AsRef/Deref), lifetime patterns, and the orphan rule.

## Rules for Type System Usage (LLM)

1. **ALWAYS use `pin-project-lite` over `pin-project`** for Pin projection — declarative macro only, zero proc-macro deps, used by tokio/hyper/tower; use `pin-project` only if you need tuple structs or custom Unpin
2. **NEVER implement `Unpin` manually for self-referential types** — this breaks Pin's safety guarantee; let the compiler decide or use `PhantomPinned`
3. **PREFER native async traits over `async-trait` crate** for new code — zero heap allocation; only use `async-trait` when you need `dyn Trait` dispatch
4. **ALWAYS use `trait_variant::make`** when you need both generic and dyn-compatible async trait versions — generates a Send-bounded variant automatically
5. **PREFER const generics over associated constants** for array sizes — `fn foo<const N: usize>(arr: [T; N])` is clearer and more flexible than trait-based `const DIM: usize`
6. **ALWAYS seal traits** when you need to add methods in future versions without breaking changes — the `mod sealed` pattern prevents external implementations
7. **PREFER type state over runtime state checks** for protocol state machines with <5 states — compile-time enforcement eliminates entire classes of bugs
8. **ALWAYS implement `From<T>`, never `Into<T>` directly** — implementing From gives you Into for free via blanket impl; implementing Into directly doesn't give you From
9. **PREFER `impl Into<T>` in function parameters, `From<T>` in implementations** — Into in params accepts more types; From in impl defines the canonical conversion
10. **NEVER abuse `Deref` for "inheritance"** — Deref is for smart pointer types (Box, Arc, newtypes wrapping a single field), not for subtyping or delegation between unrelated types

### Common Mistakes (BAD/GOOD)

**Wrong pin projection:**
```rust
// BAD: manual pin projection — easy to make unsound
impl<F: Future> Future for Wrapper<F> {
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        let inner = unsafe { &mut self.get_unchecked_mut().inner };
        let pinned = unsafe { Pin::new_unchecked(inner) };
        pinned.poll(cx)
    }
}

// GOOD: pin-project-lite handles projection safely
pin_project! {
    struct Wrapper<F> {
        #[pin]
        inner: F,
    }
}
impl<F: Future> Future for Wrapper<F> {
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        self.project().inner.poll(cx)  // safe, zero overhead
    }
}
```

**Using async-trait unnecessarily:**
```rust
// BAD: heap allocation on every call, proc macro compile time
#[async_trait]
trait Store {
    async fn get(&self, key: &str) -> Option<Vec<u8>>;
}
fn use_store(s: &impl Store) { /* ... */ }  // generic dispatch — no dyn needed

// GOOD: native async trait — zero allocation, no proc macro
trait Store {
    async fn get(&self, key: &str) -> Option<Vec<u8>>;
}
fn use_store(s: &impl Store) { /* ... */ }
```

**Manual state validation vs type state:**
```rust
// BAD: runtime panics for invalid state transitions
struct Connection { state: State }
impl Connection {
    fn query(&self) -> Result<Data> {
        if self.state != State::Authenticated {
            panic!("must authenticate first!");
        }
        // ...
    }
}

// GOOD: invalid transitions are compile-time errors
impl Connection<Authenticated> {
    fn query(&self) -> Result<Data> { /* ... */ }
}
// Connection<Connected>::query() doesn't exist
```

**Implementing Into instead of From:**
```rust
// BAD: implementing Into directly — doesn't give you From
impl Into<String> for MyType {
    fn into(self) -> String { self.0.clone() }
}

// GOOD: implement From — Into comes for free
impl From<MyType> for String {
    fn from(val: MyType) -> String { val.0.clone() }
}
// Both work: String::from(my_val) and my_val.into()
```

**Orphan rule violation:**
```rust
// BAD: foreign trait on foreign type — won't compile
impl std::fmt::Display for Vec<u8> { /* orphan rule violation */ }

// GOOD: newtype wrapper
struct Bytes(Vec<u8>);
impl std::fmt::Display for Bytes {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{:?}", self.0)
    }
}
```

**Non-object-safe trait:**
```rust
// BAD: can't use as dyn Trait — returns Self and has generic method
trait Processor {
    fn clone_self(&self) -> Self;          // Returns Self
    fn process<T>(&self, data: T) -> T;    // Generic method
}

// GOOD: object-safe design
trait Processor {
    fn clone_boxed(&self) -> Box<dyn Processor>;
    fn process(&self, data: &[u8]) -> Vec<u8>;
}
```

**'static misconception:**
```rust
// BAD: thinking 'static means "lives forever"
// T: 'static does NOT mean T is a static reference — it means T owns all its data
fn spawn_task<T: Send + 'static>(data: T) { /* ... */ }

// This compiles — String is 'static because it owns its data (no borrowed refs)
spawn_task(String::from("hello"));  // OK: String: 'static

// This fails — &str with non-'static lifetime is not 'static
let local = String::from("hello");
// spawn_task(&local);  // ERROR: &local is not 'static

// 'static references ARE 'static (string literals, leaked values)
spawn_task("hello");  // OK: &'static str
```

### Section Index

| Section | Topics |
|---------|--------|
| [Trait Patterns](#trait-patterns) | Extension traits, blanket impls, orphan rule, trait objects, object safety, supertraits, associated type constraints, unconditional impls, sealed traits, conditional bounds |
| [Type Conversions](#type-conversions) | From/Into, TryFrom, AsRef/AsMut, Deref coercion, borrow guards, conversion hierarchy |
| [Type State Pattern](#type-state-pattern) | Basic type state, builder with type state, protocol state machine, when to use |
| [Generic Associated Types](#generic-associated-types-gats) | Lending iterator, generic collection trait, type-parameterized, async GATs |
| [Const Generics](#const-generics) | Fixed-size arrays, matrix, fixed-capacity buffer, default values, nightly expressions |
| [Pin and Unpin](#pin-and-unpin) | Why Pin exists, creating pinned values, manual Future, pin projection, pin-project-lite |
| [Native Async Traits](#native-async-traits-vs-async-trait-crate) | RPITIT, limitations, Send bounds, when to use async-trait, Rust 2024 capture rules |
| [Lifetime Patterns](#lifetime-patterns) | Elision rules, struct lifetimes, 'static, variance, PhantomData (6 uses), HRTBs, DeserializeOwned pattern |
| [Combining Patterns](#combining-patterns) | Sealed + const generics + type state composition |

## Trait Patterns

### Extension Traits

```rust
// Add methods to types you don't own
trait StringExt {
    fn truncate_with_ellipsis(&self, max_len: usize) -> String;
    fn is_blank(&self) -> bool;
}

impl StringExt for str {
    fn truncate_with_ellipsis(&self, max_len: usize) -> String {
        if self.len() <= max_len {
            self.to_string()
        } else {
            format!("{}...", &self[..max_len.saturating_sub(3)])
        }
    }

    fn is_blank(&self) -> bool {
        self.trim().is_empty()
    }
}

// Now available on all &str and String (via Deref)
let title = "A Very Long Title That Should Be Truncated".truncate_with_ellipsis(20);
```

### Blanket Implementations

```rust
// Implement for all types satisfying a bound
trait Loggable {
    fn log(&self);
}

// Every Debug type is automatically Loggable
impl<T: std::fmt::Debug> Loggable for T {
    fn log(&self) {
        tracing::info!("{:?}", self);
    }
}

// Common pattern: extend Iterator for all iterators
trait IterStats: Iterator<Item = f64> + Sized {
    fn mean(self) -> f64 {
        let (sum, count) = self.fold((0.0, 0u64), |(s, c), x| (s + x, c + 1));
        if count == 0 { 0.0 } else { sum / count as f64 }
    }
}
impl<I: Iterator<Item = f64>> IterStats for I {}

// Usage: automatic
let avg = vec![1.0, 2.0, 3.0].into_iter().mean();
```

### Trait Coherence and the Orphan Rule

```rust
// You can implement a trait for a type only if:
// - You own the trait, OR
// - You own the type

// WORKS: your trait on foreign type
trait MySerialize { fn serialize(&self) -> String; }
impl MySerialize for Vec<u8> { /* ... */ }

// WORKS: foreign trait on your type
struct MyType(u64);
impl std::fmt::Display for MyType { /* ... */ }

// FAILS: foreign trait on foreign type
// impl std::fmt::Display for Vec<u8> { /* ... */ }  // Orphan rule violation

// Workaround: newtype wrapper
struct Bytes(Vec<u8>);
impl std::fmt::Display for Bytes {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{:?}", self.0)
    }
}
```

### Trait Object Patterns

```rust
// Heterogeneous collection
trait Plugin: Send + Sync {
    fn name(&self) -> &str;
    fn execute(&self, input: &str) -> String;
}

struct PluginRegistry {
    plugins: Vec<Box<dyn Plugin>>,
}

impl PluginRegistry {
    fn register(&mut self, plugin: impl Plugin + 'static) {
        self.plugins.push(Box::new(plugin));
    }

    fn run_all(&self, input: &str) -> Vec<String> {
        self.plugins.iter()
            .map(|p| p.execute(input))
            .collect()
    }
}
```

### Object Safety Rules

A trait is object-safe (usable as `dyn Trait`) only if:

| Requirement | Why |
|-------------|-----|
| No methods returning `Self` | Size of Self unknown at runtime |
| No generic methods | Can't monomorphize through vtable |
| Methods take `&self`, `&mut self`, or `Box<Self>` | Need known receiver type |
| No associated functions without `self` | No way to call without concrete type |
| Trait itself has no `Sized` supertrait | Trait objects are `!Sized` |

```rust
// BAD: not object-safe
trait NotSafe {
    fn clone_self(&self) -> Self;      // Returns Self — size unknown
    fn process<T>(&self, data: T);     // Generic method — can't dispatch
}

// GOOD: object-safe
trait Safe {
    fn clone_boxed(&self) -> Box<dyn Safe>;
    fn process(&self, data: &dyn std::any::Any);
}

// Workaround: where Self: Sized excludes method from dyn dispatch
trait MostlySafe {
    fn normal_method(&self) -> String;
    fn clone_self(&self) -> Self where Self: Sized;  // Excluded from vtable
}
// Box<dyn MostlySafe> works — clone_self just isn't available
```

### Supertraits and Trait Composition

```rust
// Supertrait — require another trait as prerequisite
trait Saveable: std::fmt::Debug + Clone {
    fn save(&self) -> Result<(), Error>;
}
// Any impl of Saveable must also impl Debug + Clone

// Multiple trait bounds in function signatures
fn process(item: &(impl Saveable + std::fmt::Display)) {
    println!("Processing: {item}");
    item.save().unwrap();
}

// Trait aliases (not yet stable — use blanket impl workaround)
trait JsonSerializable: serde::Serialize + serde::de::DeserializeOwned + Send + Sync + 'static {}
impl<T> JsonSerializable for T
where T: serde::Serialize + serde::de::DeserializeOwned + Send + Sync + 'static {}

// Now use JsonSerializable as a single bound instead of 5
fn store<T: JsonSerializable>(value: &T) -> Result<(), Error> { todo!() }
```

### Associated Type Equality Constraints

Constrain associated types across trait hierarchies — used extensively in serde's Serializer:

```rust
// Problem: a Serializer has many sub-serializers (for sequences, maps, structs).
// Each sub-serializer must produce the same Ok/Error types as the parent.
// Without constraints, they could diverge.

trait Serializer {
    type Ok;
    type Error: std::error::Error;

    // Associated type equality: SerializeSeq's Ok and Error MUST match Self's
    type SerializeSeq: SerializeSeq<Ok = Self::Ok, Error = Self::Error>;
    type SerializeMap: SerializeMap<Ok = Self::Ok, Error = Self::Error>;

    fn serialize_seq(self, len: Option<usize>) -> Result<Self::SerializeSeq, Self::Error>;
    fn serialize_map(self, len: Option<usize>) -> Result<Self::SerializeMap, Self::Error>;
}

trait SerializeSeq {
    type Ok;
    type Error: std::error::Error;

    fn serialize_element<T: Serialize + ?Sized>(&mut self, value: &T) -> Result<(), Self::Error>;
    fn end(self) -> Result<Self::Ok, Self::Error>;
}

// This guarantees type safety across the entire serialization chain:
// Serializer::Ok == SerializeSeq::Ok == SerializeMap::Ok
// No runtime type mismatches possible
```

This pattern is essential when designing trait families where multiple traits must agree on types. The equality constraint `Ok = Self::Ok` is enforced at compile time, so implementors can't accidentally return incompatible types from sub-serializers.

### Unconditional Trait Implementations

Implement traits for wrappers without requiring the inner type to implement the trait:

```rust
use std::sync::Arc;

// tokio pattern: Sender wraps Arc internally
struct Sender<T> {
    shared: Arc<SharedState<T>>,
}

// Clone only clones the Arc — T doesn't need Clone
impl<T> Clone for Sender<T> {
    fn clone(&self) -> Self {
        Self { shared: Arc::clone(&self.shared) }
    }
}

// Common mistake: adding unnecessary bounds
// BAD: impl<T: Clone> Clone for Sender<T>  // Needlessly restricts T
// GOOD: impl<T> Clone for Sender<T>         // Arc::clone doesn't need T: Clone

// Same pattern works for Debug, Display, etc.
impl<T> std::fmt::Debug for Sender<T> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("Sender").finish()  // Don't need T: Debug
    }
}

// Rule: Only add bounds that the implementation actually requires.
// Generic structs that wrap Arc, Rc, or raw pointers often need
// fewer bounds than you'd expect.
```

### Sealed Traits

Prevent external crates from implementing your trait — enables adding methods without breaking changes.

#### Basic Sealed Trait

```rust
mod sealed {
    // This module is private — external code can't access it
    pub trait Sealed {}
}

// Public trait with private supertrait
pub trait Format: sealed::Sealed {
    fn content_type(&self) -> &'static str;
    fn serialize(&self, data: &[u8]) -> Vec<u8>;
}

// Only types in this crate can implement Sealed, therefore only
// types in this crate can implement Format.

pub struct Json;
impl sealed::Sealed for Json {}
impl Format for Json {
    fn content_type(&self) -> &'static str { "application/json" }
    fn serialize(&self, data: &[u8]) -> Vec<u8> { data.to_vec() }
}

pub struct Msgpack;
impl sealed::Sealed for Msgpack {}
impl Format for Msgpack {
    fn content_type(&self) -> &'static str { "application/msgpack" }
    fn serialize(&self, data: &[u8]) -> Vec<u8> { data.to_vec() }
}

// External crates can USE Format but not IMPLEMENT it:
// fn use_format(f: &impl Format) { ... }  // ✓ works
// impl Format for MyType { ... }           // ✗ can't implement Sealed
```

#### Sealed Trait with Provided Methods

Since no external impls exist, you can add methods with default impls without breaking changes:

```rust
mod sealed { pub trait Sealed {} }

pub trait DatabaseDriver: sealed::Sealed {
    fn connect(&self, url: &str) -> Connection;

    // Can add this in a later version — no external impls to break
    fn connect_with_timeout(&self, url: &str, timeout: std::time::Duration) -> Connection {
        // Default implementation
        self.connect(url)
    }
}
# struct Connection;
```

#### Partially Sealed Trait

Allow implementing some methods but not others:

```rust
mod sealed {
    pub trait SealedExt {}
}

pub trait Storage {
    // These can be implemented by anyone:
    fn read(&self, key: &str) -> Option<Vec<u8>>;
    fn write(&self, key: &str, value: &[u8]);
}

// Extension trait that's sealed — only this crate provides impls
pub trait StorageExt: Storage + sealed::SealedExt {
    fn read_string(&self, key: &str) -> Option<String> {
        self.read(key).and_then(|v| String::from_utf8(v).ok())
    }

    fn write_string(&self, key: &str, value: &str) {
        self.write(key, value.as_bytes());
    }
}

// Blanket impl — every Storage automatically gets StorageExt
impl<T: Storage> sealed::SealedExt for T {}
impl<T: Storage> StorageExt for T {}
```

#### Sealed Enum Pattern

For exhaustive matching guarantees:

```rust
mod sealed { pub trait Sealed {} }

pub trait Event: sealed::Sealed {
    fn event_type(&self) -> &'static str;
}

pub struct Created { pub id: u64 }
pub struct Updated { pub id: u64, pub field: String }
pub struct Deleted { pub id: u64 }

impl sealed::Sealed for Created {}
impl sealed::Sealed for Updated {}
impl sealed::Sealed for Deleted {}

impl Event for Created { fn event_type(&self) -> &'static str { "created" } }
impl Event for Updated { fn event_type(&self) -> &'static str { "updated" } }
impl Event for Deleted { fn event_type(&self) -> &'static str { "deleted" } }

// Users know the trait is sealed, so they can match on concrete types
// knowing no other implementations exist
```

### Conditional Trait Bounds via cfg

Vary trait bounds based on features or target — used by serde for no_std support:

```rust
// serde pattern: Error trait changes bounds based on std availability

#[cfg(feature = "std")]
pub trait Error: std::error::Error + Send + Sync + 'static {
    fn custom<T: std::fmt::Display>(msg: T) -> Self;
}

#[cfg(not(feature = "std"))]
pub trait Error: std::fmt::Debug + std::fmt::Display + 'static {
    fn custom<T: std::fmt::Display>(msg: T) -> Self;
}

// Alternative: conditional supertrait with cfg_attr
pub trait Error:
    #[cfg_attr(feature = "std", std::error::Error)]
    std::fmt::Debug + std::fmt::Display + Send + Sync + 'static
{
    fn custom<T: std::fmt::Display>(msg: T) -> Self;
}

// Conditional derive — common in no_std crates
#[derive(Debug)]
#[cfg_attr(feature = "std", derive(thiserror::Error))]
pub enum ParseError {
    #[cfg_attr(feature = "std", error("invalid format: {0}"))]
    InvalidFormat(String),
    #[cfg_attr(feature = "std", error("out of range"))]
    OutOfRange,
}

// Conditional impl blocks
struct MyType;

#[cfg(feature = "serde")]
impl serde::Serialize for MyType {
    fn serialize<S: serde::Serializer>(&self, s: S) -> Result<S::Ok, S::Error> {
        todo!()
    }
}
```

This pattern enables library code to work across std/no_std and optional feature dependencies without code duplication.

## Type Conversions

### The Conversion Hierarchy

```rust
// From<T> — explicit, infallible conversion
// Into<T> — automatic from From, used in bounds
// TryFrom<T> — fallible conversion
// AsRef<T> — cheap reference conversion
// Deref<Target=T> — implicit reference coercion

// Implementing From gives you Into for free
struct Celsius(f64);
struct Fahrenheit(f64);

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self {
        Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
    }
}

let f: Fahrenheit = Celsius(100.0).into();  // Uses Into (auto from From)
let f = Fahrenheit::from(Celsius(100.0));   // Uses From directly
```

### impl Into<T> in Parameters

```rust
// Accept anything convertible to the target type
struct Config {
    name: String,
    timeout: Duration,
}

impl Config {
    // Accept &str, String, Cow<str>, etc.
    fn new(name: impl Into<String>) -> Self {
        Self { name: name.into(), timeout: Duration::from_secs(30) }
    }

    fn with_name(mut self, name: impl Into<String>) -> Self {
        self.name = name.into();
        self
    }
}

// Callers can pass any string type without explicit conversion
let c1 = Config::new("static");           // &str → String
let c2 = Config::new(dynamic_string);     // String → String (no-op)
let c3 = Config::new(cow_string);         // Cow<str> → String
```

### TryFrom for Validated Types

```rust
use std::convert::TryFrom;

struct Port(u16);

impl TryFrom<u32> for Port {
    type Error = PortError;

    fn try_from(value: u32) -> Result<Self, Self::Error> {
        if value == 0 {
            Err(PortError::Zero)
        } else if value > 65535 {
            Err(PortError::OutOfRange(value))
        } else {
            Ok(Port(value as u16))
        }
    }
}

// Usage
let port = Port::try_from(8080)?;       // Ok(Port(8080))
let port = Port::try_from(0)?;          // Err(PortError::Zero)
let port = Port::try_from(100_000)?;    // Err(PortError::OutOfRange(100000))

// TryFrom<&str> for parsing
impl TryFrom<&str> for Port {
    type Error = PortError;
    fn try_from(s: &str) -> Result<Self, Self::Error> {
        let n: u32 = s.parse().map_err(|_| PortError::InvalidFormat)?;
        Port::try_from(n)
    }
}
```

### AsRef and AsMut

```rust
// AsRef — cheap reference conversion, doesn't allocate
// Use in function parameters to accept multiple types

fn read_file(path: impl AsRef<std::path::Path>) -> std::io::Result<String> {
    std::fs::read_to_string(path.as_ref())
}

// Accepts: &str, String, PathBuf, &Path, OsString, etc.
read_file("config.toml")?;
read_file(PathBuf::from("/etc/config.toml"))?;
read_file(&some_string)?;

// AsRef for newtypes
struct Username(String);

impl AsRef<str> for Username {
    fn as_ref(&self) -> &str { &self.0 }
}

// Now Username works anywhere &str is expected via AsRef
fn greet(name: impl AsRef<str>) {
    println!("Hello, {}!", name.as_ref());
}
greet(Username("Alice".into()));
greet("Bob");
greet(String::from("Charlie"));
```

### Deref Coercion

```rust
use std::ops::Deref;

// Deref enables transparent forwarding
struct Email(String);

impl Deref for Email {
    type Target = str;
    fn deref(&self) -> &str { &self.0 }
}

let email = Email("alice@example.com".into());
println!("{}", email.len());           // str::len() via Deref
println!("{}", email.contains('@'));   // str::contains() via Deref

// Deref chain: Box<String> → String → str
let boxed: Box<String> = Box::new("hello".into());
let s: &str = &boxed;  // Deref coercion through Box → String → str

// CAUTION: Don't abuse Deref for "inheritance"
// Deref is for smart pointer types, not for subtyping
// BAD: struct Admin(User); impl Deref for Admin { type Target = User; }
// GOOD: struct Admin { user: User, permissions: Permissions }
```

### Borrow Guard Pattern

Custom RAII guard that derefs to the borrowed value — used in tokio's watch channel:

```rust
use std::ops::Deref;
use std::sync::Arc;

// Guard that tracks borrowing via reference counting
pub struct Ref<'a, T> {
    inner: &'a T,
    // When Ref is dropped, it decrements the refcount, signaling
    // that the value is no longer being read
    _ref_count: Arc<()>,
}

impl<T> Deref for Ref<'_, T> {
    type Target = T;
    fn deref(&self) -> &T {
        self.inner
    }
}

impl<T> Drop for Ref<'_, T> {
    fn drop(&mut self) {
        // Decrement refcount — may signal writer that reads are done
        // (Arc::strong_count decreases when this Ref is dropped)
    }
}

// Usage: transparent read access with automatic cleanup
fn read_config(rx: &WatchReceiver<Config>) {
    let config: Ref<'_, Config> = rx.borrow();
    println!("{}", config.host);    // Deref to &Config
    println!("{}", config.port);    // Still holding borrow guard
}   // Ref dropped here — signals that read is complete
```

This pattern is useful when you need to:
- Track outstanding borrows (watch channels, read-write locks)
- Perform cleanup when a borrow ends (release notifications)
- Provide transparent `&T` access while maintaining bookkeeping

### Conversion Decision Guide

| Want to... | Use |
|------------|-----|
| Convert owned value, always succeeds | `From<T>` / `.into()` |
| Convert owned value, may fail | `TryFrom<T>` / `.try_into()` |
| Borrow as reference cheaply | `AsRef<T>` / `.as_ref()` |
| Get mutable reference cheaply | `AsMut<T>` / `.as_mut()` |
| Implicit coercion (smart pointers) | `Deref` (compiler applies automatically) |
| Accept flexible input in function params | `impl Into<T>` or `impl AsRef<T>` |
| Parse from string | `FromStr` / `.parse()` |

## Type State Pattern

Encode state machines in the type system so invalid transitions are compile-time errors.

### Basic Type State

```rust
use std::marker::PhantomData;

// States — zero-sized types, exist only at compile time
struct Draft;
struct Review;
struct Published;

struct Document<State> {
    title: String,
    content: String,
    _state: PhantomData<State>,
}

// Only Draft documents can be edited
impl Document<Draft> {
    fn new(title: String) -> Self {
        Self { title, content: String::new(), _state: PhantomData }
    }

    fn set_content(mut self, content: String) -> Self {
        self.content = content;
        self
    }

    fn submit_for_review(self) -> Document<Review> {
        Document { title: self.title, content: self.content, _state: PhantomData }
    }
}

// Only Review documents can be approved or rejected
impl Document<Review> {
    fn approve(self) -> Document<Published> {
        Document { title: self.title, content: self.content, _state: PhantomData }
    }

    fn reject(self) -> Document<Draft> {
        Document { title: self.title, content: self.content, _state: PhantomData }
    }
}

// Only Published documents can be read publicly
impl Document<Published> {
    fn public_url(&self) -> String {
        format!("/documents/{}", self.title.to_lowercase().replace(' ', "-"))
    }
}

// Methods available in ALL states
impl<S> Document<S> {
    fn title(&self) -> &str {
        &self.title
    }
}

fn usage() {
    let doc = Document::<Draft>::new("My Post".into())
        .set_content("Hello world".into())
        .submit_for_review()
        .approve();

    println!("{}", doc.public_url());

    // Won't compile — Published documents can't be edited:
    // doc.set_content("changed".into());

    // Won't compile — Draft documents can't be approved:
    // Document::<Draft>::new("x".into()).approve();
}
```

### Builder with Type State

Enforce required fields at compile time — no runtime "missing field" errors.

```rust
use std::marker::PhantomData;

// Field presence markers
struct Missing;
struct Present;

struct ConnectionBuilder<Host, Port> {
    host: Option<String>,
    port: Option<u16>,
    timeout_ms: u64,
    tls: bool,
    _marker: PhantomData<(Host, Port)>,
}

impl ConnectionBuilder<Missing, Missing> {
    fn new() -> Self {
        Self {
            host: None,
            port: None,
            timeout_ms: 5000,
            tls: false,
            _marker: PhantomData,
        }
    }
}

impl<Port> ConnectionBuilder<Missing, Port> {
    fn host(self, host: impl Into<String>) -> ConnectionBuilder<Present, Port> {
        ConnectionBuilder {
            host: Some(host.into()),
            port: self.port,
            timeout_ms: self.timeout_ms,
            tls: self.tls,
            _marker: PhantomData,
        }
    }
}

impl<Host> ConnectionBuilder<Host, Missing> {
    fn port(self, port: u16) -> ConnectionBuilder<Host, Present> {
        ConnectionBuilder {
            host: self.host,
            port: Some(port),
            timeout_ms: self.timeout_ms,
            tls: self.tls,
            _marker: PhantomData,
        }
    }
}

// Optional fields — available in any state
impl<Host, Port> ConnectionBuilder<Host, Port> {
    fn timeout(mut self, ms: u64) -> Self {
        self.timeout_ms = ms;
        self
    }

    fn tls(mut self, enabled: bool) -> Self {
        self.tls = enabled;
        self
    }
}

struct Connection {
    host: String,
    port: u16,
    timeout_ms: u64,
    tls: bool,
}

// build() only available when ALL required fields are present
impl ConnectionBuilder<Present, Present> {
    fn build(self) -> Connection {
        Connection {
            host: self.host.unwrap(),
            port: self.port.unwrap(),
            timeout_ms: self.timeout_ms,
            tls: self.tls,
        }
    }
}

fn usage() {
    // Compiles — both required fields set
    let conn = ConnectionBuilder::new()
        .host("localhost")
        .port(5432)
        .tls(true)
        .timeout(3000)
        .build();

    // Won't compile — port missing:
    // ConnectionBuilder::new().host("localhost").build();
}
```

### Protocol State Machine

```rust
use std::marker::PhantomData;

struct Disconnected;
struct Connected;
struct Authenticated;

struct Session<State> {
    addr: String,
    token: Option<String>,
    _state: PhantomData<State>,
}

impl Session<Disconnected> {
    fn new(addr: impl Into<String>) -> Self {
        Self { addr: addr.into(), token: None, _state: PhantomData }
    }

    fn connect(self) -> Result<Session<Connected>, std::io::Error> {
        // TCP handshake...
        Ok(Session { addr: self.addr, token: None, _state: PhantomData })
    }
}

impl Session<Connected> {
    fn authenticate(self, credentials: &str) -> Result<Session<Authenticated>, AuthError> {
        let token = validate(credentials)?;
        Ok(Session { addr: self.addr, token: Some(token), _state: PhantomData })
    }

    fn disconnect(self) -> Session<Disconnected> {
        Session { addr: self.addr, token: None, _state: PhantomData }
    }
}

impl Session<Authenticated> {
    fn query(&self, sql: &str) -> Result<Vec<Row>, QueryError> {
        // Only authenticated sessions can query
        todo!()
    }

    fn disconnect(self) -> Session<Disconnected> {
        Session { addr: self.addr, token: None, _state: PhantomData }
    }
}

// Type state enforces: connect → authenticate → query
// Can't query without authenticating, can't authenticate without connecting
# struct Row;
# struct AuthError;
# struct QueryError;
# fn validate(_: &str) -> Result<String, AuthError> { todo!() }
```

### When to Use Type State

| Use Type State | Don't Use Type State |
|----------------|---------------------|
| Protocol state machines | Many possible states (>5) |
| Builder pattern with required fields | State changes at runtime based on user input |
| Resource lifecycle (open/closed) | States need to be stored in collections |
| Permission levels | State transitions are data-dependent |
| Compile-time guarantees matter | Simplicity matters more than safety |

**Limitation:** Type state objects can't be stored in a `Vec<Session<_>>` because each state is a different type. Use an enum wrapper if you need heterogeneous collections.

## Generic Associated Types (GATs)

GATs allow associated types in traits to have their own generic parameters. Stable since Rust 1.65.

### The Problem GATs Solve

```rust
// WITHOUT GATs — can't express "returns a reference with the container's lifetime"

// This trait can't express lending iteration:
trait LendingIterator {
    type Item;  // No way to tie Item's lifetime to &self
    fn next(&mut self) -> Option<Self::Item>;
}

// WITH GATs — Item can be parameterized by lifetime
trait LendingIterator {
    type Item<'a> where Self: 'a;
    fn next(&mut self) -> Option<Self::Item<'_>>;
}
```

### Lending Iterator

The canonical GAT example — an iterator that lends from itself:

```rust
trait LendingIterator {
    type Item<'a> where Self: 'a;
    fn next(&mut self) -> Option<Self::Item<'_>>;
}

// Iterate over overlapping windows of a slice
struct WindowsMut<'s, T> {
    data: &'s mut [T],
    pos: usize,
    window_size: usize,
}

impl<'s, T> LendingIterator for WindowsMut<'s, T> {
    type Item<'a> = &'a mut [T] where Self: 'a;

    fn next(&mut self) -> Option<Self::Item<'_>> {
        if self.pos + self.window_size > self.data.len() {
            return None;
        }
        let start = self.pos;
        self.pos += 1;
        // Reborrow from &mut self — each call borrows from the iterator
        Some(&mut self.data[start..start + self.window_size])
    }
}
```

### Generic Collection Trait

```rust
trait Collection {
    type Item;
    type Iter<'a>: Iterator<Item = &'a Self::Item> where Self: 'a;
    type IterMut<'a>: Iterator<Item = &'a mut Self::Item> where Self: 'a;

    fn iter(&self) -> Self::Iter<'_>;
    fn iter_mut(&mut self) -> Self::IterMut<'_>;
    fn push(&mut self, item: Self::Item);
    fn len(&self) -> usize;
    fn is_empty(&self) -> bool { self.len() == 0 }
}

impl<T> Collection for Vec<T> {
    type Item = T;
    type Iter<'a> = std::slice::Iter<'a, T> where T: 'a;
    type IterMut<'a> = std::slice::IterMut<'a, T> where T: 'a;

    fn iter(&self) -> Self::Iter<'_> { self.as_slice().iter() }
    fn iter_mut(&mut self) -> Self::IterMut<'_> { self.as_mut_slice().iter_mut() }
    fn push(&mut self, item: T) { Vec::push(self, item); }
    fn len(&self) -> usize { Vec::len(self) }
}

// Generic function over any Collection
fn sum_all<C: Collection<Item = i32>>(col: &C) -> i32 {
    col.iter().copied().sum()
}
```

### Type-Parameterized Associated Types

```rust
trait Deserializer {
    type Error;
    // GAT with type parameter — output type depends on what you're deserializing into
    type Output<T: serde::de::DeserializeOwned>;

    fn deserialize<T: serde::de::DeserializeOwned>(
        &self,
        input: &[u8],
    ) -> Result<Self::Output<T>, Self::Error>;
}

// A deserializer that wraps results in a timestamped envelope
struct TimestampedDeserializer;

struct Timestamped<T> {
    value: T,
    deserialized_at: std::time::Instant,
}

impl Deserializer for TimestampedDeserializer {
    type Error = serde_json::Error;
    type Output<T: serde::de::DeserializeOwned> = Timestamped<T>;

    fn deserialize<T: serde::de::DeserializeOwned>(
        &self,
        input: &[u8],
    ) -> Result<Timestamped<T>, Self::Error> {
        let value: T = serde_json::from_slice(input)?;
        Ok(Timestamped { value, deserialized_at: std::time::Instant::now() })
    }
}
```

### Async Trait with GATs

```rust
trait AsyncIterator {
    type Item<'a> where Self: 'a;

    // Return a future whose lifetime is tied to &mut self
    fn next(&mut self) -> impl Future<Output = Option<Self::Item<'_>>>;
}
```

## Const Generics

Parameterize types and functions by compile-time constant values. Stable for integers, `bool`, and `char` since Rust 1.51.

### Fixed-Size Arrays

```rust
// Generic over array size
struct Matrix<const ROWS: usize, const COLS: usize> {
    data: [[f64; COLS]; ROWS],
}

impl<const ROWS: usize, const COLS: usize> Matrix<ROWS, COLS> {
    fn new() -> Self {
        Self { data: [[0.0; COLS]; ROWS] }
    }

    fn get(&self, row: usize, col: usize) -> f64 {
        self.data[row][col]
    }

    fn set(&mut self, row: usize, col: usize, value: f64) {
        self.data[row][col] = value;
    }

    // Transpose — note how dimensions swap in the return type
    fn transpose(&self) -> Matrix<COLS, ROWS> {
        let mut result = Matrix::<COLS, ROWS>::new();
        for r in 0..ROWS {
            for c in 0..COLS {
                result.data[c][r] = self.data[r][c];
            }
        }
        result
    }
}

// Matrix multiplication — dimensions must be compatible at compile time
impl<const M: usize, const N: usize> Matrix<M, N> {
    fn multiply<const P: usize>(&self, other: &Matrix<N, P>) -> Matrix<M, P> {
        let mut result = Matrix::<M, P>::new();
        for i in 0..M {
            for j in 0..P {
                let mut sum = 0.0;
                for k in 0..N {
                    sum += self.data[i][k] * other.data[k][j];
                }
                result.data[i][j] = sum;
            }
        }
        result
    }
}

fn usage() {
    let a = Matrix::<2, 3>::new();  // 2×3
    let b = Matrix::<3, 4>::new();  // 3×4
    let c = a.multiply(&b);         // 2×4 — correct!

    // Won't compile — incompatible dimensions:
    // let d = Matrix::<2, 3>::new();
    // a.multiply(&d);  // Error: expected Matrix<3, _>, got Matrix<2, 3>
}
```

### Fixed-Capacity Buffer

```rust
struct FixedBuf<T, const CAP: usize> {
    data: [Option<T>; CAP],
    len: usize,
}

impl<T, const CAP: usize> FixedBuf<T, CAP> {
    fn new() -> Self
    where
        T: Copy,
    {
        Self { data: [None; CAP], len: 0 }
    }

    fn push(&mut self, value: T) -> Result<(), T> {
        if self.len >= CAP {
            return Err(value);
        }
        self.data[self.len] = Some(value);
        self.len += 1;
        Ok(())
    }

    fn pop(&mut self) -> Option<T> {
        if self.len == 0 {
            return None;
        }
        self.len -= 1;
        self.data[self.len].take()
    }

    fn len(&self) -> usize { self.len }
    fn capacity(&self) -> usize { CAP }
    fn is_full(&self) -> bool { self.len == CAP }
}

fn usage() {
    let mut buf = FixedBuf::<u8, 64>::new();
    buf.push(42).unwrap();
}
```

### Default Const Generic Values

```rust
struct RingBuffer<T, const N: usize = 256> {
    data: Vec<Option<T>>,
    head: usize,
    tail: usize,
}

impl<T, const N: usize> RingBuffer<T, N> {
    fn new() -> Self {
        Self {
            data: (0..N).map(|_| None).collect(),
            head: 0,
            tail: 0,
        }
    }

    fn push(&mut self, value: T) {
        self.data[self.tail] = Some(value);
        self.tail = (self.tail + 1) % N;
        if self.tail == self.head {
            self.head = (self.head + 1) % N;  // Overwrite oldest
        }
    }
}

fn usage() {
    let mut buf = RingBuffer::<u8>::new();       // Default 256
    let mut big = RingBuffer::<u8, 4096>::new(); // Custom size
}
```

### Const Generic Expressions (Nightly)

Some const expressions require nightly Rust:

```rust
#![feature(generic_const_exprs)]

// Padding to alignment boundary
struct Aligned<T, const ALIGN: usize>
where
    [(); ALIGN - std::mem::size_of::<T>() % ALIGN]:,  // Requires nightly
{
    value: T,
    _padding: [u8; ALIGN - std::mem::size_of::<T>() % ALIGN],
}
```

### Const Generics with Traits

```rust
// Stable alternative — use const generic directly
struct StablePoint<const N: usize> {
    coords: [f64; N],
}

type Point2D = StablePoint<2>;
type Point3D = StablePoint<3>;

impl<const N: usize> StablePoint<N> {
    fn distance(&self, other: &Self) -> f64 {
        self.coords.iter()
            .zip(other.coords.iter())
            .map(|(a, b)| (a - b).powi(2))
            .sum::<f64>()
            .sqrt()
    }
}
```

## Pin and Unpin

### Why Pin Exists

Rust futures are state machines that hold references across `.await` points. If a future is moved in memory after creation, those internal references would dangle. `Pin` prevents moving.

```rust
// Conceptual: what the compiler generates for an async block
async fn example() {
    let data = vec![1, 2, 3];
    let reference = &data;      // reference points to data
    some_async_op().await;      // ← suspend point
    println!("{reference:?}");  // reference must still be valid after resume
}

// The generated future struct looks roughly like:
enum ExampleFuture {
    // Before first .await
    State0 { data: Vec<i32>, reference: *const Vec<i32> },
    // After first .await
    State1 { data: Vec<i32>, reference: *const Vec<i32> },
    Complete,
}
// If this struct is moved, `reference` would point to the OLD location of `data`.
// Pin prevents this move.
```

### Pin<P> and Unpin

```rust
use std::pin::Pin;
use std::marker::Unpin;

// Pin<P> wraps a pointer P and guarantees the pointee won't be moved.
// Pin<&mut T> — pinned mutable reference
// Pin<Box<T>> — pinned heap allocation

// Unpin is an auto-trait: "this type is safe to move even when pinned"
// Most types are Unpin (i32, String, Vec, etc.)
// Self-referential types (like futures) are !Unpin

// If T: Unpin, Pin<&mut T> is equivalent to &mut T — no restriction
fn move_unpin<T: Unpin>(pinned: Pin<&mut T>) -> &mut T {
    // Pin::get_mut is safe for Unpin types
    Pin::get_mut(pinned)
}

// If T: !Unpin, you can only access it through Pin — can't get &mut T safely
fn access_not_unpin<T>(pinned: Pin<&mut T>) {
    // Pin::get_mut requires T: Unpin — won't compile for !Unpin types
    // Pin::get_unchecked_mut exists but is unsafe
}
```

### Creating Pinned Values

```rust
use std::pin::{Pin, pin};

// Stack pinning (Rust 1.68+)
let future = async { 42 };
let pinned = pin!(future);  // Pin<&mut impl Future<Output = i32>>
// `pinned` cannot be moved — it's pinned to the stack frame

// Heap pinning
let boxed: Pin<Box<dyn Future<Output = i32>>> = Box::pin(async { 42 });
// The future is on the heap and will never move

// Before pin! macro (manual stack pinning, rarely needed)
let mut future = async { 42 };
// SAFETY: we won't move `future` after this point
let pinned = unsafe { Pin::new_unchecked(&mut future) };
```

### Implementing Future Manually

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct Delay {
    deadline: std::time::Instant,
}

impl Delay {
    fn new(duration: std::time::Duration) -> Self {
        Self { deadline: std::time::Instant::now() + duration }
    }
}

impl Future for Delay {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if std::time::Instant::now() >= self.deadline {
            Poll::Ready(())
        } else {
            // In real code, register a waker with the runtime timer
            let waker = cx.waker().clone();
            let deadline = self.deadline;
            std::thread::spawn(move || {
                std::thread::sleep(deadline - std::time::Instant::now());
                waker.wake();
            });
            Poll::Pending
        }
    }
}

// Delay is Unpin because it has no self-referential fields.
// Most manually implemented futures are Unpin.
// Compiler-generated futures (from async blocks) are usually !Unpin.
```

### Pin Projection

Accessing fields of a pinned struct:

```rust
use std::pin::Pin;

// With pin-project-lite (preferred — declarative macro, zero deps, used by tokio):
use pin_project_lite::pin_project;

pin_project! {
    struct TwoFutures<A, B> {
        #[pin]   // This field is pinned when the struct is pinned
        future_a: A,
        #[pin]
        future_b: B,
        ready_a: bool,  // Not pinned — can be moved freely
        ready_b: bool,
    }
}

impl<A: Future, B: Future> Future for TwoFutures<A, B> {
    type Output = (A::Output, B::Output);

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.project();  // Safe pin projection

        // this.future_a: Pin<&mut A>  — pinned access
        // this.future_b: Pin<&mut B>  — pinned access
        // this.ready_a: &mut bool     — unpinned access

        // ... poll both futures
        todo!()
    }
}
```

### Enforcing !Unpin

```rust
use pin_project_lite::pin_project;

pin_project! {
    // #[project(!Unpin)] ensures this struct is !Unpin even if all fields are Unpin
    #[project(!Unpin)]
    struct SelfReferential {
        #[pin]
        data: Vec<u8>,
        // Imagine a pointer into `data` here
    }
}
```

### PinnedDrop

Custom drop logic when a value is pinned (e.g., deregistering from a reactor):

```rust
use pin_project_lite::pin_project;

pin_project! {
    struct TimerEntry {
        #[pin]
        state: TimerState,
        registered: bool,
    }

    impl PinnedDrop for TimerEntry {
        fn drop(this: Pin<&mut Self>) {
            let this = this.project();
            if *this.registered {
                // Deregister from timer wheel before drop
                deregister_timer(this.state);
            }
        }
    }
}
# struct TimerState;
# fn deregister_timer(_: Pin<&mut TimerState>) {}
```

### Enum Projection

```rust
use pin_project_lite::pin_project;

pin_project! {
    #[project = StateProj]  // Name the projection type
    enum State<F1, F2> {
        First { #[pin] fut: F1 },
        Second { #[pin] fut: F2 },
        Done,
    }
}

// Usage in poll:
fn poll_state<F1: Future, F2: Future>(
    state: Pin<&mut State<F1, F2>>,
    cx: &mut Context<'_>,
) {
    match state.project() {
        StateProj::First { fut } => { /* fut: Pin<&mut F1> */ }
        StateProj::Second { fut } => { /* fut: Pin<&mut F2> */ }
        StateProj::Done => {}
    }
}
```

### pin-project vs pin-project-lite

| Feature | `pin-project-lite` | `pin-project` |
|---------|-------------------|---------------|
| Macro type | Declarative (`macro_rules!`) | Procedural |
| Dependencies | Zero | `syn`, `quote`, `proc-macro2` |
| Compile time | Faster | Slower |
| Tuple structs | Not supported | Supported |
| Error messages | Basic | Detailed |
| `!Unpin` enforcement | `#[project(!Unpin)]` | `UnsafeUnpin` |
| Used by | tokio, hyper, tower | Any |

**Prefer `pin-project-lite`** unless you need tuple structs or custom `Unpin` logic.

### Pin Summary

| Type | Unpin? | Pin behavior |
|------|--------|-------------|
| `i32`, `String`, `Vec<T>` | Yes | `Pin<&mut T>` ≡ `&mut T`, no restriction |
| `async {}` blocks | No | Must be pinned before polling, can't move |
| Manual `impl Future` (no self-ref) | Yes | Usually Unpin, Pin is transparent |
| Self-referential structs | No | Must use Pin, use `pin-project-lite` for field access |

**Rule of thumb:** You rarely need to think about Pin unless you're implementing `Future` manually or building async primitives. `tokio::spawn`, `tokio::select!`, and `.await` handle pinning for you.

## Native Async Traits vs async-trait Crate

### Native Async Traits (Stable since Rust 1.75)

Rust supports `async fn` in traits natively using Return Position Impl Trait in Traits (RPITIT):

```rust
trait Database {
    async fn get(&self, id: i64) -> Option<Record>;
    async fn insert(&self, record: &Record) -> Result<i64, DbError>;
}

struct PgDatabase { pool: sqlx::PgPool }

impl Database for PgDatabase {
    async fn get(&self, id: i64) -> Option<Record> {
        sqlx::query_as("SELECT * FROM records WHERE id = $1")
            .bind(id)
            .fetch_optional(&self.pool)
            .await
            .ok()
            .flatten()
    }

    async fn insert(&self, record: &Record) -> Result<i64, DbError> {
        // ...
        todo!()
    }
}
# struct Record;
# struct DbError;
```

### Limitations of Native Async Traits

**Cannot use `dyn Trait` directly:**

```rust
trait Database {
    async fn get(&self, id: i64) -> Option<Record>;
}

// Won't compile — async trait methods return opaque types,
// can't be made into trait objects:
// fn take_db(db: &dyn Database) { ... }

// Workaround 1: Use generics (monomorphization)
fn take_db(db: &impl Database) { /* works */ }

// Workaround 2: Use trait_variant for object-safe version
```

**Send bound issue:**

```rust
trait Service {
    async fn process(&self) -> Result<(), Error>;
}

// The returned future may or may not be Send, depending on the impl.
// If you need Send (for tokio::spawn), you must bound it:

// Option A: Use #[trait_variant] (from trait_variant crate)
#[trait_variant::make(SendService: Send)]
trait Service {
    async fn process(&self) -> Result<(), Error>;
}
// This generates both Service (non-Send) and SendService (Send)

// Option B: Explicit desugaring
trait Service {
    fn process(&self) -> impl Future<Output = Result<(), Error>> + Send + '_;
}
# struct Error;
```

### When to Use async-trait Crate

```rust
// The async-trait crate boxes the future, enabling dyn dispatch:
use async_trait::async_trait;

#[async_trait]
trait Database: Send + Sync {
    async fn get(&self, id: i64) -> Option<Record>;
}

// Now works as trait object:
async fn query(db: &dyn Database) -> Option<Record> {
    db.get(42).await
}
```

**Trade-off:**

| Feature | Native async trait | async-trait crate |
|---------|-------------------|-------------------|
| Heap allocation | None | Box per call |
| `dyn Trait` support | No (without workarounds) | Yes |
| Send bounds | Manual | Automatic (Send by default) |
| Compile time | Faster | Slower (proc macro) |
| Syntax | `async fn` | `#[async_trait]` + `async fn` |

**Guidance:**
- Use **native async traits** when you use `impl Trait` (generics) at call sites — zero overhead
- Use **async-trait crate** when you need `dyn Trait` (trait objects) — e.g., plugin systems, dynamic dispatch
- Use **trait_variant** when you need both generic and dyn-compatible versions

### RPITIT (Return Position Impl Trait in Traits)

Native async traits are syntactic sugar for RPITIT:

```rust
// These are equivalent:
trait Processor {
    async fn process(&self, data: &[u8]) -> Vec<u8>;
}

trait Processor {
    fn process(&self, data: &[u8]) -> impl Future<Output = Vec<u8>> + '_;
}

// RPITIT also works for non-async return types:
trait Parser {
    fn parse(&self, input: &str) -> impl Iterator<Item = Token> + '_;
}

struct JsonParser;
impl Parser for JsonParser {
    fn parse(&self, input: &str) -> impl Iterator<Item = Token> + '_ {
        input.split(',').map(|s| Token(s.trim().to_string()))
    }
}
# struct Token(String);
```

### Rust 2024 RPIT Lifetime Capture Rules

In Rust 2024 edition, RPIT (return-position `impl Trait`) automatically captures all in-scope type and lifetime parameters. This eliminates the `Captures` workaround needed in Rust 2021:

```rust
// Rust 2021 — bare lifetime 'a is NOT automatically captured by impl Trait
// This fails in 2021:
fn foo<'a>(x: &'a str) -> impl Sized { x }

// Rust 2021 workaround — the Captures trick:
trait Captures<U> {}
impl<T: ?Sized, U> Captures<U> for T {}
fn foo<'a>(x: &'a str) -> impl Sized + Captures<&'a ()> { x }

// Rust 2024 — ALL lifetimes and types are captured automatically:
fn foo<'a>(x: &'a str) -> impl Sized { x }  // Just works

// If you need to opt OUT of capturing a lifetime in 2024 edition,
// use explicit `use<>` syntax to list exactly what to capture:
fn bar<'a, T>(x: &'a T) -> impl Sized + use<T> { /* captures T but not 'a */ }
```

**Practical impact:** Converting between `async fn` and RPIT is simpler in 2024 — `async fn foo(&self) -> T` and `fn foo(&self) -> impl Future<Output = T> + '_` now behave identically with respect to lifetime capture.

## Lifetime Patterns

### Lifetime Elision Rules

The compiler infers lifetimes automatically in most cases:

```rust
// Rule 1: Each reference parameter gets its own lifetime
fn foo(x: &str, y: &str) -> ...
// becomes: fn foo<'a, 'b>(x: &'a str, y: &'b str) -> ...

// Rule 2: If exactly one input lifetime, output gets it
fn first_word(s: &str) -> &str { ... }
// becomes: fn first_word<'a>(s: &'a str) -> &'a str { ... }

// Rule 3: If &self or &mut self exists, output gets self's lifetime
impl Parser {
    fn parse(&self, input: &str) -> &Token { ... }
    // becomes: fn parse<'a, 'b>(&'a self, input: &'b str) -> &'a Token
}

// When elision fails — must annotate explicitly
fn longest(x: &str, y: &str) -> &str { ... }  // ERROR: which lifetime?
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str { ... }  // OK
```

### Struct Lifetimes

```rust
// A struct holding references must declare lifetimes
struct Request<'a> {
    url: &'a str,
    headers: &'a [Header],
}

// The struct can't outlive the data it borrows
fn process() {
    let url = String::from("https://example.com");
    let req = Request { url: &url, headers: &[] };
    // req.url is valid as long as `url` exists
    println!("{}", req.url);
}  // url dropped here, req would be invalid after this

// Multiple lifetimes — when fields borrow from different sources
struct Comparison<'a, 'b> {
    original: &'a str,
    modified: &'b str,
}

// Bound: output lives at least as long as the shorter input
impl<'a, 'b> Comparison<'a, 'b> {
    fn diff(&self) -> &str {
        if self.original == self.modified { "same" } else { "different" }
    }
}
```

### 'static — What It Really Means

```rust
// 'static has TWO meanings depending on context:

// 1. &'static T — a reference that lives for the entire program
let s: &'static str = "I live forever";  // String literal

// Leaked values are also 'static
let leaked: &'static Vec<i32> = Box::leak(Box::new(vec![1, 2, 3]));

// 2. T: 'static — T owns all its data (contains no borrowed references)
// This does NOT mean T lives forever — it means T is self-contained
fn spawn<T: Send + 'static>(val: T) { /* ... */ }

// These are 'static (they own their data):
spawn(String::from("owned"));        // String: 'static ✓
spawn(42i32);                         // i32: 'static ✓
spawn(vec![1, 2, 3]);                // Vec<i32>: 'static ✓
spawn(Arc::new(Mutex::new(data)));   // Arc<Mutex<T>>: 'static ✓

// These are NOT 'static (they borrow data):
let local = String::from("hello");
// spawn(&local);                     // &String: NOT 'static ✗

// Common misconception: 'static does NOT mean "never deallocated"
{
    let s = String::from("hello");   // String: 'static
}  // s is deallocated here! 'static just means "no borrowed refs"
```

### Lifetime Variance

Variance determines how lifetimes relate in subtyping. A longer lifetime can be used where a shorter one is expected:

```rust
// 'long: 'short means 'long outlives 'short
// &'long T can be used where &'short T is expected (covariant)

fn demonstrate<'long, 'short>(long_ref: &'long str, short_ref: &'short str)
where
    'long: 'short,  // 'long outlives 'short
{
    // Can pass &'long str where &'short str expected
    let _: &'short str = long_ref;  // OK: longer lifetime is a subtype
    // let _: &'long str = short_ref;  // ERROR: can't extend a lifetime
}

// Variance types:
// Covariant (most references): &'a T — longer lifetime → shorter is OK
// Invariant (&'a mut T):       can't substitute different lifetimes
// This is why you can't reborrow &'a mut T as &'b mut T where 'a ≠ 'b
```

### PhantomData Uses

```rust
use std::marker::PhantomData;

// 1. Type state (most common) — see Type State Pattern section
struct Builder<State> {
    _state: PhantomData<State>,
}

// 2. Lifetime binding — struct logically "borrows" but doesn't store a reference
struct Iter<'a, T> {
    ptr: *const T,
    end: *const T,
    _marker: PhantomData<&'a T>,  // Tells compiler: acts like it borrows &'a T
}

// 3. Ownership marker — struct logically "owns" T even without storing one
struct MyAllocator<T> {
    ptr: *mut u8,
    _owns: PhantomData<T>,  // Tells compiler: dropping this may drop a T
}

// 4. Variance control
struct ContravariantLifetime<'a> {
    _marker: PhantomData<fn(&'a ())>,  // Contravariant in 'a
}

// 5. Send/Sync control
struct NotSend {
    _marker: PhantomData<*const ()>,  // *const is !Send, making struct !Send
}

// 6. Blanket impl adapter (serde pattern)
// DeserializeSeed lets you pass state into deserialization.
// PhantomData<T> implements it for any Deserialize type, acting as
// a zero-cost adapter from stateful to stateless deserialization.
trait DeserializeSeed<'de> {
    type Value;
    fn deserialize<D: Deserializer<'de>>(self, deserializer: D) -> Result<Self::Value, D::Error>;
}

// PhantomData<T> carries no state but satisfies the trait interface
impl<'de, T: Deserialize<'de>> DeserializeSeed<'de> for PhantomData<T> {
    type Value = T;
    fn deserialize<D: Deserializer<'de>>(self, deserializer: D) -> Result<T, D::Error> {
        T::deserialize(deserializer)  // Delegates — PhantomData carries nothing
    }
}
// This pattern: use PhantomData<T> as a blanket "no-state" impl
// for traits that optionally carry state
```

### Higher-Rank Trait Bounds (HRTBs)

```rust
// for<'a> means "for ALL possible lifetimes"
fn apply<F>(f: F) where F: for<'a> Fn(&'a str) -> &'a str {
    let owned = String::from("hello");
    println!("{}", f(&owned));      // Works with temporary
    println!("{}", f("static"));    // Works with 'static
}

// Common in closure-accepting APIs and trait objects
fn get_processor() -> Box<dyn for<'a> Fn(&'a str) -> usize> {
    Box::new(|s| s.len())
}

// Most common implicit use: Fn(&T) desugars to for<'a> Fn(&'a T)
fn filter_names(names: &[String], pred: impl Fn(&str) -> bool) -> Vec<&String> {
    // The Fn(&str) bound is actually for<'a> Fn(&'a str) -> bool
    names.iter().filter(|n| pred(n)).collect()
}

// Real-world pattern from serde: DeserializeOwned
// Problem: Deserialize<'de> borrows from input. But sometimes you need
// a type that can be deserialized from ANY input, regardless of lifetime.
//
// Solution: HRTB — "works for all possible lifetimes"
trait DeserializeOwned: for<'de> Deserialize<'de> {}

// Blanket impl: any type that works for all lifetimes is "owned"
impl<T> DeserializeOwned for T where T: for<'de> Deserialize<'de> {}

// Now use DeserializeOwned as a clean bound:
fn from_json<T: DeserializeOwned>(json: &str) -> Result<T, Error> {
    serde_json::from_str(json)  // T doesn't borrow from json
}

// vs Deserialize<'de> when you CAN borrow from input:
fn from_json_borrowed<'de, T: Deserialize<'de>>(json: &'de str) -> Result<T, Error> {
    serde_json::from_str(json)  // T may contain &'de str fields
}
// This HRTB + blanket impl + trait alias pattern is reusable:
// "Type that satisfies Trait for all lifetimes" = "Type that owns all its data"
```

## Combining Patterns

These patterns compose well together:

```rust
use std::marker::PhantomData;

mod sealed { pub trait Sealed {} }

// Sealed trait + const generics + type state
pub trait FixedPoint: sealed::Sealed {
    const DECIMAL_PLACES: u32;
}

pub struct USD;
impl sealed::Sealed for USD {}
impl FixedPoint for USD { const DECIMAL_PLACES: u32 = 2; }

pub struct BTC;
impl sealed::Sealed for BTC {}
impl FixedPoint for BTC { const DECIMAL_PLACES: u32 = 8; }

pub struct Amount<C: FixedPoint> {
    // Store as smallest unit (cents for USD, satoshis for BTC)
    raw: i64,
    _currency: PhantomData<C>,
}

impl<C: FixedPoint> Amount<C> {
    pub fn new(raw: i64) -> Self {
        Self { raw, _currency: PhantomData }
    }

    pub fn from_float(value: f64) -> Self {
        let factor = 10_i64.pow(C::DECIMAL_PLACES);
        Self::new((value * factor as f64).round() as i64)
    }

    pub fn to_float(&self) -> f64 {
        let factor = 10_i64.pow(C::DECIMAL_PLACES);
        self.raw as f64 / factor as f64
    }
}

// Can add USD amounts, but can't add USD + BTC — different types
impl<C: FixedPoint> std::ops::Add for Amount<C> {
    type Output = Self;
    fn add(self, rhs: Self) -> Self {
        Self::new(self.raw + rhs.raw)
    }
}

fn usage() {
    let price = Amount::<USD>::from_float(19.99);
    let tax = Amount::<USD>::from_float(1.60);
    let total = price + tax;  // OK: same currency

    let btc = Amount::<BTC>::from_float(0.00042);
    // price + btc;  // Won't compile: mismatched types Amount<USD> vs Amount<BTC>
}
```

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: ownership, traits basics, error handling, iterators, pattern matching
- **[language-patterns.md](language-patterns.md)** — Everyday idioms: pattern matching extended, ownership patterns, closures, RAII
- **[async-concurrency.md](async-concurrency.md)** — Async runtime, tokio, channels, concurrent patterns
- **[architecture.md](architecture.md)** — Workspace design, DI, application layering
- **[macros.md](macros.md)** — Declarative and procedural macros
