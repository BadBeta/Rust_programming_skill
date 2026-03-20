# Unsafe Rust and FFI

When and how to use `unsafe`, raw pointers, Foreign Function Interface, C interop, byte manipulation, network protocols, and maintaining safety guarantees.

## Rules for Unsafe & FFI (LLM)

1. **ALWAYS add a `// SAFETY:` comment on every `unsafe` block** — explain why the invariants are upheld; this is the Rust community standard and required by `clippy::undocumented_unsafe_blocks`
2. **NEVER dereference a raw pointer without proving it is valid, aligned, and non-null** — check or document these three properties before every dereference
3. **ALWAYS minimize unsafe scope** — wrap only the specific unsafe operation, not the entire function; keep unsafe blocks as small as possible so safe code can be audited separately
4. **ALWAYS wrap unsafe operations in a safe public API** — users of your code should never need to write `unsafe`; the safe wrapper upholds invariants and validates inputs
5. **ALWAYS use `catch_unwind` at FFI boundaries** — Rust panics across `extern "C"` boundaries are undefined behavior; every `#[no_mangle] extern "C"` function must catch panics
6. **NEVER use `static mut`** — use `AtomicT`, `Mutex<T>`, or `LazyLock<T>` instead; `static mut` is almost always unsound in multithreaded contexts
7. **ALWAYS use `CString`/`CStr` for C string interop** — never cast `&str` to `*const c_char` directly; Rust strings are not null-terminated
8. **ALWAYS run `cargo +nightly miri test` on any crate containing unsafe** — MIRI catches undefined behavior (use-after-free, misaligned access, data races) that tests and sanitizers miss
9. **PREFER `cxx::bridge` over raw `extern "C"` for C++ interop** — cxx generates both sides of the FFI boundary, provides type-safe smart pointers (`UniquePtr`, `SharedPtr`), and prevents ABI mismatches at compile time
10. **PREFER `zerocopy` derive macros over manual transmute/pointer casts** — `#[derive(FromBytes, IntoBytes)]` with `#[repr(C)]` provides zero-copy binary parsing with compile-time safety verification
11. **ALWAYS wrap raw file descriptors in `OwnedFd`** — prevents descriptor leaks via `Drop`; use `AsFd`/`BorrowedFd` for borrowing (the nix pattern for safe POSIX wrappers)
12. **ALWAYS use `unsafe extern` blocks in Rust 2024 edition** — each function declaration requires individual `safe` or `unsafe` annotation, preventing accidental calls to unsafe foreign functions

### Section Index

| Section | Topics |
|---------|--------|
| [When to Use Unsafe](#when-to-use-unsafe) | Five unsafe superpowers, justification criteria |
| [Raw Pointers](#raw-pointers) | Creating, dereferencing, pointer arithmetic, null checks |
| [FFI](#foreign-function-interface-ffi) | extern "C", #[link], CString/CStr, repr(C), bindgen |
| [Mutable Static Variables](#mutable-static-variables) | LazyLock alternatives, when statics are necessary |
| [Unsafe Traits](#implementing-unsafe-traits) | Send, Sync, custom unsafe traits, safety contracts |
| [Union Types](#union-types) | C-compatible unions, ManuallyDrop |
| [Best Practices for Unsafe](#best-practices-for-unsafe-code) | Minimize scope, SAFETY comments, safe wrappers |
| [Common FFI Patterns](#common-ffi-patterns) | Opaque types, callbacks, error codes, resource management |
| [Critical FFI Safety Rules](#critical-ffi-safety-rules) | Panic across FFI, string handling, alignment |
| [C Libraries (cdylib)](#building-dynamic-c-libraries-cdylib) | Building Rust as C library, header generation |
| [Python Integration](#python-integration-with-ctypes) | ctypes bindings, PyO3 |
| [C++ Interop](#c-interop-with-cxx) | cxx crate, shared types |
| [Rust 2024: unsafe extern](#rust-2024-edition-unsafe-extern-blocks) | New extern block syntax |
| [POSIX Wrappers](#safe-posix-wrappers-nix-pattern) | nix crate, OwnedFd, safe syscalls |
| [Byte Manipulation](#byte-manipulation) | Endianness, zerocopy, byteorder |
| [Binary Serialization](#manual-binary-serialization) | Wire protocols, manual parsing |
| [Network Protocol Design](#network-protocol-design) | Custom protocols, framing, codec |
| [TCP/TLS Implementation](#tcp-serverclient-implementation) | TCP server, rustls, connection handling |
| [Unsafe Checklist](#unsafe-checklist) | Pre-commit verification steps |

### Common Mistakes (BAD/GOOD)

**Oversized unsafe blocks:**
```rust
// BAD: entire function is unsafe — impossible to audit
unsafe fn process(ptr: *const u8, len: usize) -> Vec<u8> {
    let slice = std::slice::from_raw_parts(ptr, len);
    let mut result = Vec::new();
    for &b in slice { result.push(b.wrapping_add(1)); }
    result
}

// GOOD: unsafe only around the operation that needs it
fn process(ptr: *const u8, len: usize) -> Vec<u8> {
    // SAFETY: caller guarantees ptr is valid for len bytes and properly aligned
    let slice = unsafe { std::slice::from_raw_parts(ptr, len) };
    slice.iter().map(|b| b.wrapping_add(1)).collect()
}
```

**Missing panic safety at FFI boundary:**
```rust
// BAD: panic in extern "C" is undefined behavior
#[no_mangle]
pub extern "C" fn parse(input: *const c_char) -> i32 {
    let s = unsafe { CStr::from_ptr(input) }.to_str().unwrap(); // may panic!
    s.len() as i32
}

// GOOD: catch_unwind prevents panics from crossing FFI boundary
#[no_mangle]
pub extern "C" fn parse(input: *const c_char) -> i32 {
    std::panic::catch_unwind(|| {
        let s = unsafe { CStr::from_ptr(input) }.to_str().ok()?;
        Some(s.len() as i32)
    })
    .ok()
    .flatten()
    .unwrap_or(-1)
}
```

**Manual transmute vs zerocopy:**
```rust
// BAD: manual transmute — no alignment/size checking
let header: &PacketHeader = unsafe { &*(bytes.as_ptr() as *const PacketHeader) };

// GOOD: zerocopy — compile-time safety, zero runtime cost
#[derive(zerocopy::FromBytes, zerocopy::KnownLayout, zerocopy::Immutable)]
#[repr(C)]
struct PacketHeader { magic: u32, length: u16 }

let header = PacketHeader::ref_from_bytes(&bytes[..6])
    .map_err(|e| io::Error::new(io::ErrorKind::InvalidData, e))?;
```

**Raw fd leak:**
```rust
// BAD: raw fd leaks if function returns early
let fd = unsafe { libc::socket(libc::AF_INET, libc::SOCK_STREAM, 0) };
if fd == -1 { return Err(io::Error::last_os_error()); }
do_something(fd)?;  // if this fails, fd leaks!
unsafe { libc::close(fd); }

// GOOD: OwnedFd closes on drop, even on early return (nix pattern)
use std::os::fd::OwnedFd;
let fd = unsafe { OwnedFd::from_raw_fd(
    match libc::socket(libc::AF_INET, libc::SOCK_STREAM, 0) {
        -1 => return Err(io::Error::last_os_error()),
        fd => fd,
    }
) };
do_something(fd.as_fd())?;  // fd closed automatically on drop
```

## When to Use Unsafe

The `unsafe` keyword unlocks five operations the compiler cannot verify:

1. **Dereferencing raw pointers**
2. **Calling unsafe functions** (including extern functions)
3. **Accessing or modifying mutable static variables**
4. **Implementing unsafe traits** (Send, Sync)
5. **Accessing union fields**

### Valid Use Cases

```rust
// 1. Interfacing with C/C++ libraries
extern "C" {
    fn external_function(ptr: *const u8) -> i32;
}

// 2. Performance-critical sections (bypass bounds checking)
unsafe fn get_unchecked(slice: &[u8], index: usize) -> u8 {
    *slice.get_unchecked(index)
}

// 3. Low-level systems programming (hardware, OS)
unsafe fn read_port(port: u16) -> u8 {
    // Platform-specific I/O
    0 // placeholder
}

// 4. Implementing data structures requiring pointer manipulation
struct LinkedList<T> {
    head: *mut Node<T>,
    tail: *mut Node<T>,
}
```

## Raw Pointers

### Creating Raw Pointers

```rust
fn main() {
    let mut value = 42;

    // Creating raw pointers (safe)
    let r1: *const i32 = &value;      // Immutable raw pointer
    let r2: *mut i32 = &mut value;    // Mutable raw pointer

    // From arbitrary address (usually invalid)
    let address = 0x012345usize;
    let r3 = address as *const i32;

    // Dereferencing requires unsafe
    unsafe {
        println!("r1 = {}", *r1);
        *r2 = 100;
        println!("value = {}", value);
        // *r3 would likely crash - invalid memory
    }
}
```

### Raw Pointer Operations

```rust
use std::slice;

fn main() {
    let mut data = vec![1, 2, 3, 4, 5];
    let ptr = data.as_mut_ptr();
    let len = data.len();

    unsafe {
        // Create slice from raw parts
        let slice = slice::from_raw_parts_mut(ptr, len);

        // Pointer arithmetic
        let second = ptr.add(1);  // Move forward by 1 element
        *second = 20;

        // Offset (can go negative)
        let third = ptr.offset(2);
        *third = 30;

        // Check for null
        if !ptr.is_null() {
            println!("First element: {}", *ptr);
        }
    }

    println!("data = {:?}", data);  // [1, 20, 30, 4, 5]
}
```

### Pointer Safety Rules

```rust
// Raw pointers must be:
// 1. Valid (point to allocated memory)
// 2. Aligned (for the type)
// 3. Not dangling (memory not freed)
// 4. Properly initialized (for reads)

fn safe_dereference(ptr: *const i32) -> Option<i32> {
    if ptr.is_null() {
        return None;
    }

    // Alignment check
    if (ptr as usize) % std::mem::align_of::<i32>() != 0 {
        return None;
    }

    // Still unsafe - we can't verify memory is valid
    unsafe { Some(*ptr) }
}
```

## Foreign Function Interface (FFI)

### Declaring External Functions

```rust
use libc::{c_int, c_char, size_t};

// Declare C functions
extern "C" {
    fn abs(input: c_int) -> c_int;
    fn strlen(s: *const c_char) -> size_t;
    fn printf(format: *const c_char, ...) -> c_int;
}

fn main() {
    unsafe {
        let result = abs(-42);
        println!("abs(-42) = {}", result);
    }
}
```

### Rust 2024: unsafe extern Blocks

```rust
// Rust 2024 edition: extern blocks are implicitly unsafe
// All declarations inside are unsafe — the block makes this explicit
unsafe extern "C" {
    fn abs(input: c_int) -> c_int;
    // Can mark individual items as safe if the function is always safe to call
    safe fn strlen(s: *const c_char) -> size_t;
}
```

### The #[repr(C)] Attribute

```rust
use libc::c_int;

// Ensure C-compatible memory layout
#[repr(C)]
struct Point {
    x: c_int,
    y: c_int,
}

#[repr(C)]
struct Rectangle {
    top_left: Point,
    bottom_right: Point,
}

// Declare C function using our struct
extern "C" {
    fn calculate_area(rect: *const Rectangle) -> c_int;
}

fn main() {
    let rect = Rectangle {
        top_left: Point { x: 0, y: 0 },
        bottom_right: Point { x: 10, y: 20 },
    };

    unsafe {
        let area = calculate_area(&rect);
        println!("Area: {}", area);
    }
}
```

### String Handling with CString and CStr

```rust
use std::ffi::{CString, CStr};
use std::os::raw::c_char;

extern "C" {
    fn puts(s: *const c_char) -> i32;
    fn getenv(name: *const c_char) -> *const c_char;
}

// Rust String -> C string
fn send_to_c(rust_string: &str) {
    // CString adds null terminator, fails if string contains null bytes
    let c_string = CString::new(rust_string)
        .expect("String contains null byte");

    unsafe {
        puts(c_string.as_ptr());
    }
    // c_string lives until end of scope - pointer remains valid
}

// C string -> Rust String
fn receive_from_c(name: &str) -> Option<String> {
    let c_name = CString::new(name).ok()?;

    unsafe {
        let ptr = getenv(c_name.as_ptr());

        if ptr.is_null() {
            return None;
        }

        // CStr borrows the C string (doesn't take ownership)
        let c_str = CStr::from_ptr(ptr);

        // Convert to Rust String (handles invalid UTF-8 gracefully)
        Some(c_str.to_string_lossy().into_owned())
    }
}

fn main() {
    send_to_c("Hello from Rust!");

    if let Some(path) = receive_from_c("PATH") {
        println!("PATH = {}", path);
    }
}
```

### Common C Type Mappings

```rust
use libc::{
    c_char,    // i8 or u8 (platform-dependent)
    c_int,     // i32 (usually)
    c_uint,    // u32 (usually)
    c_long,    // i32 or i64 (platform-dependent)
    c_ulong,   // u32 or u64 (platform-dependent)
    c_float,   // f32
    c_double,  // f64
    c_void,    // () or opaque type
    size_t,    // usize
    ssize_t,   // isize
};

// Opaque types (C struct you don't need to see inside)
#[repr(C)]
pub struct OpaqueHandle {
    _private: [u8; 0],  // Zero-sized, prevents construction
}

extern "C" {
    fn create_handle() -> *mut OpaqueHandle;
    fn destroy_handle(handle: *mut OpaqueHandle);
    fn use_handle(handle: *mut OpaqueHandle) -> c_int;
}
```

### Callbacks from C to Rust

```rust
use libc::c_int;

// Function pointer type for C callback
type Callback = extern "C" fn(c_int) -> c_int;

extern "C" {
    fn register_callback(cb: Callback);
    fn trigger_callback(value: c_int) -> c_int;
}

// Rust function with C calling convention
extern "C" fn my_callback(x: c_int) -> c_int {
    println!("Callback called with: {}", x);
    x * 2
}

fn main() {
    unsafe {
        register_callback(my_callback);
        let result = trigger_callback(21);
        println!("Result: {}", result);  // 42
    }
}
```

### Exposing Rust Functions to C

```rust
use std::ffi::{CStr, CString};
use std::os::raw::c_char;

// No name mangling, C calling convention
#[no_mangle]
pub extern "C" fn rust_add(a: i32, b: i32) -> i32 {
    a + b
}

#[no_mangle]
pub extern "C" fn rust_greet(name: *const c_char) -> *mut c_char {
    let c_str = unsafe {
        if name.is_null() {
            return std::ptr::null_mut();
        }
        CStr::from_ptr(name)
    };

    let name_str = c_str.to_str().unwrap_or("Unknown");
    let greeting = format!("Hello, {}!", name_str);

    // Caller is responsible for freeing this!
    CString::new(greeting)
        .map(|s| s.into_raw())
        .unwrap_or(std::ptr::null_mut())
}

// Must provide a way to free the string
#[no_mangle]
pub extern "C" fn rust_free_string(s: *mut c_char) {
    if !s.is_null() {
        unsafe {
            let _ = CString::from_raw(s);
            // CString dropped here, memory freed
        }
    }
}
```

## Mutable Static Variables

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Mutex;

// UNSAFE: No synchronization
static mut COUNTER: u32 = 0;

fn unsafe_increment() {
    unsafe {
        COUNTER += 1;  // Data race if called from multiple threads!
    }
}

// SAFE: Atomic operations
static ATOMIC_COUNTER: AtomicUsize = AtomicUsize::new(0);

fn safe_increment() {
    ATOMIC_COUNTER.fetch_add(1, Ordering::SeqCst);
}

// SAFE: Mutex-protected (requires lazy initialization)
use std::sync::LazyLock;

static PROTECTED_DATA: LazyLock<Mutex<Vec<String>>> =
    LazyLock::new(|| Mutex::new(Vec::new()));

fn safe_push(item: String) {
    PROTECTED_DATA.lock().unwrap().push(item);
}
```

## Implementing Unsafe Traits

### Send and Sync

```rust
use std::cell::UnsafeCell;

// Wrapper around raw pointer
struct RawPointerWrapper<T> {
    ptr: *mut T,
}

// UNSAFE: We guarantee thread safety manually
// Send: Safe to transfer ownership to another thread
unsafe impl<T: Send> Send for RawPointerWrapper<T> {}

// Sync: Safe to share references between threads
unsafe impl<T: Sync> Sync for RawPointerWrapper<T> {}

// Example: Thread-safe wrapper with internal synchronization
struct ThreadSafeCounter {
    value: UnsafeCell<u64>,
}

// We implement our own synchronization
impl ThreadSafeCounter {
    fn new(value: u64) -> Self {
        Self { value: UnsafeCell::new(value) }
    }

    // In real code, use atomic operations or mutex
    fn get(&self) -> u64 {
        unsafe { *self.value.get() }
    }
}

// Only safe because we ensure proper synchronization in methods
unsafe impl Sync for ThreadSafeCounter {}
```

### When Manual Send/Sync is Needed

```rust
// Types that are NOT automatically Send/Sync:
// - Rc<T>: Not Send (reference count not atomic)
// - *const T, *mut T: Not Send or Sync
// - Cell<T>, RefCell<T>: Not Sync (interior mutability not thread-safe)

// If you wrap a non-Send type but ensure safety:
struct MyWrapper {
    // *mut T is not Send, but we synchronize access
    data: *mut u8,
    len: usize,
}

// Document WHY this is safe!
/// Safety: MyWrapper's data is only accessed through synchronized methods.
/// The underlying memory is allocated and never shared without synchronization.
unsafe impl Send for MyWrapper {}
```

## Union Types

```rust
#[repr(C)]
union IntOrFloat {
    i: i32,
    f: f32,
}

fn main() {
    let mut u = IntOrFloat { i: 42 };

    // Reading union fields is unsafe - compiler doesn't know active variant
    unsafe {
        println!("As int: {}", u.i);
        // Reading as float interprets same bits differently
        println!("As float: {}", u.f);  // Garbage value
    }

    // Writing is safe
    u.f = 3.14;

    unsafe {
        println!("As float: {}", u.f);
    }
}

// Common FFI use: C unions
#[repr(C)]
union Value {
    int_val: i64,
    float_val: f64,
    ptr_val: *mut std::ffi::c_void,
}

#[repr(C)]
struct TaggedValue {
    tag: u8,  // 0=int, 1=float, 2=ptr
    value: Value,
}

impl TaggedValue {
    fn get_int(&self) -> Option<i64> {
        if self.tag == 0 {
            Some(unsafe { self.value.int_val })
        } else {
            None
        }
    }
}
```

## Best Practices for Unsafe Code

### 1. Minimize Unsafe Scope

```rust
// BAD: Entire function is unsafe
unsafe fn process_data(ptr: *const u8, len: usize) -> Vec<u8> {
    let slice = std::slice::from_raw_parts(ptr, len);
    slice.to_vec()
}

// GOOD: Only unsafe operation is in unsafe block
fn process_data(ptr: *const u8, len: usize) -> Vec<u8> {
    // Safety: Caller guarantees ptr is valid for len bytes
    let slice = unsafe { std::slice::from_raw_parts(ptr, len) };
    slice.to_vec()  // Safe operations outside unsafe block
}
```

### 2. Document Safety Requirements

```rust
/// Copies `count` bytes from `src` to `dst`.
///
/// # Safety
///
/// - `src` must be valid for reads of `count` bytes
/// - `dst` must be valid for writes of `count` bytes
/// - `src` and `dst` must not overlap
/// - Both pointers must be properly aligned
pub unsafe fn copy_bytes(src: *const u8, dst: *mut u8, count: usize) {
    std::ptr::copy_nonoverlapping(src, dst, count);
}
```

### 3. Create Safe Abstractions

```rust
/// A safe wrapper around a C string buffer.
pub struct CStringBuffer {
    ptr: *mut libc::c_char,
    capacity: usize,
}

impl CStringBuffer {
    pub fn new(capacity: usize) -> Option<Self> {
        let ptr = unsafe { libc::malloc(capacity) as *mut libc::c_char };
        if ptr.is_null() {
            None
        } else {
            // Initialize to empty string
            unsafe { *ptr = 0; }
            Some(Self { ptr, capacity })
        }
    }

    pub fn as_ptr(&self) -> *const libc::c_char {
        self.ptr
    }

    pub fn as_str(&self) -> &str {
        unsafe {
            let c_str = std::ffi::CStr::from_ptr(self.ptr);
            c_str.to_str().unwrap_or("")
        }
    }
}

impl Drop for CStringBuffer {
    fn drop(&mut self) {
        unsafe { libc::free(self.ptr as *mut libc::c_void); }
    }
}

// Now users interact with safe API only
fn main() {
    let buffer = CStringBuffer::new(256).unwrap();
    println!("Buffer: {}", buffer.as_str());
}  // Automatically freed
```

### 4. Use Tools for FFI

```rust
// bindgen: Auto-generate bindings from C headers
// Add to build.rs:
fn main() {
    println!("cargo:rerun-if-changed=wrapper.h");
    let bindings = bindgen::Builder::default()
        .header("wrapper.h")
        .generate()
        .expect("Unable to generate bindings");

    let out_path = std::path::PathBuf::from(
        std::env::var("OUT_DIR").unwrap()
    );
    bindings
        .write_to_file(out_path.join("bindings.rs"))
        .expect("Couldn't write bindings");
}

// In lib.rs:
// include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
```

### 5. Testing Unsafe Code

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_raw_pointer_operations() {
        let mut data = [1, 2, 3, 4, 5];
        let ptr = data.as_mut_ptr();

        unsafe {
            // Test valid operations
            assert_eq!(*ptr, 1);
            assert_eq!(*ptr.add(2), 3);
        }
    }

    #[test]
    fn test_null_handling() {
        let result = safe_dereference(std::ptr::null());
        assert!(result.is_none());
    }

    // Use Miri for detecting undefined behavior:
    // cargo +nightly miri test
}
```

## Common FFI Patterns

### Error Handling

```rust
use libc::c_int;

extern "C" {
    fn risky_operation() -> c_int;
}

#[derive(Debug)]
pub enum FfiError {
    NullPointer,
    InvalidArgument,
    Unknown(i32),
}

fn safe_wrapper() -> Result<(), FfiError> {
    let result = unsafe { risky_operation() };

    match result {
        0 => Ok(()),
        -1 => Err(FfiError::NullPointer),
        -2 => Err(FfiError::InvalidArgument),
        code => Err(FfiError::Unknown(code)),
    }
}
```

### Resource Management with Drop

```rust
use libc::c_void;

extern "C" {
    fn create_resource() -> *mut c_void;
    fn destroy_resource(handle: *mut c_void);
    fn use_resource(handle: *mut c_void) -> i32;
}

pub struct Resource {
    handle: *mut c_void,
}

impl Resource {
    pub fn new() -> Option<Self> {
        let handle = unsafe { create_resource() };
        if handle.is_null() {
            None
        } else {
            Some(Self { handle })
        }
    }

    pub fn use_it(&self) -> i32 {
        unsafe { use_resource(self.handle) }
    }
}

impl Drop for Resource {
    fn drop(&mut self) {
        if !self.handle.is_null() {
            unsafe { destroy_resource(self.handle); }
        }
    }
}

// Resource automatically freed when dropped
fn main() {
    if let Some(res) = Resource::new() {
        println!("Result: {}", res.use_it());
    }  // destroy_resource called here
}
```

## Critical FFI Safety Rules

### Never Panic Across FFI Boundaries

**This is the most critical FFI safety rule.** When Rust panics and stack unwinding crosses an FFI boundary, it causes **undefined behavior**. Depending on the host environment, this can:

- Crash the entire host process (Python, Ruby, Node.js interpreter)
- Crash the entire runtime (Erlang BEAM VM, taking down all processes)
- Corrupt memory or leave resources in invalid states
- Cause silent data corruption

```rust
// BAD: Panic can escape to foreign caller
#[no_mangle]
pub extern "C" fn dangerous_function(input: *const c_char) -> i32 {
    let s = unsafe { CStr::from_ptr(input) };
    let rust_str = s.to_str().unwrap();  // PANICS on invalid UTF-8!
    rust_str.len() as i32
}

// GOOD: Catch panics at FFI boundary
#[no_mangle]
pub extern "C" fn safe_function(input: *const c_char) -> i32 {
    let result = std::panic::catch_unwind(|| {
        if input.is_null() {
            return -1;
        }
        let s = unsafe { CStr::from_ptr(input) };
        match s.to_str() {
            Ok(rust_str) => rust_str.len() as i32,
            Err(_) => -2,  // Invalid UTF-8 error code
        }
    });

    match result {
        Ok(value) => value,
        Err(_) => -99,  // Panic occurred, return error code
    }
}
```

### Catch-Unwind Pattern for All Public FFI Functions

Wrap all externally-callable functions with `catch_unwind`:

```rust
use std::panic::{catch_unwind, AssertUnwindSafe};

/// Helper macro for FFI functions that returns error code on panic
macro_rules! ffi_safe {
    ($body:expr, $error_value:expr) => {
        match catch_unwind(AssertUnwindSafe(|| $body)) {
            Ok(result) => result,
            Err(_) => $error_value,
        }
    };
}

#[no_mangle]
pub extern "C" fn process_data(ptr: *const u8, len: usize) -> i32 {
    ffi_safe!({
        if ptr.is_null() || len == 0 {
            return -1;
        }
        let slice = unsafe { std::slice::from_raw_parts(ptr, len) };
        // Process slice...
        slice.iter().sum::<u8>() as i32
    }, -99)  // -99 = panic occurred
}

// For functions returning pointers, return null on panic
#[no_mangle]
pub extern "C" fn create_string(input: *const c_char) -> *mut c_char {
    ffi_safe!({
        // ... create string logic ...
        CString::new("result").unwrap().into_raw()
    }, std::ptr::null_mut())
}
```

### AssertUnwindSafe for Captured State

When `catch_unwind` captures references or mutable state, wrap them in `AssertUnwindSafe`:

```rust
use std::panic::AssertUnwindSafe;

#[no_mangle]
pub extern "C" fn process_with_state(state: *mut MyState, input: i32) -> i32 {
    let state_ref = unsafe { &mut *state };

    // AssertUnwindSafe is needed because &mut references aren't UnwindSafe
    let result = std::panic::catch_unwind(AssertUnwindSafe(|| {
        state_ref.process(input)
    }));

    match result {
        Ok(value) => value,
        Err(_) => {
            // Reset state to known-good value after panic
            state_ref.reset();
            -1
        }
    }
}
```

## Building Dynamic C Libraries (cdylib)

Use `cdylib` to create shared libraries (.so/.dylib/.dll) that can be loaded by other languages.

### Cargo Configuration

```toml
# Cargo.toml
[package]
name = "my-rust-lib"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]  # Build as dynamic C library

[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

### Complete FFI Module Example

```rust
// src/lib.rs
use std::ffi::{CStr, CString};
use std::os::raw::c_char;
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
pub struct TodoItem {
    pub title: String,
    pub status: String,
}

/// Create a new todo item, returns JSON string.
///
/// # Safety
/// - `title` must be a valid null-terminated C string
/// - Caller must free the returned string with `free_string`
#[no_mangle]
pub extern "C" fn create_todo(title: *const c_char) -> *mut c_char {
    // Convert C string to Rust string
    let title_str = unsafe {
        if title.is_null() {
            return std::ptr::null_mut();
        }
        match CStr::from_ptr(title).to_str() {
            Ok(s) => s.to_string(),
            Err(_) => return std::ptr::null_mut(),
        }
    };

    let todo = TodoItem {
        title: title_str,
        status: "PENDING".to_string(),
    };

    // Serialize to JSON and return as C string
    match serde_json::to_string(&todo) {
        Ok(json) => CString::new(json)
            .map(|s| s.into_raw())
            .unwrap_or(std::ptr::null_mut()),
        Err(_) => std::ptr::null_mut(),
    }
}

/// Mark a todo as done, returns updated JSON string.
#[no_mangle]
pub extern "C" fn mark_done(todo_json: *const c_char) -> *mut c_char {
    let json_str = unsafe {
        if todo_json.is_null() {
            return std::ptr::null_mut();
        }
        match CStr::from_ptr(todo_json).to_str() {
            Ok(s) => s,
            Err(_) => return std::ptr::null_mut(),
        }
    };

    // Parse, modify, and return
    match serde_json::from_str::<TodoItem>(json_str) {
        Ok(mut todo) => {
            todo.status = "DONE".to_string();
            match serde_json::to_string(&todo) {
                Ok(json) => CString::new(json)
                    .map(|s| s.into_raw())
                    .unwrap_or(std::ptr::null_mut()),
                Err(_) => std::ptr::null_mut(),
            }
        }
        Err(_) => std::ptr::null_mut(),
    }
}

/// Free a string allocated by this library.
///
/// # Safety
/// - `s` must have been returned by a function in this library
/// - Must not be called twice on the same pointer
#[no_mangle]
pub extern "C" fn free_string(s: *mut c_char) {
    if !s.is_null() {
        unsafe {
            let _ = CString::from_raw(s);
            // String is dropped and memory freed
        }
    }
}
```

### Build the Library

```bash
# Build release version
cargo build --release

# Output locations by platform:
# Linux:   target/release/libmy_rust_lib.so
# macOS:   target/release/libmy_rust_lib.dylib
# Windows: target/release/my_rust_lib.dll
```

### Thread Safety for Stateful Libraries

When your cdylib maintains state across calls (connection pools, caches, configuration), you must handle concurrent access from the host language. Foreign callers may invoke your library from multiple threads simultaneously.

```rust
use std::sync::{Mutex, RwLock, LazyLock};
use std::collections::HashMap;

// Global state must be thread-safe
static CACHE: LazyLock<RwLock<HashMap<String, String>>> =
    LazyLock::new(|| RwLock::new(HashMap::new()));

static CONNECTION_POOL: LazyLock<Mutex<Vec<Connection>>> =
    LazyLock::new(|| Mutex::new(Vec::new()));

#[no_mangle]
pub extern "C" fn cache_get(key: *const c_char) -> *mut c_char {
    std::panic::catch_unwind(|| {
        let key_str = unsafe { CStr::from_ptr(key) }.to_str().ok()?;

        // Read lock allows concurrent reads
        let cache = CACHE.read().ok()?;
        let value = cache.get(key_str)?;

        CString::new(value.clone()).ok().map(|s| s.into_raw())
    })
    .ok()
    .flatten()
    .unwrap_or(std::ptr::null_mut())
}

#[no_mangle]
pub extern "C" fn cache_set(key: *const c_char, value: *const c_char) -> i32 {
    std::panic::catch_unwind(|| {
        let key_str = unsafe { CStr::from_ptr(key) }.to_str().ok()?;
        let value_str = unsafe { CStr::from_ptr(value) }.to_str().ok()?;

        // Write lock for exclusive access
        let mut cache = CACHE.write().ok()?;
        cache.insert(key_str.to_string(), value_str.to_string());
        Some(0)
    })
    .ok()
    .flatten()
    .unwrap_or(-1)
}
```

**Key patterns for thread-safe cdylib:**

| Pattern | Use Case |
|---------|----------|
| `LazyLock<Mutex<T>>` | Exclusive access to mutable state |
| `LazyLock<RwLock<T>>` | Many readers, few writers |
| `AtomicUsize` / `AtomicBool` | Simple counters and flags |
| `Arc<T>` returned as opaque handle | Per-instance state (caller manages lifetime) |

**Per-instance state pattern** (safer than global state):

```rust
pub struct Context {
    data: Mutex<Vec<u8>>,
}

#[no_mangle]
pub extern "C" fn context_new() -> *mut Context {
    Box::into_raw(Box::new(Context {
        data: Mutex::new(Vec::new()),
    }))
}

#[no_mangle]
pub extern "C" fn context_free(ctx: *mut Context) {
    if !ctx.is_null() {
        unsafe { let _ = Box::from_raw(ctx); }
    }
}

#[no_mangle]
pub extern "C" fn context_push(ctx: *mut Context, value: u8) -> i32 {
    std::panic::catch_unwind(|| {
        let ctx = unsafe { &*ctx };
        ctx.data.lock().ok()?.push(value);
        Some(0)
    })
    .ok()
    .flatten()
    .unwrap_or(-1)
}
```

## Python Integration with ctypes

### Loading the Rust Library

```python
# main.py
import ctypes
import platform
import json

def load_rust_library():
    """Load the Rust cdylib based on platform."""
    system = platform.system()

    if system == "Linux":
        path = "./target/release/libmy_rust_lib.so"
    elif system == "Darwin":  # macOS
        path = "./target/release/libmy_rust_lib.dylib"
    elif system == "Windows":
        path = "./target/release/my_rust_lib.dll"
    else:
        raise OSError(f"Unsupported platform: {system}")

    return ctypes.CDLL(path)

# Load library
lib = load_rust_library()

# Define function signatures
lib.create_todo.argtypes = [ctypes.c_char_p]
lib.create_todo.restype = ctypes.c_char_p

lib.mark_done.argtypes = [ctypes.c_char_p]
lib.mark_done.restype = ctypes.c_char_p

lib.free_string.argtypes = [ctypes.c_char_p]
lib.free_string.restype = None
```

### Python Wrapper Functions

```python
def create_todo(title: str) -> dict:
    """Create a new todo item using Rust library."""
    # Encode string to bytes (C expects null-terminated)
    result = lib.create_todo(title.encode('utf-8'))

    if result is None:
        raise RuntimeError("Failed to create todo")

    # Decode bytes to string
    json_str = result.decode('utf-8')
    return json.loads(json_str)

def mark_done(todo: dict) -> dict:
    """Mark a todo as done using Rust library."""
    json_bytes = json.dumps(todo).encode('utf-8')
    result = lib.mark_done(json_bytes)

    if result is None:
        raise RuntimeError("Failed to mark todo as done")

    json_str = result.decode('utf-8')
    return json.loads(json_str)

# Usage
if __name__ == "__main__":
    todo = create_todo("Learn Rust FFI")
    print(f"Created: {todo}")
    # Output: Created: {'title': 'Learn Rust FFI', 'status': 'PENDING'}

    completed = mark_done(todo)
    print(f"Completed: {completed}")
    # Output: Completed: {'title': 'Learn Rust FFI', 'status': 'DONE'}
```

### Memory Management Considerations

```python
class RustString:
    """Context manager for Rust-allocated strings."""

    def __init__(self, lib, ptr):
        self.lib = lib
        self.ptr = ptr

    def __enter__(self):
        return self.ptr.decode('utf-8') if self.ptr else None

    def __exit__(self, *args):
        if self.ptr:
            self.lib.free_string(self.ptr)

# Usage with explicit memory management
def create_todo_safe(title: str) -> dict:
    ptr = lib.create_todo(title.encode('utf-8'))
    with RustString(lib, ptr) as json_str:
        if json_str is None:
            raise RuntimeError("Failed to create todo")
        return json.loads(json_str)
```

## Alternative: PyO3 for Production

For production Python bindings, use PyO3 which provides automatic type conversions, native Python exceptions, GIL management, and `maturin` for easy packaging.

```toml
# Cargo.toml for PyO3
[lib]
name = "my_rust_lib"
crate-type = ["cdylib"]

[dependencies]
pyo3 = { version = "0.20", features = ["extension-module"] }
```

```rust
use pyo3::prelude::*;

#[pyfunction]
fn create_todo(title: String) -> PyResult<String> {
    let todo = TodoItem {
        title,
        status: "PENDING".to_string(),
    };
    serde_json::to_string(&todo)
        .map_err(|e| PyErr::new::<pyo3::exceptions::PyValueError, _>(e.to_string()))
}

#[pymodule]
fn my_rust_lib(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(create_todo, m)?)?;
    Ok(())
}
```

Build with maturin:
```bash
pip install maturin
maturin develop  # For development
maturin build    # For distribution
```

## Other Language Integration

The same `cdylib` can be used with:

| Language | FFI Method |
|----------|------------|
| Python | ctypes, cffi, PyO3 |
| Node.js | node-ffi, napi-rs |
| Ruby | ffi gem |
| Go | cgo |
| Java | JNI, JNA |
| C# | P/Invoke |
| Elixir | Rustler (NIFs) |

## C++ Interop with cxx

For C++ interop, `cxx` is safer than raw `extern "C"` — it generates both sides of the FFI bridge, with compile-time type checking across the language boundary.

### Bridge Module

```rust
// src/lib.rs
#[cxx::bridge]
mod ffi {
    // Shared types — owned by neither language, defined once
    struct BlobMetadata {
        size: usize,
        tags: Vec<String>,
    }

    // Functions implemented in C++, callable from Rust
    unsafe extern "C++" {
        include!("myproject/include/blobstore.h");

        type BlobstoreClient;

        fn new_blobstore_client() -> UniquePtr<BlobstoreClient>;
        fn put(&self, parts: &mut MultiBuf) -> u64;
        fn tag(&self, blob_id: u64, tag: &str);
        fn metadata(&self, blob_id: u64) -> BlobMetadata;
    }

    // Functions implemented in Rust, callable from C++
    extern "Rust" {
        type MultiBuf;

        fn next_chunk(buf: &mut MultiBuf) -> &[u8];
    }
}
```

### Key Types

| cxx Type | C++ Type | Ownership |
|----------|----------|-----------|
| `UniquePtr<T>` | `std::unique_ptr<T>` | C++ owns, Rust borrows |
| `SharedPtr<T>` | `std::shared_ptr<T>` | Shared ownership |
| `CxxString` | `std::string` | Pass by reference only |
| `CxxVector<T>` | `std::vector<T>` | Pass by reference only |
| `Box<T>` | `rust::Box<T>` | Rust owns, C++ borrows |
| `String` | `rust::String` | Rust string |
| `Vec<T>` | `rust::Vec<T>` | Rust vector |

### Build Integration

```toml
# Cargo.toml
[dependencies]
cxx = "1.0"

[build-dependencies]
cxx-build = "1.0"
```

```rust
// build.rs
fn main() {
    cxx_build::bridge("src/lib.rs")
        .file("src/blobstore.cc")  // C++ implementation
        .std("c++17")
        .compile("myproject");
}
```

**When to use cxx vs raw FFI:**
- **cxx**: C++ interop, complex types, safety-critical code
- **Raw `extern "C"`**: Pure C libraries, simple types, maximum control
- **bindgen**: Auto-generate bindings from C headers (large APIs)

## Rust 2024 Edition: `unsafe extern` Blocks

In Rust 2024 edition, `extern` blocks require explicit safety annotations per function:

```rust
// Rust 2024 — each declaration must be annotated
unsafe extern "C" {
    // safe: compiler trusts this is always safe to call
    safe fn abs(input: i32) -> i32;

    // unsafe (default): caller must verify preconditions
    unsafe fn strlen(s: *const c_char) -> usize;

    // safe static: always safe to read
    safe static environ: *const *const c_char;
}

// Now abs() can be called without unsafe block:
let x = abs(-42);  // OK — declared safe

// strlen() still requires unsafe:
let len = unsafe { strlen(c_str.as_ptr()) };
```

This replaces the old pattern where all extern functions were implicitly unsafe to call, even when they had no preconditions (like `abs`).

## Safe POSIX Wrappers (nix Pattern)

The `nix` crate demonstrates the gold standard for wrapping unsafe syscalls in safe Rust APIs. The pattern: minimize unsafe scope, use owned types for resources, convert errors via `Errno`.

### Pattern: Owned File Descriptors

```rust
use std::os::fd::{OwnedFd, RawFd, AsRawFd, FromRawFd, AsFd, BorrowedFd};

/// Safe wrapper around libc::socket
pub fn socket(
    domain: libc::c_int,
    ty: libc::c_int,
    protocol: libc::c_int,
) -> io::Result<OwnedFd> {
    // SAFETY: socket() is a well-defined syscall with no pointer arguments
    let fd = unsafe { libc::socket(domain, ty, protocol) };
    if fd == -1 {
        Err(io::Error::last_os_error())
    } else {
        // SAFETY: we just created this fd, it's valid
        Ok(unsafe { OwnedFd::from_raw_fd(fd) })
    }
}

// OwnedFd implements Drop — fd is closed automatically
// BorrowedFd<'_> borrows without taking ownership
// AsFd trait enables generic code over any fd-owning type

/// Safe wrapper around libc::listen
pub fn listen(sock: impl AsFd, backlog: i32) -> io::Result<()> {
    // SAFETY: AsFd guarantees the fd is valid for the duration of the call
    let res = unsafe { libc::listen(sock.as_fd().as_raw_fd(), backlog) };
    if res == -1 {
        Err(io::Error::last_os_error())
    } else {
        Ok(())
    }
}
```

### Pattern: Type-Safe Addresses

```rust
/// Trait for sockaddr-like types (nix's SockaddrLike)
pub trait SockaddrLike {
    fn as_ptr(&self) -> *const libc::sockaddr;
    fn len(&self) -> libc::socklen_t;
}

/// Safe wrapper around libc::bind using trait abstraction
pub fn bind(sock: impl AsFd, addr: &dyn SockaddrLike) -> io::Result<()> {
    // SAFETY: SockaddrLike guarantees ptr/len are valid
    let res = unsafe {
        libc::bind(sock.as_fd().as_raw_fd(), addr.as_ptr(), addr.len())
    };
    if res == -1 {
        Err(io::Error::last_os_error())
    } else {
        Ok(())
    }
}
```

### Pattern: Errno Result Conversion

```rust
/// Convert libc return value to Result using errno
fn errno_result(ret: libc::c_int) -> io::Result<libc::c_int> {
    if ret == -1 {
        Err(io::Error::last_os_error())
    } else {
        Ok(ret)
    }
}

// Usage — makes wrappers concise:
pub fn dup(fd: impl AsFd) -> io::Result<OwnedFd> {
    let raw = errno_result(unsafe { libc::dup(fd.as_fd().as_raw_fd()) })?;
    // SAFETY: dup() returned a valid fd
    Ok(unsafe { OwnedFd::from_raw_fd(raw) })
}
```

## Byte Manipulation

### Working with Byte Slices

```rust
use std::io::{self, Read};
use std::net::TcpStream;

fn process_network_data(mut stream: TcpStream) -> io::Result<()> {
    let mut buffer = [0u8; 1024];  // Fixed-size buffer

    // Read data into buffer
    let bytes_read = stream.read(&mut buffer)?;

    if bytes_read > 0 {
        // Work with the valid portion as a byte slice
        let data_slice = &buffer[..bytes_read];
        println!("Received {} bytes: {:?}", bytes_read, data_slice);
    }

    Ok(())
}
```

### Endianness Conversion

Multi-byte integers have different byte orderings:
- **Big-endian (network byte order)**: Most significant byte first
- **Little-endian**: Least significant byte first (x86, ARM default)

```rust
// Standard library methods for endianness conversion
fn endianness_examples() {
    let value: u32 = 0x12345678;

    // Convert to big-endian bytes (network byte order)
    let be_bytes: [u8; 4] = value.to_be_bytes();
    assert_eq!(be_bytes, [0x12, 0x34, 0x56, 0x78]);

    // Convert to little-endian bytes
    let le_bytes: [u8; 4] = value.to_le_bytes();
    assert_eq!(le_bytes, [0x78, 0x56, 0x34, 0x12]);

    // Convert back from bytes
    let from_be = u32::from_be_bytes(be_bytes);
    let from_le = u32::from_le_bytes(le_bytes);
    assert_eq!(from_be, value);
    assert_eq!(from_le, value);
}

// Reading a big-endian u32 from a byte slice
fn read_u32_be(bytes: &[u8]) -> Option<u32> {
    if bytes.len() >= 4 {
        let mut word_bytes = [0u8; 4];
        word_bytes.copy_from_slice(&bytes[0..4]);
        Some(u32::from_be_bytes(word_bytes))
    } else {
        None
    }
}
```

### Available Conversion Methods

| Type | To Big-Endian | From Big-Endian | To Little-Endian | From Little-Endian |
|------|---------------|-----------------|------------------|-------------------|
| `u16` | `to_be_bytes()` | `from_be_bytes()` | `to_le_bytes()` | `from_le_bytes()` |
| `u32` | `to_be_bytes()` | `from_be_bytes()` | `to_le_bytes()` | `from_le_bytes()` |
| `u64` | `to_be_bytes()` | `from_be_bytes()` | `to_le_bytes()` | `from_le_bytes()` |
| `i16` | `to_be_bytes()` | `from_be_bytes()` | `to_le_bytes()` | `from_le_bytes()` |
| `i32` | `to_be_bytes()` | `from_be_bytes()` | `to_le_bytes()` | `from_le_bytes()` |
| `i64` | `to_be_bytes()` | `from_be_bytes()` | `to_le_bytes()` | `from_le_bytes()` |
| `f32` | `to_be_bytes()` | `from_be_bytes()` | `to_le_bytes()` | `from_le_bytes()` |
| `f64` | `to_be_bytes()` | `from_be_bytes()` | `to_le_bytes()` | `from_le_bytes()` |

### Zero-Copy Binary Parsing with zerocopy

The `zerocopy` crate provides safe, zero-cost binary parsing via derive macros — no unsafe needed:

```toml
[dependencies]
zerocopy = { version = "0.8", features = ["derive"] }
```

```rust
use zerocopy::{FromBytes, IntoBytes, KnownLayout, Immutable, Unaligned};

/// Network packet header — zero-copy parsing from raw bytes
#[derive(FromBytes, IntoBytes, KnownLayout, Immutable, Clone, Copy, Debug)]
#[repr(C)]
struct EthernetHeader {
    dst_mac: [u8; 6],
    src_mac: [u8; 6],
    ether_type: [u8; 2],  // Use byte arrays for endian-neutral storage
}

impl EthernetHeader {
    fn ether_type(&self) -> u16 {
        u16::from_be_bytes(self.ether_type)
    }
}

/// Parse header from a byte buffer — zero-copy, zero-unsafe
fn parse_ethernet(packet: &[u8]) -> Option<&EthernetHeader> {
    EthernetHeader::ref_from_prefix(packet).ok().map(|(hdr, _rest)| hdr)
}

/// Variable-length packet with trailing data
#[derive(FromBytes, KnownLayout, Immutable)]
#[repr(C)]
struct Packet {
    header: EthernetHeader,
    payload: [u8],  // DST — dynamically sized trailing slice
}

fn parse_packet(data: &[u8]) -> Option<&Packet> {
    Packet::ref_from_bytes(data).ok()
}
```

**Key traits and when to derive them:**

| Trait | Meaning | Required For |
|-------|---------|-------------|
| `FromBytes` | Any byte pattern is valid | Parsing from bytes |
| `IntoBytes` | Can be viewed as bytes | Serializing to bytes |
| `KnownLayout` | Layout is statically known | All ref-from operations |
| `Immutable` | No interior mutability | Immutable references |
| `Unaligned` | Alignment = 1 | Unaligned access |
| `TryFromBytes` | Validate at runtime | Enums, booleans |

**When to use zerocopy vs manual parsing:**
- **zerocopy**: Fixed-layout binary formats, network protocols, memory-mapped I/O, any `#[repr(C)]` struct
- **Manual**: Variable-length fields, complex encoding (VLQ, UTF-8), non-`repr(C)` types

## Manual Binary Serialization

### Using Write Trait for Serialization

```rust
use std::io::{self, Write, Read, Cursor};

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct SimpleMessage {
    pub id: u16,
    pub payload_len: u32,
    pub payload: Vec<u8>,
}

impl SimpleMessage {
    // Serialize to bytes (big-endian format)
    pub fn serialize<W: Write>(&self, writer: &mut W) -> io::Result<()> {
        // Write id as big-endian u16
        writer.write_all(&self.id.to_be_bytes())?;
        // Write payload length as big-endian u32
        writer.write_all(&self.payload_len.to_be_bytes())?;
        // Write payload bytes
        writer.write_all(&self.payload)?;
        Ok(())
    }

    // Deserialize from bytes
    pub fn deserialize<R: Read>(reader: &mut R) -> io::Result<Self> {
        // Read id
        let mut id_bytes = [0u8; 2];
        reader.read_exact(&mut id_bytes)?;
        let id = u16::from_be_bytes(id_bytes);

        // Read payload length
        let mut len_bytes = [0u8; 4];
        reader.read_exact(&mut len_bytes)?;
        let payload_len = u32::from_be_bytes(len_bytes);

        // Read payload
        let mut payload = vec![0u8; payload_len as usize];
        reader.read_exact(&mut payload)?;

        Ok(SimpleMessage { id, payload_len, payload })
    }
}

// Usage with Cursor for in-memory buffers
fn roundtrip_example() -> io::Result<()> {
    let message = SimpleMessage {
        id: 0x0102,
        payload_len: 5,
        payload: vec![b'H', b'e', b'l', b'l', b'o'],
    };

    // Serialize to Vec<u8>
    let mut buffer = Vec::new();
    message.serialize(&mut buffer)?;
    println!("Serialized: {:?}", buffer);
    // Output: [1, 2, 0, 0, 0, 5, 72, 101, 108, 108, 111]

    // Deserialize using Cursor
    let mut reader = Cursor::new(buffer);
    let deserialized = SimpleMessage::deserialize(&mut reader)?;

    assert_eq!(message, deserialized);
    Ok(())
}
```

### Using io::Cursor

`Cursor` wraps an in-memory buffer (`Vec<u8>`, `&[u8]`) to provide `Read`/`Write` traits:

```rust
use std::io::{Cursor, Read, Write, Seek, SeekFrom};

fn cursor_examples() -> io::Result<()> {
    // Write to a Cursor-wrapped Vec
    let mut cursor = Cursor::new(Vec::new());
    cursor.write_all(b"Hello")?;
    cursor.write_all(b" World")?;

    // Get the inner buffer
    let buffer = cursor.into_inner();
    assert_eq!(buffer, b"Hello World");

    // Read from a Cursor
    let mut cursor = Cursor::new(b"Test data".to_vec());
    let mut buf = [0u8; 4];
    cursor.read_exact(&mut buf)?;
    assert_eq!(&buf, b"Test");

    // Seek within cursor
    cursor.seek(SeekFrom::Start(0))?;  // Back to beginning
    cursor.seek(SeekFrom::Current(2))?; // Forward 2 bytes
    cursor.seek(SeekFrom::End(-2))?;    // 2 bytes from end

    Ok(())
}
```

### Using byteorder Crate

For more ergonomic endianness handling:

```rust
// Cargo.toml: byteorder = "1.5"
use byteorder::{BigEndian, LittleEndian, ReadBytesExt, WriteBytesExt};
use std::io::{Cursor, Read, Write};

fn byteorder_examples() -> std::io::Result<()> {
    let mut buffer = Vec::new();

    // Write with specified endianness
    buffer.write_u16::<BigEndian>(0x1234)?;
    buffer.write_u32::<BigEndian>(0x12345678)?;
    buffer.write_f32::<LittleEndian>(3.14)?;

    // Read with specified endianness
    let mut cursor = Cursor::new(&buffer);
    let val_u16 = cursor.read_u16::<BigEndian>()?;
    let val_u32 = cursor.read_u32::<BigEndian>()?;
    let val_f32 = cursor.read_f32::<LittleEndian>()?;

    assert_eq!(val_u16, 0x1234);
    assert_eq!(val_u32, 0x12345678);
    assert!((val_f32 - 3.14).abs() < 0.001);

    Ok(())
}
```

## Network Protocol Design

### Modeling Messages with Structs and Enums

```rust
use serde::{Deserialize, Serialize};

// Simple message with fixed fields
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct TextMessage {
    pub sender: String,
    pub recipient: String,
    pub content: String,
}

// Protocol message enum - represents all possible message types
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum ChatMessage {
    Text(TextMessage),
    Notify(Notification),
}

// Optional fields
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct ConnectRequest {
    pub protocol_version: u16,
    pub authentication_token: Option<String>,  // May or may not be present
}

// Request/Response pattern
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Request {
    GetUser { user_id: u32 },
    ListUsers { page: u32, limit: u32 },
    CreateUser { username: String, email: String },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Response {
    User(User),
    UserList { users: Vec<User>, total: u32 },
    Created { id: u32 },
    Error { code: u16, message: String },
}
```

### Binary Serialization with bincode

`bincode` provides fast, compact binary serialization:

```rust
// Cargo.toml: bincode = "1.3"
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug, PartialEq)]
struct UserProfile {
    user_id: u64,
    username: String,
    email: Option<String>,
    active: bool,
}

fn bincode_serialization() -> Result<(), Box<dyn std::error::Error>> {
    let user = UserProfile {
        user_id: 12345,
        username: "alice_smith".to_string(),
        email: Some("alice@example.com".to_string()),
        active: true,
    };

    // Serialize to bytes (compact binary format)
    let encoded: Vec<u8> = bincode::serialize(&user)?;
    println!("Encoded size: {} bytes", encoded.len());

    // Deserialize from bytes
    let decoded: UserProfile = bincode::deserialize(&encoded)?;
    assert_eq!(user, decoded);

    Ok(())
}
```

### Serialization Formats

| Format | Crate | Use Case |
|--------|-------|----------|
| JSON | `serde_json` | Human-readable, web APIs |
| bincode | `bincode` | Fast binary, Rust-to-Rust |
| MessagePack | `rmp-serde` | Compact binary, cross-language |
| CBOR | `serde_cbor` | Compact binary, IoT/embedded |
| Protocol Buffers | `prost` | Schema-defined, cross-language |

## Protocol Versioning

### Versioned Message Enums

```rust
use serde::{Deserialize, Serialize};

// Version 1: Original message
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct TextMessageV1 {
    pub sender: String,
    pub recipient: String,
    pub content: String,
}

// Version 2: Added timestamp field
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct TextMessageV2 {
    pub sender: String,
    pub recipient: String,
    pub content: String,
    pub timestamp: u64,  // Unix timestamp
}

// Versioned enum for protocol evolution
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum ChatMessageVersioned {
    V1(TextMessageV1),
    V2(TextMessageV2),
}

impl ChatMessageVersioned {
    // Convert older versions to latest
    pub fn to_v2(self) -> TextMessageV2 {
        match self {
            ChatMessageVersioned::V1(v1) => TextMessageV2 {
                sender: v1.sender,
                recipient: v1.recipient,
                content: v1.content,
                timestamp: 0,  // Default for upgraded messages
            },
            ChatMessageVersioned::V2(v2) => v2,
        }
    }
}
```

### Schema Evolution with Serde Attributes

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug, PartialEq)]
struct UserProfileV2 {
    user_id: u64,
    username: String,
    email: Option<String>,
    active: bool,

    // New field with default - compatible with old data
    #[serde(default)]
    is_premium: bool,

    // Renamed field - accepts old name during deserialization
    #[serde(alias = "user_level")]
    account_tier: Option<String>,

    // Skip serializing if default
    #[serde(default, skip_serializing_if = "Vec::is_empty")]
    tags: Vec<String>,
}

fn schema_evolution_example() -> Result<(), serde_json::Error> {
    // Old data without new fields
    let old_json = r#"{
        "user_id": 123,
        "username": "alice",
        "email": null,
        "active": true
    }"#;

    // Deserializes successfully - is_premium defaults to false
    let user: UserProfileV2 = serde_json::from_str(old_json)?;
    assert_eq!(user.is_premium, false);

    Ok(())
}
```

### Version Field in Protocol Header

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProtocolHeader {
    pub version: u16,
    pub message_type: u16,
    pub payload_length: u32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProtocolMessage {
    pub header: ProtocolHeader,
    #[serde(with = "serde_bytes")]
    pub payload: Vec<u8>,
}

impl ProtocolMessage {
    pub fn decode_payload<T: for<'de> Deserialize<'de>>(
        &self,
    ) -> Result<T, bincode::Error> {
        bincode::deserialize(&self.payload)
    }
}

// Dispatch based on version and type
fn handle_message(msg: ProtocolMessage) -> Result<(), Box<dyn std::error::Error>> {
    match (msg.header.version, msg.header.message_type) {
        (1, 1) => {
            let text: TextMessageV1 = msg.decode_payload()?;
            handle_text_v1(text);
        }
        (2, 1) => {
            let text: TextMessageV2 = msg.decode_payload()?;
            handle_text_v2(text);
        }
        (v, t) => {
            return Err(format!("Unknown version {} type {}", v, t).into());
        }
    }
    Ok(())
}
```

## Protocol Error Handling

### Custom Protocol Error Type

```rust
use thiserror::Error;
use std::io;

#[derive(Debug, Error)]
pub enum ProtocolError {
    #[error("Invalid UTF-8 sequence")]
    Utf8Error(#[from] std::str::Utf8Error),

    #[error("IO error: {0}")]
    IoError(#[from] io::Error),

    #[error("Invalid message length: expected {expected}, got {got}")]
    InvalidLength { expected: usize, got: usize },

    #[error("Unknown message type: {0}")]
    UnknownMessageType(u16),

    #[error("Unsupported protocol version: {0}")]
    UnsupportedVersion(u16),

    #[error("Checksum mismatch: expected {expected:#x}, got {got:#x}")]
    ChecksumMismatch { expected: u32, got: u32 },

    #[error("Unexpected end of input")]
    UnexpectedEof,

    #[error("Serialization error: {0}")]
    Serialization(#[from] bincode::Error),
}

// Result alias for protocol operations
pub type ProtocolResult<T> = Result<T, ProtocolError>;
```

### Error Handling in Deserialization

```rust
use std::io::{Cursor, Read};

fn deserialize_header(bytes: &[u8]) -> ProtocolResult<ProtocolHeader> {
    if bytes.len() < 8 {
        return Err(ProtocolError::InvalidLength {
            expected: 8,
            got: bytes.len(),
        });
    }

    let mut cursor = Cursor::new(bytes);

    let mut version_bytes = [0u8; 2];
    cursor.read_exact(&mut version_bytes)?;
    let version = u16::from_be_bytes(version_bytes);

    // Validate version
    if version > 2 {
        return Err(ProtocolError::UnsupportedVersion(version));
    }

    let mut type_bytes = [0u8; 2];
    cursor.read_exact(&mut type_bytes)?;
    let message_type = u16::from_be_bytes(type_bytes);

    let mut len_bytes = [0u8; 4];
    cursor.read_exact(&mut len_bytes)?;
    let payload_length = u32::from_be_bytes(len_bytes);

    Ok(ProtocolHeader {
        version,
        message_type,
        payload_length,
    })
}
```

## Complete Protocol Example

```rust
use serde::{Deserialize, Serialize};
use std::io::{self, Cursor, Read, Write};
use thiserror::Error;

// --- Protocol Constants ---
const PROTOCOL_VERSION: u16 = 1;
const MSG_TYPE_PING: u16 = 1;
const MSG_TYPE_PONG: u16 = 2;
const MSG_TYPE_DATA: u16 = 3;

// --- Error Type ---
#[derive(Debug, Error)]
pub enum ProtocolError {
    #[error("IO error: {0}")]
    Io(#[from] io::Error),
    #[error("Serialization error: {0}")]
    Serialization(#[from] bincode::Error),
    #[error("Unknown message type: {0}")]
    UnknownType(u16),
}

// --- Message Types ---
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PingMessage {
    pub sequence: u32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PongMessage {
    pub sequence: u32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DataMessage {
    pub id: u64,
    pub payload: Vec<u8>,
}

#[derive(Debug, Clone)]
pub enum Message {
    Ping(PingMessage),
    Pong(PongMessage),
    Data(DataMessage),
}

// --- Wire Format ---
// Header: version (2) + type (2) + length (4) = 8 bytes
// Payload: variable length bincode-serialized message

impl Message {
    pub fn encode(&self) -> Result<Vec<u8>, ProtocolError> {
        let (msg_type, payload) = match self {
            Message::Ping(m) => (MSG_TYPE_PING, bincode::serialize(m)?),
            Message::Pong(m) => (MSG_TYPE_PONG, bincode::serialize(m)?),
            Message::Data(m) => (MSG_TYPE_DATA, bincode::serialize(m)?),
        };

        let mut buffer = Vec::with_capacity(8 + payload.len());

        // Write header
        buffer.write_all(&PROTOCOL_VERSION.to_be_bytes())?;
        buffer.write_all(&msg_type.to_be_bytes())?;
        buffer.write_all(&(payload.len() as u32).to_be_bytes())?;

        // Write payload
        buffer.write_all(&payload)?;

        Ok(buffer)
    }

    pub fn decode(bytes: &[u8]) -> Result<Self, ProtocolError> {
        let mut cursor = Cursor::new(bytes);

        // Read header
        let mut version_bytes = [0u8; 2];
        cursor.read_exact(&mut version_bytes)?;
        let _version = u16::from_be_bytes(version_bytes);

        let mut type_bytes = [0u8; 2];
        cursor.read_exact(&mut type_bytes)?;
        let msg_type = u16::from_be_bytes(type_bytes);

        let mut len_bytes = [0u8; 4];
        cursor.read_exact(&mut len_bytes)?;
        let payload_len = u32::from_be_bytes(len_bytes) as usize;

        // Read payload
        let mut payload = vec![0u8; payload_len];
        cursor.read_exact(&mut payload)?;

        // Decode based on type
        match msg_type {
            MSG_TYPE_PING => Ok(Message::Ping(bincode::deserialize(&payload)?)),
            MSG_TYPE_PONG => Ok(Message::Pong(bincode::deserialize(&payload)?)),
            MSG_TYPE_DATA => Ok(Message::Data(bincode::deserialize(&payload)?)),
            _ => Err(ProtocolError::UnknownType(msg_type)),
        }
    }
}

// --- Usage ---
fn protocol_roundtrip() -> Result<(), ProtocolError> {
    let original = Message::Data(DataMessage {
        id: 42,
        payload: vec![1, 2, 3, 4, 5],
    });

    let encoded = original.encode()?;
    println!("Encoded {} bytes", encoded.len());

    let decoded = Message::decode(&encoded)?;

    if let (Message::Data(orig), Message::Data(dec)) = (&original, &decoded) {
        assert_eq!(orig.id, dec.id);
        assert_eq!(orig.payload, dec.payload);
    }

    Ok(())
}
```

## TCP Server/Client Implementation

### Building a TCP Server with Tokio

Complete pattern for accepting connections and processing binary messages:

```rust
use tokio::{
    io::{AsyncReadExt, AsyncWriteExt},
    net::{TcpListener, TcpStream}
};
use std::sync::{Arc, Mutex};
use std::collections::HashMap;

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:9000").await.unwrap();
    let state = Arc::new(Mutex::new(HashMap::<Vec<u8>, Vec<u8>>::new()));

    loop {
        let (socket, _) = listener.accept().await.unwrap();
        let state_ref = state.clone();
        tokio::spawn(async move {
            println!("New connection");
            process(socket, state_ref).await;
        });
    }
}

async fn process(
    mut socket: TcpStream,
    state: Arc<Mutex<HashMap<Vec<u8>, Vec<u8>>>>
) {
    // Read length-prefixed message
    let mut len_buf = [0u8; 4];
    if socket.read_exact(&mut len_buf).await.is_err() {
        return;
    }
    let len = u32::from_be_bytes(len_buf) as usize;

    let mut buffer = vec![0u8; len];
    if socket.read_exact(&mut buffer).await.is_err() {
        return;
    }

    // Deserialize and process
    let message: KvMessage = bincode::deserialize(&buffer).unwrap();
    let response = handle_message(message, &state);

    // Send response
    let response_data = response.package();
    let _ = socket.write_all(&response_data).await;
}
```

### Length-Prefixed Message Framing

Standard pattern for binary protocols — prepend 4-byte length header:

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
pub enum KvMessage {
    Get(Vec<u8>),
    Put((Vec<u8>, Vec<u8>)),
    Del(Vec<u8>),
    Success(bool),
    ReturnValue(Option<Vec<u8>>),
    Error(String),
}

impl KvMessage {
    /// Package message with 4-byte length prefix
    pub fn package(&self) -> Vec<u8> {
        let data = bincode::serialize(&self).unwrap();
        let len = data.len() as u32;
        let mut buf = Vec::with_capacity(4 + data.len());
        buf.extend_from_slice(&len.to_be_bytes());
        buf.extend_from_slice(&data);
        buf
    }
}

/// Read length-prefixed message from stream
async fn read_message<R: tokio::io::AsyncReadExt + Unpin>(
    stream: &mut R
) -> std::io::Result<Vec<u8>> {
    let mut len_buf = [0u8; 4];
    stream.read_exact(&mut len_buf).await?;
    let len = u32::from_be_bytes(len_buf) as usize;

    let mut buffer = vec![0u8; len];
    stream.read_exact(&mut buffer).await?;
    Ok(buffer)
}
```

### TCP Client

```rust
use tokio::net::TcpStream;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

async fn send_message(
    message: KvMessage,
    addr: &str
) -> Result<KvMessage, String> {
    let mut socket = TcpStream::connect(addr).await
        .map_err(|e| format!("Failed to connect: {}", e))?;

    // Send packaged message
    let data = message.package();
    socket.write_all(&data).await
        .map_err(|e| format!("Failed to write: {}", e))?;

    // Read response
    let mut len_buf = [0u8; 4];
    socket.read_exact(&mut len_buf).await
        .map_err(|_| "Failed to read length".to_string())?;
    let len = u32::from_be_bytes(len_buf) as usize;

    let mut buffer = vec![0u8; len];
    socket.read_exact(&mut buffer).await
        .map_err(|_| "Failed to read message".to_string())?;

    bincode::deserialize(&buffer)
        .map_err(|e| format!("Failed to deserialize: {}", e))
}

#[tokio::main]
async fn main() -> Result<(), String> {
    // Key doesn't exist yet
    println!("{:?}", send_message(
        KvMessage::Get(b"key".to_vec()),
        "127.0.0.1:9000"
    ).await?);  // ReturnValue(None)

    // Insert key
    println!("{:?}", send_message(
        KvMessage::Put((b"key".to_vec(), b"value".to_vec())),
        "127.0.0.1:9000"
    ).await?);  // Success(true)

    // Now key exists
    println!("{:?}", send_message(
        KvMessage::Get(b"key".to_vec()),
        "127.0.0.1:9000"
    ).await?);  // ReturnValue(Some([118, 97, 108, 117, 101]))

    Ok(())
}
```

### HTTP as Text Protocol on TCP

HTTP is a text-based protocol over TCP. Understanding this helps debug networking issues:

```rust
use std::io::{Read, Write};
use std::net::TcpStream;

fn http_request_raw() {
    let mut stream = TcpStream::connect("127.0.0.1:8080")
        .expect("Failed to connect");

    // HTTP request is just formatted text
    let json_body = r#"{"message": "Hello"}"#;
    let request = format!(
        "POST /api/data HTTP/1.1\r\n\
         Host: 127.0.0.1\r\n\
         Content-Type: application/json\r\n\
         Content-Length: {}\r\n\
         Connection: close\r\n\r\n\
         {}",
        json_body.len(),
        json_body
    );

    stream.write_all(request.as_bytes()).unwrap();

    // Read response
    let mut buffer = [0u8; 512];
    let mut response = String::new();
    while let Ok(bytes_read) = stream.read(&mut buffer) {
        if bytes_read == 0 { break; }
        response.push_str(&String::from_utf8_lossy(&buffer[..bytes_read]));
    }
    println!("Response:\n{}", response);
}
```

## TLS/HTTPS Implementation

### Server-Side HTTPS with OpenSSL

For Actix Web with HTTPS:

```rust
use actix_web::{web, App, HttpServer, HttpResponse};
use openssl::ssl::{SslAcceptor, SslFiletype, SslMethod};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // Configure TLS
    let mut builder = SslAcceptor::mozilla_intermediate(
        SslMethod::tls()
    ).unwrap();
    builder.set_private_key_file("key.pem", SslFiletype::PEM).unwrap();
    builder.set_certificate_chain_file("cert.pem").unwrap();

    HttpServer::new(|| {
        App::new()
            .route("/api/data", web::post().to(handler))
    })
    .bind_openssl("127.0.0.1:8080", builder)?
    .run()
    .await
}
```

Dependencies:
```toml
[dependencies]
actix-web = { version = "4.9", features = ["openssl"] }
openssl = "0.10"
```

### Generating Self-Signed Certificates

For development and testing:

```bash
# Generate private key (2048 bits)
openssl genrsa -out key.pem 2048

# Generate self-signed certificate (valid 365 days)
openssl req -new -x509 -key key.pem -out cert.pem -days 365
```

The `-x509` flag creates a self-signed certificate instead of a certificate signing request.

### Client-Side HTTPS with native-tls

```rust
use native_tls::TlsConnector;
use std::io::{Read, Write};
use std::net::TcpStream;

fn https_client() -> Result<(), Box<dyn std::error::Error>> {
    // Build TLS connector
    let connector = TlsConnector::builder()
        .danger_accept_invalid_certs(true)  // Development only!
        .build()?;

    // Connect TCP, then wrap with TLS
    let stream = TcpStream::connect("127.0.0.1:8080")?;
    let mut stream = connector.connect("127.0.0.1", stream)?;

    // Now use stream like regular TCP, but encrypted
    let request = "GET / HTTP/1.1\r\nHost: 127.0.0.1\r\nConnection: close\r\n\r\n";
    stream.write_all(request.as_bytes())?;

    let mut response = String::new();
    stream.read_to_string(&mut response)?;
    println!("{}", response);

    Ok(())
}
```

Dependencies:
```toml
[dependencies]
native-tls = "0.2"
```

### TLS Configuration Options

| Option | Description |
|--------|-------------|
| `mozilla_intermediate` | Good default, supports older browsers |
| `mozilla_modern_v5` | Latest security, requires OpenSSL 1.1.1+ |
| `danger_accept_invalid_certs` | Skip cert validation (dev only!) |

## Unsafe Checklist

| Check | Description |
|-------|-------------|
| **Minimize scope** | Only wrap the specific unsafe operation |
| **Document safety** | `# Safety` section on every `unsafe fn` |
| **Validate inputs** | Check null, alignment, bounds before unsafe |
| **Wrap in safe API** | Users should never need to write `unsafe` |
| **Test boundaries** | Test null, zero-length, max-size, misaligned |
| **Use MIRI** | `cargo +nightly miri test` catches UB (see below) |
| **Prefer safe alternatives** | Atomics over `static mut`, slices over raw ptrs |
| **Catch panics at FFI** | `catch_unwind` on all `extern "C"` functions |
| **Use length-prefix framing** | 4-byte big-endian length for binary protocols |
| **Specify endianness** | Always use `to_be_bytes`/`from_be_bytes` explicitly |
| **Version protocols** | Include version field in protocol headers |
| **Never skip TLS validation** | `danger_accept_invalid_certs` is dev-only |

### Running Miri

Miri is an interpreter that detects undefined behavior in unsafe code at runtime:

```bash
# Install Miri (requires nightly)
rustup +nightly component add miri

# Run all tests under Miri
cargo +nightly miri test

# Run a specific test
cargo +nightly miri test test_my_unsafe_fn

# Common Miri flags
MIRIFLAGS="-Zmiri-disable-isolation" cargo +nightly miri test  # allow file/env access
MIRIFLAGS="-Zmiri-tag-gc=1" cargo +nightly miri test           # faster for large tests
```

**What Miri detects:**
- Use-after-free and double-free
- Out-of-bounds memory access
- Misaligned pointer dereference
- Violation of aliasing rules (Stacked Borrows / Tree Borrows)
- Data races (with `-Zmiri-preemption-rate=0.1`)
- Reading uninitialized memory
- Invalid discriminant values for enums

**What Miri does NOT detect:**
- Memory leaks (use `valgrind` or `-Zmiri-leak-check`)
- Integer overflow (unless you use checked ops)
- Logic errors in unsafe code that happen to be defined behavior
- FFI calls (Miri can't interpret foreign functions — stub them with `#[cfg(miri)]`)

```rust
// Pattern for testing unsafe code with Miri
#[cfg(test)]
mod tests {
    #[test]
    fn test_raw_pointer_roundtrip() {
        let data = vec![1u32, 2, 3, 4, 5];
        let ptr = data.as_ptr();
        let len = data.len();

        // SAFETY: ptr and len came from a valid Vec, still in scope
        let slice = unsafe { std::slice::from_raw_parts(ptr, len) };
        assert_eq!(slice, &[1, 2, 3, 4, 5]);
    }

    // Stub FFI for Miri testing
    #[cfg(miri)]
    fn external_process(data: &[u8]) -> Vec<u8> {
        data.to_vec()  // Mock implementation
    }

    #[cfg(not(miri))]
    fn external_process(data: &[u8]) -> Vec<u8> {
        unsafe { ffi::real_process(data.as_ptr(), data.len()) }
    }
}
```

## Best Practices Summary

| Practice | Description |
|----------|-------------|
| **Use serde for common formats** | JSON, MessagePack, bincode for standard serialization |
| **Manual serialization for wire protocols** | Full control over byte layout for interoperability |
| **Use thiserror for errors** | Type-safe, descriptive protocol errors |
| **Validate before deserializing** | Check lengths and magic numbers first |
| **Use Cursor for in-memory streams** | Clean Read/Write interface for byte buffers |
| **Test with property-based testing** | Ensure roundtrip serialization works |

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: ownership, lifetimes, smart pointers, Send/Sync traits
- **[testing.md](testing.md)** — Miri testing, property-based testing for roundtrip serialization
- **[services.md](services.md)** — TCP/TLS networking, binary protocol framing
- **[architecture.md](architecture.md)** — Safe API design patterns, encapsulation boundaries
- **[type-system.md](type-system.md)** — Pin/Unpin internals, type state for safety
- **[rust-nif](../../rust-nif/SKILL.md)** — Rustler NIF interop with Elixir/BEAM (higher-level FFI)
