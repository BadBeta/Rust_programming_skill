# Macros in Rust

Declarative macros, procedural macros (derive, attribute, function-like), DSL patterns, and debugging techniques.

## Rules for Macros (LLM)

1. **NEVER use macros for what generics or traits can do** — macros are harder to debug, don't get type checking at definition site, and produce opaque error messages; only use when the type system cannot express the pattern
2. **ALWAYS test macro expansion with `cargo expand`** — verify the generated code is what you expect; install with `cargo install cargo-expand`
3. **ALWAYS use `$crate::` to reference items in macro definitions** — ensures the macro works when called from other crates; bare paths break cross-crate usage
4. **PREFER `darling` for proc macro attribute parsing** — hand-parsing `syn::Attribute` is error-prone and verbose; `darling` provides derive-based parsing with defaults and validation
5. **ALWAYS provide meaningful compile errors with `compile_error!` or `syn::Error`** — never silently generate wrong code; fail loudly with a span pointing to the problematic input

### Common Mistakes (BAD/GOOD)

**Error spans pointing to wrong location:**
```rust
// BAD: error points to macro crate, not user code
return Err(syn::Error::new(
    Span::call_site(),  // Points to macro invocation, not the problem
    "field must be pub",
));

// GOOD: error points to the field that caused the problem
return Err(syn::Error::new_spanned(
    &field.ident,  // Points directly at the offending field
    "field must be pub",
));
```

**Multiple evaluation of macro arguments:**
```rust
// BAD: evaluates expression multiple times — side effects doubled
macro_rules! max {
    ($a:expr, $b:expr) => { if $a > $b { $a } else { $b } };
}
max!(expensive_fn(), other_fn());  // Both called twice!

// GOOD: bind to variables first
macro_rules! max {
    ($a:expr, $b:expr) => {{
        let a = $a;
        let b = $b;
        if a > b { a } else { b }
    }};
}
```

**Missing trailing comma support:**
```rust
// BAD: rejects common Rust style with trailing comma
macro_rules! list {
    ($($item:expr),*) => { vec![$($item),*] };
}
list![1, 2, 3,];  // Error!

// GOOD: $(,)? accepts optional trailing comma
macro_rules! list {
    ($($item:expr),* $(,)?) => { vec![$($item),*] };
}
```

### Section Index

| Section | Topics |
|---------|--------|
| [Declarative Macros](#declarative-macros-macro_rules) | Basic syntax, fragment specifiers, repetition, recursive macros, TT muncher |
| [Code Generation Patterns](#code-generation-patterns) | Enum dispatch, impl blocks, trait derives, boilerplate reduction |
| [Procedural Macros](#procedural-macros) | TokenStream, syn/quote, derive macros, attribute macros, function-like |
| [Domain-Specific Languages](#domain-specific-languages-dsls) | SQL-like DSLs, HTML builders, config DSLs, DSL design principles |
| [Debugging Macros](#debugging-macros) | cargo expand, trace_macros!, log_syntax!, trybuild |
| [When to Use Macros vs Alternatives](#when-to-use-macros-vs-alternatives) | Macros vs generics vs traits vs const generics decision guide |
| [Real-World Macro Examples](#real-world-macro-examples-from-the-ecosystem) | serde, clap, tokio select!, sqlx query patterns |
| [Macro Best Practices](#macro-best-practices) | Design guidelines, common pitfalls, hygiene |
| [Key Crates](#key-crates-for-macro-development) | syn, quote, proc-macro2, darling, paste |

## Declarative Macros (macro_rules!)

### Basic Syntax

```rust
// Simplest macro — no arguments
macro_rules! say_hello {
    () => {
        println!("Hello, world!");
    };
}

say_hello!();  // Expands to: println!("Hello, world!");
```

### Fragment Specifiers (Designators)

All available designators for capturing macro input:

| Designator | Matches | Example |
|-----------|---------|---------|
| `expr` | Any expression | `1 + 2`, `foo()`, `vec![1]` |
| `ident` | Identifier | `my_var`, `MyStruct` |
| `ty` | Type | `i32`, `Vec<String>`, `&str` |
| `stmt` | Statement | `let x = 1;` |
| `block` | Block expression | `{ let x = 1; x }` |
| `path` | Type path | `std::collections::HashMap` |
| `tt` | Single token tree | Any single token or `()`/`[]`/`{}`-delimited group |
| `literal` | Literal value | `42`, `"hello"`, `true` |
| `pat` | Pattern | `Some(x)`, `(a, b)`, `_` |
| `vis` | Visibility qualifier | `pub`, `pub(crate)`, (empty) |
| `lifetime` | Lifetime | `'a`, `'static` |
| `meta` | Attribute content | `derive(Debug)`, `cfg(test)` |
| `item` | Top-level item | `fn foo() {}`, `struct Bar;` |

```rust
macro_rules! create_function {
    ($func_name:ident, $return_type:ty, $value:expr) => {
        fn $func_name() -> $return_type {
            $value
        }
    };
}

create_function!(get_answer, i32, 42);
create_function!(get_pi, f64, 3.14159);

// Using visibility specifier
macro_rules! create_pub_function {
    ($vis:vis $func_name:ident, $return_type:ty, $value:expr) => {
        $vis fn $func_name() -> $return_type {
            $value
        }
    };
}

create_pub_function!(pub get_version, &'static str, "1.0.0");
create_pub_function!(pub(crate) get_name, &'static str, "myapp");
```

### Repetition Patterns

```rust
// $(...)* — zero or more
// $(...)+ — one or more
// $(...)? — zero or one

macro_rules! vec_of_strings {
    ($($element:expr),* $(,)?) => {{
        let mut v = Vec::new();
        $(v.push($element.to_string());)*
        v
    }};
}

let v = vec_of_strings!["a", "b", "c"];

// HashMap literal with separator
macro_rules! hash_map {
    ($($key:expr => $value:expr),* $(,)?) => {{
        let mut map = std::collections::HashMap::new();
        $(map.insert($key, $value);)*
        map
    }};
}

let map = hash_map! {
    "a" => 1,
    "b" => 2,
};

// BTreeMap literal
macro_rules! btree_map {
    ($($key:expr => $value:expr),* $(,)?) => {{
        let mut map = std::collections::BTreeMap::new();
        $(map.insert($key, $value);)*
        map
    }};
}

// HashSet literal
macro_rules! hash_set {
    ($($element:expr),* $(,)?) => {{
        let mut set = std::collections::HashSet::new();
        $(set.insert($element);)*
        set
    }};
}

let active_users = hash_set!["alice", "bob", "charlie"];
```

### Multiple Match Arms

```rust
macro_rules! log {
    ($msg:expr) => {
        println!("[INFO] {}", $msg);
    };
    ($level:ident, $msg:expr) => {
        println!("[{}] {}", stringify!($level), $msg);
    };
    ($level:ident, $fmt:expr, $($arg:expr),*) => {
        println!("[{}] {}", stringify!($level), format!($fmt, $($arg),*));
    };
}

log!("Simple message");
log!(WARN, "Warning message");
log!(ERROR, "Error: {}", "file not found");
```

Arm order matters — the macro tries each arm top to bottom:

```rust
macro_rules! overloaded {
    // Most specific first
    ($x:expr, $y:expr, $z:expr) => { $x + $y + $z };
    ($x:expr, $y:expr) => { $x + $y };
    ($x:expr) => { $x };
    () => { 0 };
}
```

### Macro Hygiene

Declarative macros in Rust are partially hygienic — local variable bindings inside a macro won't conflict with identifiers in the caller's scope:

```rust
macro_rules! swap {
    ($a:expr, $b:expr) => {{
        let temp = $a;  // Hygienic: won't conflict with caller's 'temp'
        $a = $b;
        $b = temp;
    }};
}

let mut x = 1;
let mut y = 2;
let temp = 100;  // No conflict with macro's internal 'temp'
swap!(x, y);
assert_eq!(x, 2);
assert_eq!(y, 1);
assert_eq!(temp, 100);  // Unchanged
```

However, type names and function names are NOT hygienic — they resolve in the caller's scope:

```rust
macro_rules! create_struct {
    ($name:ident) => {
        // This struct name IS visible to the caller
        struct $name {
            value: i32,
        }
    };
}

create_struct!(Point);
let p = Point { value: 42 };  // Works — struct name leaks into caller scope
```

### Recursive Macros

```rust
// Counting elements
macro_rules! count {
    () => { 0usize };
    ($head:tt $($tail:tt)*) => { 1usize + count!($($tail)*) };
}

assert_eq!(count!(a b c d), 4);

// TT muncher pattern — process tokens one at a time
macro_rules! print_all {
    () => {};
    ($head:expr) => {
        println!("{}", $head);
    };
    ($head:expr, $($tail:expr),*) => {
        println!("{}", $head);
        print_all!($($tail),*);
    };
}

print_all!("one", "two", "three");
```

### Internal Rules (Private Arms)

Use `@` prefix for internal helper arms that shouldn't be called directly:

```rust
macro_rules! parse_list {
    // Public entry point
    ([$($items:expr),*]) => {
        parse_list!(@build [] $($items),*)
    };

    // Internal: accumulate processed items
    (@build [$($processed:expr),*]) => {
        vec![$($processed),*]
    };
    (@build [$($processed:expr),*] $head:expr $(, $tail:expr)*) => {
        parse_list!(@build [$($processed,)* $head.to_string()] $($tail),*)
    };
}

let v = parse_list!([1, 2, 3]);  // vec!["1", "2", "3"]
```

## Code Generation Patterns

### Generating Enum + Display

```rust
macro_rules! impl_display {
    ($name:ident { $($variant:ident),* }) => {
        enum $name {
            $($variant,)*
        }

        impl std::fmt::Display for $name {
            fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
                match self {
                    $(Self::$variant => write!(f, stringify!($variant)),)*
                }
            }
        }
    };
}

impl_display!(Status { Pending, Running, Complete, Failed });
```

### Generating Getters/Setters with paste

The `paste` crate enables identifier manipulation (concatenation, case conversion) inside macros:

```rust
// Cargo.toml: paste = "1.0"

macro_rules! accessors {
    ($struct:ident { $($field:ident: $type:ty),* }) => {
        impl $struct {
            $(
                pub fn $field(&self) -> &$type {
                    &self.$field
                }

                paste::paste! {
                    pub fn [<set_ $field>](&mut self, value: $type) {
                        self.$field = value;
                    }

                    pub fn [<with_ $field>](mut self, value: $type) -> Self {
                        self.$field = value;
                        self
                    }
                }
            )*
        }
    };
}

struct Config {
    host: String,
    port: u16,
}

accessors!(Config { host: String, port: u16 });

// Generates:
//   fn host(&self) -> &String
//   fn set_host(&mut self, value: String)
//   fn with_host(mut self, value: String) -> Self
//   fn port(&self) -> &u16
//   fn set_port(&mut self, value: u16)
//   fn with_port(mut self, value: u16) -> Self
```

### Test Case Generation

```rust
macro_rules! test_cases {
    ($test_fn:ident, $($name:ident: $input:expr => $expected:expr),* $(,)?) => {
        $(
            #[test]
            fn $name() {
                assert_eq!($test_fn($input), $expected);
            }
        )*
    };
}

fn double(x: i32) -> i32 { x * 2 }

test_cases!(double,
    test_zero: 0 => 0,
    test_positive: 5 => 10,
    test_negative: -3 => -6,
);

// Parameterized test with multiple inputs
macro_rules! test_matrix {
    ($name:ident, $test_fn:ident, [$(($($arg:expr),+) => $expected:expr),* $(,)?]) => {
        mod $name {
            use super::*;
            $(
                paste::paste! {
                    #[test]
                    fn [<test_ $expected>]() {
                        assert_eq!($test_fn($($arg),+), $expected);
                    }
                }
            )*
        }
    };
}
```

### Error Definition Macro

```rust
macro_rules! define_errors {
    ($($name:ident($msg:expr)),* $(,)?) => {
        $(
            #[derive(Debug)]
            pub struct $name;

            impl std::fmt::Display for $name {
                fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
                    write!(f, $msg)
                }
            }

            impl std::error::Error for $name {}
        )*
    };
}

define_errors!(
    NotFoundError("Resource not found"),
    InvalidInputError("Invalid input provided"),
    TimeoutError("Operation timed out"),
);
```

### Trait Implementation Macro

```rust
// Generate From implementations for enum variants
macro_rules! impl_from_variants {
    ($enum:ident { $($variant:ident($inner:ty)),* $(,)? }) => {
        $(
            impl From<$inner> for $enum {
                fn from(val: $inner) -> Self {
                    $enum::$variant(val)
                }
            }
        )*
    };
}

enum AppError {
    Io(std::io::Error),
    Parse(std::num::ParseIntError),
    Custom(String),
}

impl_from_variants!(AppError {
    Io(std::io::Error),
    Parse(std::num::ParseIntError),
    Custom(String),
});
```

### Configuration Builder Macro

```rust
macro_rules! config {
    (
        $name:ident {
            $($field:ident: $type:ty = $default:expr),* $(,)?
        }
    ) => {
        #[derive(Debug, Clone)]
        pub struct $name {
            $(pub $field: $type,)*
        }

        impl Default for $name {
            fn default() -> Self {
                Self {
                    $($field: $default,)*
                }
            }
        }

        impl $name {
            pub fn new() -> Self {
                Self::default()
            }

            $(
                pub fn $field(mut self, value: $type) -> Self {
                    self.$field = value;
                    self
                }
            )*
        }
    };
}

// Usage: Fluent configuration builder
config! {
    ServerConfig {
        host: String = "127.0.0.1".to_string(),
        port: u16 = 8080,
        max_connections: usize = 100,
        timeout_ms: u64 = 30000,
        debug: bool = false,
    }
}

let config = ServerConfig::new()
    .host("0.0.0.0".to_string())
    .port(3000)
    .debug(true);
```

### Bitflag Macro

```rust
macro_rules! bitflags {
    (
        $vis:vis struct $name:ident: $ty:ty {
            $($flag:ident = $value:expr),* $(,)?
        }
    ) => {
        #[derive(Debug, Clone, Copy, PartialEq, Eq)]
        $vis struct $name($ty);

        impl $name {
            $(pub const $flag: Self = Self($value);)*

            pub fn empty() -> Self { Self(0) }
            pub fn contains(self, other: Self) -> bool { (self.0 & other.0) == other.0 }
            pub fn insert(&mut self, other: Self) { self.0 |= other.0; }
            pub fn remove(&mut self, other: Self) { self.0 &= !other.0; }
            pub fn toggle(&mut self, other: Self) { self.0 ^= other.0; }
        }

        impl std::ops::BitOr for $name {
            type Output = Self;
            fn bitor(self, rhs: Self) -> Self { Self(self.0 | rhs.0) }
        }

        impl std::ops::BitAnd for $name {
            type Output = Self;
            fn bitand(self, rhs: Self) -> Self { Self(self.0 & rhs.0) }
        }
    };
}

bitflags! {
    pub struct Permissions: u32 {
        READ    = 0b001,
        WRITE   = 0b010,
        EXECUTE = 0b100,
    }
}

let mut perms = Permissions::READ | Permissions::WRITE;
assert!(perms.contains(Permissions::READ));
perms.remove(Permissions::WRITE);
```

## Procedural Macros

Proc macros operate on the token stream at compile time. They require a separate crate with `proc-macro = true`.

### Crate Setup

```toml
# Cargo.toml for the proc-macro crate
[lib]
proc-macro = true

[dependencies]
syn = { version = "2.0", features = ["full"] }
quote = "1.0"
proc-macro2 = "1.0"
```

Typical workspace layout:

```
my-project/
├── Cargo.toml          # workspace
├── my-lib/
│   ├── Cargo.toml      # depends on my-lib-derive
│   └── src/lib.rs
├── my-lib-derive/
│   ├── Cargo.toml      # proc-macro = true
│   └── src/lib.rs
```

### The syn/quote/proc-macro2 Ecosystem

- **`syn`** — Parses Rust token streams into an AST. `DeriveInput` for derives, `ItemFn` for functions, `ItemStruct` for structs.
- **`quote`** — Generates Rust code from templates. `quote! { ... }` produces a `TokenStream`. Use `#variable` for interpolation.
- **`proc-macro2`** — Re-exports of `proc_macro` types that work in unit tests (the real `proc_macro` types only work during compilation).

```rust
use proc_macro2::TokenStream;  // For testability
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

// quote interpolation:
let name = quote! { MyStruct };
let expanded = quote! {
    impl #name {
        fn new() -> Self { Self {} }
    }
};

// Repetition in quote:
let fields = vec![quote! { x: i32 }, quote! { y: f64 }];
let expanded = quote! {
    struct Point {
        #(#fields,)*
    }
};
```

### Derive Macros

The most common proc macro type. Generates `impl` blocks for annotated types.

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput, Data, Fields};

#[proc_macro_derive(Builder)]
pub fn derive_builder(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;
    let builder_name = format!("{}Builder", name);
    let builder_ident = syn::Ident::new(&builder_name, name.span());

    let fields = match &input.data {
        Data::Struct(data) => match &data.fields {
            Fields::Named(fields) => &fields.named,
            _ => panic!("Only named fields supported"),
        },
        _ => panic!("Only structs supported"),
    };

    let builder_fields = fields.iter().map(|f| {
        let name = &f.ident;
        let ty = &f.ty;
        quote! { #name: Option<#ty> }
    });

    let builder_methods = fields.iter().map(|f| {
        let name = &f.ident;
        let ty = &f.ty;
        quote! {
            pub fn #name(mut self, #name: #ty) -> Self {
                self.#name = Some(#name);
                self
            }
        }
    });

    let build_fields = fields.iter().map(|f| {
        let name = &f.ident;
        quote! {
            #name: self.#name.ok_or(concat!(
                stringify!(#name), " is not set"
            ))?
        }
    });

    let expanded = quote! {
        impl #name {
            pub fn builder() -> #builder_ident {
                #builder_ident::default()
            }
        }

        #[derive(Default)]
        pub struct #builder_ident {
            #(#builder_fields,)*
        }

        impl #builder_ident {
            #(#builder_methods)*

            pub fn build(self) -> Result<#name, &'static str> {
                Ok(#name {
                    #(#build_fields,)*
                })
            }
        }
    };

    TokenStream::from(expanded)
}

// Usage:
// #[derive(Builder)]
// struct Config {
//     host: String,
//     port: u16,
// }
//
// let config = Config::builder()
//     .host("localhost".into())
//     .port(8080)
//     .build()?;
```

### Handling Helper Attributes in Derives

Derive macros can register helper attributes that are parsed on fields/variants:

```rust
#[proc_macro_derive(MyTrait, attributes(my_attr))]
pub fn derive_my_trait(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);

    if let Data::Struct(data) = &input.data {
        if let Fields::Named(fields) = &data.fields {
            for field in &fields.named {
                for attr in &field.attrs {
                    if attr.path().is_ident("my_attr") {
                        // Parse attribute arguments
                        let nested: syn::Ident = attr.parse_args().unwrap();
                        if nested == "skip" {
                            // Skip this field in code generation
                        }
                    }
                }
            }
        }
    }

    // ...
    TokenStream::new()
}

// Usage:
// #[derive(MyTrait)]
// struct Example {
//     #[my_attr(skip)]
//     internal: String,
//     name: String,
// }
```

### Attribute Macros

Transform entire items (functions, structs, modules). Unlike derive macros, attribute macros can modify the original item.

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, ItemFn};

#[proc_macro_attribute]
pub fn log_calls(_args: TokenStream, input: TokenStream) -> TokenStream {
    let input_fn = parse_macro_input!(input as ItemFn);
    let fn_name = &input_fn.sig.ident;
    let fn_block = &input_fn.block;
    let fn_sig = &input_fn.sig;
    let fn_vis = &input_fn.vis;
    let fn_attrs = &input_fn.attrs;

    let expanded = quote! {
        #(#fn_attrs)*
        #fn_vis #fn_sig {
            println!("Calling function: {}", stringify!(#fn_name));
            let _start = std::time::Instant::now();
            let result = (|| #fn_block)();
            println!("{} completed in {:?}", stringify!(#fn_name), _start.elapsed());
            result
        }
    };

    TokenStream::from(expanded)
}

// Usage:
// #[log_calls]
// fn calculate_sum(a: i32, b: i32) -> i32 {
//     a + b
// }
```

Attribute macro with arguments:

```rust
#[proc_macro_attribute]
pub fn rate_limit(args: TokenStream, input: TokenStream) -> TokenStream {
    let limit: syn::LitInt = parse_macro_input!(args as syn::LitInt);
    let input_fn = parse_macro_input!(input as ItemFn);
    let fn_name = &input_fn.sig.ident;
    let fn_block = &input_fn.block;
    let fn_sig = &input_fn.sig;

    let expanded = quote! {
        #fn_sig {
            static LIMITER: std::sync::LazyLock<tokio::sync::Semaphore> =
                std::sync::LazyLock::new(|| tokio::sync::Semaphore::new(#limit));

            let _permit = LIMITER.acquire().await.unwrap();
            #fn_block
        }
    };

    TokenStream::from(expanded)
}

// Usage:
// #[rate_limit(10)]
// async fn handle_request() -> Response { ... }
```

### Function-Like Proc Macros

Look like regular macro invocations but are implemented as proc macros:

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, LitStr};

#[proc_macro]
pub fn make_greeting(input: TokenStream) -> TokenStream {
    let name = parse_macro_input!(input as LitStr);
    let greeting = format!("Hello, {}!", name.value());

    let expanded = quote! {
        #greeting
    };

    TokenStream::from(expanded)
}

// Usage:
// let msg = make_greeting!("World");  // "Hello, World!"
```

SQL-like function proc macro (compile-time query validation):

```rust
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
    let query = parse_macro_input!(input as LitStr);
    let query_str = query.value();

    // Validate SQL at compile time
    if !query_str.to_uppercase().starts_with("SELECT")
        && !query_str.to_uppercase().starts_with("INSERT")
        && !query_str.to_uppercase().starts_with("UPDATE")
        && !query_str.to_uppercase().starts_with("DELETE")
    {
        return syn::Error::new(
            query.span(),
            "Query must start with SELECT, INSERT, UPDATE, or DELETE"
        ).to_compile_error().into();
    }

    let expanded = quote! {
        sqlx::query(#query_str)
    };

    TokenStream::from(expanded)
}

// Usage:
// let rows = sql!("SELECT id, name FROM users WHERE active = true")
//     .fetch_all(&pool).await?;
```

### Error Handling in Proc Macros

Always use `syn::Error` for proper error reporting with span information:

```rust
#[proc_macro_derive(Validate)]
pub fn derive_validate(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);

    match &input.data {
        Data::Struct(data) => match &data.fields {
            Fields::Named(fields) => {
                // Generate validation code
                generate_validation(&input, fields)
            }
            Fields::Unnamed(_) => {
                syn::Error::new_spanned(
                    &input,
                    "Validate cannot be derived for tuple structs"
                ).to_compile_error().into()
            }
            Fields::Unit => {
                syn::Error::new_spanned(
                    &input,
                    "Validate cannot be derived for unit structs"
                ).to_compile_error().into()
            }
        },
        Data::Enum(_) => {
            syn::Error::new_spanned(
                &input,
                "Validate can only be derived for structs, not enums"
            ).to_compile_error().into()
        }
        Data::Union(_) => {
            syn::Error::new_spanned(
                &input,
                "Validate cannot be derived for unions"
            ).to_compile_error().into()
        }
    }
}

// Collecting multiple errors:
fn validate_fields(fields: &syn::FieldsNamed) -> Result<TokenStream, syn::Error> {
    let mut errors = Vec::new();

    for field in &fields.named {
        if field.ident.as_ref().map_or(false, |i| i.to_string().starts_with('_')) {
            errors.push(syn::Error::new_spanned(
                field,
                "Fields starting with _ cannot be validated"
            ));
        }
    }

    if let Some(first) = errors.into_iter().reduce(|mut a, b| { a.combine(b); a }) {
        Err(first)
    } else {
        Ok(quote! { /* generated code */ }.into())
    }
}
```

### Span-Preserving Code Generation (`quote_spanned!`)

Use `quote_spanned!` to ensure compiler errors point to the user's source code, not your macro crate. This is the difference between a helpful error and "error in generated code in <proc-macro>":

```rust
use quote::quote_spanned;
use syn::spanned::Spanned;

// BAD: error points to macro crate, not user code
let expanded = quote! {
    impl std::fmt::Display for #name {
        fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
            write!(f, "{}", self.#field_name)
        }
    }
};

// GOOD: error points to the field that caused the problem
let field_span = field.span();
let expanded = quote_spanned! {field_span=>
    impl std::fmt::Display for #name {
        fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
            write!(f, "{}", self.#field_name)
        }
    }
};
// If #field_name's type doesn't impl Display, the error points
// to the field declaration in the user's code, not to the macro.
```

### Conditional Code Generation with `Option<TokenStream>`

Generate optional trait methods without `if/else` boilerplate — `None` expands to nothing when interpolated:

```rust
use quote::quote;
use proc_macro2::TokenStream;

fn generate_impl(input: &DeriveInput, has_source: bool, has_backtrace: bool) -> TokenStream {
    // Optional methods — None produces no output
    let source_method: Option<TokenStream> = if has_source {
        let field = &source_field;
        Some(quote! {
            fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
                Some(&self.#field)
            }
        })
    } else {
        None
    };

    let backtrace_method: Option<TokenStream> = if has_backtrace {
        Some(quote! {
            fn backtrace(&self) -> Option<&std::backtrace::Backtrace> {
                Some(&self.backtrace)
            }
        })
    } else {
        None
    };

    // Interpolating None produces nothing — clean and composable
    quote! {
        impl std::error::Error for #name {
            #source_method
            #backtrace_method
        }
    }
}
```

### Modular Proc Macro Architecture

Production proc macros (serde, thiserror) split into distinct phases:

```
my_derive/
├── src/
│   ├── lib.rs        # Entry point — parse_macro_input!, dispatch, error conversion
│   ├── parse.rs      # Attribute parsing — extract config from syn types
│   ├── validate.rs   # Validate attribute combinations, field constraints
│   └── expand.rs     # Code generation — all quote! calls live here
```

```rust
// lib.rs — minimal entry point
#[proc_macro_derive(MyDerive, attributes(my_attr))]
pub fn derive_my(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    expand::derive(&input)
        .unwrap_or_else(syn::Error::into_compile_error)
        .into()
}

// expand.rs — returns Result for clean error propagation
pub fn derive(input: &DeriveInput) -> syn::Result<proc_macro2::TokenStream> {
    let config = parse::extract_config(input)?;
    validate::check_constraints(&config)?;
    let expanded = generate_impl(input, &config);
    Ok(expanded)
}
```

**Benefits:** testable without proc-macro harness, composable error handling with `?`, clear separation of concerns.

### Testing Proc Macros

Use `trybuild` for compile-fail tests and regular tests for expansion verification:

```rust
// In tests/
#[test]
fn test_derive_compiles() {
    let t = trybuild::TestCases::new();
    t.pass("tests/pass/*.rs");
    t.compile_fail("tests/fail/*.rs");
}

// tests/pass/basic.rs
use my_derive::Builder;

#[derive(Builder)]
struct Config {
    host: String,
    port: u16,
}

fn main() {
    let config = Config::builder()
        .host("localhost".into())
        .port(8080)
        .build()
        .unwrap();
}

// tests/fail/not_struct.rs
use my_derive::Builder;

#[derive(Builder)]  // Should fail: not a struct
enum Status {
    Active,
    Inactive,
}

fn main() {}
```

## Domain-Specific Languages (DSLs)

### State Machine DSL

```rust
macro_rules! state_machine {
    (
        $machine_name:ident {
            initial: $initial:ident,
            states: { $($state:ident),+ $(,)? },
            events: { $($event:ident),+ $(,)? },
            transitions: {
                $($from:ident + $evt:ident => $to:ident),+ $(,)?
            }
        }
    ) => {
        #[derive(Debug, Clone, Copy, PartialEq, Eq)]
        pub enum State {
            $($state),+
        }

        #[derive(Debug, Clone, Copy, PartialEq, Eq)]
        pub enum Event {
            $($event),+
        }

        pub struct $machine_name {
            state: State,
        }

        impl $machine_name {
            pub fn new() -> Self {
                Self { state: State::$initial }
            }

            pub fn state(&self) -> State {
                self.state
            }

            pub fn transition(&mut self, event: Event) -> Result<State, &'static str> {
                let new_state = match (self.state, event) {
                    $((State::$from, Event::$evt) => State::$to,)+
                    _ => return Err("Invalid transition"),
                };
                self.state = new_state;
                Ok(new_state)
            }
        }

        impl Default for $machine_name {
            fn default() -> Self {
                Self::new()
            }
        }
    };
}

// Usage
state_machine! {
    TrafficLight {
        initial: Red,
        states: { Red, Yellow, Green },
        events: { Timer, Emergency },
        transitions: {
            Red + Timer => Green,
            Green + Timer => Yellow,
            Yellow + Timer => Red,
            Red + Emergency => Red,
            Green + Emergency => Red,
            Yellow + Emergency => Red,
        }
    }
}

fn main() {
    let mut light = TrafficLight::new();
    assert_eq!(light.state(), State::Red);

    light.transition(Event::Timer).unwrap();
    assert_eq!(light.state(), State::Green);

    light.transition(Event::Emergency).unwrap();
    assert_eq!(light.state(), State::Red);
}
```

### Query Builder DSL

```rust
macro_rules! query {
    // SELECT fields FROM table
    (SELECT $($field:ident),+ FROM $table:ident) => {
        Query::new(stringify!($table))
            $(.select(stringify!($field)))+
    };

    // SELECT fields FROM table WHERE condition
    (SELECT $($field:ident),+ FROM $table:ident WHERE $cond_field:ident = $value:expr) => {
        Query::new(stringify!($table))
            $(.select(stringify!($field)))+
            .where_eq(stringify!($cond_field), $value)
    };

    // SELECT fields FROM table ORDER BY field direction
    (SELECT $($field:ident),+ FROM $table:ident ORDER BY $order:ident $dir:ident) => {
        Query::new(stringify!($table))
            $(.select(stringify!($field)))+
            .order_by(stringify!($order), stringify!($dir))
    };
}

#[derive(Debug)]
struct Query {
    table: String,
    fields: Vec<String>,
    conditions: Vec<(String, String)>,
    order: Option<(String, String)>,
}

impl Query {
    fn new(table: &str) -> Self {
        Self {
            table: table.to_string(),
            fields: vec![],
            conditions: vec![],
            order: None,
        }
    }

    fn select(mut self, field: &str) -> Self {
        self.fields.push(field.to_string());
        self
    }

    fn where_eq(mut self, field: &str, value: &str) -> Self {
        self.conditions.push((field.to_string(), value.to_string()));
        self
    }

    fn order_by(mut self, field: &str, direction: &str) -> Self {
        self.order = Some((field.to_string(), direction.to_string()));
        self
    }

    fn to_sql(&self) -> String {
        let fields = self.fields.join(", ");
        let mut sql = format!("SELECT {} FROM {}", fields, self.table);

        if !self.conditions.is_empty() {
            let conds: Vec<_> = self.conditions.iter()
                .map(|(f, v)| format!("{} = '{}'", f, v))
                .collect();
            sql.push_str(&format!(" WHERE {}", conds.join(" AND ")));
        }

        if let Some((field, dir)) = &self.order {
            sql.push_str(&format!(" ORDER BY {} {}", field, dir));
        }

        sql
    }
}

fn main() {
    let q1 = query!(SELECT id, name, email FROM users);
    println!("{}", q1.to_sql());
    // SELECT id, name, email FROM users

    let q2 = query!(SELECT id, name FROM users WHERE status = "active");
    println!("{}", q2.to_sql());
    // SELECT id, name FROM users WHERE status = 'active'

    let q3 = query!(SELECT name, created_at FROM posts ORDER BY created_at DESC);
    println!("{}", q3.to_sql());
    // SELECT name, created_at FROM posts ORDER BY created_at DESC
}
```

### HTML Builder DSL

```rust
macro_rules! html {
    // Self-closing tag
    ($tag:ident /) => {
        format!("<{} />", stringify!($tag))
    };

    // Tag with attributes only (self-closing)
    ($tag:ident [ $($attr:ident = $value:expr),* ] /) => {
        {
            let attrs = vec![$(format!("{}=\"{}\"", stringify!($attr), $value)),*];
            format!("<{} {} />", stringify!($tag), attrs.join(" "))
        }
    };

    // Tag with content
    ($tag:ident { $($content:tt)* }) => {
        format!("<{}>{}</{}>", stringify!($tag), html!($($content)*), stringify!($tag))
    };

    // Tag with attributes and content
    ($tag:ident [ $($attr:ident = $value:expr),* ] { $($content:tt)* }) => {
        {
            let attrs = vec![$(format!("{}=\"{}\"", stringify!($attr), $value)),*];
            format!("<{} {}>{}</{}>",
                stringify!($tag),
                attrs.join(" "),
                html!($($content)*),
                stringify!($tag))
        }
    };

    // Text content
    ($text:expr) => {
        $text.to_string()
    };

    // Multiple elements
    ($($element:tt)+) => {
        {
            let mut result = String::new();
            $(result.push_str(&html!($element));)+
            result
        }
    };
}

fn main() {
    let page = html! {
        div [class = "container"] {
            h1 { "Welcome" }
            p [id = "intro"] { "Hello, world!" }
            br /
            a [href = "https://example.com"] { "Click here" }
        }
    };

    println!("{}", page);
}
```

### Message/Protocol Routing DSL

A practical pattern for mapping data contracts to handler functions, useful for TCP protocols, message queues, or module interfaces:

```rust
// Data contracts with input and output fields
#[derive(Debug)]
pub struct CreateUserContract {
    pub username: String,
    pub result: Option<Result<u64, String>>,
}

#[derive(Debug)]
pub struct DeleteUserContract {
    pub user_id: u64,
    pub result: Option<Result<(), String>>,
}

// Enum wrapping all contract types for transport
#[derive(Debug)]
pub enum ContractHandler {
    CreateUser(CreateUserContract),
    DeleteUser(DeleteUserContract),
}

// Handler functions for each contract
fn handle_create_user(mut contract: CreateUserContract) -> CreateUserContract {
    // Business logic here
    contract.result = Some(Ok(42));
    contract
}

fn handle_delete_user(mut contract: DeleteUserContract) -> DeleteUserContract {
    contract.result = Some(Ok(()));
    contract
}

// Macro to generate routing function
#[macro_export]
macro_rules! register_contract_routes {
    (
        $handler_enum:ident,
        $fn_name:ident,
        $( $contract:ident => $handler_fn:path ),* $(,)?
    ) => {
        pub fn $fn_name(received_msg: $handler_enum) -> $handler_enum {
            match received_msg {
                $(
                    $handler_enum::$contract(inner) => {
                        let executed = $handler_fn(inner);
                        $handler_enum::$contract(executed)
                    }
                )*
            }
        }
    };
}

// Generate the routing function
register_contract_routes!(
    ContractHandler,
    handle_contract,
    CreateUser => handle_create_user,
    DeleteUser => handle_delete_user,
);

fn main() {
    let contract = CreateUserContract {
        username: "alice".to_string(),
        result: None,
    };
    let response = handle_contract(ContractHandler::CreateUser(contract));
    println!("{:?}", response);
}
```

This pattern enables:
- Clean separation between transport and business logic
- Easy addition of new contracts (just add to enum and macro call)
- Type-safe routing without manual match statements
- Configurable handler functions via trait bounds for DI

### DSL Design Guidelines

1. **Keep syntax familiar and intuitive** — mimic the domain (SQL, HTML, config files):
```rust
// GOOD: Resembles actual SQL
query!(SELECT name FROM users WHERE id = "123")

// BAD: Cryptic custom syntax
query!(users -> [name] ? id == "123")
```

2. **Provide good error messages** — use `compile_error!` with helpful text:
```rust
macro_rules! validated_config {
    () => {
        compile_error!(
            "validated_config! requires at least one field. \
             Usage: validated_config! { field: value, ... }"
        );
    };
    ($($field:ident: $value:expr),+ $(,)?) => {
        // ... generate config
    };
}
```

3. **Support incremental complexity** — simple usage should be simple, advanced features opt-in:
```rust
// Simple
config!(port: 8080)
// Advanced
config!(port: 8080, host: "0.0.0.0", timeout: 30, tls: true)
```

4. **Document the DSL syntax** — always include a doc comment showing the grammar:
```rust
/// Creates a state machine with the given configuration.
///
/// # Syntax
/// ```ignore
/// state_machine! {
///     MachineName {
///         initial: StateName,
///         states: { State1, State2, ... },
///         events: { Event1, Event2, ... },
///         transitions: {
///             FromState + Event => ToState,
///             ...
///         }
///     }
/// }
/// ```
macro_rules! state_machine { /* ... */ }
```

## Debugging Macros

### cargo expand

The primary tool for understanding macro expansion:

```bash
# Install
cargo install cargo-expand

# Expand all macros in the crate
cargo expand

# Expand a specific module
cargo expand module_name

# Expand a specific item
cargo expand ::path::to::item

# Expand only a specific type's derives
cargo expand --test test_name
```

### compile_error! for Debugging

Deliberately trigger a compile error to see what the macro receives:

```rust
macro_rules! debug_input {
    ($input:expr) => {
        compile_error!(concat!("Input received: ", stringify!($input)));
    };
}

// Compile error message shows exactly what was captured
debug_input!(some_complex_expression);
// error: Input received: some_complex_expression
```

### Tracing Macro Expansion at Runtime

```rust
macro_rules! trace_macro {
    ($($arg:tt)*) => {
        eprintln!("TRACE [{}:{}]: {}", file!(), line!(), stringify!($($arg)*));
        $($arg)*
    };
}

let x = trace_macro!(1 + 2);
// Prints: TRACE [src/main.rs:5]: 1 + 2
```

### Debugging Proc Macros

```rust
// In proc macro code, use eprintln! to output during compilation
#[proc_macro_derive(Debug)]
pub fn derive_debug(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);

    // Print during compilation (shows in cargo build output)
    eprintln!("=== DERIVE INPUT ===");
    eprintln!("Name: {}", input.ident);
    eprintln!("Fields: {:?}", match &input.data {
        Data::Struct(data) => data.fields.iter()
            .filter_map(|f| f.ident.as_ref().map(|i| i.to_string()))
            .collect::<Vec<_>>(),
        _ => vec![],
    });

    // ... generate code
    TokenStream::new()
}
```

## When to Use Macros vs Alternatives

| Use Case | Macro | Generic/Trait | Function |
|----------|-------|---------------|----------|
| Variadic arguments | **Yes** | No | No |
| Code generation | **Yes** | Sometimes | No |
| DSLs | **Yes** | No | No |
| Compile-time validation | Proc macro | Const generics | No |
| Eliminating boilerplate | Yes | **Often better** | **Often better** |
| Type-level operations | No | **Yes** (GATs, etc.) | No |
| Conditional compilation | **Yes** (`cfg!`) | No | No |
| String manipulation at compile time | **Yes** (`concat!`, `stringify!`) | No | No |

**Good use cases for macros:**
- Eliminating repetitive boilerplate that can't be abstracted with generics
- Domain-specific languages (state machines, query builders, routers)
- Code generation based on compile-time information
- Variadic functions (accepting any number of arguments)
- Reducing trait implementation boilerplate for many types
- Compile-time string processing

**Avoid macros when:**
- A function or generic would work just as well
- The macro is complex and hard to debug
- Error messages would be confusing for users
- You're using a macro to avoid learning the type system

**Rule of thumb:** If a function or generic works, prefer it. Macros are for when you need to generate code, accept variable syntax, or work at the token level.

### Decision Flowchart

```
Need to accept varying number/types of arguments?
├── Yes → Use declarative macro (macro_rules!)
└── No
    ├── Need to generate impl blocks from struct definitions?
    │   └── Yes → Use derive macro
    ├── Need to transform/wrap entire functions?
    │   └── Yes → Use attribute macro
    ├── Need compile-time code validation?
    │   └── Yes → Use proc macro
    ├── Can solve with generics + trait bounds?
    │   └── Yes → Use generics (preferred)
    └── Can solve with a regular function?
        └── Yes → Use a function (preferred)
```

## Real-World Macro Examples from the Ecosystem

### serde's Serialize/Deserialize

The most widely used derive macros in Rust. Key patterns to learn from:

```rust
// serde uses helper attributes extensively
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]  // Container attribute
struct ApiResponse {
    #[serde(rename = "id")]          // Field attribute
    response_id: u64,
    #[serde(skip_serializing_if = "Option::is_none")]
    metadata: Option<String>,
    #[serde(default)]
    tags: Vec<String>,
}
```

### clap's Parser derive

```rust
// clap uses derives + helper attributes for CLI definition
#[derive(Parser)]
#[command(name = "myapp", version, about)]
struct Cli {
    #[arg(short, long, default_value = "info")]
    log_level: String,
    #[command(subcommand)]
    command: Commands,
}
```

### thiserror's Error derive

```rust
// thiserror generates Display + Error impls from attributes
#[derive(thiserror::Error, Debug)]
enum AppError {
    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),
    #[error("not found: {entity} with id {id}")]
    NotFound { entity: &'static str, id: i64 },
}
```

### tokio's select! macro

```rust
// tokio::select! is a complex declarative macro handling async branching
tokio::select! {
    result = async_operation_1() => { /* handle */ },
    result = async_operation_2() => { /* handle */ },
    _ = tokio::time::sleep(Duration::from_secs(5)) => { /* timeout */ },
}
```

## Macro Best Practices

### Design Guidelines

1. **Keep macros focused** — each macro should have a single, clear purpose
2. **Document with examples** — macro syntax is not self-evident:
```rust
/// Creates a configuration struct with fluent builder pattern.
///
/// # Example
/// ```
/// config! {
///     DbConfig {
///         host: String = "localhost".to_string(),
///         port: u16 = 5432,
///     }
/// }
/// let cfg = DbConfig::new().port(3306);
/// ```
macro_rules! config { /* ... */ }
```

3. **Use descriptive names** that indicate what the macro generates:
```rust
macro_rules! generate_database_queries { /* good */ }
macro_rules! do_stuff { /* bad */ }
```

4. **Provide good error messages** in proc macros:
```rust
if !is_valid {
    return syn::Error::new(
        span,
        "Expected a struct with named fields. \
         Tuple structs and unit structs are not supported."
    ).to_compile_error().into();
}
```

5. **Use compile_error! in declarative macros** for catch-all arms:
```rust
macro_rules! my_macro {
    (struct $name:ident { $($body:tt)* }) => { /* ... */ };
    (enum $name:ident { $($body:tt)* }) => { /* ... */ };
    ($($other:tt)*) => {
        compile_error!("Expected `struct Name { ... }` or `enum Name { ... }`");
    };
}
```

6. **Export with `#[macro_export]`** for cross-crate usage:
```rust
#[macro_export]
macro_rules! my_public_macro {
    // This will be available as `my_crate::my_public_macro!`
    ($($tt:tt)*) => { /* ... */ };
}
```

7. **Prefer `tt` for forwarding** — when passing tokens through to another macro:
```rust
macro_rules! wrapper {
    ($($tt:tt)*) => {
        inner_macro!($($tt)*);
    };
}
```

8. **Test macro expansion** — use `cargo expand`, `trybuild`, and unit tests to verify output

### Common Pitfalls

**Forgetting trailing commas in repetitions:**
```rust
// BAD: Doesn't accept trailing comma
macro_rules! list {
    ($($item:expr),*) => { /* ... */ };
}
list![1, 2, 3,];  // Error!

// GOOD: Optional trailing comma
macro_rules! list {
    ($($item:expr),* $(,)?) => { /* ... */ };
}
list![1, 2, 3,];  // Works
```

**Expression hygiene with multiple evaluation:**
```rust
// BAD: Evaluates expression multiple times
macro_rules! max {
    ($a:expr, $b:expr) => {
        if $a > $b { $a } else { $b }
    };
}
max!(expensive_fn(), other_fn());  // Both called twice!

// GOOD: Bind to variables first
macro_rules! max {
    ($a:expr, $b:expr) => {{
        let a = $a;
        let b = $b;
        if a > b { a } else { b }
    }};
}
```

**Missing braces in macro body:**
```rust
// BAD: No outer braces — can leak variables
macro_rules! init {
    ($name:ident) => {
        let $name = Vec::new();
        // Other statements can leak
    };
}

// GOOD: Wrap in braces for block scope
macro_rules! init {
    ($name:ident) => {{
        let $name = Vec::new();
        $name
    }};
}
```

## Key Crates for Macro Development

| Crate | Purpose | When to Use |
|-------|---------|-------------|
| `syn` | Parse Rust syntax into AST | All proc macros |
| `quote` | Generate Rust code from templates | All proc macros |
| `proc-macro2` | Testable proc-macro types | Proc macro unit tests |
| `paste` | Identifier concatenation in macros | `[<set_ $field>]` patterns |
| `darling` | Attribute parsing for derives | Complex derive attributes |
| `trybuild` | Compile-fail test harness | Testing proc macros |
| `cargo-expand` | Expand macros in source | Debugging any macro |
| `proc-macro-error` | Better error reporting | Proc macro error handling |

### darling for Complex Attribute Parsing

When your derive macro has many attributes, `darling` greatly simplifies parsing:

```rust
use darling::{FromDeriveInput, FromField};

#[derive(FromDeriveInput)]
#[darling(attributes(my_derive))]
struct MyDeriveInput {
    ident: syn::Ident,
    generics: syn::Generics,
    data: darling::ast::Data<(), MyField>,
    #[darling(default)]
    rename_all: Option<String>,
}

#[derive(FromField)]
#[darling(attributes(my_derive))]
struct MyField {
    ident: Option<syn::Ident>,
    ty: syn::Type,
    #[darling(default)]
    skip: bool,
    #[darling(default)]
    rename: Option<String>,
}

// Usage in proc macro:
#[proc_macro_derive(MyDerive, attributes(my_derive))]
pub fn derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let parsed = MyDeriveInput::from_derive_input(&input).unwrap();

    // Fields are already parsed with all attributes resolved
    if let darling::ast::Data::Struct(fields) = &parsed.data {
        for field in fields.iter() {
            if field.skip { continue; }
            // Use field.rename, field.ty, etc.
        }
    }

    TokenStream::new()
}
```

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: traits, generics, derive macros usage, common macro invocations
- **[language-patterns.md](language-patterns.md)** — Common macro invocation patterns, when traits/generics suffice instead
- **[type-system.md](type-system.md)** — Type-level programming, sealed traits, const generics — alternatives to macros
- **[architecture.md](architecture.md)** — `safe_eject!` macro pattern, workspace organization for proc macro crates
- **[testing.md](testing.md)** — Testing macro expansions, `trybuild` for compile-fail tests
