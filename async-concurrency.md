# Async & Concurrency in Rust

Tokio runtime, async/await, threads, Send/Sync, channels, rayon, atomics, deadlock prevention, Tower service pattern, graceful shutdown, backpressure, actors, Hyper HTTP, bridging sync/async, and async testing.

## Rules for Async & Concurrency (LLM)

1. **NEVER hold a `MutexGuard` across `.await`** — std `Mutex` is not async-aware; holding it across await points blocks the executor thread and risks deadlock; use `tokio::sync::Mutex` only when you must hold across await, otherwise scope the guard before the await
2. **ALWAYS use `JoinSet` over `FuturesUnordered`** — `JoinSet` is cancel-safe, tracks task handles, supports `abort_all`/`shutdown`, and is the idiomatic tokio pattern for managing dynamic task collections
3. **ALWAYS use `CancellationToken` for shutdown signaling** — prefer `tokio_util::sync::CancellationToken` over broadcast channels for shutdown; it's clone-cheap, hierarchical (child tokens), and purpose-built for cancellation
4. **ALWAYS use `spawn_blocking` for CPU-bound or blocking work** — blocking the tokio runtime starves other tasks; wrap file I/O, compression, hashing, and synchronous computations in `spawn_blocking`
5. **NEVER use `Rc`, `RefCell`, or `!Send` types in spawned tasks** — `tokio::spawn` requires `Send`; use `Arc`/`Mutex` instead, or use `spawn_local` on a `LocalSet` if you truly need `!Send`
6. **ALWAYS bound channels** — unbounded channels (`mpsc::unbounded_channel`) can cause OOM under load; use bounded channels with explicit capacity and handle `SendError`/backpressure
7. **PREFER `tokio::select!` with `biased;`** when branch priority matters — without `biased`, branches are polled in random order; shutdown branches should have priority
8. **ALWAYS use `borrow_and_update()` (not `borrow()`) after `changed()` on watch channels** — `borrow()` doesn't mark the value as seen, causing `changed()` to return immediately on next call, creating a busy-loop

### Common Mistakes (BAD/GOOD)

**Holding MutexGuard across await:**
```rust
// BAD: std::sync::MutexGuard held across .await — blocks executor thread
let guard = data.lock().unwrap();
do_async_work(&guard).await;  // other tasks on this thread are blocked!
drop(guard);

// GOOD: scope the lock, clone/copy what you need
let value = {
    let guard = data.lock().unwrap();
    guard.clone()  // release lock immediately
};
do_async_work(&value).await;
```

**Spawning blocking work on the async runtime:**
```rust
// BAD: CPU-bound work blocks the tokio worker thread
tokio::spawn(async {
    let hash = sha256(&large_file);  // blocks executor for seconds!
    save_hash(hash).await;
});

// GOOD: offload to blocking threadpool
tokio::spawn(async {
    let hash = tokio::task::spawn_blocking(move || sha256(&large_file)).await?;
    save_hash(hash).await;
});
```

**Unbounded channel as work queue:**
```rust
// BAD: unbounded channel — producer can OOM the process
let (tx, mut rx) = tokio::sync::mpsc::unbounded_channel();
// Under load, millions of messages queue up

// GOOD: bounded channel with explicit backpressure
let (tx, mut rx) = tokio::sync::mpsc::channel(1000);
// tx.send().await blocks when buffer is full — natural backpressure
```

### Section Index

| Section | Content |
|---------|---------|
| [Threads](#threads) | Spawning, `Arc<Mutex<T>>`, scoped threads, thread pools |
| [Send and Sync](#send-and-sync) | Thread safety markers, when to implement, common types |
| [Channels](#channels) | mpsc, broadcast, watch, oneshot, flume, crossbeam |
| [Async/Await](#asyncawait) | `async fn`, `.await`, `Future` trait, pinning basics |
| [Futures and Streams](#futures-and-streams) | `Future` combinators, `Stream` trait, `StreamExt` |
| [Tokio Runtime Architecture](#tokio-runtime-architecture) | Work-stealing, `current_thread` vs `multi_thread`, `spawn_blocking` |
| [Task Management with JoinSet](#task-management-with-joinset) | `JoinSet`, `TaskTracker`, structured concurrency |
| [Pin/Unpin](#pinunpin) | Self-referential futures, `Pin<Box<dyn Future>>`, `Unpin` |
| [Parallel Iterators (Rayon)](#parallel-iterators-rayon) | `par_iter`, `par_bridge`, CPU-bound parallelism |
| [Atomic Operations](#atomic-operations) | `AtomicU64`, `Ordering`, lock-free patterns |
| [Deadlock Prevention](#deadlock-prevention) | Lock ordering, `try_lock`, timeout patterns |
| [Tower Service Pattern](#tower-service-pattern) | `Service` trait, layers, middleware composition |
| [Graceful Shutdown](#graceful-shutdown) | `CancellationToken`, signal handling, drain connections |
| [Backpressure and Flow Control](#backpressure-and-flow-control) | Bounded channels, semaphores, rate limiting |
| [Async Testing Patterns](#async-testing-patterns) | `#[tokio::test]`, time control, mock services |
| [Hyper Low-Level HTTP Server](#hyper-low-level-http-server) | Custom HTTP handling, streaming bodies |
| [Bridging Sync and Async](#bridging-sync-and-async) | `block_on`, `spawn_blocking`, `Handle::current()` |
| [Timeouts, Retries, and Rate Limiting](#timeouts-retries-and-rate-limiting) | `tokio::time::timeout`, exponential backoff, governor |
| [Debugging Async Systems](#debugging-async-systems) | `tokio-console`, task dumps, deadlock detection |
| [Tokio Scheduler Tuning](#tokio-scheduler-tuning) | Worker threads, blocking pool, runtime builder |
| [Common Patterns](#common-patterns) | Select, fan-out/fan-in, pipeline, producer-consumer |
| [Actor Model](#actor-model) | Channel-based actors, actix, supervision |
| [Sans-I/O Pattern](#sans-io-pattern) | Protocol logic without async/threads, testable state machines, runtime loop |

## Threads

### Spawning Threads

```rust
use std::thread;

let handle = thread::spawn(|| {
    println!("Hello from thread!");
    42  // Return value
});

let result = handle.join().unwrap();  // Wait and get result
println!("Thread returned: {}", result);
```

### Sharing Data Between Threads

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        let mut num = counter.lock().unwrap();
        *num += 1;
    });
    handles.push(handle);
}

for handle in handles {
    handle.join().unwrap();
}

println!("Result: {}", *counter.lock().unwrap());
```

### Scoped Threads

Scoped threads can borrow from the stack without `Arc` because the scope guarantees all threads complete before returning:

```rust
use std::thread;

let data = vec![1, 2, 3, 4, 5];

thread::scope(|s| {
    // Can borrow data without Arc because scope ensures threads complete
    s.spawn(|| {
        println!("First: {:?}", &data[..2]);
    });
    s.spawn(|| {
        println!("Last: {:?}", &data[3..]);
    });
});  // All threads joined here

// data still valid here
```

Scoped threads with mutable borrows:

```rust
fn parallel_process(data: &mut [u32]) {
    let mid = data.len() / 2;
    let (left, right) = data.split_at_mut(mid);

    std::thread::scope(|s| {
        s.spawn(|| left.iter_mut().for_each(|x| *x *= 2));
        s.spawn(|| right.iter_mut().for_each(|x| *x *= 2));
    });
}
```

## Send and Sync

### Marker Traits

```rust
// Send: Type can be transferred to another thread
// Sync: Type can be shared between threads (&T is Send)

// Automatically implemented for most types
// NOT Send: Rc<T>, raw pointers
// NOT Sync: Cell<T>, RefCell<T>

// Manual implementation (requires unsafe)
unsafe impl Send for MyType {}
unsafe impl Sync for MyType {}
```

| Trait | Meaning | Example |
|-------|---------|---------|
| `Send` | Can be transferred to another thread | Most types |
| `Sync` | Can be shared between threads via `&T` | Most types |
| `!Send` | Cannot be sent between threads | `Rc<T>`, `*mut T` |
| `!Sync` | Cannot be shared between threads | `Cell<T>`, `RefCell<T>` |

Key relationships:
- `Arc<T>` is `Send + Sync` when `T: Send + Sync`
- `Mutex<T>` is `Send + Sync` when `T: Send`
- `Rc<T>` is neither `Send` nor `Sync`

### Thread-Safe Wrappers

```rust
// Arc<T>: Thread-safe Rc (atomic reference counting)
// Mutex<T>: Mutual exclusion (one accessor at a time)
// RwLock<T>: Read-write lock (many readers OR one writer)

use std::sync::{Arc, Mutex, RwLock};
use std::collections::HashMap;

let shared = Arc::new(Mutex::new(vec![]));
let read_heavy = Arc::new(RwLock::new(HashMap::new()));

// Multiple readers
let data = read_heavy.read().unwrap();
// Exclusive writer
let mut data = read_heavy.write().unwrap();
```

### Send/Sync in Clean Architecture

Repository traits and services must be thread-safe for use with async runtimes and web frameworks:

```rust
use std::sync::Arc;
use async_trait::async_trait;

// Repository trait requires Send + Sync for thread safety
#[async_trait]
pub trait UserRepository: Send + Sync {
    async fn find_by_id(&self, id: u64) -> Result<Option<User>, Error>;
    async fn save(&self, user: &User) -> Result<(), Error>;
}

// Service wraps repository in Arc for shared ownership across tasks
pub struct UserService {
    repository: Arc<dyn UserRepository>,
    metrics: Arc<Mutex<Metrics>>,  // Thread-safe metrics tracking
}

impl UserService {
    pub fn new(repository: Arc<dyn UserRepository>) -> Self {
        Self {
            repository,
            metrics: Arc::new(Mutex::new(Metrics::new())),
        }
    }

    pub async fn get_user(&self, id: u64) -> Result<Option<User>, Error> {
        // Metrics updated atomically
        {
            let mut metrics = self.metrics.lock().unwrap();
            metrics.record_request();
        }  // MutexGuard dropped here, lock released

        self.repository.find_by_id(id).await
    }
}

// Web handler receives cloned Arc
async fn get_user_handler(
    service: web::Data<UserService>,  // Arc<UserService> under the hood
    path: web::Path<u64>,
) -> HttpResponse {
    // Each request shares the same service instance
    match service.get_user(*path).await {
        Ok(Some(user)) => HttpResponse::Ok().json(user),
        Ok(None) => HttpResponse::NotFound().finish(),
        Err(e) => HttpResponse::InternalServerError().body(e.to_string()),
    }
}

// Application wiring
#[tokio::main]
async fn main() {
    // Create shared repository
    let repository: Arc<dyn UserRepository> = Arc::new(PostgresUserRepository::new(pool));

    // Create service with shared repository
    let service = Arc::new(UserService::new(repository));

    // Clone Arc for each worker thread
    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::from(service.clone()))
            .route("/users/{id}", web::get().to(get_user_handler))
    })
    .workers(4)  // 4 threads share the same Arc<UserService>
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

### AtomicUsize for Lock-Free Metrics

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

pub struct RequestMetrics {
    total_requests: AtomicUsize,
    successful_requests: AtomicUsize,
    failed_requests: AtomicUsize,
}

impl RequestMetrics {
    pub fn new() -> Self {
        Self {
            total_requests: AtomicUsize::new(0),
            successful_requests: AtomicUsize::new(0),
            failed_requests: AtomicUsize::new(0),
        }
    }

    pub fn record_success(&self) {
        self.total_requests.fetch_add(1, Ordering::Relaxed);
        self.successful_requests.fetch_add(1, Ordering::Relaxed);
    }

    pub fn record_failure(&self) {
        self.total_requests.fetch_add(1, Ordering::Relaxed);
        self.failed_requests.fetch_add(1, Ordering::Relaxed);
    }

    pub fn get_stats(&self) -> (usize, usize, usize) {
        (
            self.total_requests.load(Ordering::Relaxed),
            self.successful_requests.load(Ordering::Relaxed),
            self.failed_requests.load(Ordering::Relaxed),
        )
    }
}

// Shared across all request handlers without locking
pub struct ServiceWithMetrics<R: UserRepository> {
    repository: R,
    metrics: Arc<RequestMetrics>,
}

impl<R: UserRepository> ServiceWithMetrics<R> {
    pub async fn get_user(&self, id: u64) -> Result<Option<User>, Error> {
        match self.repository.find_by_id(id).await {
            Ok(result) => {
                self.metrics.record_success();
                Ok(result)
            }
            Err(e) => {
                self.metrics.record_failure();
                Err(e)
            }
        }
    }
}
```

### RwLock Patterns

```rust
use std::sync::{Arc, RwLock};
use std::thread;
use std::time::Duration;

// Shared cache - read-heavy workload
let cache = Arc::new(RwLock::new(vec![1, 2, 3]));

// Multiple concurrent readers
let mut reader_handles = vec![];
for i in 0..5 {
    let cache_clone = Arc::clone(&cache);
    reader_handles.push(thread::spawn(move || {
        // Multiple threads can hold read locks simultaneously
        let data = cache_clone.read().unwrap();
        println!("Reader {}: {:?}", i, *data);
        // Lock released when guard drops
    }));
}

// Writer blocks until all readers release
let cache_writer = Arc::clone(&cache);
let writer_handle = thread::spawn(move || {
    thread::sleep(Duration::from_millis(10));  // Let readers start
    println!("Writer: waiting for lock...");
    let mut data = cache_writer.write().unwrap();  // Blocks until readers done
    println!("Writer: acquired lock");
    data.push(4);
    data.push(5);
});

for handle in reader_handles {
    handle.join().unwrap();
}
writer_handle.join().unwrap();

// try_read/try_write for non-blocking attempts
let cache = Arc::new(RwLock::new(0));
match cache.try_read() {
    Ok(guard) => println!("Got read lock: {}", *guard),
    Err(_) => println!("Would block"),
}
```

## Channels

### MPSC (Standard Library)

```rust
use std::sync::mpsc;
use std::thread;

let (tx, rx) = mpsc::channel();

// Clone sender for multiple producers
let tx2 = tx.clone();

thread::spawn(move || {
    tx.send("Hello from thread 1").unwrap();
});

thread::spawn(move || {
    tx2.send("Hello from thread 2").unwrap();
});

// Receive messages
for msg in rx {
    println!("Got: {}", msg);
}
```

### Bounded Channels (Standard Library)

```rust
use std::sync::mpsc;

// Bounded channel (blocks when full)
let (tx, rx) = mpsc::sync_channel(10);

thread::spawn(move || {
    for i in 0..100 {
        tx.send(i).unwrap();  // Blocks if buffer full
    }
});

// Non-blocking receive
match rx.try_recv() {
    Ok(msg) => println!("Got: {}", msg),
    Err(mpsc::TryRecvError::Empty) => println!("No message"),
    Err(mpsc::TryRecvError::Disconnected) => println!("Channel closed"),
}
```

### Tokio mpsc (Async)

```rust
use tokio::sync::mpsc;

#[derive(Debug)]
enum Command {
    Get { key: String, resp: tokio::sync::oneshot::Sender<Option<String>> },
    Set { key: String, value: String },
}

async fn run_store(mut rx: mpsc::Receiver<Command>) {
    let mut store = std::collections::HashMap::new();
    while let Some(cmd) = rx.recv().await {
        match cmd {
            Command::Get { key, resp } => {
                let _ = resp.send(store.get(&key).cloned());
            }
            Command::Set { key, value } => {
                store.insert(key, value);
            }
        }
    }
}

async fn client(tx: mpsc::Sender<Command>) {
    // Set a value
    tx.send(Command::Set { key: "foo".into(), value: "bar".into() }).await.unwrap();

    // Get with response channel (request-response pattern)
    let (resp_tx, resp_rx) = tokio::sync::oneshot::channel();
    tx.send(Command::Get { key: "foo".into(), resp: resp_tx }).await.unwrap();
    let val = resp_rx.await.unwrap();
    assert_eq!(val, Some("bar".to_string()));
}
```

### Broadcast Channels (Tokio)

```rust
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    let (tx, mut rx1) = broadcast::channel(16);
    let mut rx2 = tx.subscribe();

    tx.send("Hello").unwrap();

    // Both receivers get the message
    assert_eq!(rx1.recv().await.unwrap(), "Hello");
    assert_eq!(rx2.recv().await.unwrap(), "Hello");
}
```

### Watch Channel (Tokio)

Single value, multiple observers — useful for configuration updates:

```rust
use tokio::sync::watch;

let (tx, mut rx) = watch::channel("initial".to_string());

// Watcher gets notified when value changes
tokio::spawn(async move {
    while rx.changed().await.is_ok() {
        // IMPORTANT: use borrow_and_update() — not borrow() — after changed()
        // borrow() doesn't mark as seen, causing changed() to return immediately (busy-loop!)
        let val = rx.borrow_and_update().clone();
        println!("config updated: {val}");
    }
});

tx.send("updated".to_string()).unwrap();

// wait_for: block until a condition is met on the watched value
let mut rx2 = tx.subscribe();
rx2.wait_for(|val| val == "ready").await.unwrap();
// Useful for startup synchronization, health checks, feature flags
```

### Oneshot Channel (Tokio)

Single value, single consumer — used for request-response:

```rust
use tokio::sync::oneshot;

let (tx, rx) = oneshot::channel();
tokio::spawn(async move {
    let result = expensive_async_work().await;
    let _ = tx.send(result);
});
let value = rx.await.unwrap();
```

### Crossbeam Channels

`crossbeam-channel` provides high-performance bounded/unbounded channels with MPMC support:

```rust
use crossbeam_channel::{bounded, unbounded, select, Receiver, Sender};
use std::thread;
use std::time::Duration;

// Bounded channel with explicit capacity (provides backpressure)
let (tx, rx): (Sender<i32>, Receiver<i32>) = bounded(10);

// Unbounded channel (no capacity limit)
let (tx_unbounded, rx_unbounded) = unbounded();

// Producer blocks when bounded channel is full
thread::spawn(move || {
    for i in 0..100 {
        tx.send(i).unwrap();  // Blocks if buffer full
    }
});

// Consumer
for msg in rx {
    println!("Received: {}", msg);
}
```

### Crossbeam Select

```rust
use crossbeam_channel::{bounded, select, Receiver};
use std::time::Duration;

fn process_multiple_channels(rx1: Receiver<i32>, rx2: Receiver<String>) {
    loop {
        select! {
            recv(rx1) -> msg => {
                match msg {
                    Ok(n) => println!("Channel 1: {}", n),
                    Err(_) => break,  // Channel closed
                }
            }
            recv(rx2) -> msg => {
                match msg {
                    Ok(s) => println!("Channel 2: {}", s),
                    Err(_) => break,
                }
            }
            default(Duration::from_secs(1)) => {
                println!("Timeout, no messages");
            }
        }
    }
}
```

### CSP Data Processing Pipeline

Multi-stage pipeline using channels for concurrent data processing:

```rust
use tokio::sync::mpsc::{self, Receiver, Sender};

// Stage 1: Reader - produces raw data
async fn reader_stage(tx: Sender<String>) {
    for i in 0..100 {
        let data = format!("item:{}", i);
        if tx.send(data).await.is_err() {
            break;  // Downstream closed
        }
    }
    // Sender dropped here, signals completion
}

// Stage 2: Parser - transforms data
async fn parser_stage(mut rx: Receiver<String>, tx: Sender<ParsedItem>) {
    while let Some(raw) = rx.recv().await {
        if let Some(parsed) = parse_item(&raw) {
            if tx.send(parsed).await.is_err() {
                break;
            }
        }
    }
}

// Stage 3: Filter - applies business logic
async fn filter_stage(mut rx: Receiver<ParsedItem>, tx: Sender<ParsedItem>) {
    while let Some(item) = rx.recv().await {
        if item.is_valid() {
            if tx.send(item).await.is_err() {
                break;
            }
        }
    }
}

// Stage 4: Writer - consumes final output
async fn writer_stage(mut rx: Receiver<ParsedItem>) {
    while let Some(item) = rx.recv().await {
        write_item(&item).await;
    }
}

// Assemble the pipeline
#[tokio::main]
async fn main() {
    let (tx1, rx1) = mpsc::channel(32);  // Bounded for backpressure
    let (tx2, rx2) = mpsc::channel(32);
    let (tx3, rx3) = mpsc::channel(32);

    // Spawn all stages concurrently
    let reader = tokio::spawn(reader_stage(tx1));
    let parser = tokio::spawn(parser_stage(rx1, tx2));
    let filter = tokio::spawn(filter_stage(rx2, tx3));
    let writer = tokio::spawn(writer_stage(rx3));

    // Wait for pipeline completion
    // Shutdown propagates: reader finishes -> drops tx1 ->
    // parser sees None -> drops tx2 -> filter sees None -> etc.
    let _ = tokio::join!(reader, parser, filter, writer);
}
```

## Async/Await

### Basic Async Functions

```rust
async fn fetch_data(url: &str) -> Result<String, Error> {
    let response = reqwest::get(url).await?;
    let body = response.text().await?;
    Ok(body)
}

// Async blocks
let future = async {
    let data = fetch_data("https://api.example.com").await?;
    process(data).await
};
```

### Running Async Code

```rust
// Using tokio runtime
#[tokio::main]
async fn main() {
    let result = fetch_data("https://example.com").await;
    println!("{:?}", result);
}

// Manual runtime creation
fn main() {
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        fetch_data("https://example.com").await
    });
}
```

### Spawning Async Tasks

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    // Spawn concurrent task
    let handle = task::spawn(async {
        expensive_computation().await
    });

    // Do other work...
    do_something_else().await;

    // Wait for spawned task
    let result = handle.await.unwrap();
}
```

### Concurrent Execution

```rust
use tokio::join;
use futures::future::join_all;

// Run multiple futures concurrently
let (a, b, c) = join!(
    fetch_data("url1"),
    fetch_data("url2"),
    fetch_data("url3"),
);

// Join dynamic number of futures
let urls = vec!["url1", "url2", "url3"];
let futures: Vec<_> = urls.iter().map(|u| fetch_data(u)).collect();
let results = join_all(futures).await;

// Select first to complete
use tokio::select;
select! {
    result = fetch_data("url1") => println!("First: {:?}", result),
    result = fetch_data("url2") => println!("Second: {:?}", result),
}
```

## Futures and Streams

### The Future Trait

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Instant;

// Future is a value that will resolve to a result
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

// Custom future (rarely needed)
struct Delay {
    when: Instant,
}

impl Future for Delay {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if Instant::now() >= self.when {
            Poll::Ready(())
        } else {
            // Schedule wake-up
            let waker = cx.waker().clone();
            let when = self.when;
            std::thread::spawn(move || {
                std::thread::sleep(when - Instant::now());
                waker.wake();
            });
            Poll::Pending
        }
    }
}
```

### Streams

```rust
use futures::stream::{self, StreamExt};

// Stream is async iterator
async fn process_stream() {
    let stream = stream::iter(vec![1, 2, 3, 4, 5]);

    // Process each item
    stream.for_each(|item| async move {
        println!("Item: {}", item);
    }).await;

    // Map and collect
    let stream = stream::iter(vec![1, 2, 3]);
    let results: Vec<_> = stream.map(|x| x * 2).collect().await;

    // Filter
    let stream = stream::iter(1..10);
    let evens: Vec<_> = stream.filter(|x| async move { x % 2 == 0 }).collect().await;
}
```

## Tokio Runtime Architecture

### Work-Stealing Scheduler

Tokio uses a multi-threaded work-stealing scheduler based on the Chase-Lev algorithm:

```rust
// Default: one worker thread per CPU core
#[tokio::main]
async fn main() {
    // Tasks are distributed across worker threads
    // Idle workers "steal" tasks from busy workers' queues
}

// LIFO slot: recently spawned tasks run on same thread (cache locality)
// FIFO queue: older tasks available for stealing
```

### Runtime Configuration

```rust
use tokio::runtime::Builder;

// Multi-threaded runtime with custom settings
let runtime = Builder::new_multi_thread()
    .worker_threads(4)              // Number of worker threads
    .max_blocking_threads(512)      // Blocking thread pool size
    .thread_name("my-worker")       // Thread naming for debugging
    .thread_stack_size(3 * 1024 * 1024)
    .enable_all()                   // Enable I/O and time drivers
    .build()
    .unwrap();

runtime.block_on(async {
    // Your async code here
});

// Single-threaded runtime (useful for testing, embedded)
let rt = Builder::new_current_thread()
    .enable_all()
    .build()
    .unwrap();
```

Attribute macro shortcuts:

```rust
// Default multi-threaded runtime
#[tokio::main]
async fn main() { }

// Custom worker threads
#[tokio::main(flavor = "multi_thread", worker_threads = 4)]
async fn main() { }

// Single-threaded (for embedded or constrained environments)
#[tokio::main(flavor = "current_thread")]
async fn main() { }
```

### Cooperative Scheduling

Tokio uses cooperative scheduling — tasks must yield voluntarily. Long-running CPU work blocks the executor:

```rust
// BAD: CPU-bound work blocks the executor
async fn bad_handler() -> String {
    let result = expensive_computation(); // Blocks thread!
    format!("result: {result}")
}

// GOOD: Use spawn_blocking for CPU-bound work
async fn good_handler() -> String {
    let result = tokio::task::spawn_blocking(|| {
        expensive_computation()
    }).await.unwrap();
    format!("result: {result}")
}
```

For long-running async computations, yield periodically:

```rust
async fn cpu_intensive_work(data: &[u32]) -> u64 {
    let mut sum: u64 = 0;
    for (i, &item) in data.iter().enumerate() {
        sum += item as u64;

        // Yield every 1000 iterations to prevent starvation
        if i % 1000 == 0 {
            tokio::task::yield_now().await;
        }
    }
    sum
}
```

### Runtime Introspection with tokio-console

```rust
// Cargo.toml
// [dependencies]
// console-subscriber = "0.2"
// tokio = { version = "1", features = ["full", "tracing"] }

#[tokio::main]
async fn main() {
    // Initialize console subscriber for runtime introspection
    console_subscriber::init();

    // Now run: tokio-console to view task states, poll times, etc.
    my_app().await;
}
```

## Task Management with JoinSet

### Managing Dynamic Task Collections

```rust
use tokio::task::JoinSet;

#[tokio::main]
async fn main() {
    let mut join_set = JoinSet::new();

    // Spawn tasks dynamically
    for i in 0..10 {
        join_set.spawn(async move {
            tokio::time::sleep(tokio::time::Duration::from_millis(100 * i)).await;
            format!("Task {} completed", i)
        });
    }

    // Process results as tasks complete (not in spawn order)
    while let Some(result) = join_set.join_next().await {
        match result {
            Ok(output) => println!("{}", output),
            Err(e) => eprintln!("Task failed: {}", e),
        }
    }
}
```

### JoinSet with Early Termination

```rust
use tokio::task::JoinSet;

async fn find_first_success<T, E>(
    tasks: impl IntoIterator<Item = impl std::future::Future<Output = Result<T, E>> + Send + 'static>,
) -> Option<T>
where
    T: Send + 'static,
    E: Send + 'static,
{
    let mut join_set = JoinSet::new();

    for task in tasks {
        join_set.spawn(task);
    }

    // Return first successful result, abort remaining tasks
    while let Some(result) = join_set.join_next().await {
        if let Ok(Ok(value)) = result {
            join_set.abort_all();  // Cancel remaining tasks
            return Some(value);
        }
    }

    None
}
```

### JoinSet with Timeout

```rust
use tokio::task::JoinSet;
use tokio::time::{timeout, Duration};

async fn process_with_deadline<T>(
    mut join_set: JoinSet<T>,
    deadline: Duration,
) -> Vec<T>
where
    T: Send + 'static,
{
    let mut results = Vec::new();
    let deadline = tokio::time::Instant::now() + deadline;

    loop {
        let remaining = deadline.saturating_duration_since(tokio::time::Instant::now());
        if remaining.is_zero() {
            break;
        }

        match timeout(remaining, join_set.join_next()).await {
            Ok(Some(Ok(result))) => results.push(result),
            Ok(Some(Err(_))) => continue,  // Task panicked
            Ok(None) => break,              // All tasks done
            Err(_) => break,                // Timeout
        }
    }

    // Abort any remaining tasks
    join_set.abort_all();
    results
}
```

### JoinSet for Parallel Processing

```rust
use tokio::task::JoinSet;

async fn parallel_fetch(urls: Vec<String>) -> Vec<Result<String, reqwest::Error>> {
    let mut join_set = JoinSet::new();

    for url in urls {
        join_set.spawn(async move {
            reqwest::get(&url).await?.text().await
        });
    }

    let mut results = Vec::new();
    while let Some(result) = join_set.join_next().await {
        match result {
            Ok(fetch_result) => results.push(fetch_result),
            Err(e) => results.push(Err(e.into())),
        }
    }

    results
}
```

### JoinSet with Timeout via select!

```rust
async fn fetch_with_timeout(urls: Vec<String>) -> Vec<String> {
    let mut set = JoinSet::new();
    for url in urls {
        set.spawn(async move { reqwest::get(&url).await?.text().await });
    }

    let mut results = Vec::new();
    let deadline = tokio::time::Instant::now() + std::time::Duration::from_secs(10);

    loop {
        tokio::select! {
            Some(res) = set.join_next() => {
                if let Ok(Ok(body)) = res { results.push(body); }
            }
            _ = tokio::time::sleep_until(deadline) => {
                set.abort_all();
                break;
            }
            else => break,
        }
    }
    results
}
```

### JoinSet vs join_all

```rust
use tokio::task::JoinSet;
use futures::future::join_all;

// join_all: Fixed set of futures, preserves order
async fn with_join_all(items: Vec<i32>) -> Vec<i32> {
    let futures: Vec<_> = items.into_iter()
        .map(|x| async move { x * 2 })
        .collect();
    join_all(futures).await  // Results in same order as input
}

// JoinSet: Dynamic, results in completion order
async fn with_join_set(items: Vec<i32>) -> Vec<i32> {
    let mut join_set = JoinSet::new();
    for x in items {
        join_set.spawn(async move { x * 2 });
    }

    let mut results = Vec::new();
    while let Some(Ok(result)) = join_set.join_next().await {
        results.push(result);  // Completion order, not input order
    }
    results
}

// Use JoinSet when:
// - Tasks are spawned dynamically during processing
// - You want to process results as they complete
// - You need to abort remaining tasks
// - Task count may change

// Use join_all when:
// - Fixed set of futures known upfront
// - Order of results must match input order
// - Simpler, less overhead for small fixed sets
```

### JoinSet Advanced Methods

```rust
use tokio::task::JoinSet;

let mut set = JoinSet::new();
for i in 0..10 {
    set.spawn(async move { expensive_work(i).await });
}

// join_all: await ALL tasks, panic on failure (consumes JoinSet)
let all_results: Vec<Output> = set.join_all().await;

// shutdown: abort all + wait for completion (ignores panics)
set.shutdown().await;

// detach_all: tasks continue running but JoinSet forgets them
set.detach_all();

// abort_all: signal cancellation (tasks still need join_next to drain)
set.abort_all();
while set.join_next().await.is_some() {} // drain cancelled tasks

// try_join_next: non-blocking poll (returns None if nothing ready)
if let Some(result) = set.try_join_next() {
    handle(result?);
}
```

**Cancel-safety:** `join_next()` is cancel-safe — if used in `tokio::select!` and another branch completes first, no tasks are lost from the set. This makes it safe to combine with timeouts and shutdown signals.

## Pin/Unpin

### Why Pin Exists

Self-referential types (like futures containing references to their own data) break if moved in memory. `Pin<P>` guarantees the value won't move.

```rust
use std::pin::Pin;
use std::future::Future;

// Most types are Unpin (safe to move) — Pin has no effect
// Futures generated by async blocks are !Unpin — Pin is required

// When you need to store a future:
struct TaskQueue {
    tasks: Vec<Pin<Box<dyn Future<Output = ()> + Send>>>,
}

impl TaskQueue {
    fn add<F: Future<Output = ()> + Send + 'static>(&mut self, f: F) {
        self.tasks.push(Box::pin(f));
    }
}

// pin! macro for stack pinning
async fn example() {
    let fut = async { 42 };
    tokio::pin!(fut);
    // fut is now Pin<&mut impl Future<Output = i32>>
    let result = (&mut fut).await;
}
```

### When You Need Pin

| Situation | Pin needed? |
|-----------|-------------|
| `async fn` parameters | No — handled automatically |
| Storing `dyn Future` in struct | Yes — `Pin<Box<dyn Future>>` |
| `tokio::select!` on futures | Yes — `tokio::pin!(fut)` |
| Implementing `Future` manually | Yes — `self: Pin<&mut Self>` |
| Regular structs/enums | No — most types are `Unpin` |

For deeper Pin/Unpin internals (pin projection, pin-project crate, manual Future implementation), see [type-system.md](type-system.md).

## Parallel Iterators (Rayon)

### Basic Parallel Iteration

```rust
use rayon::prelude::*;

// Parallel map
let results: Vec<_> = data.par_iter()
    .map(|x| expensive_computation(x))
    .collect();

// Parallel filter
let filtered: Vec<_> = data.par_iter()
    .filter(|x| x.is_valid())
    .collect();

// Parallel fold/reduce
let sum: i32 = data.par_iter().sum();
let max = data.par_iter().max();

// Parallel for_each
data.par_iter().for_each(|item| {
    process(item);
});

// Control chunk size
let results: Vec<_> = data.par_chunks(100)
    .map(|chunk| process_chunk(chunk))
    .collect();

// Parallel sort
let mut data = vec![5, 3, 1, 4, 2];
data.par_sort();
```

### Custom Thread Pools with Rayon

```rust
use rayon::ThreadPoolBuilder;

// Create custom thread pool
let pool = ThreadPoolBuilder::new()
    .num_threads(4)                    // Explicit thread count
    .thread_name(|i| format!("worker-{}", i))
    .stack_size(8 * 1024 * 1024)       // 8MB stack per thread
    .build()
    .expect("Failed to build thread pool");

// Execute parallel work on custom pool
pool.install(|| {
    // All rayon operations in this closure use this pool
    let results: Vec<_> = data.par_iter()
        .map(|x| expensive_computation(x))
        .collect();
    results
});

// Spawn tasks on custom pool
pool.spawn(|| {
    println!("Running on custom pool");
});

// Scoped execution with join
pool.scope(|s| {
    s.spawn(|_| println!("Task 1"));
    s.spawn(|_| println!("Task 2"));
});  // Waits for all spawned tasks
```

### Rayon Global Pool Configuration

```rust
use rayon::ThreadPoolBuilder;

// Configure global pool (call once at startup)
ThreadPoolBuilder::new()
    .num_threads(std::thread::available_parallelism().map_or(4, |n| n.get()))
    .build_global()
    .expect("Failed to initialize global thread pool");

// Now par_iter() uses the configured global pool
let sum: i32 = (0..1000).into_par_iter().sum();
```

### Bridging Async and Rayon

```rust
// Bridge async → rayon
async fn compute_parallel(data: Vec<f64>) -> f64 {
    tokio::task::spawn_blocking(move || {
        data.par_iter().map(|x| x.sqrt()).sum()
    }).await.unwrap()
}
```

## Atomic Operations

### Basic Atomics

```rust
use std::sync::atomic::{AtomicUsize, AtomicBool, AtomicU64, Ordering};

static COUNTER: AtomicUsize = AtomicUsize::new(0);
static FLAG: AtomicBool = AtomicBool::new(false);
static REQUEST_COUNT: AtomicU64 = AtomicU64::new(0);

fn increment() {
    COUNTER.fetch_add(1, Ordering::SeqCst);
}

fn set_flag() {
    FLAG.store(true, Ordering::Release);
}

fn check_flag() -> bool {
    FLAG.load(Ordering::Acquire)
}

// Compare and swap
fn try_increment(expected: usize, new: usize) -> bool {
    COUNTER.compare_exchange(expected, new, Ordering::SeqCst, Ordering::SeqCst).is_ok()
}
```

### Memory Ordering

```rust
use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering};
use std::sync::Arc;
use std::thread;

// Relaxed: Only guarantees atomicity, no synchronization
// Use for counters where only the final value matters
static COUNTER: AtomicUsize = AtomicUsize::new(0);
COUNTER.fetch_add(1, Ordering::Relaxed);

// Acquire/Release: Synchronize data between threads
// Release: All writes before this are visible to Acquire loads
// Acquire: Sees all writes that happened before the Release
let data_ready = Arc::new(AtomicBool::new(false));
let shared_data = Arc::new(std::sync::Mutex::new(Vec::new()));

// Producer thread
let data_ready_p = Arc::clone(&data_ready);
let shared_p = Arc::clone(&shared_data);
thread::spawn(move || {
    shared_p.lock().unwrap().push(42);  // Write data
    data_ready_p.store(true, Ordering::Release);  // Signal ready
});

// Consumer thread
let data_ready_c = Arc::clone(&data_ready);
let shared_c = Arc::clone(&shared_data);
thread::spawn(move || {
    while !data_ready_c.load(Ordering::Acquire) {
        std::hint::spin_loop();
    }
    // Guaranteed to see the push(42) from producer
    let data = shared_c.lock().unwrap();
    assert_eq!(data[0], 42);
});

// SeqCst: Strongest ordering, total order across all threads
// Use when multiple atomics must be seen in consistent order
static A: AtomicBool = AtomicBool::new(false);
static B: AtomicBool = AtomicBool::new(false);
A.store(true, Ordering::SeqCst);
B.store(true, Ordering::SeqCst);
```

Memory ordering guide:
- **Relaxed** — no ordering guarantees, fine for counters
- **Acquire/Release** — paired for synchronization (lock/unlock)
- **SeqCst** — total ordering, safest, slight overhead

### Compare-Exchange for Lock-Free Operations

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

static VALUE: AtomicUsize = AtomicUsize::new(0);

// Atomically increment only if current value matches expected
fn conditional_increment(expected: usize) -> Result<usize, usize> {
    VALUE.compare_exchange(
        expected,           // Expected current value
        expected + 1,       // New value if match
        Ordering::AcqRel,   // Success ordering
        Ordering::Relaxed,  // Failure ordering
    )
    // Returns Ok(old) if swapped, Err(actual) if not
}

// Lock-free increment with retry loop
fn atomic_increment() -> usize {
    loop {
        let current = VALUE.load(Ordering::Relaxed);
        match VALUE.compare_exchange_weak(
            current,
            current + 1,
            Ordering::AcqRel,
            Ordering::Relaxed,
        ) {
            Ok(_) => return current + 1,
            Err(_) => continue,  // Retry on failure
        }
    }
}

// compare_exchange_weak may fail spuriously (more efficient on some CPUs)
// Use in loops where you retry anyway
// compare_exchange never fails spuriously (use for single attempts)
```

### AtomicPtr for Lock-Free Data Structures

```rust
use std::sync::atomic::{AtomicPtr, Ordering};
use std::ptr;

struct Node<T> {
    data: T,
    next: AtomicPtr<Node<T>>,
}

struct LockFreeStack<T> {
    head: AtomicPtr<Node<T>>,
}

impl<T> LockFreeStack<T> {
    fn new() -> Self {
        LockFreeStack {
            head: AtomicPtr::new(ptr::null_mut()),
        }
    }

    fn push(&self, data: T) {
        let new_node = Box::into_raw(Box::new(Node {
            data,
            next: AtomicPtr::new(ptr::null_mut()),
        }));

        loop {
            let old_head = self.head.load(Ordering::Relaxed);
            unsafe { (*new_node).next.store(old_head, Ordering::Relaxed) };

            match self.head.compare_exchange_weak(
                old_head,
                new_node,
                Ordering::Release,
                Ordering::Relaxed,
            ) {
                Ok(_) => break,
                Err(_) => continue,
            }
        }
    }

    fn pop(&self) -> Option<T> {
        loop {
            let head = self.head.load(Ordering::Acquire);
            if head.is_null() {
                return None;
            }

            let next = unsafe { (*head).next.load(Ordering::Relaxed) };

            match self.head.compare_exchange_weak(
                head,
                next,
                Ordering::AcqRel,
                Ordering::Acquire,
            ) {
                Ok(_) => {
                    let node = unsafe { Box::from_raw(head) };
                    return Some(node.data);
                }
                Err(_) => continue,
            }
        }
    }
}

// Note: Real lock-free structures need memory reclamation (epoch-based, hazard pointers)
```

## Deadlock Prevention

Deadlocks occur when threads hold locks while waiting for other locks, creating circular dependencies. Prevention strategies focus on eliminating the conditions that allow deadlocks.

### The Deadlock Problem

```rust
use std::sync::{Arc, Mutex};
use std::thread;

// BAD: Potential deadlock - threads acquire locks in different order
fn bad_transfer(
    from: &Arc<Mutex<Account>>,
    to: &Arc<Mutex<Account>>,
    amount: f64,
) {
    let mut from_guard = from.lock().unwrap();  // Thread A locks account 1
    let mut to_guard = to.lock().unwrap();      // Thread A waits for account 2

    // Meanwhile Thread B might call transfer(to, from, amount):
    // Thread B locks account 2 (already held by... nobody yet, but will try)
    // Thread B waits for account 1 (held by Thread A)
    // DEADLOCK if timing is right!

    from_guard.balance -= amount;
    to_guard.balance += amount;
}
```

### Consistent Lock Ordering

Always acquire locks in the same order across all code paths:

```rust
use std::sync::{Arc, Mutex};

#[derive(Debug)]
pub struct Account {
    pub id: u64,
    pub balance: Mutex<f64>,
}

impl Account {
    pub fn new(id: u64, initial_balance: f64) -> Arc<Self> {
        Arc::new(Account {
            id,
            balance: Mutex::new(initial_balance),
        })
    }
}

/// Transfer money between accounts - deadlock-free via consistent ordering
pub fn transfer(from: &Arc<Account>, to: &Arc<Account>, amount: f64) -> Result<(), TransferError> {
    if from.id == to.id {
        return Err(TransferError::SameAccount);
    }

    // Always lock lower ID first - guarantees consistent ordering
    let (first, second, from_is_first) = if from.id < to.id {
        (from, to, true)
    } else {
        (to, from, false)
    };

    // Acquire locks in consistent order
    let mut first_balance = first.balance.lock().unwrap();
    let mut second_balance = second.balance.lock().unwrap();

    // Now perform the transfer
    let (from_balance, to_balance) = if from_is_first {
        (&mut *first_balance, &mut *second_balance)
    } else {
        (&mut *second_balance, &mut *first_balance)
    };

    if *from_balance < amount {
        return Err(TransferError::InsufficientFunds);
    }

    *from_balance -= amount;
    *to_balance += amount;

    Ok(())
}

#[derive(Debug, thiserror::Error)]
pub enum TransferError {
    #[error("Cannot transfer to same account")]
    SameAccount,
    #[error("Insufficient funds")]
    InsufficientFunds,
}

// Usage is now safe from deadlocks
fn concurrent_transfers() {
    let account_a = Account::new(1, 1000.0);
    let account_b = Account::new(2, 500.0);

    let a1 = Arc::clone(&account_a);
    let b1 = Arc::clone(&account_b);
    let handle1 = thread::spawn(move || {
        for _ in 0..100 {
            let _ = transfer(&a1, &b1, 10.0);
        }
    });

    let a2 = Arc::clone(&account_a);
    let b2 = Arc::clone(&account_b);
    let handle2 = thread::spawn(move || {
        for _ in 0..100 {
            let _ = transfer(&b2, &a2, 10.0);  // Opposite direction - still safe!
        }
    });

    handle1.join().unwrap();
    handle2.join().unwrap();
}
```

### Lock-Free Alternative with try_lock

Avoid deadlocks by refusing to block:

```rust
use std::sync::{Arc, Mutex, TryLockError};
use std::thread;
use std::time::Duration;

/// Transfer using try_lock with retry - never blocks indefinitely
pub fn transfer_with_retry(
    from: &Arc<Mutex<f64>>,
    to: &Arc<Mutex<f64>>,
    amount: f64,
    max_retries: usize,
) -> Result<(), &'static str> {
    for attempt in 0..max_retries {
        // Try to acquire both locks
        let from_result = from.try_lock();
        let to_result = to.try_lock();

        match (from_result, to_result) {
            (Ok(mut from_guard), Ok(mut to_guard)) => {
                // Got both locks!
                if *from_guard < amount {
                    return Err("Insufficient funds");
                }
                *from_guard -= amount;
                *to_guard += amount;
                return Ok(());
            }
            _ => {
                // Couldn't get both locks - back off and retry
                // Randomized backoff helps reduce contention
                let backoff = Duration::from_micros(10 * (1 << attempt.min(10)));
                thread::sleep(backoff);
            }
        }
    }

    Err("Max retries exceeded")
}
```

### Hierarchical Locking

Assign a hierarchy level to each lock type; always acquire higher levels before lower:

```rust
/// Lock hierarchy (higher number = acquired later)
/// 1. GlobalConfig
/// 2. UserManager
/// 3. Individual User
/// 4. Session

pub struct Application {
    config: Mutex<GlobalConfig>,        // Level 1
    user_manager: Mutex<UserManager>,   // Level 2
    sessions: Mutex<SessionStore>,      // Level 4
}

impl Application {
    /// Safe: acquires locks in hierarchy order
    pub fn update_user_with_config(&self, user_id: u64, new_setting: Setting) {
        let config = self.config.lock().unwrap();           // Level 1
        let mut users = self.user_manager.lock().unwrap();  // Level 2 (after 1)

        if config.allows_setting(&new_setting) {
            users.update_setting(user_id, new_setting);
        }
    }

    /// Safe: skips level 1, but level 4 is still after level 2
    pub fn user_session_operation(&self, user_id: u64) {
        let users = self.user_manager.lock().unwrap();      // Level 2
        let mut sessions = self.sessions.lock().unwrap();   // Level 4 (after 2)

        if let Some(user) = users.get(user_id) {
            sessions.refresh_for_user(user);
        }
    }

    // WOULD BE WRONG (but prevented by design):
    // fn bad_operation(&self) {
    //     let sessions = self.sessions.lock().unwrap();    // Level 4
    //     let config = self.config.lock().unwrap();        // Level 1 - WRONG ORDER!
    // }
}
```

### Using parking_lot for Deadlock Detection

The `parking_lot` crate provides deadlock detection in debug builds:

```rust
// Cargo.toml
// [dependencies]
// parking_lot = { version = "0.12", features = ["deadlock_detection"] }

use parking_lot::{Mutex, deadlock};
use std::thread;
use std::time::Duration;

fn setup_deadlock_detection() {
    // Spawn a background thread to check for deadlocks
    thread::spawn(move || {
        loop {
            thread::sleep(Duration::from_secs(10));

            let deadlocks = deadlock::check_deadlock();
            if deadlocks.is_empty() {
                continue;
            }

            eprintln!("{} deadlocks detected!", deadlocks.len());
            for (i, threads) in deadlocks.iter().enumerate() {
                eprintln!("Deadlock #{}", i + 1);
                for t in threads {
                    eprintln!("Thread {:?}:", t.thread_id());
                    eprintln!("{:#?}", t.backtrace());
                }
            }
        }
    });
}

fn main() {
    setup_deadlock_detection();

    // Your application code...
}
```

### Avoiding Deadlocks: Summary

| Strategy | When to Use |
|----------|-------------|
| Consistent ordering | Multiple locks of same type (accounts, resources) |
| try_lock with retry | Can tolerate retry latency, low contention |
| Hierarchical locking | Different lock types with clear dependency order |
| Lock-free structures | High contention, simple operations |
| Message passing | Complex coordination, actor-style systems |

## False Sharing Prevention

When threads access different fields of the same cache line, hardware forces constant cache invalidation — "false sharing." This silently destroys parallel throughput:

```rust
use crossbeam_utils::CachePadded;

// BAD: counters share cache lines — threads invalidate each other
struct SharedCounters {
    reads: AtomicU64,
    writes: AtomicU64,
}

// GOOD: each counter gets its own cache line (typically 64 bytes)
struct PaddedCounters {
    reads: CachePadded<AtomicU64>,
    writes: CachePadded<AtomicU64>,
}
```

**When to use CachePadded:**
- Atomics updated frequently by different threads
- Per-thread counters aggregated at read time
- Lock-free data structures with adjacent hot fields

### Shard-Per-Lock Concurrency

Instead of one lock over an entire collection, shard by key hash to reduce contention. This is how `dashmap` achieves high throughput:

```rust
use std::collections::HashMap;
use std::hash::{BuildHasher, Hash, Hasher, RandomState};
use std::sync::RwLock;

const NUM_SHARDS: usize = 64;

struct ShardedMap<K, V> {
    shards: Vec<RwLock<HashMap<K, V>>>,
    hasher: RandomState,
}

impl<K: Hash + Eq, V> ShardedMap<K, V> {
    fn new() -> Self {
        Self {
            shards: (0..NUM_SHARDS).map(|_| RwLock::new(HashMap::new())).collect(),
            hasher: RandomState::new(),
        }
    }

    fn shard_index(&self, key: &K) -> usize {
        let mut hasher = self.hasher.build_hasher();
        key.hash(&mut hasher);
        hasher.finish() as usize % NUM_SHARDS
    }

    fn insert(&self, key: K, value: V) -> Option<V> {
        let idx = self.shard_index(&key);
        self.shards[idx].write().unwrap().insert(key, value)
    }

    fn get<Q>(&self, key: &K) -> Option<V>
    where
        K: std::borrow::Borrow<K>,
        V: Clone,
    {
        let idx = self.shard_index(key);
        self.shards[idx].read().unwrap().get(key).cloned()
    }
}
```

**Guidelines:**
- 64 shards is a good default — matches typical L1 cache line count
- Use `RwLock` per shard for read-heavy workloads, `Mutex` for write-heavy
- Consider `CachePadded<RwLock<...>>` per shard if contention is extreme
- In practice, prefer `dashmap` over hand-rolling unless you need custom behavior

## Tower Service Pattern

### The Service Trait

Tower provides a composable abstraction for request/response services:

```rust
use tower::{Service, ServiceExt};
use std::task::{Context, Poll};
use std::future::Future;
use std::pin::Pin;

// Service trait: async fn(Request) -> Result<Response, Error>
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;
    fn call(&mut self, req: Request) -> Self::Future;
}

// Simple service implementation
struct MyService;

impl Service<String> for MyService {
    type Response = String;
    type Error = std::convert::Infallible;
    type Future = Pin<Box<dyn Future<Output = Result<String, Self::Error>> + Send>>;

    fn poll_ready(&mut self, _cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        Poll::Ready(Ok(()))
    }

    fn call(&mut self, req: String) -> Self::Future {
        Box::pin(async move {
            Ok(format!("Hello, {}!", req))
        })
    }
}
```

### Middleware Composition

```rust
use tower::{ServiceBuilder, timeout::TimeoutLayer, limit::RateLimitLayer};
use std::time::Duration;

// Compose middleware layers
let service = ServiceBuilder::new()
    // Layers are applied bottom-up (innermost first)
    .layer(TimeoutLayer::new(Duration::from_secs(30)))
    .layer(RateLimitLayer::new(100, Duration::from_secs(1)))
    .service(MyService);

// Common Tower middleware:
// - TimeoutLayer: Fail requests exceeding time limit
// - RateLimitLayer: Limit requests per time window
// - BufferLayer: Add request buffering/queueing
// - RetryLayer: Automatic retry with policy
// - ConcurrencyLimitLayer: Limit concurrent requests
```

### Load Balancing

```rust
use tower::balance::p2c::Balance;
use tower::discover::ServiceList;

// Power-of-two-choices load balancer
let services = vec![service1, service2, service3];
let discover = ServiceList::new(services);
let load_balanced = Balance::new(discover);

// Requests distributed based on load metrics
```

## Graceful Shutdown

### Signal Handling

```rust
use tokio::signal;
use tokio::sync::broadcast;
use std::time::Duration;

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c()
            .await
            .expect("Failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("Failed to install signal handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}

// Application with graceful shutdown
#[tokio::main]
async fn main() {
    let (shutdown_tx, _) = broadcast::channel::<()>(1);

    // Spawn workers with shutdown receiver
    let shutdown_rx = shutdown_tx.subscribe();
    let worker = tokio::spawn(async move {
        run_worker(shutdown_rx).await;
    });

    // Wait for shutdown signal
    shutdown_signal().await;
    println!("Shutdown signal received");

    // Notify all workers
    let _ = shutdown_tx.send(());

    // Wait for graceful completion with timeout
    let _ = tokio::time::timeout(
        Duration::from_secs(30),
        worker
    ).await;
}

async fn run_worker(mut shutdown: broadcast::Receiver<()>) {
    loop {
        tokio::select! {
            // biased; ensures shutdown is checked first (deterministic priority)
            // Without biased, branches are polled in random order — shutdown may be delayed
            biased;

            _ = shutdown.recv() => {
                println!("Worker shutting down");
                break;
            }
            _ = do_work() => {}
        }
    }
}
```

### Cancellation Tokens

```rust
use tokio_util::sync::CancellationToken;

pub async fn run(token: CancellationToken) {
    let worker_token = token.clone();
    let worker = tokio::spawn(async move {
        loop {
            tokio::select! {
                _ = worker_token.cancelled() => {
                    tracing::info!("shutting down");
                    break;
                }
                _ = do_work() => {}
            }
        }
    });

    // Wait for shutdown signal
    shutdown_signal().await;
    token.cancel();

    let _ = tokio::time::timeout(
        std::time::Duration::from_secs(30),
        worker,
    ).await;
}
```

### Connection Draining

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use std::time::{Duration, Instant};

struct Server {
    active_connections: Arc<AtomicUsize>,
}

impl Server {
    async fn graceful_shutdown(&self, timeout: Duration) {
        // Stop accepting new connections (handled elsewhere)

        // Wait for existing connections to complete
        let start = Instant::now();
        while self.active_connections.load(Ordering::SeqCst) > 0 {
            if start.elapsed() > timeout {
                println!("Timeout: {} connections still active",
                    self.active_connections.load(Ordering::SeqCst));
                break;
            }
            tokio::time::sleep(Duration::from_millis(100)).await;
        }
    }
}
```

## Backpressure and Flow Control

### Bounded Channels for Backpressure

```rust
use tokio::sync::mpsc;

// Bounded channel creates natural backpressure
let (tx, mut rx) = mpsc::channel::<Work>(100); // Buffer of 100

// Producer slows down when buffer full
tokio::spawn(async move {
    for item in work_items {
        // Blocks when channel is full
        tx.send(item).await.unwrap();
    }
});

// Consumer processes at its own pace
tokio::spawn(async move {
    while let Some(work) = rx.recv().await {
        process(work).await;
    }
});
```

### Semaphore for Concurrency Limiting

```rust
use tokio::sync::Semaphore;
use std::sync::Arc;

// Limit concurrent operations
let semaphore = Arc::new(Semaphore::new(10)); // Max 10 concurrent

async fn rate_limited_operation(sem: Arc<Semaphore>) {
    // Acquire permit (blocks if at limit)
    let _permit = sem.acquire().await.unwrap();

    // Permit automatically released when dropped
    do_work().await;
}

// Process many items with limited concurrency
let handles: Vec<_> = items.into_iter().map(|item| {
    let sem = semaphore.clone();
    tokio::spawn(async move {
        let _permit = sem.acquire().await.unwrap();
        process(item).await
    })
}).collect();

futures::future::join_all(handles).await;
```

### Load Shedding

```rust
use tokio::sync::Semaphore;

struct LoadShedder {
    permits: Semaphore,
}

impl LoadShedder {
    fn new(max_concurrent: usize) -> Self {
        Self {
            permits: Semaphore::new(max_concurrent),
        }
    }

    async fn try_acquire(&self) -> Option<tokio::sync::SemaphorePermit<'_>> {
        // Non-blocking: shed load if at capacity
        self.permits.try_acquire().ok()
    }
}

async fn handle_request(shedder: &LoadShedder) -> Result<Response, Error> {
    match shedder.try_acquire() {
        Some(_permit) => {
            // Process request
            Ok(process_request().await)
        }
        None => {
            // Shed load: return 503 Service Unavailable
            Err(Error::ServiceOverloaded)
        }
    }
}
```

## Async Testing Patterns

### Deterministic Time Control

```rust
#[cfg(test)]
mod tests {
    use tokio::time::{self, Duration, Instant};

    #[tokio::test]
    async fn test_timeout_behavior() {
        // Freeze time for deterministic testing
        time::pause();

        let start = Instant::now();

        let result = time::timeout(
            Duration::from_secs(5),
            async {
                // This would normally take 10 seconds
                time::sleep(Duration::from_secs(10)).await;
                "completed"
            }
        );

        // Advance time by 5 seconds (instant, no real delay)
        time::advance(Duration::from_secs(5)).await;

        // Timeout should trigger
        assert!(result.await.is_err());

        // Verify elapsed time in test
        assert!(start.elapsed() >= Duration::from_secs(5));
    }

    #[tokio::test]
    async fn test_retry_timing() {
        time::pause();

        let attempts = std::sync::Arc::new(std::sync::atomic::AtomicUsize::new(0));
        let attempts_clone = attempts.clone();

        let task = tokio::spawn(async move {
            for _ in 0..3 {
                attempts_clone.fetch_add(1, std::sync::atomic::Ordering::SeqCst);
                time::sleep(Duration::from_secs(1)).await;
            }
        });

        // Advance through all retries instantly
        time::advance(Duration::from_secs(3)).await;
        task.await.unwrap();

        assert_eq!(attempts.load(std::sync::atomic::Ordering::SeqCst), 3);
    }
}
```

### Trait-Based Async Mocking

```rust
use async_trait::async_trait;

#[async_trait]
pub trait HttpClient: Send + Sync {
    async fn get(&self, url: &str) -> Result<String, Error>;
}

// Production implementation
struct RealHttpClient;

#[async_trait]
impl HttpClient for RealHttpClient {
    async fn get(&self, url: &str) -> Result<String, Error> {
        reqwest::get(url).await?.text().await.map_err(Into::into)
    }
}

// Mock for testing
struct MockHttpClient {
    responses: std::collections::HashMap<String, String>,
}

#[async_trait]
impl HttpClient for MockHttpClient {
    async fn get(&self, url: &str) -> Result<String, Error> {
        self.responses
            .get(url)
            .cloned()
            .ok_or(Error::NotFound)
    }
}

// Service using the trait
struct MyService<C: HttpClient> {
    client: C,
}

impl<C: HttpClient> MyService<C> {
    async fn fetch_data(&self) -> Result<Data, Error> {
        let response = self.client.get("https://api.example.com/data").await?;
        Ok(serde_json::from_str(&response)?)
    }
}

#[tokio::test]
async fn test_service_with_mock() {
    let mut responses = std::collections::HashMap::new();
    responses.insert(
        "https://api.example.com/data".to_string(),
        r#"{"value": 42}"#.to_string()
    );

    let mock_client = MockHttpClient { responses };
    let service = MyService { client: mock_client };

    let data = service.fetch_data().await.unwrap();
    assert_eq!(data.value, 42);
}
```

### Barrier Synchronization in Tests

```rust
use tokio::sync::Barrier;
use std::sync::Arc;

#[tokio::test]
async fn test_concurrent_access() {
    let barrier = Arc::new(Barrier::new(3));
    let shared_state = Arc::new(tokio::sync::Mutex::new(Vec::new()));

    let mut handles = vec![];

    for i in 0..3 {
        let b = barrier.clone();
        let state = shared_state.clone();

        handles.push(tokio::spawn(async move {
            // All tasks wait here until all reach this point
            b.wait().await;

            // Now all start simultaneously
            let mut guard = state.lock().await;
            guard.push(i);
        }));
    }

    for handle in handles {
        handle.await.unwrap();
    }

    let final_state = shared_state.lock().await;
    assert_eq!(final_state.len(), 3);
}
```

## Hyper Low-Level HTTP Server

Hyper provides low-level HTTP primitives for building high-performance servers with full control over connection handling.

### HTTP/1.1 Server

```rust
use hyper::{body::Bytes, server::conn::http1, service::service_fn, Request, Response};
use hyper_util::rt::TokioIo;
use http_body_util::Full;
use std::convert::Infallible;
use tokio::net::TcpListener;

async fn handle(req: Request<hyper::body::Incoming>) -> Result<Response<Full<Bytes>>, Infallible> {
    println!("Request: {} {}", req.method(), req.uri());
    Ok(Response::new(Full::new(Bytes::from("Hello, World!"))))
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("Listening on http://127.0.0.1:8080");

    loop {
        let (stream, addr) = listener.accept().await?;
        let io = TokioIo::new(stream);

        tokio::spawn(async move {
            if let Err(err) = http1::Builder::new()
                .serve_connection(io, service_fn(handle))
                .await
            {
                eprintln!("Error serving {}: {:?}", addr, err);
            }
        });
    }
}
```

### HTTP/2 Server

```rust
use hyper::{body::Bytes, server::conn::http2, service::service_fn, Request, Response};
use hyper_util::rt::{TokioExecutor, TokioIo};
use http_body_util::Full;
use std::convert::Infallible;
use tokio::net::TcpListener;

async fn handle(req: Request<hyper::body::Incoming>) -> Result<Response<Full<Bytes>>, Infallible> {
    Ok(Response::new(Full::new(Bytes::from("Hello HTTP/2!"))))
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (stream, _) = listener.accept().await?;
        let io = TokioIo::new(stream);

        tokio::spawn(async move {
            // HTTP/2 requires TokioExecutor for spawning tasks
            if let Err(err) = http2::Builder::new(TokioExecutor::new())
                .serve_connection(io, service_fn(handle))
                .await
            {
                eprintln!("Error: {:?}", err);
            }
        });
    }
}
```

### TokioIo Adapter

The `TokioIo` wrapper adapts tokio's `AsyncRead`/`AsyncWrite` to hyper's I/O traits:

```rust
use hyper_util::rt::TokioIo;
use tokio::net::TcpStream;

// Hyper requires its own I/O traits, not tokio's directly
let stream: TcpStream = TcpListener::bind("...").await?.accept().await?.0;

// Wrap in TokioIo for hyper compatibility
let io = TokioIo::new(stream);

// Now use with http1::Builder or http2::Builder
http1::Builder::new()
    .serve_connection(io, service_fn(handler))
    .await?;
```

### TokioExecutor for HTTP/2

HTTP/2 multiplexes streams and needs to spawn tasks. The `TokioExecutor` adapts tokio's spawning:

```rust
use hyper_util::rt::TokioExecutor;

// HTTP/2 builder requires an executor
let builder = http2::Builder::new(TokioExecutor::new());

// Configure HTTP/2 settings
let builder = http2::Builder::new(TokioExecutor::new())
    .max_concurrent_streams(100)
    .initial_stream_window_size(65535)
    .initial_connection_window_size(1048576);
```

### Body Extraction

```rust
use hyper::{body::Incoming, Request};
use http_body_util::BodyExt;
use serde::de::DeserializeOwned;

pub async fn extract_json<T: DeserializeOwned>(
    req: Request<Incoming>
) -> Result<T, Box<dyn std::error::Error>> {
    // Collect entire body
    let body = req.collect().await?.aggregate();

    // Parse JSON from collected bytes
    let parsed: T = serde_json::from_reader(body.reader())?;
    Ok(parsed)
}

// Usage in handler
async fn handle(req: Request<Incoming>) -> Result<Response<Full<Bytes>>, Infallible> {
    if req.method() == hyper::Method::POST {
        match extract_json::<CreateRequest>(req).await {
            Ok(data) => {
                // Process data
                Ok(Response::new(Full::new(Bytes::from("Created"))))
            }
            Err(e) => {
                Ok(Response::builder()
                    .status(400)
                    .body(Full::new(Bytes::from(format!("Error: {}", e))))
                    .unwrap())
            }
        }
    } else {
        Ok(Response::new(Full::new(Bytes::from("Hello"))))
    }
}
```

### Graceful Shutdown with Hyper

```rust
use tokio::signal;
use tokio::sync::watch;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let (shutdown_tx, mut shutdown_rx) = watch::channel(false);

    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    // Spawn shutdown signal handler
    tokio::spawn(async move {
        signal::ctrl_c().await.expect("Failed to listen for Ctrl+C");
        println!("Shutdown signal received");
        let _ = shutdown_tx.send(true);
    });

    loop {
        tokio::select! {
            result = listener.accept() => {
                let (stream, _) = result?;
                let io = TokioIo::new(stream);
                let mut shutdown = shutdown_rx.clone();

                tokio::spawn(async move {
                    let conn = http1::Builder::new()
                        .serve_connection(io, service_fn(handle));

                    tokio::pin!(conn);

                    loop {
                        tokio::select! {
                            result = conn.as_mut() => {
                                if let Err(e) = result {
                                    eprintln!("Error: {:?}", e);
                                }
                                break;
                            }
                            _ = shutdown.changed() => {
                                // Start graceful shutdown
                                conn.as_mut().graceful_shutdown();
                            }
                        }
                    }
                });
            }
            _ = shutdown_rx.changed() => {
                println!("Stopping listener");
                break;
            }
        }
    }

    Ok(())
}
```

## Bridging Sync and Async

### spawn_blocking for CPU-Bound Work

```rust
use tokio::task;

async fn process_image(data: Vec<u8>) -> Result<Vec<u8>, Error> {
    // Offload CPU-intensive work to blocking thread pool
    task::spawn_blocking(move || {
        // This runs on a dedicated thread, won't block async runtime
        image::load_from_memory(&data)?
            .resize(800, 600, image::imageops::FilterType::Lanczos3)
            .to_bytes()
    })
    .await?
}

// Simple file read bridging
async fn read_file(path: &str) -> std::io::Result<String> {
    let path = path.to_string();
    tokio::task::spawn_blocking(move || {
        std::fs::read_to_string(&path)
    }).await.unwrap()
}
```

### Message-Passing Bridge Pattern

```rust
use tokio::sync::{mpsc, oneshot};
use std::thread;

struct SyncWorker {
    sender: mpsc::Sender<WorkRequest>,
}

struct WorkRequest {
    data: String,
    response: oneshot::Sender<String>,
}

impl SyncWorker {
    fn new() -> Self {
        let (tx, mut rx) = mpsc::channel::<WorkRequest>(100);

        // Dedicated thread for sync operations
        thread::spawn(move || {
            // Use blocking_recv in sync context
            while let Some(request) = rx.blocking_recv() {
                // Perform blocking operation
                let result = expensive_sync_operation(&request.data);
                let _ = request.response.send(result);
            }
        });

        Self { sender: tx }
    }

    async fn process(&self, data: String) -> Result<String, Error> {
        let (response_tx, response_rx) = oneshot::channel();

        self.sender.send(WorkRequest {
            data,
            response: response_tx,
        }).await?;

        Ok(response_rx.await?)
    }
}

fn expensive_sync_operation(data: &str) -> String {
    // Blocking operation that can't be made async
    std::thread::sleep(std::time::Duration::from_millis(100));
    format!("Processed: {}", data)
}
```

### Sync → Async Bridge

```rust
fn sync_code(tx: std::sync::mpsc::Sender<Request>) {
    tx.send(Request::DoSomething).unwrap();
}

async fn async_bridge(mut rx: tokio::sync::mpsc::Receiver<Request>) {
    while let Some(req) = rx.recv().await {
        handle_request(req).await;
    }
}
```

## Timeouts, Retries, and Rate Limiting

### Exponential Backoff with Jitter

```rust
use tokio::time::{sleep, Duration};
use rand::Rng;

async fn retry_with_backoff<T, E, F, Fut>(
    mut operation: F,
    max_retries: u32,
) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<T, E>>,
{
    let mut attempt = 0;

    loop {
        match operation().await {
            Ok(result) => return Ok(result),
            Err(e) if attempt < max_retries => {
                attempt += 1;

                // Exponential backoff: 2^attempt * 100ms
                let base_delay = Duration::from_millis(100 * (1 << attempt));

                // Add jitter: random 0-50% of base delay
                let jitter = rand::thread_rng()
                    .gen_range(0..base_delay.as_millis() / 2) as u64;
                let delay = base_delay + Duration::from_millis(jitter);

                // Cap at 30 seconds
                let delay = delay.min(Duration::from_secs(30));

                sleep(delay).await;
            }
            Err(e) => return Err(e),
        }
    }
}
```

### Token Bucket Rate Limiter

```rust
use tokio::sync::Semaphore;
use tokio::time::{interval, Duration};
use std::sync::Arc;

struct TokenBucket {
    tokens: Arc<Semaphore>,
}

impl TokenBucket {
    fn new(capacity: usize, refill_rate: Duration) -> Self {
        let tokens = Arc::new(Semaphore::new(capacity));
        let tokens_clone = tokens.clone();

        // Background task to refill tokens
        tokio::spawn(async move {
            let mut ticker = interval(refill_rate);
            loop {
                ticker.tick().await;
                // Add token if below capacity
                if tokens_clone.available_permits() < capacity {
                    tokens_clone.add_permits(1);
                }
            }
        });

        Self { tokens }
    }

    async fn acquire(&self) {
        self.tokens.acquire().await.unwrap().forget();
    }

    fn try_acquire(&self) -> bool {
        self.tokens.try_acquire().map(|p| p.forget()).is_ok()
    }
}
```

## Debugging Async Systems

### Deadlock Detection Pattern

```rust
use std::collections::{HashMap, HashSet};
use std::sync::Mutex;

// Track lock dependencies to detect potential deadlocks
struct LockTracker {
    // Thread -> Set of locks it's waiting for
    waiting_for: Mutex<HashMap<std::thread::ThreadId, HashSet<usize>>>,
}

impl LockTracker {
    fn before_lock(&self, lock_id: usize) {
        let thread_id = std::thread::current().id();
        let mut waiting = self.waiting_for.lock().unwrap();
        waiting.entry(thread_id).or_default().insert(lock_id);

        // Check for cycles (simplified)
        if self.has_cycle(&waiting) {
            eprintln!("Potential deadlock detected!");
        }
    }

    fn after_lock(&self, lock_id: usize) {
        let thread_id = std::thread::current().id();
        let mut waiting = self.waiting_for.lock().unwrap();
        if let Some(locks) = waiting.get_mut(&thread_id) {
            locks.remove(&lock_id);
        }
    }

    fn has_cycle(&self, _graph: &HashMap<std::thread::ThreadId, HashSet<usize>>) -> bool {
        // Implement cycle detection (DFS/Tarjan's algorithm)
        false
    }
}
```

### Task Leak Detection

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::time::Duration;

static ACTIVE_TASKS: AtomicUsize = AtomicUsize::new(0);

async fn tracked_task<F, T>(name: &str, f: F) -> T
where
    F: std::future::Future<Output = T>,
{
    ACTIVE_TASKS.fetch_add(1, Ordering::SeqCst);
    let result = f.await;
    ACTIVE_TASKS.fetch_sub(1, Ordering::SeqCst);
    result
}

// Periodic health check
async fn monitor_tasks() {
    let mut interval = tokio::time::interval(Duration::from_secs(60));
    loop {
        interval.tick().await;
        let count = ACTIVE_TASKS.load(Ordering::SeqCst);
        if count > 1000 {
            eprintln!("Warning: {} active tasks (potential leak)", count);
        }
    }
}
```

### Structured Logging with Tracing

```rust
use tracing::{info, instrument, span, Level};

#[instrument(skip(data), fields(data_len = data.len()))]
async fn process_request(request_id: u64, data: &[u8]) -> Result<Response, Error> {
    info!("Processing request");

    let result = async {
        let span = span!(Level::DEBUG, "parse", request_id);
        let _enter = span.enter();
        parse_data(data).await
    }.await?;

    info!(result_size = result.len(), "Request processed");
    Ok(Response::new(result))
}

fn setup_tracing() {
    tracing_subscriber::fmt()
        .with_max_level(Level::DEBUG)
        .with_target(true)
        .with_thread_ids(true)
        .init();
}
```

## Tokio Scheduler Tuning

### Thread Pool Sizing

```rust
use tokio::runtime::Builder;

// CPU-bound workloads: match available cores
let parallelism = std::thread::available_parallelism().map_or(4, |n| n.get());
let cpu_bound_rt = Builder::new_multi_thread()
    .worker_threads(parallelism)
    .build()
    .unwrap();

// I/O-bound workloads: can exceed core count
let io_bound_rt = Builder::new_multi_thread()
    .worker_threads(parallelism * 2)
    .max_blocking_threads(512)
    .build()
    .unwrap();
```

### NUMA-Aware Configuration

```rust
// For NUMA systems, consider pinning threads to cores
// and creating separate runtimes per NUMA node

use tokio::runtime::Builder;

fn create_numa_runtime(node: usize, cores: &[usize]) -> tokio::runtime::Runtime {
    Builder::new_multi_thread()
        .worker_threads(cores.len())
        .thread_name(format!("numa-{}-worker", node))
        .on_thread_start(move || {
            // Pin thread to specific cores (platform-specific)
            // core_affinity::set_for_current(cores[0]);
        })
        .build()
        .unwrap()
}
```

### Task Batching for Reduced Overhead

```rust
use tokio::sync::mpsc;

// Instead of spawning many tiny tasks, batch them
async fn batch_processor(mut rx: mpsc::Receiver<Work>) {
    let mut batch = Vec::with_capacity(100);

    loop {
        // Collect batch
        while batch.len() < 100 {
            match rx.try_recv() {
                Ok(work) => batch.push(work),
                Err(_) => break,
            }
        }

        if batch.is_empty() {
            // Wait for at least one item
            if let Some(work) = rx.recv().await {
                batch.push(work);
            } else {
                break; // Channel closed
            }
        }

        // Process batch
        process_batch(&batch).await;
        batch.clear();

        // Yield to allow other tasks to run
        tokio::task::yield_now().await;
    }
}
```

## Common Patterns

### Worker Pool

```rust
use std::sync::mpsc;
use std::sync::{Arc, Mutex};
use std::thread;

struct ThreadPool {
    workers: Vec<thread::JoinHandle<()>>,
    sender: mpsc::Sender<Box<dyn FnOnce() + Send>>,
}

impl ThreadPool {
    fn new(size: usize) -> Self {
        let (sender, receiver) = mpsc::channel::<Box<dyn FnOnce() + Send>>();
        let receiver = Arc::new(Mutex::new(receiver));

        let workers = (0..size).map(|_| {
            let receiver = Arc::clone(&receiver);
            thread::spawn(move || {
                while let Ok(job) = receiver.lock().unwrap().recv() {
                    job();
                }
            })
        }).collect();

        ThreadPool { workers, sender }
    }

    fn execute<F: FnOnce() + Send + 'static>(&self, f: F) {
        self.sender.send(Box::new(f)).unwrap();
    }
}
```

### Async Mutex

```rust
use tokio::sync::Mutex;
use std::sync::Arc;

// Async-safe mutex (doesn't block thread)
let data = Arc::new(Mutex::new(vec![]));

let data_clone = Arc::clone(&data);
tokio::spawn(async move {
    let mut guard = data_clone.lock().await;
    guard.push(1);
});
```

## Actor Model

### Actor Model Concepts

The Actor Model defines computation as independent, asynchronous entities (actors) that:
- Encapsulate internal state with no direct external access
- Communicate exclusively through immutable messages
- Process messages sequentially from a mailbox queue
- Can create other actors, send messages, or change their own behavior

This eliminates race conditions by ensuring an actor's state is only modified in response to one message at a time.

### Manual Actor with Tokio

```rust
use tokio::sync::{mpsc, oneshot};

enum Command {
    Get { resp: oneshot::Sender<String> },
    Set { value: String },
}

struct StateActor {
    receiver: mpsc::Receiver<Command>,
    state: String,
}

impl StateActor {
    fn new(receiver: mpsc::Receiver<Command>) -> Self {
        StateActor {
            receiver,
            state: String::new(),
        }
    }

    async fn run(mut self) {
        while let Some(cmd) = self.receiver.recv().await {
            match cmd {
                Command::Get { resp } => {
                    let _ = resp.send(self.state.clone());
                }
                Command::Set { value } => {
                    self.state = value;
                }
            }
        }
    }
}

#[derive(Clone)]
struct StateHandle {
    sender: mpsc::Sender<Command>,
}

impl StateHandle {
    fn new() -> (Self, StateActor) {
        let (sender, receiver) = mpsc::channel(32);
        (StateHandle { sender }, StateActor::new(receiver))
    }

    async fn get(&self) -> Option<String> {
        let (resp_tx, resp_rx) = oneshot::channel();
        self.sender.send(Command::Get { resp: resp_tx }).await.ok()?;
        resp_rx.await.ok()
    }

    async fn set(&self, value: String) {
        let _ = self.sender.send(Command::Set { value }).await;
    }
}

#[tokio::main]
async fn main() {
    let (handle, actor) = StateHandle::new();

    // Spawn actor task
    tokio::spawn(actor.run());

    // Use handle
    handle.set("Hello, World!".into()).await;
    if let Some(value) = handle.get().await {
        println!("State: {}", value);
    }
}
```

### Actix Framework

#### Defining Actors

```rust
use actix::prelude::*;

// Define a message with its return type
#[derive(Message)]
#[rtype(result = "String")]  // Handler returns String
struct Greet {
    name: String,
}

// Define the actor with internal state
struct GreeterActor {
    greeting_count: usize,
}

// Implement the Actor trait
impl Actor for GreeterActor {
    type Context = Context<Self>;

    // Called when actor starts
    fn started(&mut self, _ctx: &mut Self::Context) {
        println!("GreeterActor started");
    }

    // Called when actor stops
    fn stopped(&mut self, _ctx: &mut Self::Context) {
        println!("GreeterActor stopped after {} greetings", self.greeting_count);
    }
}

// Implement message handler
impl Handler<Greet> for GreeterActor {
    type Result = String;  // Must match #[rtype(result = "...")]

    fn handle(&mut self, msg: Greet, _ctx: &mut Context<Self>) -> Self::Result {
        self.greeting_count += 1;
        format!("Hello, {}! (greeting #{})", msg.name, self.greeting_count)
    }
}
```

#### Starting Actors and Sending Messages

```rust
use actix::prelude::*;

#[actix::main]
async fn main() {
    // Start actor and get its address
    let addr: Addr<GreeterActor> = GreeterActor { greeting_count: 0 }.start();

    // send() - Request/response, returns Future
    let response = addr.send(Greet { name: "Alice".into() }).await;
    match response {
        Ok(greeting) => println!("{}", greeting),
        Err(e) => eprintln!("Mailbox error: {}", e),
    }

    // do_send() - Fire-and-forget, no response
    addr.do_send(Greet { name: "Bob".into() });

    // try_send() - Non-blocking, returns error if mailbox full
    if addr.try_send(Greet { name: "Charlie".into() }).is_err() {
        eprintln!("Mailbox full");
    }
}
```

#### Message Types

```rust
use actix::prelude::*;

// Message with no return value
#[derive(Message)]
#[rtype(result = "()")]
struct Increment;

// Message returning a value
#[derive(Message)]
#[rtype(result = "usize")]
struct GetCount;

// Message returning Result
#[derive(Message)]
#[rtype(result = "Result<String, std::io::Error>")]
struct ReadFile {
    path: String,
}

// Counter actor handling multiple message types
struct Counter {
    count: usize,
}

impl Actor for Counter {
    type Context = Context<Self>;
}

impl Handler<Increment> for Counter {
    type Result = ();

    fn handle(&mut self, _msg: Increment, _ctx: &mut Context<Self>) {
        self.count += 1;
    }
}

impl Handler<GetCount> for Counter {
    type Result = usize;

    fn handle(&mut self, _msg: GetCount, _ctx: &mut Context<Self>) -> Self::Result {
        self.count
    }
}
```

#### Self-Scheduling and Delayed Messages

```rust
use actix::prelude::*;
use std::time::Duration;

#[derive(Message)]
#[rtype(result = "()")]
struct Tick;

struct TickActor {
    tick_count: usize,
}

impl Actor for TickActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        // Schedule periodic tick every second
        ctx.run_interval(Duration::from_secs(1), |actor, _ctx| {
            actor.tick_count += 1;
            println!("Tick #{}", actor.tick_count);
        });

        // Schedule one-time delayed action
        ctx.run_later(Duration::from_secs(5), |actor, ctx| {
            println!("Stopping after {} ticks", actor.tick_count);
            ctx.stop();
        });
    }
}

impl Handler<Tick> for TickActor {
    type Result = ();

    fn handle(&mut self, _msg: Tick, _ctx: &mut Context<Self>) {
        self.tick_count += 1;
    }
}
```

#### Async Operations in Handlers

```rust
use actix::prelude::*;

#[derive(Message)]
#[rtype(result = "Result<String, reqwest::Error>")]
struct FetchUrl(String);

struct WebFetcher {
    client: reqwest::Client,
}

impl Actor for WebFetcher {
    type Context = Context<Self>;
}

impl Handler<FetchUrl> for WebFetcher {
    type Result = ResponseFuture<Result<String, reqwest::Error>>;

    fn handle(&mut self, msg: FetchUrl, _ctx: &mut Context<Self>) -> Self::Result {
        let client = self.client.clone();
        let url = msg.0;

        // Return a future that will be awaited by the runtime
        Box::pin(async move {
            let response = client.get(&url).send().await?;
            response.text().await
        })
    }
}
```

#### Actor Supervision

```rust
use actix::prelude::*;

struct Supervisor;
struct Worker {
    id: usize,
}

#[derive(Message)]
#[rtype(result = "()")]
struct DoWork;

impl Actor for Supervisor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        // Spawn child actors
        for id in 0..3 {
            let worker = Worker { id }.start();
            // Optionally link workers to supervisor
        }
    }
}

impl Actor for Worker {
    type Context = Context<Self>;

    fn started(&mut self, _ctx: &mut Self::Context) {
        println!("Worker {} started", self.id);
    }

    fn stopped(&mut self, _ctx: &mut Self::Context) {
        println!("Worker {} stopped", self.id);
    }
}

impl Handler<DoWork> for Worker {
    type Result = ();

    fn handle(&mut self, _msg: DoWork, _ctx: &mut Context<Self>) {
        println!("Worker {} processing", self.id);
    }
}
```

### Custom Actors with async-std

Build actors from scratch using channels for maximum flexibility:

```rust
use async_std::channel::{bounded, Receiver, Sender};
use async_std::task;

// Message enum with reply channels
enum CounterMessage {
    Increment,
    Decrement,
    GetCount(Sender<usize>),  // Include reply sender
    Stop,
}

struct Counter {
    count: usize,
    receiver: Receiver<CounterMessage>,
}

impl Counter {
    fn new(receiver: Receiver<CounterMessage>) -> Self {
        Counter { count: 0, receiver }
    }

    // Actor's main loop
    async fn run(mut self) {
        println!("Counter actor started");

        while let Ok(message) = self.receiver.recv().await {
            match message {
                CounterMessage::Increment => {
                    self.count += 1;
                }
                CounterMessage::Decrement => {
                    if self.count > 0 {
                        self.count -= 1;
                    }
                }
                CounterMessage::GetCount(reply_tx) => {
                    let _ = reply_tx.send(self.count).await;
                }
                CounterMessage::Stop => {
                    println!("Counter actor stopping");
                    break;
                }
            }
        }

        println!("Counter actor stopped with count: {}", self.count);
    }
}

// Actor handle for sending messages
#[derive(Clone)]
struct CounterHandle {
    sender: Sender<CounterMessage>,
}

impl CounterHandle {
    fn new(capacity: usize) -> (Self, Counter) {
        let (sender, receiver) = bounded(capacity);
        let handle = CounterHandle { sender };
        let actor = Counter::new(receiver);
        (handle, actor)
    }

    async fn increment(&self) {
        let _ = self.sender.send(CounterMessage::Increment).await;
    }

    async fn decrement(&self) {
        let _ = self.sender.send(CounterMessage::Decrement).await;
    }

    async fn get_count(&self) -> usize {
        let (reply_tx, reply_rx) = bounded(1);
        let _ = self.sender.send(CounterMessage::GetCount(reply_tx)).await;
        reply_rx.recv().await.unwrap_or(0)
    }

    async fn stop(&self) {
        let _ = self.sender.send(CounterMessage::Stop).await;
    }
}

fn main() {
    task::block_on(async {
        let (handle, counter) = CounterHandle::new(32);

        // Spawn actor
        let actor_task = task::spawn(counter.run());

        // Use the handle
        handle.increment().await;
        handle.increment().await;
        handle.increment().await;
        handle.decrement().await;

        let count = handle.get_count().await;
        println!("Current count: {}", count);

        handle.stop().await;
        actor_task.await;
    });
}
```

### Actor Patterns

#### Request-Response Pattern

```rust
use actix::prelude::*;

#[derive(Message)]
#[rtype(result = "Result<UserData, UserError>")]
struct GetUser { id: u64 }

struct UserService {
    db: DatabasePool,
}

impl Handler<GetUser> for UserService {
    type Result = ResponseFuture<Result<UserData, UserError>>;

    fn handle(&mut self, msg: GetUser, _ctx: &mut Context<Self>) -> Self::Result {
        let db = self.db.clone();
        Box::pin(async move {
            db.find_user(msg.id).await
        })
    }
}

// Usage
async fn get_user(service: Addr<UserService>, id: u64) -> Result<UserData, Error> {
    service.send(GetUser { id }).await?
        .map_err(|e| Error::User(e))
}
```

#### Pub/Sub Pattern

```rust
use actix::prelude::*;
use std::collections::HashSet;

#[derive(Message, Clone)]
#[rtype(result = "()")]
struct Event {
    topic: String,
    payload: String,
}

#[derive(Message)]
#[rtype(result = "()")]
struct Subscribe {
    topic: String,
    subscriber: Recipient<Event>,
}

struct EventBus {
    subscribers: std::collections::HashMap<String, HashSet<Recipient<Event>>>,
}

impl Actor for EventBus {
    type Context = Context<Self>;
}

impl Handler<Subscribe> for EventBus {
    type Result = ();

    fn handle(&mut self, msg: Subscribe, _ctx: &mut Context<Self>) {
        self.subscribers
            .entry(msg.topic)
            .or_default()
            .insert(msg.subscriber);
    }
}

impl Handler<Event> for EventBus {
    type Result = ();

    fn handle(&mut self, msg: Event, _ctx: &mut Context<Self>) {
        if let Some(subs) = self.subscribers.get(&msg.topic) {
            for sub in subs {
                sub.do_send(msg.clone());
            }
        }
    }
}
```

#### Actor Pool Pattern

```rust
use actix::prelude::*;

struct WorkerPool {
    workers: Vec<Addr<Worker>>,
    next: usize,
}

impl WorkerPool {
    fn new(size: usize) -> Self {
        let workers = (0..size)
            .map(|id| Worker { id }.start())
            .collect();
        WorkerPool { workers, next: 0 }
    }

    fn next_worker(&mut self) -> &Addr<Worker> {
        let worker = &self.workers[self.next];
        self.next = (self.next + 1) % self.workers.len();
        worker
    }
}

impl Actor for WorkerPool {
    type Context = Context<Self>;
}

impl Handler<Work> for WorkerPool {
    type Result = ();

    fn handle(&mut self, msg: Work, _ctx: &mut Context<Self>) {
        // Round-robin distribution
        self.next_worker().do_send(msg);
    }
}
```

### When to Use Actors

**Use actors when:**
- You need isolated, concurrent state management
- Building distributed systems with message-passing
- Implementing complex workflows with supervision
- Managing many independent concurrent entities (game entities, user sessions)

**Consider alternatives when:**
- Simple shared state (use `Arc<Mutex<T>>`)
- Pure data parallelism (use Rayon)
- Simple async pipelines (use channels directly)
- Performance-critical hot paths (actor overhead may matter)

### Actor vs Alternative Comparison

| Pattern | Use When |
|---------|----------|
| Actor (mpsc + task) | Stateful component, message-driven, needs encapsulation |
| `Arc<Mutex<T>>` | Simple shared state, infrequent contention |
| `DashMap` | High-concurrency key-value access |
| Channels only | Pipeline/dataflow, no persistent state |
| `RwLock` | Read-heavy workloads with occasional writes |

### Comparison: actix vs Custom Actors

| Feature | actix | Custom (channels) |
|---------|-------|-------------------|
| Boilerplate | More macros/traits | Less, more explicit |
| Message types | Strongly typed | Enum-based |
| Supervision | Built-in | Manual |
| Lifecycle | Automatic | Manual |
| Flexibility | Framework conventions | Full control |
| Dependencies | Heavier | Minimal |

## Sans-I/O Pattern

The sans-I/O pattern separates protocol logic from all I/O operations. The protocol implementation has no async, no threads, no network calls — just pure state machine transitions driven by an external runtime loop. Used by production crates like `str0m` (WebRTC) and `atm0s-media-server`.

**Why sans-I/O?**
- **Testable** — test protocol logic without network, async runtime, or mocks
- **Portable** — same protocol code works with tokio, async-std, or bare-metal
- **Debuggable** — deterministic: given the same inputs and timestamps, output is identical
- **Composable** — multiple protocol instances share one runtime loop

**The pattern: trait with `on_tick`, `on_input`, `pop_output`**

```rust
use std::time::Instant;

/// Events coming into the protocol from the outside world
pub enum TransportInput {
    Net(Vec<u8>),                    // Received network packet
    Timer,                            // Periodic tick
    UserCommand(Command),             // Application-level command
}

/// Events the protocol wants to send to the outside world
pub enum TransportOutput {
    Net(Vec<u8>),                    // Send this packet
    Event(ProtocolEvent),             // Notify application
    Timeout(Duration),                // "Wake me up in this long"
}

/// Sans-I/O protocol — no async, no threads, no network
pub struct Protocol {
    state: State,
    output_queue: VecDeque<TransportOutput>,
}

impl Protocol {
    /// Advance internal timers — call periodically from the runtime loop
    pub fn on_tick(&mut self, now: Instant) {
        if self.state.needs_keepalive(now) {
            self.output_queue.push_back(TransportOutput::Net(keepalive_packet()));
        }
    }

    /// Feed external events into the protocol
    pub fn on_input(&mut self, now: Instant, input: TransportInput) {
        match input {
            TransportInput::Net(data) => {
                if let Some(event) = self.state.process_packet(&data, now) {
                    self.output_queue.push_back(TransportOutput::Event(event));
                }
            }
            TransportInput::UserCommand(cmd) => {
                let packets = self.state.handle_command(cmd, now);
                for pkt in packets {
                    self.output_queue.push_back(TransportOutput::Net(pkt));
                }
            }
            TransportInput::Timer => self.on_tick(now),
        }
    }

    /// Drain output events — runtime loop sends packets, delivers events
    pub fn pop_output(&mut self) -> Option<TransportOutput> {
        self.output_queue.pop_front()
    }

    /// Graceful shutdown
    pub fn on_shutdown(&mut self, now: Instant) {
        self.output_queue.push_back(TransportOutput::Net(close_packet()));
    }
}
```

**Runtime loop (the async part — separate from protocol):**

```rust
async fn run_protocol(udp: UdpSocket, mut proto: Protocol) {
    let mut buf = vec![0u8; 2048];
    let mut tick = tokio::time::interval(Duration::from_millis(50));

    loop {
        tokio::select! {
            _ = tick.tick() => {
                proto.on_tick(Instant::now());
            }
            result = udp.recv_from(&mut buf) => {
                if let Ok((len, _addr)) = result {
                    proto.on_input(Instant::now(), TransportInput::Net(buf[..len].to_vec()));
                }
            }
        }

        // Drain all outputs
        while let Some(output) = proto.pop_output() {
            match output {
                TransportOutput::Net(data) => { udp.send(&data).await.ok(); }
                TransportOutput::Event(event) => { handle_event(event).await; }
                TransportOutput::Timeout(_dur) => { /* adjust tick interval */ }
            }
        }
    }
}
```

**Testing sans-I/O protocols — no async, no network:**

```rust
#[test]
fn protocol_sends_keepalive_after_timeout() {
    let mut proto = Protocol::new();
    let t0 = Instant::now();

    // Simulate 30 seconds passing with no input
    proto.on_tick(t0 + Duration::from_secs(30));

    // Protocol should emit a keepalive packet — no network needed to test
    let output = proto.pop_output().unwrap();
    assert!(matches!(output, TransportOutput::Net(_)));
}

#[test]
fn protocol_handles_incoming_packet() {
    let mut proto = Protocol::new();
    proto.on_input(Instant::now(), TransportInput::Net(valid_packet()));

    let output = proto.pop_output().unwrap();
    assert!(matches!(output, TransportOutput::Event(ProtocolEvent::Connected)));
}
```

**Production examples:** `str0m` (WebRTC SFU), `quinn` (QUIC), `rustls` (TLS). The `str0m::Rtc` instance has no network calls — all I/O is driven by the caller.

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: async/await basics, Send/Sync, concurrency primitives overview
- **[architecture.md](architecture.md)** — Production patterns, graceful shutdown integration, tracing setup
- **[services.md](services.md)** — Microservices communication, distributed tracing, resilience patterns
- **[web-apis.md](web-apis.md)** — Tower middleware with Axum, async handlers, WebSocket, WebRTC signaling
- **[type-system.md](type-system.md)** — Pin/Unpin internals, async trait patterns
- **[testing.md](testing.md)** — `#[tokio::test]`, async mock patterns, time control
