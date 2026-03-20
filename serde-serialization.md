# Serde Serialization

Comprehensive serde patterns: derive macros, field attributes, custom serialization, enum representations, format-specific usage, and zero-copy deserialization.

## Rules for Serde (LLM)

1. **ALWAYS use `#[serde(deny_unknown_fields)]` on config and API input structs** — catches typos and prevents silent data loss from misspelled fields
2. **ALWAYS use `#[serde(rename_all = "camelCase")]` for JSON APIs** — Rust convention is snake_case, JavaScript/JSON convention is camelCase; explicit rename prevents mismatches
3. **NEVER use `#[serde(untagged)]` when good error messages matter** — untagged enums produce "data did not match any variant" with no detail about what failed
4. **ALWAYS use `#[serde(transparent)]` for newtype wrappers** — without it, `UserId(42)` serializes as `{"0": 42}` instead of `42`
5. **ALWAYS use `#[serde(skip_serializing_if = "Option::is_none")]` on Option fields** — avoids cluttering output with `"field": null` entries
6. **PREFER `Cow<'a, str>` with `#[serde(borrow)]` for zero-copy deserialization** — borrows from input when possible, owns when escaping required
7. **NEVER deserialize untrusted input without size limits** — use `bincode::options().with_limit()` or validate lengths; unbounded deserialization enables DoS
8. **ALWAYS test serde round-trips** — serialize then deserialize and assert equality; catches asymmetric implementations

### Common Mistakes (BAD/GOOD)

**Missing `transparent` on newtypes:**
```rust
// BAD: UserId(42) serializes as {"0": 42}
#[derive(Serialize, Deserialize)]
struct UserId(u64);

// GOOD: UserId(42) serializes as 42
#[derive(Serialize, Deserialize)]
#[serde(transparent)]
struct UserId(u64);
```

**Using `untagged` for important error messages:**
```rust
// BAD: error is "data did not match any variant" — useless to debug
#[derive(Deserialize)]
#[serde(untagged)]
enum Config { Inline(String), Full { host: String, port: u16 } }

// GOOD: internally tagged — error says which variant and which field failed
#[derive(Deserialize)]
#[serde(tag = "type")]
enum Config {
    #[serde(rename = "inline")]
    Inline { value: String },
    #[serde(rename = "full")]
    Full { host: String, port: u16 },
}
```

**Indexing `Value` expecting `None`:**
```rust
// BAD: value["missing"] returns Value::Null, not None — can silently propagate
let name = &data["users"][0]["name"];  // Null if any key is wrong

// GOOD: use .get() for explicit None handling
let name = data.get("users")
    .and_then(|u| u.get(0))
    .and_then(|u| u.get("name"))
    .and_then(|n| n.as_str());
```

## Derive Basics

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct User {
    pub id: u64,
    pub username: String,
    pub email: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

// Serialize to JSON
let user = User { id: 1, username: "alice".into(), email: "a@b.com".into(), created_at: chrono::Utc::now() };
let json = serde_json::to_string(&user)?;
let pretty = serde_json::to_string_pretty(&user)?;

// Deserialize from JSON
let user: User = serde_json::from_str(&json)?;

// Works with any serde-compatible format
let toml_str = toml::to_string(&user)?;
let yaml_str = serde_yaml::to_string(&user)?;
let bytes = bincode::serialize(&user)?;
```

## Field Attributes

### Renaming

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]  // All fields: snake_case -> camelCase
pub struct ApiResponse {
    pub user_id: u64,           // → "userId"
    pub display_name: String,   // → "displayName"
    pub is_active: bool,        // → "isActive"

    #[serde(rename = "type")]   // Override individual field
    pub kind: String,           // → "type" (reserved word in Rust)
}

// Other rename_all options:
// "snake_case", "camelCase", "PascalCase", "SCREAMING_SNAKE_CASE",
// "kebab-case", "SCREAMING-KEBAB-CASE", "lowercase", "UPPERCASE"
```

### Default Values

```rust
#[derive(Serialize, Deserialize)]
pub struct Config {
    pub host: String,

    #[serde(default = "default_port")]
    pub port: u16,

    #[serde(default)]  // Uses Default::default() → 0, false, "", Vec::new(), etc.
    pub workers: usize,

    #[serde(default = "default_log_level")]
    pub log_level: String,
}

fn default_port() -> u16 { 8080 }
fn default_log_level() -> String { "info".to_string() }

// Missing fields use defaults:
// {"host": "localhost"} → Config { host: "localhost", port: 8080, workers: 0, log_level: "info" }
```

### Skip, Flatten, Alias

```rust
#[derive(Serialize, Deserialize)]
pub struct Document {
    pub title: String,

    #[serde(skip)]  // Skip both serialization and deserialization
    pub internal_cache: Option<Vec<u8>>,

    #[serde(skip_serializing)]  // Include when deserializing, omit when serializing
    pub password_hash: String,

    #[serde(skip_deserializing, default)]  // Include when serializing, skip when deserializing
    pub computed_field: String,

    #[serde(skip_serializing_if = "Option::is_none")]  // Omit if None
    pub description: Option<String>,

    #[serde(skip_serializing_if = "Vec::is_empty")]
    pub tags: Vec<String>,

    #[serde(flatten)]  // Merge fields from nested struct into parent
    pub metadata: Metadata,

    #[serde(alias = "user_name", alias = "userName")]  // Accept multiple names
    pub username: String,
}

#[derive(Serialize, Deserialize)]
pub struct Metadata {
    pub created_at: String,
    pub updated_at: String,
}

// With flatten, JSON looks like:
// {"title": "...", "created_at": "...", "updated_at": "...", ...}
// NOT: {"title": "...", "metadata": {"created_at": "...", "updated_at": "..."}}
```

### Custom Serialization Functions

```rust
use serde::{Serializer, Deserializer};

#[derive(Serialize, Deserialize)]
pub struct Event {
    pub name: String,

    #[serde(serialize_with = "serialize_duration", deserialize_with = "deserialize_duration")]
    pub duration: std::time::Duration,

    #[serde(with = "chrono::serde::ts_seconds")]  // Use module with both ser/de
    pub timestamp: chrono::DateTime<chrono::Utc>,
}

fn serialize_duration<S: Serializer>(d: &std::time::Duration, s: S) -> Result<S::Ok, S::Error> {
    s.serialize_f64(d.as_secs_f64())
}

fn deserialize_duration<'de, D: Deserializer<'de>>(d: D) -> Result<std::time::Duration, D::Error> {
    let secs: f64 = Deserialize::deserialize(d)?;
    Ok(std::time::Duration::from_secs_f64(secs))
}
```

### Serde `with` Module Pattern

```rust
// Reusable serialization module
mod hex_bytes {
    use serde::{Serializer, Deserializer, Deserialize};

    pub fn serialize<S: Serializer>(bytes: &[u8], s: S) -> Result<S::Ok, S::Error> {
        s.serialize_str(&hex::encode(bytes))
    }

    pub fn deserialize<'de, D: Deserializer<'de>>(d: D) -> Result<Vec<u8>, D::Error> {
        let s = String::deserialize(d)?;
        hex::decode(&s).map_err(serde::de::Error::custom)
    }
}

#[derive(Serialize, Deserialize)]
pub struct CryptoKey {
    #[serde(with = "hex_bytes")]
    pub key_data: Vec<u8>,  // Serializes as hex string: "a1b2c3..."
}
```

## Enum Representations

Serde supports four enum serialization strategies:

### Externally Tagged (Default)

```rust
#[derive(Serialize, Deserialize)]
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Point,
}

// JSON: {"Circle": {"radius": 5.0}}
// JSON: {"Rectangle": {"width": 10.0, "height": 20.0}}
// JSON: "Point"
```

### Internally Tagged

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
enum Event {
    UserCreated { user_id: u64, email: String },
    OrderPlaced { order_id: u64, total: f64 },
    SystemStarted,
}

// JSON: {"type": "UserCreated", "user_id": 42, "email": "a@b.com"}
// JSON: {"type": "OrderPlaced", "order_id": 1, "total": 99.99}
// JSON: {"type": "SystemStarted"}
```

### Adjacently Tagged

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type", content = "data")]
enum Message {
    Text(String),
    Image { url: String, width: u32, height: u32 },
    Ping,
}

// JSON: {"type": "Text", "data": "hello"}
// JSON: {"type": "Image", "data": {"url": "...", "width": 800, "height": 600}}
// JSON: {"type": "Ping"}
```

### Untagged

```rust
#[derive(Serialize, Deserialize)]
#[serde(untagged)]
enum Value {
    Integer(i64),
    Float(f64),
    Text(String),
    Boolean(bool),
    Array(Vec<Value>),
}

// JSON: 42          → Value::Integer(42)
// JSON: "hello"     → Value::Text("hello")
// JSON: [1, "two"]  → Value::Array(...)
// Note: tries variants in order, first match wins
// Error messages are poor — "data did not match any variant"
```

### Enum Variant Attributes

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "status")]
enum OrderStatus {
    #[serde(rename = "pending")]
    Pending,

    #[serde(rename = "processing")]
    Processing { started_at: String },

    #[serde(rename = "shipped")]
    Shipped { tracking_number: String },

    #[serde(other)]  // Catch-all for unknown variants (deserialization only)
    Unknown,

    #[serde(skip)]  // Never serialize/deserialize this variant
    Internal(InternalState),
}
```

## Custom Serialize/Deserialize

### Implement Serialize

```rust
use serde::{Serialize, Serializer};
use serde::ser::SerializeStruct;

struct Color { r: u8, g: u8, b: u8 }

impl Serialize for Color {
    fn serialize<S: Serializer>(&self, serializer: S) -> Result<S::Ok, S::Error> {
        // Option 1: Serialize as string "#rrggbb"
        serializer.serialize_str(&format!("#{:02x}{:02x}{:02x}", self.r, self.g, self.b))
    }
}

// Or as a struct:
impl Serialize for Color {
    fn serialize<S: Serializer>(&self, serializer: S) -> Result<S::Ok, S::Error> {
        let mut state = serializer.serialize_struct("Color", 3)?;
        state.serialize_field("r", &self.r)?;
        state.serialize_field("g", &self.g)?;
        state.serialize_field("b", &self.b)?;
        state.end()
    }
}
```

### Implement Deserialize (Visitor Pattern)

```rust
use serde::{Deserialize, Deserializer};
use serde::de::{self, Visitor};

impl<'de> Deserialize<'de> for Color {
    fn deserialize<D: Deserializer<'de>>(deserializer: D) -> Result<Self, D::Error> {
        struct ColorVisitor;

        impl<'de> Visitor<'de> for ColorVisitor {
            type Value = Color;

            fn expecting(&self, formatter: &mut std::fmt::Formatter) -> std::fmt::Result {
                formatter.write_str("a color string like '#ff0000' or an RGB object")
            }

            // Accept string "#rrggbb"
            fn visit_str<E: de::Error>(self, value: &str) -> Result<Color, E> {
                let hex = value.trim_start_matches('#');
                if hex.len() != 6 {
                    return Err(E::custom(format!("invalid color length: {}", hex.len())));
                }
                let r = u8::from_str_radix(&hex[0..2], 16).map_err(E::custom)?;
                let g = u8::from_str_radix(&hex[2..4], 16).map_err(E::custom)?;
                let b = u8::from_str_radix(&hex[4..6], 16).map_err(E::custom)?;
                Ok(Color { r, g, b })
            }

            // Accept map {"r": 255, "g": 0, "b": 0}
            fn visit_map<A: de::MapAccess<'de>>(self, mut map: A) -> Result<Color, A::Error> {
                let mut r = None;
                let mut g = None;
                let mut b = None;

                while let Some(key) = map.next_key::<String>()? {
                    match key.as_str() {
                        "r" => r = Some(map.next_value()?),
                        "g" => g = Some(map.next_value()?),
                        "b" => b = Some(map.next_value()?),
                        _ => { let _: serde::de::IgnoredAny = map.next_value()?; }
                    }
                }

                Ok(Color {
                    r: r.ok_or_else(|| de::Error::missing_field("r"))?,
                    g: g.ok_or_else(|| de::Error::missing_field("g"))?,
                    b: b.ok_or_else(|| de::Error::missing_field("b"))?,
                })
            }
        }

        deserializer.deserialize_any(ColorVisitor)
    }
}
```

### String-or-Struct Pattern

Accept both `"simple_value"` and `{"complex": "value"}`:

```rust
use serde::{Deserialize, Deserializer};
use serde::de::{self, Visitor};

#[derive(Debug, Deserialize)]
pub struct DatabaseConfig {
    pub host: String,
    pub port: u16,
    pub name: String,
}

impl DatabaseConfig {
    fn from_url(url: &str) -> Result<Self, String> {
        // Parse "postgres://host:port/name"
        // ...
    }
}

// Accept either a URL string or a full config object
fn deserialize_db_config<'de, D: Deserializer<'de>>(d: D) -> Result<DatabaseConfig, D::Error> {
    struct DbVisitor;

    impl<'de> Visitor<'de> for DbVisitor {
        type Value = DatabaseConfig;
        fn expecting(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
            f.write_str("a database URL string or config object")
        }

        fn visit_str<E: de::Error>(self, v: &str) -> Result<DatabaseConfig, E> {
            DatabaseConfig::from_url(v).map_err(E::custom)
        }

        fn visit_map<A: de::MapAccess<'de>>(self, map: A) -> Result<DatabaseConfig, A::Error> {
            Deserialize::deserialize(de::value::MapAccessDeserializer::new(map))
        }
    }

    d.deserialize_any(DbVisitor)
}
```

## Zero-Copy Deserialization

Borrow from input data instead of allocating new strings:

```rust
use std::borrow::Cow;

#[derive(Deserialize)]
pub struct LogEntry<'a> {
    #[serde(borrow)]
    pub message: &'a str,  // Zero-copy: points into input buffer

    #[serde(borrow)]
    pub source: Cow<'a, str>,  // Zero-copy when possible, owned when escaping needed

    pub level: u8,  // Copy types are always copied
}

// Usage
let input = r#"{"message": "hello", "source": "app", "level": 3}"#;
let entry: LogEntry = serde_json::from_str(input)?;
// entry.message borrows directly from `input` — no allocation

// Cow enables mixed ownership:
let input_escaped = r#"{"message": "hello", "source": "app \"v2\"", "level": 3}"#;
let entry: LogEntry = serde_json::from_str(input_escaped)?;
// entry.source is Cow::Owned because JSON escape processing required allocation
```

### When Zero-Copy Works

| Format | Zero-copy support |
|--------|------------------|
| `serde_json::from_str` | Yes (for unescaped strings) |
| `serde_json::from_slice` | Yes (for unescaped strings) |
| `serde_json::from_reader` | No (reader consumes input) |
| `toml::from_str` | Limited |
| `bincode` | No (binary format) |
| `serde_yaml` | No |

## Format-Specific Patterns

### JSON (serde_json)

```rust
use serde_json::{json, Value};

// Dynamic JSON with json! macro
let payload = json!({
    "name": "Alice",
    "age": 30,
    "tags": ["admin", "user"],
    "address": {
        "city": "Portland"
    }
});

// Access dynamic values
if let Some(name) = payload.get("name").and_then(Value::as_str) {
    println!("Name: {name}");
}

// Parse unknown structure
let value: Value = serde_json::from_str(input)?;
match &value {
    Value::Object(map) => { /* iterate fields */ }
    Value::Array(items) => { /* iterate items */ }
    _ => {}
}

// Streaming serialization for large data
let mut writer = std::io::BufWriter::new(file);
serde_json::to_writer(&mut writer, &data)?;

// Streaming deserialization
let reader = std::io::BufReader::new(file);
let data: MyStruct = serde_json::from_reader(reader)?;
```

**Feature flags:**

```toml
# Cargo.toml — preserve insertion order for JSON objects
serde_json = { version = "1", features = ["preserve_order"] }
# Without this, Object fields are sorted by BTreeMap (alphabetical order)
# With this, Object uses IndexMap (insertion order preserved)
```

**Value indexing behavior:**

```rust
let data = json!({"name": "Alice"});

// Index operator returns Value::Null on missing keys (never panics)
assert_eq!(data["missing"], Value::Null);
assert_eq!(data["missing"]["deep"], Value::Null);  // chains safely

// .get() returns Option — use this when you need to distinguish missing from null
assert_eq!(data.get("missing"), None);
assert_eq!(data.get("name"), Some(&json!("Alice")));
```

### JSON Pointer (RFC6901)

Navigate deeply nested `Value` without chained `.get()` calls:

```rust
use serde_json::json;

let data = json!({
    "users": [
        {"name": "Alice", "address": {"city": "Portland"}},
        {"name": "Bob", "address": {"city": "Seattle"}},
    ]
});

// Navigate with "/" separated path — indices work on arrays
let city = data.pointer("/users/0/address/city");
assert_eq!(city, Some(&json!("Portland")));

// Mutate deeply nested values
let mut data = data;
if let Some(city) = data.pointer_mut("/users/1/address/city") {
    *city = json!("Denver");
}

// Returns None for missing paths — no panics
assert_eq!(data.pointer("/users/99/name"), None);
```

### Stream Deserializer (NDJSON / Multi-Value Streams)

Process multiple JSON values from a single input — essential for NDJSON (newline-delimited JSON) log files, streaming APIs, and message queues:

```rust
use serde::Deserialize;
use serde_json::Deserializer;

#[derive(Deserialize, Debug)]
struct LogEntry {
    level: String,
    message: String,
}

// NDJSON: one JSON object per line
let ndjson = r#"{"level":"info","message":"started"}
{"level":"warn","message":"slow query"}
{"level":"error","message":"connection lost"}"#;

let stream = Deserializer::from_str(ndjson).into_iter::<LogEntry>();
for entry in stream {
    let entry = entry?;
    println!("[{}] {}", entry.level, entry.message);
}

// byte_offset() enables resumable parsing
let mut de = Deserializer::from_str(ndjson);
let mut stream = de.into_iter::<LogEntry>();
while let Some(Ok(entry)) = stream.next() {
    println!("Parsed up to byte {}", stream.byte_offset());
}
```

### Converting Between Typed and Dynamic (`to_value` / `from_value`)

```rust
use serde_json::{to_value, from_value, Value};

#[derive(Serialize, Deserialize, PartialEq, Debug)]
struct Config { host: String, port: u16 }

let config = Config { host: "localhost".into(), port: 8080 };

// Typed → Value (for dynamic manipulation)
let mut val: Value = to_value(&config)?;
val["port"] = json!(9090);  // Modify dynamically

// Value → Typed (after manipulation)
let modified: Config = from_value(val)?;
assert_eq!(modified.port, 9090);

// Value::take() — move out without cloning
let mut obj = json!({"data": [1, 2, 3], "meta": "info"});
let data = obj["data"].take();  // obj["data"] is now Null
assert_eq!(data, json!([1, 2, 3]));
```

### Deserialization-Time Validation

Reject invalid data during deserialization, not after:

```rust
use serde::{Deserialize, Deserializer};

#[derive(Deserialize)]
pub struct CreateUser {
    #[serde(deserialize_with = "non_empty_string")]
    pub username: String,

    #[serde(deserialize_with = "valid_port")]
    pub port: u16,
}

fn non_empty_string<'de, D: Deserializer<'de>>(d: D) -> Result<String, D::Error> {
    let s = String::deserialize(d)?;
    if s.trim().is_empty() {
        return Err(serde::de::Error::custom("must not be empty"));
    }
    Ok(s)
}

fn valid_port<'de, D: Deserializer<'de>>(d: D) -> Result<u16, D::Error> {
    let port = u16::deserialize(d)?;
    if port == 0 {
        return Err(serde::de::Error::custom("port must be non-zero"));
    }
    Ok(port)
}
```

### Round-Trip Testing Pattern

Always test that `serialize → deserialize` produces the original value:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn assert_round_trip<T>(value: &T)
    where
        T: Serialize + for<'de> Deserialize<'de> + PartialEq + std::fmt::Debug,
    {
        let json = serde_json::to_string(value).expect("serialize");
        let back: T = serde_json::from_str(&json).expect("deserialize");
        assert_eq!(value, &back, "round-trip failed for: {json}");
    }

    // Data-driven: test multiple values in one function
    fn test_encode_ok<T: Serialize + std::fmt::Debug>(cases: &[(T, &str)]) {
        for (value, expected) in cases {
            let json = serde_json::to_string(value).unwrap();
            assert_eq!(&json, *expected, "encoding {:?}", value);
        }
    }

    #[test]
    fn user_round_trips() {
        assert_round_trip(&User {
            id: UserId(42),
            username: "alice".into(),
            email: Email("a@b.com".into()),
        });
    }

    #[test]
    fn edge_cases_round_trip() {
        assert_round_trip(&Config { host: "".into(), port: 1 });  // empty string
        assert_round_trip(&Config { host: "日本語".into(), port: 65535 });  // unicode
    }
}
```

### TOML

```rust
// Cargo.toml: toml = "0.8"

#[derive(Serialize, Deserialize)]
struct AppConfig {
    #[serde(default)]
    server: ServerConfig,
    #[serde(default)]
    database: DatabaseConfig,
}

#[derive(Serialize, Deserialize)]
struct ServerConfig {
    #[serde(default = "default_host")]
    host: String,
    #[serde(default = "default_port")]
    port: u16,
}

// Read config file
let content = std::fs::read_to_string("config.toml")?;
let config: AppConfig = toml::from_str(&content)?;

// Write config file
let toml_string = toml::to_string_pretty(&config)?;
```

### CSV

```rust
// Cargo.toml: csv = "1"

#[derive(Serialize, Deserialize)]
struct Record {
    name: String,
    value: f64,
    category: String,
}

// Read CSV
let mut reader = csv::Reader::from_path("data.csv")?;
for result in reader.deserialize::<Record>() {
    let record = result?;
    println!("{}: {}", record.name, record.value);
}

// Write CSV
let mut writer = csv::Writer::from_path("output.csv")?;
writer.serialize(Record { name: "test".into(), value: 42.0, category: "A".into() })?;
writer.flush()?;
```

### bincode (Binary)

```rust
// Cargo.toml: bincode = "1"

#[derive(Serialize, Deserialize)]
struct Packet {
    header: u32,
    payload: Vec<u8>,
    checksum: u16,
}

// Compact binary serialization
let encoded: Vec<u8> = bincode::serialize(&packet)?;
let decoded: Packet = bincode::deserialize(&encoded)?;

// Size-limited deserialization (prevent DoS)
let decoded: Packet = bincode::options()
    .with_limit(1024 * 1024)  // 1MB max
    .deserialize(&encoded)?;
```

**bincode for internal messaging protocols:**

When building systems where components exchange messages (media servers, game engines, IPC), bincode provides compact, fast serialization without the overhead of JSON. Pattern from atm0s-media-server:

```rust
use derivative::Derivative;
use serde::{Deserialize, Serialize};

/// Peer-to-peer message packet — compact binary, not JSON
#[derive(Derivative, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[derivative(Debug)]
pub struct MessageChannelPacket {
    pub from: PeerId,
    #[derivative(Debug = "ignore")]  // Don't log binary payloads
    pub data: Vec<u8>,               // Opaque payload — could be nested serde types
}

impl MessageChannelPacket {
    pub fn serialize(&self) -> Vec<u8> {
        bincode::serialize(self).expect("MessageChannelPacket always serializable")
    }

    pub fn deserialize(data: &[u8]) -> Option<Self> {
        bincode::deserialize::<Self>(data).ok()  // Graceful failure on corrupt data
    }
}
```

**When bincode vs JSON:**
- **bincode**: Internal messages, IPC, protocol packets, performance-critical paths. No human readability needed.
- **JSON**: External APIs, config files, debugging, interop with other languages.
- **bincode + serde**: Same `#[derive(Serialize, Deserialize)]` structs work with both — switch format at the serialization call site, not the type definition.

## Container Attributes Reference

| Attribute | Effect |
|-----------|--------|
| `#[serde(rename_all = "...")]` | Rename all fields |
| `#[serde(deny_unknown_fields)]` | Error on unexpected fields |
| `#[serde(default)]` | Use Default for all missing fields |
| `#[serde(tag = "...")]` | Internal enum tagging |
| `#[serde(tag = "...", content = "...")]` | Adjacent enum tagging |
| `#[serde(untagged)]` | No tag, match by structure |
| `#[serde(bound = "...")]` | Custom trait bounds |
| `#[serde(from = "Type")]` | Deserialize via intermediate type |
| `#[serde(into = "Type")]` | Serialize via intermediate type |
| `#[serde(remote = "Type")]` | Derive for external types |
| `#[serde(transparent)]` | Newtype wrapper (serialize inner directly) |

## Field Attributes Reference

| Attribute | Effect |
|-----------|--------|
| `#[serde(rename = "...")]` | Rename this field |
| `#[serde(alias = "...")]` | Accept alternate name (deser only) |
| `#[serde(default)]` | Use Default if missing |
| `#[serde(default = "fn")]` | Use custom function if missing |
| `#[serde(flatten)]` | Merge nested struct into parent |
| `#[serde(skip)]` | Skip entirely |
| `#[serde(skip_serializing)]` | Skip when serializing |
| `#[serde(skip_deserializing)]` | Skip when deserializing |
| `#[serde(skip_serializing_if = "fn")]` | Conditional skip |
| `#[serde(serialize_with = "fn")]` | Custom serializer |
| `#[serde(deserialize_with = "fn")]` | Custom deserializer |
| `#[serde(with = "module")]` | Module with both ser/de |
| `#[serde(borrow)]` | Zero-copy borrowing |
| `#[serde(bound = "...")]` | Custom trait bounds |

## Derive for Remote Types

Serialize types from other crates without modifying them:

```rust
// Cannot add Serialize to chrono::Duration — it's external
// Use serde(remote) with a local definition

mod duration_serde {
    use serde::{Serialize, Deserialize, Serializer, Deserializer};

    #[derive(Serialize, Deserialize)]
    #[serde(remote = "chrono::Duration")]
    struct DurationDef {
        #[serde(getter = "chrono::Duration::num_milliseconds")]
        millis: i64,
    }

    impl From<DurationDef> for chrono::Duration {
        fn from(def: DurationDef) -> Self {
            chrono::Duration::milliseconds(def.millis)
        }
    }
}

#[derive(Serialize, Deserialize)]
struct Task {
    name: String,
    #[serde(with = "duration_serde")]
    timeout: chrono::Duration,
}
```

## Newtype Wrappers with transparent

```rust
#[derive(Debug, Serialize, Deserialize)]
#[serde(transparent)]
pub struct UserId(pub u64);

#[derive(Debug, Serialize, Deserialize)]
#[serde(transparent)]
pub struct Email(pub String);

#[derive(Serialize, Deserialize)]
pub struct User {
    pub id: UserId,      // Serializes as number: 42
    pub email: Email,    // Serializes as string: "a@b.com"
}

// JSON: {"id": 42, "email": "a@b.com"}
// NOT: {"id": {"0": 42}, "email": {"0": "a@b.com"}}
```

## Intermediate Types with from/into

```rust
// Deserialize via intermediate type for validation
#[derive(Deserialize)]
struct RawPort(u16);

#[derive(Debug, Serialize, Deserialize)]
#[serde(try_from = "RawPort")]
pub struct Port(u16);

impl TryFrom<RawPort> for Port {
    type Error = String;
    fn try_from(raw: RawPort) -> Result<Self, Self::Error> {
        if raw.0 == 0 {
            Err("Port cannot be 0".to_string())
        } else {
            Ok(Port(raw.0))
        }
    }
}

// Deserializing port: 0 now produces a serde error

// #[serde(from)] — infallible conversion (no validation needed)
#[derive(Deserialize)]
struct RawTimestamp(i64);

#[derive(Debug, Serialize, Deserialize)]
#[serde(from = "RawTimestamp")]
pub struct Timestamp(chrono::DateTime<chrono::Utc>);

impl From<RawTimestamp> for Timestamp {
    fn from(raw: RawTimestamp) -> Self {
        Timestamp(chrono::DateTime::from_timestamp(raw.0, 0).unwrap_or_default())
    }
}

// Use try_from when conversion can fail (validation)
// Use from when conversion is infallible (type transformation)
```

## Common Patterns

### Deny Unknown Fields (Strict Parsing)

```rust
#[derive(Deserialize)]
#[serde(deny_unknown_fields)]
pub struct StrictConfig {
    pub host: String,
    pub port: u16,
}
// Error if JSON has extra fields like "databse" (catches typos)
```

### Deserialize from Multiple Formats

```rust
#[derive(Deserialize)]
#[serde(untagged)]
enum StringOrNumber {
    String(String),
    Number(f64),
}

#[derive(Deserialize)]
struct FlexibleConfig {
    #[serde(deserialize_with = "deserialize_port")]
    port: u16,
}

fn deserialize_port<'de, D: Deserializer<'de>>(d: D) -> Result<u16, D::Error> {
    let value = StringOrNumber::deserialize(d)?;
    match value {
        StringOrNumber::Number(n) => Ok(n as u16),
        StringOrNumber::String(s) => s.parse().map_err(serde::de::Error::custom),
    }
}
// Accepts both: {"port": 8080} and {"port": "8080"}
```

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: serde essentials, derive basics, common attributes
- **[web-apis.md](web-apis.md)** — `Json<T>` extraction, API request/response serialization
- **[database.md](database.md)** — `FromRow` derive, query result mapping
- **[services.md](services.md)** — Protocol versioning, Redis value serialization, binary formats
- **[unsafe-ffi.md](unsafe-ffi.md)** — Manual byte serialization for wire protocols
