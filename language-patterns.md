# Rust Language Patterns

Extended coverage of everyday Rust idioms. SKILL.md covers essentials; this file goes deeper on pattern matching, ownership strategies, iterator composition, closures, traits, conversions, RAII, modules, and production patterns.

## Rules for Language Patterns (LLM)

1. **ALWAYS use `let-else` for guard clauses** — `let Some(x) = val else { return Err(...) };` is cleaner than nested `match` or `if let` with early return
2. **NEVER move out of a collection element** — use `.remove()`, `.swap_remove()`, or `std::mem::take()` instead of indexing and moving which won't compile
3. **ALWAYS use the entry API for HashMap insert-or-update** — `map.entry(key).or_insert_with(|| ...)` avoids double lookup; never use `if !map.contains_key() { map.insert() }`
4. **PREFER `Cow<str>` over `String` for function parameters that might not need allocation** — accepts both `&str` and `String`, only allocates when mutation is needed
5. **NEVER implement `Deref` for smart-pointer-like behavior on domain types** — `Deref` is for smart pointers (`Box`, `Arc`); use explicit methods or `AsRef` for domain type conversions
6. **ALWAYS chain `?` with `.map_err()` or `.context()` for error propagation** — bare `?` loses context about what operation failed; add context at every boundary
7. **PREFER `impl IntoIterator<Item = T>` over `&[T]` for function parameters** — accepts `Vec`, slices, arrays, ranges, and any iterator; more flexible API
8. **ALWAYS use `#[must_use]` on functions that return values which should not be ignored** — prevents silent bugs where a `Result` or computed value is accidentally dropped

### Common Mistakes (BAD/GOOD)

**Double HashMap lookup:**
```rust
// BAD: two hash lookups
if !scores.contains_key(&name) {
    scores.insert(name.clone(), 0);
}
*scores.get_mut(&name).unwrap() += 1;
```

```rust
// GOOD: single lookup with entry API
*scores.entry(name).or_insert(0) += 1;
```

**Unnecessary String allocation:**
```rust
// BAD: always allocates, even when input is already &str
fn greet(name: String) {
    println!("Hello, {name}");
}
greet("world".to_string());  // Forced allocation
```

```rust
// GOOD: accepts &str or String without allocation
fn greet(name: &str) {
    println!("Hello, {name}");
}
greet("world");       // No allocation
greet(&my_string);    // Borrows existing String
```

**Bare ? without context:**
```rust
// BAD: error just says "No such file" — which file?
fn load_config(path: &str) -> Result<Config> {
    let data = std::fs::read_to_string(path)?;
    Ok(toml::from_str(&data)?)
}
```

```rust
// GOOD: error says "failed to read config from /etc/app.toml: No such file"
fn load_config(path: &str) -> Result<Config> {
    let data = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config from {path}"))?;
    toml::from_str(&data)
        .context("failed to parse config TOML")?
}
```

**Iterator collect then re-iterate:**
```rust
// BAD: collects into Vec just to iterate again
let names: Vec<String> = users.iter().map(|u| u.name.clone()).collect();
let upper: Vec<String> = names.iter().map(|n| n.to_uppercase()).collect();
```

```rust
// GOOD: chain iterators lazily, collect once
let upper: Vec<String> = users.iter()
    .map(|u| u.name.to_uppercase())
    .collect();
```

**Misusing Deref for inheritance:**
```rust
// BAD: Deref to simulate inheritance — confuses method resolution
struct Admin(User);
impl Deref for Admin { type Target = User; fn deref(&self) -> &User { &self.0 } }
```

```rust
// GOOD: explicit delegation or AsRef
struct Admin { user: User, permissions: Vec<Permission> }
impl Admin {
    fn name(&self) -> &str { &self.user.name }
}
impl AsRef<User> for Admin {
    fn as_ref(&self) -> &User { &self.user }
}
```

### Section Index

| Section | Topics |
|---------|--------|
| [Pattern Matching (Extended)](#pattern-matching-extended) | Match ergonomics, exhaustive matching, if-let chains, let-else, matches!, destructuring, or-patterns, tuple matching (smoltcp), range patterns (rustc lexer), ref/ref mut (tokio), const matching (SLIP protocol), BAD/GOOD mistakes |
| [Ownership & Borrowing Patterns](#ownership--borrowing-patterns) | Borrow splitting, temporary borrow scoping, Cow\<T\>, zero-copy, entry API |
| [The ? Operator & Error Chains](#the--operator--error-chains) | ? chaining, adding context, error conversion across layers, fallible iterators |
| [Iterator Composition](#iterator-composition-extended) | Adaptor chaining, custom iterators, IntoIterator, lazy evaluation |
| [Closure Capture Semantics](#closure-capture-semantics) | What closures capture, move closures, Fn/FnMut/FnOnce hierarchy |
| [Trait Patterns](#trait-patterns) | Extension traits, blanket impls, orphan rule, trait objects, supertraits |
| [From/Into/AsRef Conversions](#fromintoasref-conversions) | Conversion hierarchy, impl Into\<T\>, TryFrom, AsRef/AsMut, Deref coercion |
| [RAII, Drop & Resource Management](#raii-drop--resource-management) | Automatic cleanup, guard pattern, ManuallyDrop |
| [Module Organization & Visibility](#module-organization--visibility) | Visibility modifiers, module layout, re-exports, doc attributes |
| [Conditional Compilation](#conditional-compilation) | cfg attributes, runtime cfg, conditional macros, test vs production lints |
| [Common Macro Invocation Patterns](#common-macro-invocation-patterns) | vec!/format!/write!, include_str!/env!, assert!/todo! |
| [Production Patterns](#production-patterns) | Config loading, graceful shutdown, retry backoff, middleware, type-safe IDs, Visitor |

## Pattern Matching (Extended)

### Match Ergonomics (Auto-Ref)

```rust
// Rust automatically adds & in match patterns for references
let value = &Some(42);

// Without ergonomics — verbose
match value {
    &Some(ref n) => println!("{n}"),
    &None => println!("none"),
}

// With match ergonomics — Rust inserts & and ref for you
match value {
    Some(n) => println!("{n}"),  // n: &i32, not i32
    None => println!("none"),
}

// Same for nested references
let data: &Vec<String> = &vec!["hello".into()];
match data.first() {
    Some(s) => println!("{s}"),  // s: &String
    None => {}
}
```

### Exhaustive Matching Strategies

```rust
#[non_exhaustive]
enum ApiError {
    NotFound,
    Unauthorized,
    RateLimited,
}

// Must have wildcard for #[non_exhaustive] enums from external crates
match api_error {
    ApiError::NotFound => retry_with_backoff(),
    ApiError::Unauthorized => refresh_token(),
    ApiError::RateLimited => sleep_and_retry(),
    _ => log_unknown_error(),  // Required — future variants may be added
}

// Internal enums — prefer exhaustive matching (no wildcard)
// This way the compiler tells you when you add a variant
enum State { Running, Paused, Stopped }
match state {
    State::Running => tick(),
    State::Paused => wait(),
    State::Stopped => cleanup(),
    // No _ — adding a variant forces handling here
}
```

### if-let Chains (Rust 2024 Edition)

```rust
// Chain multiple patterns with &&
if let Some(user) = get_user(id)
    && let Some(email) = user.email.as_ref()
    && email.ends_with("@company.com")
{
    send_internal_notification(email);
}

// Equivalent to nested if-let (but much cleaner)
if let Some(user) = get_user(id) {
    if let Some(email) = user.email.as_ref() {
        if email.ends_with("@company.com") {
            send_internal_notification(email);
        }
    }
}
```

### let-else for Early Returns

```rust
// let-else — bind or diverge (return, break, continue, panic)
fn process(input: &str) -> Result<Output, Error> {
    let Some(header) = input.lines().next() else {
        return Err(Error::EmptyInput);
    };

    let Ok(config) = parse_header(header) else {
        return Err(Error::InvalidHeader);
    };

    let Some(value) = config.get("key") else {
        return Err(Error::MissingKey("key"));
    };

    // header, config, value all available here — no nesting
    Ok(Output::new(value))
}

// Compare with match/if-let — let-else keeps the happy path flat
```

### matches! Macro

```rust
// Returns bool — useful in filter/any/all contexts
let is_digit = matches!(ch, '0'..='9');
let is_keyword = matches!(word, "if" | "else" | "for" | "while" | "loop" | "match");

// With guards
let is_small_positive = matches!(n, x if x > 0 && x < 100);

// In iterators
let has_errors = results.iter().any(|r| matches!(r, Err(_)));
let errors: Vec<_> = results.iter()
    .filter(|r| matches!(r, Err(_)))
    .collect();

// Matching nested patterns
let is_ok_and_even = matches!(result, Ok(n) if n % 2 == 0);
let is_some_string = matches!(opt, Some(s) if !s.is_empty());
```

### Destructuring Complex Types

```rust
// Struct destructuring with rename
let Point { x: px, y: py } = point;

// Nested destructuring
let Config {
    database: DbConfig { host, port, .. },
    server: ServerConfig { bind_addr, .. },
    ..
} = config;

// Tuple struct in match
match command {
    Command::SetVolume(level) if level > 100 => Err(Error::TooLoud),
    Command::SetVolume(level) => set_volume(level),
    Command::Mute => set_volume(0),
}

// Slice patterns
match data.as_slice() {
    [] => println!("empty"),
    [single] => println!("one: {single}"),
    [first, .., last] => println!("range: {first}..{last}"),
}

// Reference patterns in function parameters
fn process_pair(&(ref name, ref value): &(String, i32)) {
    println!("{name}: {value}");
}
```

### Or-Patterns with Bindings

```rust
// Same binding name in each alternative — must bind same type
match event {
    Event::Click { x, y } | Event::Touch { x, y } => handle_input(x, y),
    Event::Key { code } | Event::GamepadButton { code } => handle_button(code),
    _ => {}
}

// In let statements
let (Ok(value) | Err(value)) = result_that_returns_same_type;
```

### Match on References Without Moving

```rust
// Common gotcha: matching on owned value in a reference context
let items: Vec<String> = vec!["hello".into(), "world".into()];

// BAD: tries to move out of Vec
// for item in &items {
//     match item {
//         s if s.len() > 3 => println!("long: {s}"),
//         _ => {}
//     }
// }

// GOOD: match on reference (auto-ref handles this, but explicit is clearer)
for item in &items {
    match item.as_str() {
        s if s.len() > 3 => println!("long: {s}"),
        _ => {}
    }
}

// GOOD: match on borrowed value
match &some_string {
    s if s.starts_with("http") => fetch_url(s),
    s => read_file(s),
}
```

### Tuple Matching for Multi-Value Dispatch

Match on tuples of values for state machines, protocol handlers, and multi-value dispatch. Production pattern from smoltcp (TCP/IP stack) and ripgrep:

```rust
// State machine — match (state, event) pairs
// Pattern: smoltcp TCP socket processes (state, control_flag) pairs
enum State { Idle, Connecting, Connected, Closing }
enum Event { Connect, Data(Vec<u8>), Disconnect, Timeout }

fn transition(state: State, event: Event) -> (State, Vec<Action>) {
    match (state, event) {
        (State::Idle, Event::Connect) => {
            (State::Connecting, vec![Action::SendSyn])
        }
        (State::Connecting, Event::Data(_)) => {
            (State::Connected, vec![Action::SendAck])
        }
        (State::Connected, Event::Data(payload)) => {
            (State::Connected, vec![Action::Process(payload), Action::SendAck])
        }
        (State::Connected, Event::Disconnect) => {
            (State::Closing, vec![Action::SendFin])
        }
        // Wildcard for any state receiving Timeout
        (_, Event::Timeout) => {
            (State::Idle, vec![Action::Reset])
        }
        // No-op transitions
        (state, _) => (state, vec![]),
    }
}

// Matching on two Option values — ripgrep sort ordering pattern
// Source: ripgrep/crates/core/flags/hiargs.rs
fn compare_optional(a: Option<u64>, b: Option<u64>) -> Ordering {
    match (a, b) {
        (Some(a), Some(b)) => a.cmp(&b),
        (Some(_), None) => Ordering::Less,     // Present before absent
        (None, Some(_)) => Ordering::Greater,
        (None, None) => Ordering::Equal,
    }
}

// Triple match — smoltcp matches (state, control, ack) in TCP processing
// Source: smoltcp/src/socket/tcp.rs
match (self.state, repr.control, repr.ack_number) {
    (State::SynSent, TcpControl::Rst, None) => { /* reject */ }
    (State::SynSent, TcpControl::Rst, Some(ack)) if ack == expected => { /* accept */ }
    (_, TcpControl::Rst, _) => { /* any RST with valid seq */ }
    (State::Listen, TcpControl::Syn, None) => { /* new connection */ }
    // ...
}
```

### Range Patterns

Match on numeric and character ranges. Common in parsers, lexers, and protocol handlers:

```rust
// Character classification — rustc lexer uses this pattern
// Source: compiler/rustc_lexer/src/lib.rs
let token_kind = match first_char {
    c @ '0'..='9' => {
        let kind = self.number(c);
        TokenKind::Literal { kind, suffix_start: self.pos() }
    }
    'a'..='z' | 'A'..='Z' | '_' => self.identifier(),
    _ => TokenKind::Unknown,
};

// Tag/attribute name parsing — robinson HTML parser
// Source: mbrubeck/robinson/src/html.rs
fn parse_name(input: &str) -> String {
    input.chars()
        .take_while(|c| matches!(c, 'a'..='z' | 'A'..='Z' | '0'..='9'))
        .collect()
}

// Byte-level parsing — common in binary protocols and hex decoders
fn decode_hex_digit(byte: u8) -> Option<u8> {
    match byte {
        b'0'..=b'9' => Some(byte - b'0'),
        b'a'..=b'f' => Some(byte - b'a' + 10),
        b'A'..=b'F' => Some(byte - b'A' + 10),
        _ => None,
    }
}

// HTTP status code classification
fn status_category(code: u16) -> &'static str {
    match code {
        100..=199 => "informational",
        200..=299 => "success",
        300..=399 => "redirection",
        400..=499 => "client error",
        500..=599 => "server error",
        _ => "unknown",
    }
}
```

### `ref` and `ref mut` in Patterns

Control borrowing explicitly in match arms — needed when matching on owned enums but wanting to borrow contents rather than move them. Production pattern from tokio:

```rust
// tokio's blocking I/O uses ref mut to modify state enum contents in place
// Source: tokio/src/io/blocking.rs
enum State {
    Idle(Option<Buf>),
    Busy(Receiver<(io::Result<usize>, Buf)>),
}

fn poll_read(&mut self, cx: &mut Context<'_>, dst: &mut ReadBuf<'_>) -> Poll<io::Result<()>> {
    loop {
        match self.state {
            State::Idle(ref mut buf_cell) => {
                // ref mut lets us modify the Option inside without moving out of self.state
                let mut buf = buf_cell.take().unwrap();
                if !buf.is_empty() {
                    buf.copy_to(dst);
                    *buf_cell = Some(buf);
                    return Poll::Ready(Ok(()));
                }
                // Transition to Busy...
            }
            State::Busy(ref mut rx) => {
                let (result, buf) = ready!(Pin::new(rx).poll(cx))?;
                self.state = State::Idle(Some(buf));
                return Poll::Ready(result);
            }
        }
    }
}

// ref to borrow from a match without moving (pre-ergonomics or when explicit)
match &event {
    Event::Message { sender, body, .. } => {
        log::info!("Message from {sender}: {body}");
    }
    Event::Error(ref e) => {
        // ref achieves the same as matching on &event — borrows e
        log::error!("Error: {e}");
    }
    _ => {}
}
```

### Matching on Constants

Use `const` values as match arms for protocol bytes, magic numbers, and named sentinel values. Clean alternative to matching on raw literals:

```rust
// SLIP protocol framing — named constants as match patterns
// Source: knurling-rs/nrfdfu-rs/src/slip.rs
const END: u8 = 0xC0;
const ESC: u8 = 0xDB;
const ESC_END: u8 = 0xDC;
const ESC_ESC: u8 = 0xDD;

fn encode_frame(buf: &[u8], writer: &mut impl Write) -> io::Result<()> {
    for &byte in buf {
        match byte {
            END => writer.write_all(&[ESC, ESC_END])?,
            ESC => writer.write_all(&[ESC, ESC_ESC])?,
            _ => writer.write_all(&[byte])?,
        }
    }
    writer.write_all(&[END])
}

fn decode_byte(bytes: &mut impl Iterator<Item = io::Result<u8>>) -> io::Result<Option<u8>> {
    match next_byte(bytes)? {
        ESC => match next_byte(bytes)? {
            ESC_ESC => Ok(Some(ESC)),
            ESC_END => Ok(Some(END)),
            invalid => Err(io::Error::new(
                io::ErrorKind::InvalidData,
                format!("invalid escape: 0x{invalid:02x}"),
            )),
        },
        END => Ok(None),  // Frame complete
        other => Ok(Some(other)),
    }
}

// Also works with associated constants and string constants
impl HttpMethod {
    const GET: &str = "GET";
    const POST: &str = "POST";
    const PUT: &str = "PUT";
    const DELETE: &str = "DELETE";
}

fn parse_method(s: &str) -> Option<HttpMethod> {
    match s {
        HttpMethod::GET => Some(HttpMethod::Get),
        HttpMethod::POST => Some(HttpMethod::Post),
        HttpMethod::PUT => Some(HttpMethod::Put),
        HttpMethod::DELETE => Some(HttpMethod::Delete),
        _ => None,
    }
}
```

### Pattern Matching Mistakes (BAD/GOOD)

```rust
// BAD: Wildcard hides missing match arms — new variants silently ignored
enum Command { Start, Stop, Pause, Resume }
fn handle(cmd: Command) {
    match cmd {
        Command::Start => start(),
        Command::Stop => stop(),
        _ => {}  // Pause and Resume silently do nothing — bug if unintentional
    }
}

// GOOD: Explicit match arms — compiler catches new variants
fn handle(cmd: Command) {
    match cmd {
        Command::Start => start(),
        Command::Stop => stop(),
        Command::Pause => {}   // Intentionally no-op (explicit)
        Command::Resume => {}  // Intentionally no-op (explicit)
    }
}

// BAD: match where if-let or let-else suffices
fn get_name(user: Option<&User>) -> String {
    match user {
        Some(u) => u.name.clone(),
        None => "anonymous".to_string(),
    }
}

// GOOD: map_or for simple transformation
fn get_name(user: Option<&User>) -> String {
    user.map_or_else(|| "anonymous".to_string(), |u| u.name.clone())
}

// BAD: forgetting match is an expression — assigning in each arm
let msg;
match status {
    Status::Ok => msg = "success",
    Status::Err => msg = "failure",
}

// GOOD: match IS an expression — bind directly
let msg = match status {
    Status::Ok => "success",
    Status::Err => "failure",
};

// BAD: matching String when &str works — forces allocation or ownership
fn classify(input: String) -> Category {
    match input.as_str() {  // Needs .as_str() because match arms are &str
        "error" => Category::Error,
        "warn" => Category::Warning,
        _ => Category::Info,
    }
}

// GOOD: take &str from the start — no allocation needed
fn classify(input: &str) -> Category {
    match input {
        "error" => Category::Error,
        "warn" => Category::Warning,
        _ => Category::Info,
    }
}
```

## Ownership & Borrowing Patterns

### Borrow Splitting

```rust
// The borrow checker can track borrows to individual struct fields
struct GameState {
    player: Player,
    enemies: Vec<Enemy>,
    score: u32,
}

fn update(state: &mut GameState) {
    // This works — different fields are borrowed independently
    let player = &mut state.player;
    let enemies = &state.enemies;  // Immutable borrow of different field
    player.update(enemies);
    state.score += 1;  // Mutable borrow of yet another field
}

// But it doesn't work through methods — the whole &mut self is borrowed
impl GameState {
    fn player(&mut self) -> &mut Player { &mut self.player }
    fn enemies(&self) -> &[Enemy] { &self.enemies }

    fn update(&mut self) {
        // FAILS: self.player() borrows all of self, so self.enemies() can't borrow
        // let p = self.player();
        // let e = self.enemies();

        // WORKS: access fields directly
        let p = &mut self.player;
        let e = &self.enemies;
        p.update(e);
    }
}
# struct Player;
# struct Enemy;
# impl Player { fn update(&mut self, _: &[Enemy]) {} }
```

### Temporary Borrow Scoping

```rust
// Limit borrow scope by using a block
let mut data = vec![1, 2, 3, 4, 5];

// Extract information from immutable borrow
let first = {
    let slice = &data[..];
    slice.first().copied()  // Borrow of data ends here
};

// Now we can mutate
data.push(6);

// Same pattern with Mutex — don't hold the guard
let result = {
    let guard = mutex.lock().unwrap();
    guard.clone()  // Clone what you need, drop the guard
};
// Mutex is unlocked here
do_async_work(result).await;
```

### Cow<T> — Clone on Write

```rust
use std::borrow::Cow;

// Cow defers cloning until mutation is needed
fn normalize_username(input: &str) -> Cow<str> {
    if input.chars().all(|c| c.is_lowercase()) {
        Cow::Borrowed(input)  // No allocation — just return the input
    } else {
        Cow::Owned(input.to_lowercase())  // Allocate only when needed
    }
}

// In structs — own or borrow depending on context
struct LogEntry<'a> {
    message: Cow<'a, str>,
    source: Cow<'a, str>,
}

impl<'a> LogEntry<'a> {
    // Can be created with borrowed OR owned strings
    fn new(msg: impl Into<Cow<'a, str>>, src: impl Into<Cow<'a, str>>) -> Self {
        Self { message: msg.into(), source: src.into() }
    }
}

// Usage — no allocation when possible
let entry = LogEntry::new("static message", "static source");  // Both borrowed
let entry = LogEntry::new(format!("dynamic {}", id), "static");  // First owned, second borrowed

// Common in parsers — most tokens are substrings, but some need transformation
fn parse_token(input: &str) -> Cow<str> {
    if input.contains('\\') {
        Cow::Owned(input.replace("\\n", "\n").replace("\\t", "\t"))
    } else {
        Cow::Borrowed(input)
    }
}
```

### Zero-Copy Patterns

```rust
// Take &[u8] or &str instead of Vec<u8> or String
fn process_bytes(data: &[u8]) -> Result<(), Error> {
    // Works with Vec<u8>, &[u8], arrays, slices — anything that derefs to [u8]
    for chunk in data.chunks(1024) {
        handle_chunk(chunk)?;
    }
    Ok(())
}

// Serde zero-copy deserialization
use serde::Deserialize;

#[derive(Deserialize)]
struct Message<'a> {
    #[serde(borrow)]
    text: &'a str,           // Borrows from the input buffer — no allocation
    #[serde(borrow)]
    tags: Vec<&'a str>,      // Each tag borrows from input
}
// Requires: serde_json::from_str (not from_reader, which doesn't support borrowing)

// String interning for repeated strings
use std::collections::HashSet;
fn intern<'a>(pool: &'a HashSet<String>, s: &str) -> &'a str {
    if let Some(existing) = pool.get(s) {
        existing.as_str()
    } else {
        // In real code, use a proper interning crate like `string-interner`
        panic!("not interned")
    }
}
```

### Entry Pattern for Complex Map Updates

```rust
use std::collections::HashMap;

// Build complex values only when the key is absent
let mut cache: HashMap<String, Vec<Item>> = HashMap::new();

// or_insert_with — lazy initialization
cache.entry(key.to_string())
    .or_insert_with(|| expensive_compute(&key))
    .push(new_item);

// and_modify + or_insert — update if present, insert if absent
let mut word_count: HashMap<&str, usize> = HashMap::new();
for word in text.split_whitespace() {
    word_count.entry(word)
        .and_modify(|count| *count += 1)
        .or_insert(1);
}

// Entry with complex initialization
let mut connections: HashMap<String, Connection> = HashMap::new();
let conn = connections.entry(host.to_string())
    .or_insert_with(|| {
        Connection::new(&host)
            .timeout(Duration::from_secs(30))
            .build()
            .expect("connection failed")
    });
conn.send(message)?;
```

## The ? Operator & Error Chains

### Basic ? Chaining

```rust
// ? works with both Result and Option
fn parse_config(path: &str) -> Result<Config, AppError> {
    let content = std::fs::read_to_string(path)?;  // io::Error → AppError
    let parsed: toml::Value = content.parse()?;     // toml::Error → AppError
    let port = parsed.get("port")                   // Option → None returns early
        .and_then(|v| v.as_integer())
        .ok_or(AppError::MissingField("port"))?;    // Option → Result
    Ok(Config { port: port as u16 })
}
```

### Adding Context to Errors

```rust
use anyhow::{Context, Result};

// .context() wraps the error with a static message
fn load_config() -> Result<Config> {
    let path = find_config_path().context("locating config file")?;
    let content = std::fs::read_to_string(&path)
        .with_context(|| format!("reading config from {}", path.display()))?;
    let config: Config = toml::from_str(&content)
        .context("parsing config TOML")?;
    Ok(config)
}

// Error output with context chain:
// Error: parsing config TOML
//   Caused by:
//     expected value at line 3, column 5

// .with_context() — lazy, use when formatting is expensive
fn load_user(id: u64) -> Result<User> {
    db.find(id)
        .with_context(|| format!("loading user {id} from database"))?
        .ok_or_else(|| anyhow::anyhow!("user {id} not found"))
}
```

### Error Conversion Across Layers

```rust
// Layer 1: Repository errors (thiserror)
#[derive(Debug, thiserror::Error)]
pub enum RepoError {
    #[error("not found: {entity} {id}")]
    NotFound { entity: &'static str, id: String },
    #[error(transparent)]
    Database(#[from] sqlx::Error),
}

// Layer 2: Service errors — wraps repo errors
#[derive(Debug, thiserror::Error)]
pub enum ServiceError {
    #[error(transparent)]
    Repo(#[from] RepoError),         // Auto From impl
    #[error("validation: {0}")]
    Validation(String),
    #[error("unauthorized")]
    Unauthorized,
}

// Layer 3: Handler errors — wraps service errors
#[derive(Debug, thiserror::Error)]
pub enum HandlerError {
    #[error(transparent)]
    Service(#[from] ServiceError),   // Auto From impl
    #[error("bad request: {0}")]
    BadRequest(String),
}

// Now ? converts automatically up the chain:
// sqlx::Error → RepoError → ServiceError → HandlerError
async fn handle_request(id: u64) -> Result<Response, HandlerError> {
    let user = user_service.find(id)?;   // ServiceError → HandlerError
    Ok(Response::ok(user))
}
```

### Fallible Iterators

```rust
// Collect Results — short-circuits on first error
let numbers: Result<Vec<i32>, _> = strings.iter()
    .map(|s| s.parse::<i32>())
    .collect();

// Process all, collect errors separately
let (ok, err): (Vec<_>, Vec<_>) = items.iter()
    .map(|item| process(item))
    .partition(Result::is_ok);

let successes: Vec<_> = ok.into_iter().map(Result::unwrap).collect();
let failures: Vec<_> = err.into_iter().map(Result::unwrap_err).collect();

// try_for_each — stop on first error
items.iter().try_for_each(|item| -> Result<(), Error> {
    validate(item)?;
    persist(item)?;
    Ok(())
})?;

// try_fold — accumulate with early error exit
let total = items.iter().try_fold(0u64, |acc, item| -> Result<u64, Error> {
    let value = parse_value(item)?;
    Ok(acc + value)
})?;
```

## Iterator Composition (Extended)

### Adaptor Chaining Patterns

```rust
// Deduplication — unique elements preserving order
let unique: Vec<_> = items.iter()
    .collect::<std::collections::LinkedHashSet<_>>()  // Not in std — use indexmap
    .into_iter()
    .collect();

// Alternative using a HashSet as seen-tracker
let mut seen = HashSet::new();
let unique: Vec<_> = items.iter()
    .filter(|item| seen.insert(*item))  // insert returns false if already present
    .collect();

// Batched processing
for chunk in data.chunks(100) {
    process_batch(chunk)?;
}

// Interleave two iterators
let a = [1, 3, 5];
let b = [2, 4, 6];
let interleaved: Vec<_> = a.iter()
    .zip(b.iter())
    .flat_map(|(a, b)| [a, b])
    .collect();  // [1, 2, 3, 4, 5, 6]

// Group consecutive equal elements
let groups: Vec<Vec<_>> = items.iter()
    .fold(Vec::new(), |mut groups, item| {
        match groups.last_mut() {
            Some(group) if group[0] == item => group.push(item),
            _ => groups.push(vec![item]),
        }
        groups
    });
```

### Building Custom Iterators

```rust
// Iterator over pairs (sliding window of 2)
struct Pairs<I: Iterator> {
    iter: std::iter::Peekable<I>,
}

impl<I: Iterator> Iterator for Pairs<I>
where
    I::Item: Clone,
{
    type Item = (I::Item, I::Item);

    fn next(&mut self) -> Option<Self::Item> {
        let first = self.iter.next()?;
        let second = self.iter.peek()?.clone();
        Some((first, second))
    }
}

// Extension trait to add .pairs() to any iterator
trait IteratorExt: Iterator + Sized {
    fn pairs(self) -> Pairs<Self> where Self::Item: Clone {
        Pairs { iter: self.peekable() }
    }
}
impl<I: Iterator> IteratorExt for I {}

// Usage
let slopes: Vec<_> = values.iter().pairs()
    .map(|(a, b)| b - a)
    .collect();
```

### IntoIterator for Custom Types

```rust
struct Matrix {
    data: Vec<Vec<f64>>,
}

// Consuming iterator — matrix is consumed
impl IntoIterator for Matrix {
    type Item = Vec<f64>;
    type IntoIter = std::vec::IntoIter<Vec<f64>>;

    fn into_iter(self) -> Self::IntoIter {
        self.data.into_iter()
    }
}

// Borrowing iterator — matrix is borrowed
impl<'a> IntoIterator for &'a Matrix {
    type Item = &'a Vec<f64>;
    type IntoIter = std::slice::Iter<'a, Vec<f64>>;

    fn into_iter(self) -> Self::IntoIter {
        self.data.iter()
    }
}

// Now works in for loops
for row in &matrix { println!("{row:?}"); }  // borrows
for row in matrix { process(row); }           // consumes
```

### Lazy Evaluation Patterns

```rust
// Build a pipeline, don't execute until needed
fn build_query(filters: &Filters) -> impl Iterator<Item = &Record> + '_ {
    records.iter()
        .filter(move |r| filters.matches(r))  // Lazy — nothing runs yet
        .take(filters.limit)
}

// Execute lazily
let results: Vec<_> = build_query(&filters).collect();  // Now it runs

// Chain multiple lazy stages
let pipeline = data.iter()
    .filter(|x| x.is_valid())
    .map(|x| x.transform())
    .inspect(|x| tracing::debug!("processing: {x:?}"))
    .take_while(|x| x.value < threshold);

// Nothing has happened yet — pipeline is just a description
let output: Vec<_> = pipeline.collect();  // Executes the entire chain
```

## Closure Capture Semantics

### What Closures Capture

```rust
let name = String::from("Alice");
let age = 30;

// Borrows name (Fn — most permissive)
let greet = || println!("Hello, {name}");
greet();
println!("{name}");  // Still available — only borrowed

// Mutably borrows data (FnMut)
let mut total = 0;
let mut add = |x: i32| { total += x; };  // Captures &mut total
add(5);
add(10);
println!("{total}");  // 15

// Takes ownership (FnOnce)
let name = String::from("Alice");
let consume = move || {
    drop(name);  // name is moved into the closure
};
consume();
// consume();  // ERROR: FnOnce can only be called once
// println!("{name}");  // ERROR: name was moved
```

### move Closures

```rust
// move forces ownership even when borrowing would suffice
// Required for: spawning threads, returning closures, async blocks

// Thread spawning — closure must be 'static (own all data)
let data = vec![1, 2, 3];
std::thread::spawn(move || {
    println!("{data:?}");  // data moved into thread
});

// Returning closures — can't return references to local variables
fn make_greeter(name: String) -> impl Fn() -> String {
    move || format!("Hello, {name}!")  // name moved into closure
}

// Async blocks with move
let data = Arc::clone(&shared_data);
tokio::spawn(async move {
    process(&data).await;  // data (Arc) moved into task
});

// move + Clone for partial capture
let config = config.clone();  // Clone first, then move the clone
tokio::spawn(async move {
    use_config(&config).await;
});
```

### Fn Trait Hierarchy

```rust
// Fn: borrows captured data (can call multiple times)
// FnMut: mutably borrows captured data (can call multiple times)
// FnOnce: consumes captured data (can call at most once)
// Relationship: Fn ⊂ FnMut ⊂ FnOnce

// Accept the most general trait your code needs:
fn call_once<F: FnOnce() -> i32>(f: F) -> i32 { f() }
fn call_many<F: Fn() -> i32>(f: F) -> i32 { f() + f() }
fn call_mut<F: FnMut() -> i32>(mut f: F) -> i32 { f() + f() }

// PREFER Fn for callbacks stored in structs
struct EventHandler<F: Fn(&Event)> {
    handler: F,
}

// PREFER FnOnce for one-shot callbacks (e.g., completion handlers)
fn on_complete<F: FnOnce(Result<(), Error>)>(f: F) {
    // ...
    f(Ok(()));
}

// PREFER FnMut for iterators and accumulators
fn fold<F: FnMut(i32, i32) -> i32>(items: &[i32], init: i32, mut f: F) -> i32 {
    let mut acc = init;
    for &item in items {
        acc = f(acc, item);
    }
    acc
}
```

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

// Object-safe trait design rules:
// - No methods returning Self (use Box<Self> instead)
// - No generic methods (use trait objects as parameters instead)
// - Methods must take &self, &mut self, or Box<Self>

// BAD: not object-safe
trait NotSafe {
    fn clone_self(&self) -> Self;  // Returns Self
    fn process<T>(&self, data: T);  // Generic method
}

// GOOD: object-safe
trait Safe {
    fn clone_boxed(&self) -> Box<dyn Safe>;
    fn process(&self, data: &dyn std::any::Any);
}
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

// Trait aliases (not yet stable — use type alias workaround)
trait JsonSerializable: serde::Serialize + serde::de::DeserializeOwned + Send + Sync + 'static {}
impl<T> JsonSerializable for T
where T: serde::Serialize + serde::de::DeserializeOwned + Send + Sync + 'static {}

// Now use JsonSerializable as a single bound instead of 5
fn store<T: JsonSerializable>(value: &T) -> Result<(), Error> { todo!() }
```

## From/Into/AsRef Conversions

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

## RAII, Drop & Resource Management

### Automatic Cleanup with Drop

```rust
struct TempDir {
    path: std::path::PathBuf,
}

impl TempDir {
    fn new(prefix: &str) -> std::io::Result<Self> {
        let path = std::env::temp_dir().join(format!("{prefix}-{}", rand::random::<u32>()));
        std::fs::create_dir_all(&path)?;
        Ok(Self { path })
    }

    fn path(&self) -> &std::path::Path {
        &self.path
    }
}

impl Drop for TempDir {
    fn drop(&mut self) {
        let _ = std::fs::remove_dir_all(&self.path);
    }
}

// Automatic cleanup when scope exits — even on error/panic
fn process() -> Result<(), Error> {
    let tmp = TempDir::new("myapp")?;
    write_files(tmp.path())?;        // If this fails...
    process_files(tmp.path())?;      // ...or this fails...
    Ok(())
    // tmp is dropped here — directory cleaned up regardless
}
```

### Guard Pattern

```rust
// Use a guard to ensure cleanup happens
struct MutexGuard<'a, T> {
    lock: &'a Mutex<T>,
    data: &'a mut T,
}

impl<'a, T> Drop for MutexGuard<'a, T> {
    fn drop(&mut self) {
        // Automatically unlocks when guard goes out of scope
        self.lock.unlock();
    }
}

// Real-world: std::sync::MutexGuard works exactly this way
{
    let guard = mutex.lock().unwrap();  // Lock acquired
    guard.do_work();
}  // Lock released here — even if do_work() panics

// scopeguard crate — ad-hoc cleanup
use scopeguard::defer;

fn deploy() -> Result<(), Error> {
    start_maintenance_mode()?;
    defer! { stop_maintenance_mode(); }  // Always runs on scope exit

    run_migrations()?;
    update_code()?;
    Ok(())
    // stop_maintenance_mode() called here
}
```

### ManuallyDrop and mem::forget

```rust
use std::mem::ManuallyDrop;

// ManuallyDrop — opt out of automatic Drop
struct FileMapping {
    ptr: *mut u8,
    len: usize,
}

impl FileMapping {
    fn into_vec(self) -> Vec<u8> {
        let md = ManuallyDrop::new(self);  // Prevent Drop from running
        // SAFETY: ptr and len came from a valid mapping
        unsafe { Vec::from_raw_parts(md.ptr, md.len, md.len) }
    }
}

// mem::forget — prevent drop (leaks the value)
// Use sparingly — usually ManuallyDrop is clearer
let handle = create_handle();
std::mem::forget(handle);  // handle is leaked — destructor never runs
```

## Module Organization & Visibility

### Visibility Modifiers

```rust
pub struct User {
    pub name: String,          // Public everywhere
    pub(crate) id: u64,       // Public within this crate only
    pub(super) role: Role,    // Public to parent module only
    pub(in crate::admin) level: u32,  // Public to specific module
    password_hash: String,     // Private (default)
}

// pub(crate) for internal APIs
pub(crate) fn internal_helper() { }

// Private module with public re-exports
mod internal {
    pub fn compute() -> u64 { 42 }  // pub within parent, not externally
}
pub use internal::compute;  // Re-export makes it truly public
```

### Module Layout Patterns

```rust
// Pattern 1: File per module
// src/
// ├── lib.rs       → mod user; mod auth;
// ├── user.rs      → pub struct User { ... }
// └── auth.rs      → pub fn authenticate() { ... }

// Pattern 2: Directory module (for modules with sub-modules)
// src/
// ├── lib.rs       → mod database;
// └── database/
//     ├── mod.rs    → pub mod postgres; pub mod sqlite; pub use postgres::PgPool;
//     ├── postgres.rs
//     └── sqlite.rs

// Pattern 3: Prelude module (export commonly used items)
pub mod prelude {
    pub use crate::{Config, Error, Result};
    pub use crate::traits::{Repository, Service};
}
// Users: use mylib::prelude::*;

// Pattern 4: Internal/external separation
pub mod api {        // Public interface
    pub use crate::internal::UserService;
}
mod internal {       // Implementation details
    pub struct UserService { /* fields private */ }
    impl UserService { /* methods */ }
}
```

### Re-exports for Clean APIs

```rust
// Before re-exports — users must know internal structure
// use mylib::database::postgres::PgPool;
// use mylib::database::traits::Repository;
// use mylib::error::AppError;

// After re-exports — clean public API
// src/lib.rs
mod database;
mod error;

pub use database::PgPool;           // Flatten the hierarchy
pub use database::Repository;
pub use error::AppError as Error;   // Rename on re-export

// Users: use mylib::{PgPool, Repository, Error};
```

### Documentation Attributes for Re-exports

Every production library (axum, reqwest, serde_json) uses these to control documentation:

```rust
// Show re-exported type inline in THIS crate's docs
#[doc(inline)]
pub use self::extract::Json;

// Link to the ORIGINAL crate's docs (don't duplicate)
#[doc(no_inline)]
pub use http::StatusCode;
pub use http::Method;

// Hide from docs entirely — used for macro internals
#[doc(hidden)]
pub mod __private {
    // Functions called by proc-macro generated code
    pub fn __format_err(args: std::fmt::Arguments) -> Error { /* ... */ }
}

// Feature badges on docs.rs — shows which feature enables an item
#![cfg_attr(docsrs, feature(doc_cfg))]

#[cfg(feature = "json")]
#[cfg_attr(docsrs, doc(cfg(feature = "json")))]  // Shows "Available on feature json only"
pub use self::json::Json;

// Include external docs from markdown files (axum pattern)
#[doc = include_str!("docs/extractors.md")]
pub mod extract;
```

### Implementing FromIterator and Extend

Implement these to make your collection work with `.collect()` and `.extend()`:

```rust
// FromIterator — enables: let set: MySet<T> = iter.collect();
impl<T: Eq + Hash> FromIterator<T> for MySet<T> {
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self {
        let mut set = MySet::new();
        set.extend(iter);
        set
    }
}

// Extend — enables: set.extend(vec![1, 2, 3]);
impl<T: Eq + Hash> Extend<T> for MySet<T> {
    fn extend<I: IntoIterator<Item = T>>(&mut self, iter: I) {
        for item in iter {
            self.insert(item);
        }
    }
}

// Extend for references (avoids requiring owned values)
impl<'a, T: 'a + Eq + Hash + Copy> Extend<&'a T> for MySet<T> {
    fn extend<I: IntoIterator<Item = &'a T>>(&mut self, iter: I) {
        self.extend(iter.into_iter().copied());
    }
}
```

### Operator Overloading

Implement `std::ops` traits for domain types (used by dashmap, nalgebra, num crates):

```rust
use std::ops::{Add, Mul, Neg};

#[derive(Debug, Clone, Copy, PartialEq)]
struct Vec2 { x: f64, y: f64 }

impl Add for Vec2 {
    type Output = Self;
    fn add(self, rhs: Self) -> Self {
        Vec2 { x: self.x + rhs.x, y: self.y + rhs.y }
    }
}

impl Mul<f64> for Vec2 {
    type Output = Self;
    fn mul(self, scalar: f64) -> Self {
        Vec2 { x: self.x * scalar, y: self.y * scalar }
    }
}

impl Neg for Vec2 {
    type Output = Self;
    fn neg(self) -> Self {
        Vec2 { x: -self.x, y: -self.y }
    }
}

// Index for custom containers
use std::ops::Index;

impl<T> Index<usize> for RingBuffer<T> {
    type Output = T;
    fn index(&self, idx: usize) -> &T {
        &self.data[(self.head + idx) % self.data.len()]
    }
}

// Usage: let v = Vec2 { x: 1.0, y: 2.0 } + Vec2 { x: 3.0, y: 4.0 };
// Usage: let scaled = v * 2.5;
```

## Conditional Compilation

### cfg Attributes

```rust
// Platform-specific code
#[cfg(target_os = "linux")]
fn get_memory_usage() -> u64 {
    // Read from /proc/self/status
    todo!()
}

#[cfg(target_os = "macos")]
fn get_memory_usage() -> u64 {
    // Use mach_task_basic_info
    todo!()
}

#[cfg(not(any(target_os = "linux", target_os = "macos")))]
fn get_memory_usage() -> u64 { 0 }

// Feature-gated modules
#[cfg(feature = "postgres")]
pub mod postgres;

#[cfg(feature = "sqlite")]
pub mod sqlite;

// Test-only code
#[cfg(test)]
mod tests {
    use super::*;

    // Test helpers only compiled in test mode
    fn test_fixture() -> TestData { /* ... */ }
}

// cfg_if crate — cleaner conditional blocks
cfg_if::cfg_if! {
    if #[cfg(feature = "postgres")] {
        type DefaultDb = PgPool;
    } else if #[cfg(feature = "sqlite")] {
        type DefaultDb = SqlitePool;
    } else {
        compile_error!("Enable either 'postgres' or 'sqlite' feature");
    }
}
```

### Runtime cfg Checks

```rust
// cfg! macro — evaluate at runtime (all branches must compile)
if cfg!(debug_assertions) {
    println!("Debug mode — extra validation enabled");
    expensive_validation(&data)?;
}

if cfg!(target_pointer_width = "64") {
    // 64-bit optimized path
}

// Combine with feature flags
if cfg!(feature = "tracing") {
    tracing::info!("operation started");
}
```

### Conditional Compilation Macros (reqwest pattern)

For code that spans many items across platform boundaries, use wrapper macros:

```rust
// Define once — reuse everywhere
macro_rules! if_wasm {
    ($($item:item)*) => {$(
        #[cfg(target_arch = "wasm32")]
        $item
    )*}
}

macro_rules! if_native {
    ($($item:item)*) => {$(
        #[cfg(not(target_arch = "wasm32"))]
        $item
    )*}
}

// Cleanly separates platform-specific code
if_native! {
    mod native_tls;
    mod dns;
    pub use native_tls::Certificate;
}

if_wasm! {
    mod wasm;
    pub use wasm::WebClient;
}
```

### Test vs Production Lint Profiles

Production libraries use different lint rules for test and non-test code:

```rust
// In lib.rs or main.rs:
#![cfg_attr(not(test), warn(clippy::print_stdout, clippy::dbg_macro))]
#![cfg_attr(not(test), warn(unused_crate_dependencies))]
#![cfg_attr(test, allow(clippy::print_stdout))]  // println! ok in tests
#![cfg_attr(test, deny(warnings))]                // strict in tests

// Per-function: test-only imports
#[cfg(test)]
use pretty_assertions::assert_eq;  // Better diff output in tests
```

### Flexible Key Lookups with Equivalent/Borrow

Allow `&str` lookups on `HashMap<String, V>` without allocating:

```rust
use std::borrow::Borrow;

// Standard HashMap already supports this via Borrow trait:
let mut map = HashMap::new();
map.insert("key".to_string(), 42);
let val = map.get("key");  // &str works — String: Borrow<str>

// For custom key types, implement Borrow:
#[derive(Hash, Eq, PartialEq)]
struct CaseInsensitive(String);

impl Borrow<str> for CaseInsensitive {
    fn borrow(&self) -> &str { &self.0 }
}
// Now: map.get("some_key") works with CaseInsensitive keys
```

### Try-Lock Pattern for Non-Blocking Access

```rust
use std::sync::RwLock;

let lock = RwLock::new(vec![1, 2, 3]);

// Non-blocking: returns None if lock is held
match lock.try_read() {
    Ok(data) => println!("got data: {data:?}"),
    Err(_) => println!("lock busy, skipping"),
}

// Common in hot paths where blocking is unacceptable
fn try_get_cached(&self, key: &str) -> Option<Value> {
    let cache = self.cache.try_read().ok()?;  // Skip if locked
    cache.get(key).cloned()
}
```

## Common Macro Invocation Patterns

### vec!, format!, write!

```rust
// vec! — most common collection macro
let zeros = vec![0u8; 1024];            // 1024 zeros
let items = vec![1, 2, 3];              // List initialization
let matrix = vec![vec![0; cols]; rows]; // 2D array

// format! — String construction (allocates)
let msg = format!("{name} has {count} items");
let padded = format!("{:>10}", "right");
let hex = format!("{:#x}", 255);         // "0xff"

// write! / writeln! — write to any impl Write (doesn't allocate new String)
use std::fmt::Write as FmtWrite;
let mut buf = String::new();
writeln!(buf, "line {}", 1)?;
writeln!(buf, "line {}", 2)?;

use std::io::Write as IoWrite;
let mut file = std::fs::File::create("out.txt")?;
writeln!(file, "data: {}", value)?;
```

### include_str!, include_bytes!, env!

```rust
// Embed file contents at compile time
const SCHEMA: &str = include_str!("../schema.sql");
const LOGO: &[u8] = include_bytes!("../assets/logo.png");

// Compile-time environment variables
const VERSION: &str = env!("CARGO_PKG_VERSION");
const PKG_NAME: &str = env!("CARGO_PKG_NAME");

// Optional env var (returns Option<&str>)
const CI: Option<&str> = option_env!("CI");
```

### assert!, debug_assert!, todo!

```rust
// assert! — always checked (even in release)
assert!(index < len, "index {index} out of bounds for length {len}");
assert_eq!(expected, actual, "mismatch at position {pos}");

// debug_assert! — only checked in debug builds (zero cost in release)
debug_assert!(ptr.is_aligned(), "pointer must be aligned");
debug_assert_eq!(a.len(), b.len());

// todo! — marks unfinished code (panics with file:line)
fn complex_algorithm() -> Result<(), Error> {
    todo!("implement after spec review")
}

// unimplemented! — marks intentionally unimplemented code
fn unsupported_format(&self) -> ! {
    unimplemented!("XML format not supported — use JSON")
}

// unreachable! — marks provably unreachable code
match direction {
    Direction::North | Direction::South | Direction::East | Direction::West => move_to(direction),
    // If Direction is non_exhaustive and you're sure no other values exist:
    _ => unreachable!("all directions handled"),
}
```

## Production Patterns

### Configuration Loading

```rust
// figment — layered configuration
use figment::{Figment, providers::{Env, Toml, Format}};

#[derive(serde::Deserialize)]
struct Config {
    host: String,
    port: u16,
    #[serde(default = "default_workers")]
    workers: usize,
    database_url: String,
}

fn default_workers() -> usize { num_cpus::get() }

fn load_config() -> Result<Config, figment::Error> {
    Figment::new()
        .merge(Toml::file("config.toml"))       // Base config
        .merge(Toml::file("config.local.toml"))  // Local overrides
        .merge(Env::prefixed("APP_"))            // Environment variables (highest priority)
        .extract()
}
```

### Graceful Shutdown

```rust
use tokio::signal;
use tokio::sync::watch;

async fn run_server() -> Result<(), Error> {
    let (shutdown_tx, shutdown_rx) = watch::channel(false);

    // Spawn the server
    let server = tokio::spawn(run_http_server(shutdown_rx.clone()));

    // Spawn background workers
    let worker = tokio::spawn(run_background_worker(shutdown_rx.clone()));

    // Wait for shutdown signal
    signal::ctrl_c().await?;
    tracing::info!("Shutdown signal received");

    // Notify all tasks
    shutdown_tx.send(true)?;

    // Wait for graceful shutdown with timeout
    tokio::select! {
        _ = server => tracing::info!("Server stopped"),
        _ = tokio::time::sleep(Duration::from_secs(30)) => {
            tracing::warn!("Shutdown timeout — forcing exit");
        }
    }

    Ok(())
}

async fn run_background_worker(mut shutdown: watch::Receiver<bool>) {
    loop {
        tokio::select! {
            _ = shutdown.changed() => {
                tracing::info!("Worker shutting down");
                break;
            }
            _ = do_work() => {}
        }
    }
}
# async fn run_http_server(_: watch::Receiver<bool>) {}
# async fn do_work() { tokio::time::sleep(Duration::from_secs(1)).await; }
```

### Retry with Exponential Backoff

```rust
use std::time::Duration;

async fn retry_with_backoff<F, Fut, T, E>(
    f: F,
    max_retries: u32,
    initial_delay: Duration,
) -> Result<T, E>
where
    F: Fn() -> Fut,
    Fut: std::future::Future<Output = Result<T, E>>,
    E: std::fmt::Display,
{
    let mut delay = initial_delay;

    for attempt in 0..max_retries {
        match f().await {
            Ok(value) => return Ok(value),
            Err(e) if attempt + 1 < max_retries => {
                tracing::warn!(
                    attempt = attempt + 1,
                    max_retries,
                    "retrying after error: {e}"
                );
                tokio::time::sleep(delay).await;
                delay = delay.mul_f64(2.0).min(Duration::from_secs(60));  // Cap at 60s
            }
            Err(e) => return Err(e),
        }
    }
    unreachable!()
}

// Usage
let result = retry_with_backoff(
    || async { fetch_data(&url).await },
    5,
    Duration::from_millis(100),
).await?;
```

### Middleware / Decorator Pattern

```rust
// Tower-style middleware — composable request/response processing
use std::future::Future;

trait Service<Request> {
    type Response;
    type Error;

    fn call(&self, req: Request) -> impl Future<Output = Result<Self::Response, Self::Error>>;
}

// Logging middleware wraps any service
struct LoggingMiddleware<S> {
    inner: S,
    prefix: String,
}

impl<S, Req> Service<Req> for LoggingMiddleware<S>
where
    S: Service<Req>,
    Req: std::fmt::Debug,
{
    type Response = S::Response;
    type Error = S::Error;

    async fn call(&self, req: Req) -> Result<S::Response, S::Error> {
        tracing::info!("{}: processing {:?}", self.prefix, req);
        let result = self.inner.call(req).await;
        match &result {
            Ok(_) => tracing::info!("{}: success", self.prefix),
            Err(_) => tracing::warn!("{}: failed", self.prefix),
        }
        result
    }
}

// Compose: LoggingMiddleware<RetryMiddleware<HttpClient>>
```

### Type-Safe ID Pattern

```rust
use std::marker::PhantomData;

// Generic typed ID — prevents mixing up IDs from different entities
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct Id<T> {
    value: u64,
    _phantom: PhantomData<T>,
}

impl<T> Id<T> {
    fn new(value: u64) -> Self {
        Self { value, _phantom: PhantomData }
    }
}

impl<T> std::fmt::Display for Id<T> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.value)
    }
}

// Usage — different types, can't be mixed
struct User;
struct Order;

type UserId = Id<User>;
type OrderId = Id<Order>;

fn process_order(user_id: UserId, order_id: OrderId) {
    // Can't accidentally swap these — they're different types
}

// With serde support
impl<T> serde::Serialize for Id<T> {
    fn serialize<S: serde::Serializer>(&self, s: S) -> Result<S::Ok, S::Error> {
        self.value.serialize(s)
    }
}
```

### Visitor Pattern

The Visitor pattern separates traversal logic from data structure. In Rust, traits with associated types make this type-safe and extensible — serde's deserialization is the canonical real-world example.

```rust
// The data structure defines what it can contain
trait Element {
    fn accept<V: Visitor>(&self, visitor: &mut V) -> V::Output;
}

// The visitor defines how to process each element type
trait Visitor {
    type Output;
    fn visit_text(&mut self, text: &str) -> Self::Output;
    fn visit_number(&mut self, n: f64) -> Self::Output;
    fn visit_list(&mut self, items: &[Node]) -> Self::Output;
}

enum Node {
    Text(String),
    Number(f64),
    List(Vec<Node>),
}

impl Element for Node {
    fn accept<V: Visitor>(&self, visitor: &mut V) -> V::Output {
        match self {
            Node::Text(s) => visitor.visit_text(s),
            Node::Number(n) => visitor.visit_number(*n),
            Node::List(items) => visitor.visit_list(items),
        }
    }
}

// Different visitors for different operations — no changes to Node
struct JsonPrinter;
impl Visitor for JsonPrinter {
    type Output = String;
    fn visit_text(&mut self, text: &str) -> String {
        format!("\"{}\"", text.replace('"', "\\\""))
    }
    fn visit_number(&mut self, n: f64) -> String { n.to_string() }
    fn visit_list(&mut self, items: &[Node]) -> String {
        let inner: Vec<String> = items.iter().map(|item| item.accept(self)).collect();
        format!("[{}]", inner.join(","))
    }
}

struct Counter { count: usize }
impl Visitor for Counter {
    type Output = ();
    fn visit_text(&mut self, _: &str) { self.count += 1; }
    fn visit_number(&mut self, _: f64) { self.count += 1; }
    fn visit_list(&mut self, items: &[Node]) {
        for item in items { item.accept(self); }
    }
}
```

**serde's Visitor pattern** follows this same structure but adds lifetime threading:

```rust
// Simplified version of serde's Visitor trait
trait Visitor<'de>: Sized {
    type Value;

    // Each method handles one data type the deserializer might encounter
    fn visit_bool(self, v: bool) -> Result<Self::Value, Error> {
        Err(Error::invalid_type("bool"))  // Default: reject
    }
    fn visit_str(self, v: &str) -> Result<Self::Value, Error> {
        Err(Error::invalid_type("string"))
    }
    fn visit_borrowed_str(self, v: &'de str) -> Result<Self::Value, Error> {
        self.visit_str(v)  // Default: delegate to non-borrowed
    }
    // ... visit_u64, visit_map, visit_seq, etc.
}

// The 'de lifetime enables zero-copy deserialization:
// visit_borrowed_str gives you &'de str pointing into the input buffer
// visit_str gives you &str with a shorter lifetime (may need to copy)
```

**When to use Visitor in Rust:**
- Processing heterogeneous data (ASTs, config files, serialization formats)
- Multiple operations on the same data structure (print, validate, transform)
- The set of element types is stable but operations change frequently
- **When NOT:** if the set of operations is stable but types change, use enum + match instead

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: ownership, traits, error handling, iterators, serde, async
- **[type-system.md](type-system.md)** — Type state, GATs, const generics, Pin/Unpin, sealed traits, async traits
- **[architecture.md](architecture.md)** — Workspace design, DI, application layering, production patterns
- **[async-concurrency.md](async-concurrency.md)** — Tokio runtime, channels, rayon, Tower, actors
- **[error-handling.md](error-handling.md)** — thiserror/anyhow/color-eyre, multi-layer errors
- **[macros.md](macros.md)** — Declarative and procedural macro authoring
