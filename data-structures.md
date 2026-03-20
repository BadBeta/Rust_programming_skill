# Data Structures, Algorithms, and Performance

Standard library collections, sorting, graph algorithms, algorithm design paradigms, benchmarking with criterion, profiling tools, and optimization strategies.

## Rules for Data Structures & Performance (LLM)

1. **ALWAYS use `with_capacity()` when the size is known or estimable** — `Vec::new()` starts at 0 capacity and reallocates as it grows; `Vec::with_capacity(n)` avoids O(log n) reallocations
2. **NEVER benchmark in debug mode** — `cargo bench` uses release by default, but `cargo test` uses debug; always run benchmarks separately with `--release` if using custom harness
3. **ALWAYS use `black_box()` in benchmarks** — prevents the compiler from optimizing away the computation you're measuring
4. **PREFER `sort_unstable()` over `sort()`** — unstable sort is faster and allocates no extra memory; use `sort()` only when equal elements must preserve their original order
5. **ALWAYS profile before optimizing** — use flamegraph/perf to find actual bottlenecks; never guess where time is spent
6. **PREFER `entry()` API over `get()` then `insert()`** — `entry()` does one hash lookup; separate get+insert does two
7. **ALWAYS implement `FromIterator` and `Extend` for custom collections** — enables `.collect()` and `.extend()`, integrating with the iterator ecosystem
8. **PREFER iterators over indexed loops** — iterators eliminate bounds checks and enable SIMD auto-vectorization

### Section Index

| Section | Topics |
|---------|--------|
| [Complexity Analysis](#complexity-analysis) | Big O notation, amortized analysis, complexity comparison table |
| [Standard Library Collections](#standard-library-collections) | Vec, HashMap, BTreeMap, HashSet, VecDeque, BinaryHeap, LinkedList |
| [Sorting Algorithms](#sorting-algorithms) | sort/sort_unstable, sort_by_key, custom comparators, partial_sort |
| [Graph Algorithms](#graph-algorithms) | Adjacency list, BFS, DFS, Dijkstra, topological sort |
| [Algorithm Design Paradigms](#algorithm-design-paradigms) | Divide and conquer, dynamic programming, greedy, backtracking |
| [Rust Libraries](#rust-libraries-for-data-structures-and-algorithms) | SmallVec, arrayvec, indexmap, dashmap, petgraph, roaring |
| [Benchmarking](#benchmarking-with-criterion) | criterion setup, parameterized benchmarks, comparison groups |
| [Identifying Bottlenecks](#identifying-performance-bottlenecks) | I/O, locking, allocation, cache misses, algorithmic |
| [Profiling Tools](#profiling-tools) | flamegraph, perf, DHAT, valgrind, memory profiling |
| [Best Practices](#best-practices) | Collection selection guide, optimization patterns, zero-copy |

## Complexity Analysis

### Big O Notation

```rust
// O(1) — Constant time
fn get_first(v: &[i32]) -> Option<&i32> {
    v.first()
}

// O(log n) — Logarithmic time
fn binary_search(arr: &[i32], target: i32) -> Option<usize> {
    arr.binary_search(&target).ok()
}

// O(n) — Linear time
fn linear_search(arr: &[i32], target: i32) -> Option<usize> {
    arr.iter().position(|&x| x == target)
}

// O(n log n) — Linearithmic time (merge sort, standard library sort)
fn sort_vec(v: &mut Vec<i32>) {
    v.sort_unstable();
}

// O(n²) — Quadratic time
fn bubble_sort<T: Ord>(arr: &mut [T]) {
    for i in 0..arr.len() {
        for j in 0..arr.len().saturating_sub(1 + i) {
            if arr[j] > arr[j + 1] {
                arr.swap(j, j + 1);
            }
        }
    }
}
```

### Complexity Classes

| Notation | Name | Example |
|----------|------|---------|
| O(1) | Constant | Array index access, HashMap lookup |
| O(log n) | Logarithmic | Binary search, BTreeMap operations |
| O(n) | Linear | Linear search, Vec iteration |
| O(n log n) | Linearithmic | Merge sort, quick sort (avg), `sort()` |
| O(n²) | Quadratic | Bubble sort, nested loops |
| O(2ⁿ) | Exponential | Recursive Fibonacci |
| O(n!) | Factorial | Permutation generation |

### Amortized Analysis

```rust
// Vec::push is O(1) amortized despite occasional O(n) resizing.
// Aggregate method: n pushes cost at most 2n operations = O(1) per push.

// Dynamic array with amortized O(1) append
struct DynamicArray<T> {
    data: Vec<T>,
}

impl<T> DynamicArray<T> {
    fn new() -> Self {
        DynamicArray { data: Vec::new() }
    }

    // Amortized O(1): occasional resize is spread across all operations.
    // Vec doubles capacity on resize — each element is copied at most
    // O(log n) times across all resizes, but total work across n pushes
    // is bounded by 2n.
    fn push(&mut self, value: T) {
        self.data.push(value);
    }
}

// Accounting method example: assign 3 units per push
// - 1 unit for insertion
// - 2 units saved for future resize (copy self + copy one old element)
// Each element "pays" for its own future copy during resize.
```

## Standard Library Collections

### Vec and VecDeque

```rust
use std::collections::VecDeque;

// Vec — dynamic array, O(1) amortized push, O(1) index access
let mut v: Vec<i32> = Vec::with_capacity(100); // Pre-allocate
v.push(1);              // O(1) amortized
v.pop();                // O(1)
v.insert(0, 42);        // O(n) — shifts elements right
v.remove(0);            // O(n) — shifts elements left
v.swap_remove(0);       // O(1) — swaps with last, doesn't preserve order
v.get(0);               // O(1), returns Option<&T>
v.sort_unstable();      // O(n log n)
v.binary_search(&42);   // O(log n) on sorted Vec
v.dedup();              // O(n), removes consecutive duplicates (sort first)
v.retain(|x| *x > 0);  // O(n), keep elements matching predicate
v.truncate(5);          // O(n), keep first 5 elements
v.extend_from_slice(&[1, 2, 3]); // Append slice

// Efficient batch removal
v.retain(|x| *x % 2 == 0);  // Remove all odd numbers

// Draining (removes and returns elements)
let removed: Vec<i32> = v.drain(1..3).collect();

// Split at position
let (left, right) = v.split_at(2);

// VecDeque — ring buffer, O(1) push/pop at both ends
let mut dq: VecDeque<i32> = VecDeque::new();
dq.push_back(1);        // O(1)
dq.push_front(2);       // O(1)
dq.pop_back();           // O(1)
dq.pop_front();          // O(1)
dq.make_contiguous();    // Ensures internal buffer is contiguous

// Convert between Vec and VecDeque
let v: Vec<i32> = Vec::from(dq);
let dq: VecDeque<i32> = VecDeque::from(v);
```

### Stack and Queue

```rust
use std::collections::VecDeque;

// Stack using Vec (LIFO — Last In, First Out)
struct Stack<T> {
    items: Vec<T>,
}

impl<T> Stack<T> {
    fn new() -> Self {
        Stack { items: Vec::new() }
    }

    fn push(&mut self, item: T) {
        self.items.push(item);
    }

    fn pop(&mut self) -> Option<T> {
        self.items.pop()
    }

    fn peek(&self) -> Option<&T> {
        self.items.last()
    }

    fn is_empty(&self) -> bool {
        self.items.is_empty()
    }

    fn len(&self) -> usize {
        self.items.len()
    }
}

// Queue using VecDeque (FIFO — First In, First Out)
struct Queue<T> {
    items: VecDeque<T>,
}

impl<T> Queue<T> {
    fn new() -> Self {
        Queue { items: VecDeque::new() }
    }

    fn enqueue(&mut self, item: T) {
        self.items.push_back(item);
    }

    fn dequeue(&mut self) -> Option<T> {
        self.items.pop_front()
    }

    fn peek(&self) -> Option<&T> {
        self.items.front()
    }

    fn is_empty(&self) -> bool {
        self.items.is_empty()
    }

    fn len(&self) -> usize {
        self.items.len()
    }
}
```

### HashMap and HashSet

```rust
use std::collections::{HashMap, HashSet};

// HashMap — O(1) average insert/lookup/remove
let mut map: HashMap<String, i32> = HashMap::new();
map.insert("key".into(), 42);
map.get("key");                         // O(1), returns Option<&V>
map.get_mut("key");                     // O(1), returns Option<&mut V>
map.contains_key("key");               // O(1)
map.remove("key");                      // O(1)

// Entry API — conditional insert/update without double lookup
map.entry("key".into()).or_insert(0);   // Insert if missing
map.entry("key".into())
    .and_modify(|v| *v += 1)
    .or_insert(0);                      // Increment or initialize

// Insert and get the old value
let old = map.insert("key".into(), 99); // Returns Option<V>

// Iterate
for (key, value) in &map {
    println!("{key}: {value}");
}

// Collect into HashMap
let map: HashMap<&str, i32> = vec![("a", 1), ("b", 2)].into_iter().collect();

// HashSet — O(1) average insert/contains/remove
let mut set: HashSet<i32> = HashSet::new();
set.insert(1);
set.contains(&1);          // O(1)
set.remove(&1);            // O(1)

// Set operations
let a: HashSet<_> = [1, 2, 3].into_iter().collect();
let b: HashSet<_> = [2, 3, 4].into_iter().collect();
let union: HashSet<_> = a.union(&b).cloned().collect();
let intersection: HashSet<_> = a.intersection(&b).cloned().collect();
let difference: HashSet<_> = a.difference(&b).cloned().collect();
let symmetric_diff: HashSet<_> = a.symmetric_difference(&b).cloned().collect();

// Check subset/superset
let is_subset = a.is_subset(&b);
let is_superset = a.is_superset(&b);
let is_disjoint = a.is_disjoint(&b);
```

### BTreeMap and BTreeSet

```rust
use std::collections::{BTreeMap, BTreeSet};

// BTreeMap — O(log n) operations, keys are sorted
let mut map: BTreeMap<i32, &str> = BTreeMap::new();
map.insert(3, "c");
map.insert(1, "a");
map.insert(2, "b");

// Iterate in sorted key order
for (k, v) in &map {
    println!("{k}: {v}");  // 1: a, 2: b, 3: c
}

// Range queries — the killer feature vs HashMap
let range: Vec<_> = map.range(1..3).collect();         // [(1,"a"), (2,"b")]
let from_2: Vec<_> = map.range(2..).collect();         // [(2,"b"), (3,"c")]
let up_to_2: Vec<_> = map.range(..=2).collect();       // [(1,"a"), (2,"b")]

// First and last
let first = map.first_key_value();  // Some((&1, &"a"))
let last = map.last_key_value();    // Some((&3, &"c"))

// Split at key
let mut right = map.split_off(&2); // map has keys < 2, right has keys >= 2

// BTreeSet — sorted set with range queries
let mut set: BTreeSet<i32> = BTreeSet::new();
set.insert(3);
set.insert(1);
set.insert(5);
let first = set.first();   // Some(&1)
let last = set.last();     // Some(&5)
let range: Vec<_> = set.range(1..=3).collect(); // [&1, &3]
```

### BinaryHeap (Priority Queue)

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

// Max-heap by default — peek/pop always return the largest
let mut heap = BinaryHeap::new();
heap.push(3);
heap.push(1);
heap.push(4);
heap.push(1);

assert_eq!(heap.peek(), Some(&4));  // O(1)
assert_eq!(heap.pop(), Some(4));    // O(log n)
assert_eq!(heap.pop(), Some(3));

// Min-heap using Reverse wrapper
let mut min_heap = BinaryHeap::new();
min_heap.push(Reverse(3));
min_heap.push(Reverse(1));
min_heap.push(Reverse(4));
assert_eq!(min_heap.pop(), Some(Reverse(1))); // Smallest first

// Custom priority with Ord implementation
#[derive(Eq, PartialEq)]
struct Task {
    priority: u32,
    name: String,
}

impl Ord for Task {
    fn cmp(&self, other: &Self) -> std::cmp::Ordering {
        self.priority.cmp(&other.priority)
    }
}

impl PartialOrd for Task {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        Some(self.cmp(other))
    }
}

// Now BinaryHeap<Task> pops highest-priority tasks first
let mut task_queue = BinaryHeap::new();
task_queue.push(Task { priority: 1, name: "low".into() });
task_queue.push(Task { priority: 10, name: "high".into() });
assert_eq!(task_queue.pop().unwrap().name, "high");

// Convert heap to sorted vec
let sorted: Vec<_> = heap.into_sorted_vec();
```

### Disjoint Set (Union-Find)

```rust
// Used for connected components, Kruskal's MST, cycle detection
struct DisjointSet {
    parent: Vec<usize>,
    rank: Vec<usize>,
}

impl DisjointSet {
    fn new(n: usize) -> Self {
        DisjointSet {
            parent: (0..n).collect(),
            rank: vec![0; n],
        }
    }

    // Find with path compression — O(α(n)) amortized (nearly constant)
    fn find(&mut self, x: usize) -> usize {
        if self.parent[x] != x {
            self.parent[x] = self.find(self.parent[x]); // Path compression
        }
        self.parent[x]
    }

    // Union by rank — O(α(n)) amortized
    fn union(&mut self, x: usize, y: usize) -> bool {
        let px = self.find(x);
        let py = self.find(y);

        if px == py {
            return false; // Already in same set
        }

        // Attach smaller tree under root of larger tree
        match self.rank[px].cmp(&self.rank[py]) {
            std::cmp::Ordering::Less => self.parent[px] = py,
            std::cmp::Ordering::Greater => self.parent[py] = px,
            std::cmp::Ordering::Equal => {
                self.parent[py] = px;
                self.rank[px] += 1;
            }
        }
        true
    }

    fn connected(&mut self, x: usize, y: usize) -> bool {
        self.find(x) == self.find(y)
    }
}

// Usage: find connected components
let mut ds = DisjointSet::new(5);
ds.union(0, 1);
ds.union(2, 3);
ds.union(1, 3);
assert!(ds.connected(0, 2));  // true — 0-1-3-2 are all connected
assert!(!ds.connected(0, 4)); // false — 4 is isolated
```

### When to Use Which Collection

| Collection | Best For | Avoid When |
|-----------|----------|------------|
| `Vec` | Default choice, sequential, stack | Need fast front insertion |
| `VecDeque` | Queue (FIFO), push/pop both ends | Need random access by index |
| `HashMap` | Key-value, no ordering needed | Need sorted iteration |
| `BTreeMap` | Sorted keys, range queries | Only need key lookup (HashMap faster) |
| `HashSet` | Unique values, membership test | Need ordering |
| `BTreeSet` | Unique sorted values, min/max, ranges | Only need membership test |
| `BinaryHeap` | Priority queue, always-access-largest | Need arbitrary element removal |
| `LinkedList` | Almost never — use VecDeque instead | Basically always avoid |

### Implementing Standard Collection Traits

Custom collections should implement these traits to integrate with the Rust ecosystem:

```rust
use std::iter::FromIterator;

/// A sorted, deduplicated collection (example custom type)
pub struct SortedVec<T: Ord> {
    inner: Vec<T>,
}

impl<T: Ord> SortedVec<T> {
    pub fn new() -> Self {
        Self { inner: Vec::new() }
    }

    pub fn with_capacity(cap: usize) -> Self {
        Self { inner: Vec::with_capacity(cap) }
    }

    pub fn insert(&mut self, value: T) {
        match self.inner.binary_search(&value) {
            Ok(_) => {}  // Already present — deduplicate
            Err(pos) => self.inner.insert(pos, value),
        }
    }

    pub fn contains(&self, value: &T) -> bool {
        self.inner.binary_search(value).is_ok()
    }

    pub fn len(&self) -> usize { self.inner.len() }
    pub fn is_empty(&self) -> bool { self.inner.is_empty() }
    pub fn iter(&self) -> std::slice::Iter<'_, T> { self.inner.iter() }
}

// FromIterator — enables .collect::<SortedVec<_>>()
impl<T: Ord> FromIterator<T> for SortedVec<T> {
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self {
        let mut sv = SortedVec::new();
        sv.extend(iter);
        sv
    }
}

// Extend — enables sorted_vec.extend(other_iter)
impl<T: Ord> Extend<T> for SortedVec<T> {
    fn extend<I: IntoIterator<Item = T>>(&mut self, iter: I) {
        for item in iter {
            self.insert(item);
        }
    }
}

// IntoIterator for owned values
impl<T: Ord> IntoIterator for SortedVec<T> {
    type Item = T;
    type IntoIter = std::vec::IntoIter<T>;
    fn into_iter(self) -> Self::IntoIter { self.inner.into_iter() }
}

// IntoIterator for references
impl<'a, T: Ord> IntoIterator for &'a SortedVec<T> {
    type Item = &'a T;
    type IntoIter = std::slice::Iter<'a, T>;
    fn into_iter(self) -> Self::IntoIter { self.inner.iter() }
}

// Default — enables SortedVec::default()
impl<T: Ord> Default for SortedVec<T> {
    fn default() -> Self { Self::new() }
}

// Usage — integrates seamlessly with iterators
let sv: SortedVec<i32> = vec![3, 1, 4, 1, 5].into_iter().collect();
assert_eq!(sv.iter().collect::<Vec<_>>(), &[1, 3, 4, 5]);  // Sorted, deduped
```

### Custom Iterator Implementation

```rust
/// Iterator that yields pairs of adjacent elements
pub struct Windows2<I: Iterator> {
    iter: std::iter::Peekable<I>,
}

impl<I: Iterator> Windows2<I> {
    pub fn new(iter: I) -> Self {
        Self { iter: iter.peekable() }
    }
}

impl<I> Iterator for Windows2<I>
where
    I: Iterator,
    I::Item: Clone,
{
    type Item = (I::Item, I::Item);

    fn next(&mut self) -> Option<Self::Item> {
        let current = self.iter.next()?;
        let next = self.iter.peek()?.clone();
        Some((current, next))
    }

    // size_hint enables allocation optimization in .collect()
    fn size_hint(&self) -> (usize, Option<usize>) {
        let (lower, upper) = self.iter.size_hint();
        let lower = lower.saturating_sub(1);
        let upper = upper.map(|u| u.saturating_sub(1));
        (lower, upper)
    }
}

// Extension trait — adds .windows2() to all iterators
pub trait Windows2Ext: Iterator + Sized {
    fn windows2(self) -> Windows2<Self> {
        Windows2::new(self)
    }
}

impl<I: Iterator> Windows2Ext for I {}

// Usage
let pairs: Vec<_> = [1, 2, 3, 4].iter().windows2().collect();
assert_eq!(pairs, [(&1, &2), (&2, &3), (&3, &4)]);
```

### Platform-Conditional Optimization (`cfg`)

```rust
/// Fast byte search — SIMD on supported platforms, scalar fallback otherwise
pub fn find_byte(haystack: &[u8], needle: u8) -> Option<usize> {
    #[cfg(target_arch = "x86_64")]
    {
        if is_x86_feature_detected!("avx2") {
            return unsafe { find_byte_avx2(haystack, needle) };
        }
    }

    // Scalar fallback — works everywhere
    haystack.iter().position(|&b| b == needle)
}

// In practice, prefer the `memchr` crate which handles all of this:
// memchr::memchr(needle, haystack)
```

### Type-Level Markers (Zero-Cost Abstraction)

Uninhabited enums as compile-time type parameters (pattern from byteorder):

```rust
/// Endianness marker — can never be instantiated, zero runtime cost
pub enum BigEndian {}
pub enum LittleEndian {}

pub trait ByteOrder: private::Sealed {
    fn read_u16(buf: &[u8]) -> u16;
    fn write_u16(buf: &mut [u8], val: u16);
}

impl ByteOrder for BigEndian {
    fn read_u16(buf: &[u8]) -> u16 {
        u16::from_be_bytes([buf[0], buf[1]])
    }
    fn write_u16(buf: &mut [u8], val: u16) {
        buf[..2].copy_from_slice(&val.to_be_bytes());
    }
}

impl ByteOrder for LittleEndian {
    fn read_u16(buf: &[u8]) -> u16 {
        u16::from_le_bytes([buf[0], buf[1]])
    }
    fn write_u16(buf: &mut [u8], val: u16) {
        buf[..2].copy_from_slice(&val.to_le_bytes());
    }
}

// Sealed trait prevents external implementations
mod private {
    pub trait Sealed {}
    impl Sealed for super::BigEndian {}
    impl Sealed for super::LittleEndian {}
}

// Extension trait — adds methods to any Read/Write implementor
pub trait ReadBytesExt: std::io::Read {
    fn read_u16<T: ByteOrder>(&mut self) -> std::io::Result<u16> {
        let mut buf = [0u8; 2];
        self.read_exact(&mut buf)?;
        Ok(T::read_u16(&buf))
    }
}

impl<R: std::io::Read + ?Sized> ReadBytesExt for R {}

// Usage — endianness is a type parameter, resolved at compile time
let val = cursor.read_u16::<BigEndian>()?;
```

## Sorting Algorithms

### Built-in Sort Methods

```rust
let mut v = vec![3, 1, 4, 1, 5, 9];

// Stable sort — O(n log n), preserves order of equal elements
// Uses TimSort (adaptive merge sort), good for partially sorted data
v.sort();

// Unstable sort — O(n log n), may reorder equal elements
// Uses pattern-defeating quicksort, often faster than stable sort
v.sort_unstable();

// Custom comparison
v.sort_by(|a, b| b.cmp(a));       // Descending
v.sort_unstable_by(|a, b| {
    a.abs().cmp(&b.abs())         // By absolute value
});

// Sort by key extraction
let mut items = vec![("banana", 2), ("apple", 1), ("cherry", 3)];
items.sort_by_key(|&(_, n)| n);   // Sort by second element
items.sort_by_key(|&(name, _)| name); // Sort by name (alphabetical)

// Partial sort — get k-th smallest element in O(n) average
let mut v = vec![5, 3, 8, 1, 9, 2, 7];
v.select_nth_unstable(2);
// After: elements <= v[2] are before it, elements >= v[2] are after it
// v[2] is the 3rd smallest element (0-indexed)

// Sort floats (f64 doesn't implement Ord, only PartialOrd)
let mut floats = vec![3.14, 1.0, 2.718];
floats.sort_by(|a, b| a.partial_cmp(b).unwrap());
// Or use total_cmp for NaN-safe comparison (Rust 1.62+)
floats.sort_by(f64::total_cmp);

// Check if sorted
assert!(v.is_sorted()); // Rust 1.82+ (nightly: is_sorted_by, is_sorted_by_key)
```

### Basic Sorting Implementations

```rust
// Insertion Sort — O(n²), stable, excellent for small or nearly-sorted data
// Rust's standard sort uses insertion sort for small sub-arrays
fn insertion_sort<T: Ord>(arr: &mut [T]) {
    for i in 1..arr.len() {
        let mut j = i;
        while j > 0 && arr[j - 1] > arr[j] {
            arr.swap(j - 1, j);
            j -= 1;
        }
    }
}

// Selection Sort — O(n²), not stable, minimal number of swaps
// Useful when swap cost is high (large elements)
fn selection_sort<T: Ord>(arr: &mut [T]) {
    for i in 0..arr.len() {
        let min_idx = (i..arr.len())
            .min_by_key(|&j| &arr[j])
            .unwrap();
        arr.swap(i, min_idx);
    }
}

// Bubble Sort — O(n²), stable, simple, early exit for sorted data
fn bubble_sort<T: Ord>(arr: &mut [T]) {
    let mut swapped = true;
    while swapped {
        swapped = false;
        for i in 0..arr.len().saturating_sub(1) {
            if arr[i] > arr[i + 1] {
                arr.swap(i, i + 1);
                swapped = true;
            }
        }
    }
}
```

### Advanced Sorting Implementations

```rust
// Merge Sort — O(n log n), stable, requires O(n) extra space
// Predictable performance regardless of input distribution
fn merge_sort<T: Ord + Clone>(arr: &mut [T]) {
    if arr.len() <= 1 {
        return;
    }

    let mid = arr.len() / 2;
    let mut left = arr[..mid].to_vec();
    let mut right = arr[mid..].to_vec();

    merge_sort(&mut left);
    merge_sort(&mut right);

    let (mut i, mut j, mut k) = (0, 0, 0);
    while i < left.len() && j < right.len() {
        if left[i] <= right[j] {
            arr[k] = left[i].clone();
            i += 1;
        } else {
            arr[k] = right[j].clone();
            j += 1;
        }
        k += 1;
    }

    while i < left.len() {
        arr[k] = left[i].clone();
        i += 1;
        k += 1;
    }
    while j < right.len() {
        arr[k] = right[j].clone();
        j += 1;
        k += 1;
    }
}

// Quick Sort — O(n log n) average, O(n²) worst case, in-place
// Fastest in practice for random data due to cache efficiency
fn quick_sort<T: Ord>(arr: &mut [T]) {
    if arr.len() <= 1 {
        return;
    }

    let pivot_idx = partition(arr);
    quick_sort(&mut arr[..pivot_idx]);
    quick_sort(&mut arr[pivot_idx + 1..]);
}

fn partition<T: Ord>(arr: &mut [T]) -> usize {
    let pivot = arr.len() - 1;
    let mut i = 0;
    for j in 0..pivot {
        if arr[j] <= arr[pivot] {
            arr.swap(i, j);
            i += 1;
        }
    }
    arr.swap(i, pivot);
    i
}

// Heap Sort — O(n log n), in-place, not stable
// Guaranteed O(n log n) worst case, good for memory-constrained situations
fn heap_sort<T: Ord>(arr: &mut [T]) {
    let n = arr.len();

    // Build max heap (heapify all non-leaf nodes bottom-up)
    for i in (0..n / 2).rev() {
        heapify(arr, n, i);
    }

    // Extract elements one by one from heap
    for i in (1..n).rev() {
        arr.swap(0, i);     // Move current max to end
        heapify(arr, i, 0); // Restore heap property on reduced heap
    }
}

fn heapify<T: Ord>(arr: &mut [T], n: usize, i: usize) {
    let mut largest = i;
    let left = 2 * i + 1;
    let right = 2 * i + 2;

    if left < n && arr[left] > arr[largest] {
        largest = left;
    }
    if right < n && arr[right] > arr[largest] {
        largest = right;
    }
    if largest != i {
        arr.swap(i, largest);
        heapify(arr, n, largest);
    }
}
```

### Sorting Algorithm Comparison

| Algorithm | Best | Average | Worst | Space | Stable | Notes |
|-----------|------|---------|-------|-------|--------|-------|
| Insertion | O(n) | O(n²) | O(n²) | O(1) | Yes | Best for small/nearly-sorted |
| Selection | O(n²) | O(n²) | O(n²) | O(1) | No | Minimal swaps |
| Bubble | O(n) | O(n²) | O(n²) | O(1) | Yes | Early exit when sorted |
| Merge | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes | Predictable performance |
| Quick | O(n log n) | O(n log n) | O(n²) | O(log n) | No | Cache-friendly, fastest avg |
| Heap | O(n log n) | O(n log n) | O(n log n) | O(1) | No | In-place guaranteed O(n log n) |
| Rust `sort()` | O(n) | O(n log n) | O(n log n) | O(n) | Yes | TimSort (adaptive merge) |
| Rust `sort_unstable()` | O(n) | O(n log n) | O(n log n) | O(1) | No | Pattern-defeating quicksort |

## Graph Algorithms

### Graph Representation

```rust
use std::collections::{HashMap, HashSet, VecDeque, BinaryHeap};
use std::cmp::Reverse;

// Adjacency List — most common, O(V + E) space
// Good for sparse graphs (most real-world graphs)
type Graph = HashMap<usize, Vec<usize>>;

fn add_edge(graph: &mut Graph, from: usize, to: usize) {
    graph.entry(from).or_default().push(to);
    graph.entry(to).or_default().push(from);  // Remove for directed graph
}

// Weighted graph
type WeightedGraph = HashMap<usize, Vec<(usize, i32)>>;

fn add_weighted_edge(graph: &mut WeightedGraph, from: usize, to: usize, weight: i32) {
    graph.entry(from).or_default().push((to, weight));
    graph.entry(to).or_default().push((from, weight)); // Remove for directed
}

// Adjacency Matrix — O(V²) space
// Good for dense graphs, O(1) edge lookup
struct AdjMatrix {
    matrix: Vec<Vec<bool>>,
    n: usize,
}

impl AdjMatrix {
    fn new(n: usize) -> Self {
        AdjMatrix {
            matrix: vec![vec![false; n]; n],
            n,
        }
    }

    fn add_edge(&mut self, from: usize, to: usize) {
        self.matrix[from][to] = true;
        self.matrix[to][from] = true;  // Remove for directed
    }

    fn has_edge(&self, from: usize, to: usize) -> bool {
        self.matrix[from][to]
    }
}

// Edge List — simplest, good for Kruskal's MST
struct Edge {
    from: usize,
    to: usize,
    weight: i32,
}
```

### Depth-First Search (DFS)

```rust
// Recursive DFS
fn dfs(graph: &Graph, start: usize) -> Vec<usize> {
    let mut visited = HashSet::new();
    let mut result = Vec::new();
    dfs_helper(graph, start, &mut visited, &mut result);
    result
}

fn dfs_helper(
    graph: &Graph,
    node: usize,
    visited: &mut HashSet<usize>,
    result: &mut Vec<usize>,
) {
    if !visited.insert(node) {
        return;
    }

    result.push(node);

    if let Some(neighbors) = graph.get(&node) {
        for &neighbor in neighbors {
            dfs_helper(graph, neighbor, visited, result);
        }
    }
}

// Iterative DFS using explicit stack
// Avoids stack overflow on deep graphs
fn dfs_iterative(graph: &Graph, start: usize) -> Vec<usize> {
    let mut visited = HashSet::new();
    let mut stack = vec![start];
    let mut result = Vec::new();

    while let Some(node) = stack.pop() {
        if !visited.insert(node) {
            continue;
        }

        result.push(node);

        if let Some(neighbors) = graph.get(&node) {
            // Reverse to visit in same order as recursive DFS
            for &neighbor in neighbors.iter().rev() {
                if !visited.contains(&neighbor) {
                    stack.push(neighbor);
                }
            }
        }
    }
    result
}
```

### Breadth-First Search (BFS)

```rust
fn bfs(graph: &Graph, start: usize) -> Vec<usize> {
    let mut visited = HashSet::new();
    let mut queue = VecDeque::new();
    let mut result = Vec::new();

    visited.insert(start);
    queue.push_back(start);

    while let Some(node) = queue.pop_front() {
        result.push(node);

        if let Some(neighbors) = graph.get(&node) {
            for &neighbor in neighbors {
                if visited.insert(neighbor) {
                    queue.push_back(neighbor);
                }
            }
        }
    }
    result
}

// BFS shortest path (unweighted graph)
// Returns the shortest path from start to end, or None if unreachable
fn bfs_shortest_path(
    graph: &Graph,
    start: usize,
    end: usize,
) -> Option<Vec<usize>> {
    let mut visited = HashSet::new();
    let mut queue = VecDeque::new();
    let mut parent: HashMap<usize, usize> = HashMap::new();

    visited.insert(start);
    queue.push_back(start);

    while let Some(node) = queue.pop_front() {
        if node == end {
            // Reconstruct path by following parent pointers
            let mut path = vec![end];
            let mut current = end;
            while let Some(&p) = parent.get(&current) {
                path.push(p);
                current = p;
            }
            path.reverse();
            return Some(path);
        }

        if let Some(neighbors) = graph.get(&node) {
            for &neighbor in neighbors {
                if visited.insert(neighbor) {
                    parent.insert(neighbor, node);
                    queue.push_back(neighbor);
                }
            }
        }
    }
    None // No path found
}
```

### Dijkstra's Algorithm

```rust
// Single-source shortest path for non-negative weights
// O((V + E) log V) with binary heap
fn dijkstra(
    graph: &WeightedGraph,
    start: usize,
    n: usize,
) -> Vec<i32> {
    let mut dist = vec![i32::MAX; n];
    let mut heap = BinaryHeap::new();

    dist[start] = 0;
    heap.push(Reverse((0, start)));

    while let Some(Reverse((d, node))) = heap.pop() {
        // Skip if we already found a shorter path
        if d > dist[node] {
            continue;
        }

        if let Some(neighbors) = graph.get(&node) {
            for &(neighbor, weight) in neighbors {
                let new_dist = dist[node] + weight;
                if new_dist < dist[neighbor] {
                    dist[neighbor] = new_dist;
                    heap.push(Reverse((new_dist, neighbor)));
                }
            }
        }
    }
    dist
}

// Dijkstra with path reconstruction
fn dijkstra_with_path(
    graph: &WeightedGraph,
    start: usize,
    end: usize,
    n: usize,
) -> Option<(i32, Vec<usize>)> {
    let mut dist = vec![i32::MAX; n];
    let mut prev = vec![None; n];
    let mut heap = BinaryHeap::new();

    dist[start] = 0;
    heap.push(Reverse((0, start)));

    while let Some(Reverse((d, node))) = heap.pop() {
        if node == end {
            break;
        }
        if d > dist[node] {
            continue;
        }

        if let Some(neighbors) = graph.get(&node) {
            for &(neighbor, weight) in neighbors {
                let new_dist = dist[node] + weight;
                if new_dist < dist[neighbor] {
                    dist[neighbor] = new_dist;
                    prev[neighbor] = Some(node);
                    heap.push(Reverse((new_dist, neighbor)));
                }
            }
        }
    }

    if dist[end] == i32::MAX {
        return None;
    }

    let mut path = vec![end];
    let mut current = end;
    while let Some(p) = prev[current] {
        path.push(p);
        current = p;
    }
    path.reverse();
    Some((dist[end], path))
}
```

### Topological Sort

```rust
// Kahn's algorithm — BFS-based topological sort for DAGs
// Returns None if cycle detected
fn topological_sort(graph: &Graph, n: usize) -> Option<Vec<usize>> {
    // Calculate in-degree for each node
    let mut in_degree = vec![0usize; n];

    for neighbors in graph.values() {
        for &neighbor in neighbors {
            in_degree[neighbor] += 1;
        }
    }

    // Start with nodes that have no incoming edges
    let mut queue: VecDeque<_> = (0..n)
        .filter(|&i| in_degree[i] == 0)
        .collect();

    let mut result = Vec::new();

    while let Some(node) = queue.pop_front() {
        result.push(node);

        if let Some(neighbors) = graph.get(&node) {
            for &neighbor in neighbors {
                in_degree[neighbor] -= 1;
                if in_degree[neighbor] == 0 {
                    queue.push_back(neighbor);
                }
            }
        }
    }

    if result.len() == n {
        Some(result)
    } else {
        None  // Cycle detected — not all nodes reached
    }
}

// DFS-based topological sort (alternative)
fn topological_sort_dfs(graph: &Graph, n: usize) -> Option<Vec<usize>> {
    let mut visited = vec![false; n];
    let mut on_stack = vec![false; n]; // For cycle detection
    let mut result = Vec::new();

    for i in 0..n {
        if !visited[i] {
            if !topo_dfs(graph, i, &mut visited, &mut on_stack, &mut result) {
                return None; // Cycle detected
            }
        }
    }

    result.reverse();
    Some(result)
}

fn topo_dfs(
    graph: &Graph,
    node: usize,
    visited: &mut Vec<bool>,
    on_stack: &mut Vec<bool>,
    result: &mut Vec<usize>,
) -> bool {
    visited[node] = true;
    on_stack[node] = true;

    if let Some(neighbors) = graph.get(&node) {
        for &neighbor in neighbors {
            if on_stack[neighbor] {
                return false; // Cycle
            }
            if !visited[neighbor] && !topo_dfs(graph, neighbor, visited, on_stack, result) {
                return false;
            }
        }
    }

    on_stack[node] = false;
    result.push(node);
    true
}
```

## Algorithm Design Paradigms

### Divide and Conquer

```rust
// Binary search — divide search space in half each step
fn binary_search<T: Ord>(arr: &[T], target: &T) -> Option<usize> {
    let mut low = 0;
    let mut high = arr.len();

    while low < high {
        let mid = low + (high - low) / 2;  // Avoids overflow vs (low + high) / 2
        match arr[mid].cmp(target) {
            std::cmp::Ordering::Equal => return Some(mid),
            std::cmp::Ordering::Less => low = mid + 1,
            std::cmp::Ordering::Greater => high = mid,
        }
    }
    None
}

// Maximum crossing subarray (part of divide-and-conquer max subarray)
fn max_crossing_sum(arr: &[i32], low: usize, mid: usize, high: usize) -> i32 {
    // Find max sum going left from mid
    let mut left_sum = i32::MIN;
    let mut sum = 0;
    for i in (low..=mid).rev() {
        sum += arr[i];
        left_sum = left_sum.max(sum);
    }

    // Find max sum going right from mid+1
    let mut right_sum = i32::MIN;
    sum = 0;
    for i in (mid + 1)..=high {
        sum += arr[i];
        right_sum = right_sum.max(sum);
    }

    left_sum + right_sum
}

// Kadane's algorithm — O(n) maximum subarray sum
fn max_subarray_sum(arr: &[i32]) -> i32 {
    let mut max_ending_here = arr[0];
    let mut max_so_far = arr[0];

    for &x in &arr[1..] {
        max_ending_here = x.max(max_ending_here + x);
        max_so_far = max_so_far.max(max_ending_here);
    }
    max_so_far
}
```

### Dynamic Programming

```rust
use std::collections::HashMap;

// Fibonacci with memoization (top-down DP)
fn fib_memo(n: usize, memo: &mut HashMap<usize, u64>) -> u64 {
    if n <= 1 {
        return n as u64;
    }

    if let Some(&result) = memo.get(&n) {
        return result;
    }

    let result = fib_memo(n - 1, memo) + fib_memo(n - 2, memo);
    memo.insert(n, result);
    result
}

// Fibonacci with tabulation (bottom-up DP)
fn fib_tab(n: usize) -> u64 {
    if n <= 1 {
        return n as u64;
    }

    let mut dp = vec![0u64; n + 1];
    dp[1] = 1;

    for i in 2..=n {
        dp[i] = dp[i - 1] + dp[i - 2];
    }
    dp[n]
}

// Space-optimized Fibonacci — O(1) space
fn fib_optimized(n: usize) -> u64 {
    if n <= 1 {
        return n as u64;
    }
    let (mut a, mut b) = (0u64, 1u64);
    for _ in 2..=n {
        let temp = a + b;
        a = b;
        b = temp;
    }
    b
}

// Longest Common Subsequence — classic 2D DP
fn lcs(s1: &str, s2: &str) -> usize {
    let (m, n) = (s1.len(), s2.len());
    let s1: Vec<char> = s1.chars().collect();
    let s2: Vec<char> = s2.chars().collect();
    let mut dp = vec![vec![0; n + 1]; m + 1];

    for i in 1..=m {
        for j in 1..=n {
            if s1[i - 1] == s2[j - 1] {
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                dp[i][j] = dp[i - 1][j].max(dp[i][j - 1]);
            }
        }
    }
    dp[m][n]
}

// 0/1 Knapsack — maximize value within weight capacity
fn knapsack(weights: &[usize], values: &[i32], capacity: usize) -> i32 {
    let n = weights.len();
    let mut dp = vec![vec![0; capacity + 1]; n + 1];

    for i in 1..=n {
        for w in 0..=capacity {
            if weights[i - 1] <= w {
                dp[i][w] = dp[i - 1][w].max(
                    dp[i - 1][w - weights[i - 1]] + values[i - 1],
                );
            } else {
                dp[i][w] = dp[i - 1][w];
            }
        }
    }
    dp[n][capacity]
}

// Coin Change — minimum coins to make amount
fn coin_change(coins: &[usize], amount: usize) -> Option<usize> {
    let mut dp = vec![usize::MAX; amount + 1];
    dp[0] = 0;

    for i in 1..=amount {
        for &coin in coins {
            if coin <= i && dp[i - coin] != usize::MAX {
                dp[i] = dp[i].min(dp[i - coin] + 1);
            }
        }
    }

    if dp[amount] == usize::MAX {
        None
    } else {
        Some(dp[amount])
    }
}
```

### Greedy Algorithms

```rust
// Activity Selection — maximize non-overlapping activities
fn activity_selection(
    activities: &mut [(usize, usize)],  // (start, end)
) -> Vec<(usize, usize)> {
    // Greedy choice: always pick the activity that ends earliest
    activities.sort_by_key(|&(_, end)| end);

    let mut result = vec![activities[0]];
    let mut last_end = activities[0].1;

    for &activity in &activities[1..] {
        if activity.0 >= last_end {
            result.push(activity);
            last_end = activity.1;
        }
    }
    result
}

// Huffman Coding — minimum-cost merging (simplified)
fn huffman_cost(freqs: &[u32]) -> u32 {
    let mut heap: BinaryHeap<_> = freqs.iter()
        .map(|&f| Reverse(f))
        .collect();

    let mut total_cost = 0;

    while heap.len() > 1 {
        let Reverse(a) = heap.pop().unwrap();
        let Reverse(b) = heap.pop().unwrap();
        let combined = a + b;
        total_cost += combined;
        heap.push(Reverse(combined));
    }
    total_cost
}

// Fractional Knapsack — greedy works for fractional items
fn fractional_knapsack(
    items: &[(f64, f64)],  // (weight, value)
    capacity: f64,
) -> f64 {
    // Sort by value-to-weight ratio (descending)
    let mut indexed: Vec<(usize, f64)> = items.iter()
        .enumerate()
        .map(|(i, &(w, v))| (i, v / w))
        .collect();
    indexed.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());

    let mut remaining = capacity;
    let mut total_value = 0.0;

    for (i, _ratio) in indexed {
        let (weight, value) = items[i];
        if weight <= remaining {
            total_value += value;
            remaining -= weight;
        } else {
            total_value += value * (remaining / weight);
            break;
        }
    }
    total_value
}
```

### Backtracking

```rust
// N-Queens problem — place N queens on NxN board without attacking each other
fn solve_n_queens(n: usize) -> Vec<Vec<String>> {
    let mut results = Vec::new();
    let mut board = vec![vec!['.'; n]; n];
    backtrack_queens(&mut board, 0, &mut results);
    results
}

fn backtrack_queens(
    board: &mut Vec<Vec<char>>,
    row: usize,
    results: &mut Vec<Vec<String>>,
) {
    let n = board.len();
    if row == n {
        // Found a valid placement — record it
        let solution: Vec<String> = board.iter()
            .map(|row| row.iter().collect())
            .collect();
        results.push(solution);
        return;
    }

    for col in 0..n {
        if is_safe(board, row, col) {
            board[row][col] = 'Q';
            backtrack_queens(board, row + 1, results);
            board[row][col] = '.'; // Undo (backtrack)
        }
    }
}

fn is_safe(board: &[Vec<char>], row: usize, col: usize) -> bool {
    let n = board.len();

    // Check column above
    for i in 0..row {
        if board[i][col] == 'Q' {
            return false;
        }
    }

    // Check upper-left diagonal
    let (mut i, mut j) = (row as i32 - 1, col as i32 - 1);
    while i >= 0 && j >= 0 {
        if board[i as usize][j as usize] == 'Q' {
            return false;
        }
        i -= 1;
        j -= 1;
    }

    // Check upper-right diagonal
    let (mut i, mut j) = (row as i32 - 1, col as i32 + 1);
    while i >= 0 && j < n as i32 {
        if board[i as usize][j as usize] == 'Q' {
            return false;
        }
        i -= 1;
        j += 1;
    }

    true
}

// Subset Sum — find all subsets that sum to target
fn subset_sum(nums: &[i32], target: i32) -> Vec<Vec<i32>> {
    let mut results = Vec::new();
    let mut current = Vec::new();
    subset_sum_helper(nums, target, 0, &mut current, &mut results);
    results
}

fn subset_sum_helper(
    nums: &[i32],
    remaining: i32,
    start: usize,
    current: &mut Vec<i32>,
    results: &mut Vec<Vec<i32>>,
) {
    if remaining == 0 {
        results.push(current.clone());
        return;
    }
    if remaining < 0 {
        return;
    }

    for i in start..nums.len() {
        current.push(nums[i]);
        subset_sum_helper(nums, remaining - nums[i], i + 1, current, results);
        current.pop(); // Backtrack
    }
}
```

## Rust Libraries for Data Structures and Algorithms

### Petgraph (Graph Library)

```rust
use petgraph::graph::{Graph, NodeIndex};
use petgraph::algo::{dijkstra, min_spanning_tree, toposort, is_cyclic_directed};
use petgraph::visit::Bfs;
use petgraph::Direction;

// Create a weighted directed graph
let mut graph = Graph::<&str, u32>::new();
let a = graph.add_node("A");
let b = graph.add_node("B");
let c = graph.add_node("C");
let d = graph.add_node("D");
graph.add_edge(a, b, 1);
graph.add_edge(b, c, 2);
graph.add_edge(a, c, 4);
graph.add_edge(c, d, 1);

// Dijkstra's shortest path from node A
let distances = dijkstra(&graph, a, None, |e| *e.weight());
// distances: {A: 0, B: 1, C: 3, D: 4}

// Minimum spanning tree
let mst = min_spanning_tree(&graph);

// Topological sort (for DAGs)
let sorted = toposort(&graph, None);

// Cycle detection
let has_cycle = is_cyclic_directed(&graph);

// BFS traversal
let mut bfs = Bfs::new(&graph, a);
while let Some(node) = bfs.next(&graph) {
    println!("Visited: {}", graph[node]);
}

// Get neighbors
let neighbors: Vec<_> = graph.neighbors(a).collect();

// Undirected graph
let mut undirected = Graph::<&str, u32, petgraph::Undirected>::new_undirected();
```

### Rayon (Parallel Iterators)

```rust
use rayon::prelude::*;

// Parallel sort — uses all CPU cores
let mut data: Vec<i32> = (0..1_000_000).rev().collect();
data.par_sort();            // Stable parallel sort
data.par_sort_unstable();   // Unstable parallel sort (often faster)

// Parallel map-reduce
let sum: i32 = data.par_iter()
    .map(|x| x * 2)
    .sum();

// Parallel filter
let evens: Vec<_> = data.par_iter()
    .filter(|&&x| x % 2 == 0)
    .cloned()
    .collect();

// Parallel chunks processing
data.par_chunks(1000)
    .for_each(|chunk| {
        process_chunk(chunk);
    });

// Parallel find
let first_big: Option<&i32> = data.par_iter()
    .find_any(|&&x| x > 500_000);

// Convert sequential to parallel — just change .iter() to .par_iter()
let sequential: Vec<_> = data.iter().map(|x| x * 2).collect();
let parallel: Vec<_> = data.par_iter().map(|x| x * 2).collect();

// Custom thread pool
let pool = rayon::ThreadPoolBuilder::new()
    .num_threads(4)
    .build()
    .unwrap();

pool.install(|| {
    data.par_sort();
});
```

### ndarray and nalgebra (Linear Algebra)

```rust
// ndarray — N-dimensional arrays (NumPy-like)
use ndarray::{Array2, arr2, Array1};

let a = arr2(&[[1.0, 2.0], [3.0, 4.0]]);
let b = arr2(&[[5.0, 6.0], [7.0, 8.0]]);
let c = a.dot(&b);              // Matrix multiplication
let d = &a + &b;                // Element-wise addition
let e = &a * 2.0;               // Scalar multiplication
let row = a.row(0);             // View of first row
let col = a.column(1);          // View of second column
let transposed = a.t();         // Transpose (view)

// Create from shape
let zeros = Array2::<f64>::zeros((3, 4));
let ones = Array2::<f64>::ones((3, 4));
let eye = Array2::<f64>::eye(3);  // Identity matrix

// nalgebra — linear algebra (smaller, fixed-size matrices)
use nalgebra::{Matrix2, Matrix3, Vector2, Vector3};

let m = Matrix2::new(1.0, 2.0, 3.0, 4.0);
let v = Vector2::new(1.0, 2.0);
let result = m * v;             // Matrix-vector multiply

// Inverse and determinant
let det = m.determinant();
let inv = m.try_inverse();      // Returns Option (not all matrices invertible)

// Eigenvalues (for symmetric matrices)
let symmetric = Matrix3::new(
    2.0, 1.0, 0.0,
    1.0, 3.0, 1.0,
    0.0, 1.0, 2.0,
);
let eigen = symmetric.symmetric_eigen();
println!("Eigenvalues: {}", eigen.eigenvalues);
```

## Benchmarking with Criterion

### Setup

```toml
# Cargo.toml
[dev-dependencies]
criterion = "0.5"

[[bench]]
name = "my_benchmark"
harness = false
```

### Basic Benchmark Structure

```rust
// benches/my_benchmark.rs
use criterion::{criterion_group, criterion_main, Criterion, black_box};

fn basic_benchmark(c: &mut Criterion) {
    c.bench_function("function_name", |b| {
        b.iter(|| function_to_benchmark(black_box(42)))
    });
}

criterion_group!(benches, basic_benchmark);
criterion_main!(benches);
```

Run with: `cargo bench`

### Benchmarking with Varying Inputs

Use `bench_with_input` and `BenchmarkId` for parameterized benchmarks:

```rust
use criterion::{criterion_group, criterion_main, Criterion, BenchmarkId};

fn parameterized_benchmark(c: &mut Criterion) {
    let mut group = c.benchmark_group("Sorting Performance");

    for size in [100, 1000, 10000, 100000] {
        group.bench_with_input(
            BenchmarkId::new("sort", size),
            &size,
            |b, &size| {
                b.iter_with_setup(
                    || generate_random_vec(size),
                    |mut data| data.sort(),
                );
            },
        );
    }
    group.finish();
}

fn generate_random_vec(size: usize) -> Vec<i32> {
    use rand::Rng;
    let mut rng = rand::thread_rng();
    (0..size).map(|_| rng.gen()).collect()
}

criterion_group!(benches, parameterized_benchmark);
criterion_main!(benches);
```

### Separating Setup from Measurement

Use `iter_with_setup` to exclude setup costs from timing:

```rust
fn benchmark_with_setup(c: &mut Criterion) {
    let data_size = 10000;

    c.bench_function("process_vector", |b| {
        b.iter_with_setup(
            // Setup: NOT timed
            || (0..data_size).collect::<Vec<u32>>(),
            // Measurement: timed
            |vec| process_vector(&vec),
        );
    });
}
```

### Benchmarking Async Code

```rust
use criterion::{criterion_group, criterion_main, Criterion};
use tokio::runtime::Runtime;

async fn async_operation(data: &[u8]) -> Vec<u8> {
    tokio::time::sleep(std::time::Duration::from_micros(10)).await;
    data.to_vec()
}

fn async_benchmark(c: &mut Criterion) {
    let rt = Runtime::new().unwrap();
    let data = vec![0u8; 1000];

    c.bench_function("async_operation", |b| {
        b.iter(|| rt.block_on(async_operation(&data)))
    });
}

// Alternative: isolated runtime per iteration for complete independence
fn async_benchmark_isolated(c: &mut Criterion) {
    c.bench_function("async_isolated", |b| {
        b.iter_with_setup(
            || {
                let rt = Runtime::new().unwrap();
                let data = vec![0u8; 1000];
                (rt, data)
            },
            |(rt, data)| rt.block_on(async_operation(&data)),
        );
    });
}

criterion_group!(benches, async_benchmark, async_benchmark_isolated);
criterion_main!(benches);
```

### Preventing Compiler Optimization

Use `criterion::black_box` to prevent dead code elimination:

```rust
use criterion::black_box;

c.bench_function("compute", |b| {
    b.iter(|| {
        // black_box prevents the compiler from:
        // 1. Constant-folding the input
        // 2. Eliminating the computation as dead code
        let result = expensive_computation(black_box(42));
        black_box(result)  // Prevent optimizing away the result
    });
});

// Without black_box, the compiler might:
// - Compute the result at compile time (constant propagation)
// - Remove the entire computation since the result is unused
// - Inline and optimize away the function
```

### Benchmark Groups and Comparison

Compare multiple implementations of the same operation:

```rust
fn compare_implementations(c: &mut Criterion) {
    let mut group = c.benchmark_group("String Concatenation");

    let strings: Vec<String> = (0..1000).map(|i| format!("item_{i}")).collect();

    group.bench_function("push_str", |b| {
        b.iter(|| {
            let mut result = String::new();
            for s in &strings {
                result.push_str(s);
            }
            result
        });
    });

    group.bench_function("join", |b| {
        b.iter(|| strings.join(""));
    });

    group.bench_function("collect", |b| {
        b.iter(|| strings.iter().map(|s| s.as_str()).collect::<String>());
    });

    group.bench_function("with_capacity", |b| {
        b.iter(|| {
            let total_len: usize = strings.iter().map(|s| s.len()).sum();
            let mut result = String::with_capacity(total_len);
            for s in &strings {
                result.push_str(s);
            }
            result
        });
    });

    group.finish();
}
```

### Benchmark Configuration

```rust
use criterion::{criterion_group, Criterion};
use std::time::Duration;

fn configured_benchmark(c: &mut Criterion) {
    let mut group = c.benchmark_group("Configured");

    // Set measurement time (default: 5 seconds)
    group.measurement_time(Duration::from_secs(10));

    // Set sample size (default: 100)
    group.sample_size(200);

    // Set warm-up time (default: 3 seconds)
    group.warm_up_time(Duration::from_secs(5));

    // Set confidence level (default: 0.95)
    group.confidence_level(0.99);

    // Set noise threshold (default: 0.01 = 1%)
    group.noise_threshold(0.02);

    group.bench_function("my_function", |b| {
        b.iter(|| my_function());
    });

    group.finish();
}
```

## Identifying Performance Bottlenecks

### CPU-Bound Bottlenecks

Symptoms: High CPU utilization, execution time dominated by computation.

```rust
// BAD: O(n²) bubble sort
pub fn inefficient_sort(data: &mut [i32]) {
    let n = data.len();
    for i in 0..n {
        for j in 0..n - 1 - i {
            if data[j] > data[j + 1] {
                data.swap(j, j + 1);
            }
        }
    }
}

// GOOD: O(n log n) standard library sort
pub fn efficient_sort(data: &mut [i32]) {
    data.sort_unstable(); // Pattern-defeating quicksort
}
```

### I/O-Bound Bottlenecks

Symptoms: Low CPU utilization, high wait times on I/O operations.

```rust
use std::fs::File;
use std::io::{self, BufReader, BufWriter, Read, Write};
use std::path::Path;

// BAD: byte-by-byte reading (excessive syscalls)
fn read_file_slow(path: &Path) -> io::Result<Vec<u8>> {
    let mut file = File::open(path)?;
    let mut buffer = Vec::new();
    let mut byte = [0u8; 1];

    loop {
        match file.read(&mut byte)? {
            0 => break,
            _ => buffer.push(byte[0]),
        }
    }
    Ok(buffer)
}

// GOOD: buffered reading (batches syscalls, 8KB buffer by default)
fn read_file_fast(path: &Path) -> io::Result<Vec<u8>> {
    let file = File::open(path)?;
    let mut reader = BufReader::new(file);
    let mut buffer = Vec::new();
    reader.read_to_end(&mut buffer)?;
    Ok(buffer)
}

// GOOD: buffered writing
fn write_file_fast(path: &Path, data: &[u8]) -> io::Result<()> {
    let file = File::create(path)?;
    let mut writer = BufWriter::new(file);
    writer.write_all(data)?;
    writer.flush()?;
    Ok(())
}

// Even better: pre-allocate buffer based on file size
fn read_file_preallocated(path: &Path) -> io::Result<Vec<u8>> {
    let metadata = std::fs::metadata(path)?;
    let mut buffer = Vec::with_capacity(metadata.len() as usize);
    File::open(path)?.read_to_end(&mut buffer)?;
    Ok(buffer)
}
```

### Cache-Related Bottlenecks

Symptoms: Slower than expected despite low CPU utilization, high cache miss rates in perf.

```rust
// GOOD cache locality: sequential access through contiguous memory
// Each Point is adjacent in memory — CPU prefetcher works well
#[repr(C)]
struct Point {
    x: f64,
    y: f64,
    z: f64,
}

pub fn process_contiguous(points: &[Point]) -> f64 {
    let mut sum = 0.0;
    for point in points {
        sum += point.x + point.y + point.z;
    }
    sum
}

// BAD cache locality: scattered access via pointers
// Each Box<Point> is a separate heap allocation — random memory addresses
pub fn process_scattered(points: &[Box<Point>]) -> f64 {
    let mut sum = 0.0;
    for point in points {
        sum += point.x + point.y + point.z;  // Cache miss on each deref
    }
    sum
}

// Structure of Arrays (SoA) — better for processing one field at a time
struct PointsSoA {
    x: Vec<f64>,
    y: Vec<f64>,
    z: Vec<f64>,
}

pub fn sum_x_soa(points: &PointsSoA) -> f64 {
    points.x.iter().sum()  // Perfect sequential access, SIMD-friendly
}

// Array of Structures (AoS) — better when accessing all fields together
// The Point struct above is AoS — each point's fields are adjacent
```

### Concurrency Bottlenecks (Mutex Contention)

Symptoms: Poor scaling with additional threads, threads spending time waiting for locks.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

// BAD: all threads compete for single lock constantly
pub fn high_contention(num_threads: usize, iterations: usize) -> usize {
    let counter = Arc::new(Mutex::new(0usize));
    let mut handles = vec![];

    for _ in 0..num_threads {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..iterations {
                let mut count = counter.lock().unwrap();
                *count += 1;
                // Lock held for entire iteration — massive contention
            }
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }
    *counter.lock().unwrap()
}

// GOOD: minimize critical section, accumulate locally
pub fn low_contention(num_threads: usize, iterations: usize) -> usize {
    let counter = Arc::new(Mutex::new(0usize));
    let mut handles = vec![];

    for _ in 0..num_threads {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            // Accumulate locally without any lock
            let mut local_sum = 0;
            for _ in 0..iterations {
                local_sum += 1;
            }
            // Single lock acquisition per thread
            *counter.lock().unwrap() += local_sum;
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }
    *counter.lock().unwrap()
}

// BEST: use atomics for simple counters (no lock needed)
use std::sync::atomic::{AtomicUsize, Ordering};

pub fn atomic_counter(num_threads: usize, iterations: usize) -> usize {
    let counter = Arc::new(AtomicUsize::new(0));
    let mut handles = vec![];

    for _ in 0..num_threads {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            let mut local_sum = 0;
            for _ in 0..iterations {
                local_sum += 1;
            }
            counter.fetch_add(local_sum, Ordering::Relaxed);
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }
    counter.load(Ordering::Relaxed)
}
```

## Profiling Tools

### perf (Linux)

CPU sampling profiler for identifying hot functions.

#### Setup for Rust

```bash
# Build with debug symbols in release mode for accurate profiling
# Add to Cargo.toml:
# [profile.release]
# debug = true  # Include debug info for symbol resolution

# Or create a dedicated profiling profile:
# [profile.profiling]
# inherits = "release"
# debug = true

cargo build --release
# Or: cargo build --profile profiling

# Ensure kernel allows profiling (may need root)
echo -1 | sudo tee /proc/sys/kernel/perf_event_paranoid
```

#### Recording Profiles

```bash
# Basic recording — samples call stacks at default frequency
perf record -g target/release/your_app

# With specific frequency (99Hz to avoid lockstep with timer interrupts)
perf record -F 99 -g target/release/your_app

# Record for an already-running process
perf record -g -p $(pgrep your_app)

# Record specific hardware events
perf record -e cycles,cache-misses,branch-misses -g target/release/your_app

# Record with frame pointer call graph (fast, but requires -Cforce-frame-pointers)
perf record --call-graph fp target/release/your_app

# Record with DWARF-based call graph (more accurate, larger output)
perf record --call-graph dwarf target/release/your_app

# Record for a specific duration (30 seconds)
perf record -g -- sleep 30 &
PERF_PID=$!
target/release/your_app &
APP_PID=$!
sleep 30
kill $APP_PID
wait $PERF_PID
```

#### Analyzing Results

```bash
# Interactive TUI report — navigate with arrow keys, Enter to drill in
perf report

# Top functions by sample count (non-interactive)
perf report --stdio | head -50

# Show annotated assembly for a specific function
perf annotate --symbol=hot_function

# Show call graph as text (flat profile)
perf report --stdio --no-children

# Export report to file
perf report --stdio > profile_report.txt

# Quick statistics summary (cycles, instructions, cache misses, etc.)
perf stat target/release/your_app

# Statistics for specific events
perf stat -e cache-misses,cache-references,instructions,cycles target/release/your_app
```

#### Interpreting perf Output

```
# Overhead  Command      Shared Object        Symbol
   35.21%  your_app     your_app             [.] process_data
   22.15%  your_app     libc.so.6            [.] malloc
   15.43%  your_app     your_app             [.] parse_input
    8.72%  your_app     [kernel.kallsyms]    [k] copy_user_enhanced

# [.] = user space function
# [k] = kernel space function
# High malloc % → allocation-heavy code, consider pre-allocation or arena
# High kernel % → I/O heavy, consider batching or async
# Hot function at top → focus optimization here for biggest impact
```

### flamegraph

Visual flame graph generation for understanding call hierarchies.

```bash
# Install cargo-flamegraph
cargo install flamegraph

# Generate flame graph (uses perf on Linux, dtrace on macOS)
cargo flamegraph --release
# Output: flamegraph.svg

# For a specific binary
cargo flamegraph --release --bin my_binary

# For benchmarks
cargo flamegraph --release --bench my_benchmark -- --bench

# Custom output filename
cargo flamegraph --release -o profile.svg

# With specific sampling frequency (997 to avoid alias with timer)
cargo flamegraph --release --freq 997

# Include kernel functions
cargo flamegraph --release --root

# Pass arguments to your program
cargo flamegraph --release -- --config prod.toml --workers 4
```

#### Reading Flame Graphs

- **Width** = Time spent (wider = more time)
- **Height** = Call stack depth (bottom = entry point, top = leaf functions)
- **Color** = Generally arbitrary (some tools color by module)
- Click to zoom into a stack frame
- Search (Ctrl+F in browser) to highlight specific functions

```
Common patterns to look for:
- Wide bars at TOP    → expensive leaf functions (optimize these first)
- Wide bars THROUGHOUT → expensive call paths (consider algorithmic improvement)
- Many thin spikes   → high call overhead (consider inlining or batching)
- Flat wide tops     → time in specific functions, not their callees
- Wide "malloc" bars → allocation-heavy (pre-allocate, reuse buffers)
```

### Valgrind Suite

Valgrind provides multiple profiling tools for different analysis needs. Note: Valgrind runs ~10-50x slower than native execution.

#### Callgrind (Function Profiling)

```bash
# Profile function calls and instruction counts
valgrind --tool=callgrind target/release/your_app

# With cache simulation for more accuracy
valgrind --tool=callgrind --cache-sim=yes target/release/your_app

# Output file: callgrind.out.<pid>

# View with KCachegrind (GUI — recommended)
kcachegrind callgrind.out.*

# Or with callgrind_annotate (CLI)
callgrind_annotate callgrind.out.* | head -100

# Compare two runs
callgrind_annotate --diff callgrind.out.1234 callgrind.out.5678
```

#### Cachegrind (Cache Analysis)

```bash
# Analyze cache behavior
valgrind --tool=cachegrind target/release/your_app

# Output shows cache miss rates:
# I1  = L1 instruction cache misses
# D1  = L1 data cache misses
# LL  = Last-level (L2/L3) cache misses

# Annotate source with cache miss counts
cg_annotate cachegrind.out.* src/hot_module.rs

# High D1 miss rate → poor data locality, consider SoA layout
# High LL miss rate → data doesn't fit in cache, reduce working set
```

#### Massif (Heap Profiler)

```bash
# Profile heap usage over time
valgrind --tool=massif target/release/your_app

# With stack profiling (much slower but more complete)
valgrind --tool=massif --stacks=yes target/release/your_app

# Visualize results (text-based graph)
ms_print massif.out.*

# GUI visualization (if available)
massif-visualizer massif.out.*
```

#### DHAT (Detailed Heap Analysis)

```bash
# Detailed heap allocation analysis
valgrind --tool=dhat target/release/your_app

# Opens interactive viewer in browser
# Shows: allocation sites, sizes, lifetimes, access patterns

# Identifies:
# - Short-lived allocations (could use stack)
# - Large allocations that are mostly unused
# - Allocation sites creating the most bytes
```

### Heaptrack (Heap Profiler)

Faster alternative to Valgrind for heap profiling, with ~2x overhead vs Valgrind's ~20x.

```bash
# Install
sudo apt install heaptrack heaptrack-gui    # Debian/Ubuntu
sudo pacman -S heaptrack                     # Arch Linux

# Profile your application
heaptrack target/release/your_app

# Output: heaptrack.your_app.<pid>.gz

# Analyze with GUI (recommended)
heaptrack_gui heaptrack.your_app.*.gz

# Analyze with CLI
heaptrack_print heaptrack.your_app.*.gz
```

#### Heaptrack Analysis

The GUI shows:
- **Summary**: Total allocations, peak memory, leaked memory
- **Flame graph**: Allocation call stacks (width = bytes allocated)
- **Top allocators**: Functions allocating the most memory
- **Timeline**: Memory usage over time
- **Allocation sizes**: Distribution of allocation sizes

```
What to look for:
- Functions with many small allocations → consider pooling or arena allocation
- Large peak memory vs sustained usage → optimize lifetimes
- Allocations in hot loops → move allocation outside loop
- Temporary allocations → use stack or reuse buffers
- Many Vec/String reallocations → pre-allocate with with_capacity
```

#### Heaptrack vs Valgrind Massif

| Aspect | Heaptrack | Massif |
|--------|-----------|--------|
| Overhead | ~2x slowdown | ~20x slowdown |
| Accuracy | Call stack sampling | Exact tracking |
| Output | Interactive GUI | Text-based |
| Best for | Quick analysis, exploring | Deep investigation |
| Stack tracking | Optional | Optional (slow) |

### DHAT (In-Process Heap Profiling)

For lighter-weight heap profiling without Valgrind:

```rust
// Cargo.toml:
// [features]
// dhat-heap = []
// [dependencies]
// dhat = { version = "0.3", optional = true }

#[cfg(feature = "dhat-heap")]
#[global_allocator]
static ALLOC: dhat::Alloc = dhat::Alloc;

fn main() {
    #[cfg(feature = "dhat-heap")]
    let _profiler = dhat::Profiler::new_heap();

    // ... application code ...

    // On drop, prints allocation statistics and writes dhat-heap.json
    // Open dhat-heap.json with Firefox's DHAT viewer:
    //   https://nnethercote.github.io/dh_view/dh_view.html
}
```

```bash
# Run with feature enabled
cargo run --features dhat-heap
```

### Instruments (macOS)

```bash
# CPU profiling with Time Profiler
xcrun xctrace record --template 'Time Profiler' \
    --launch -- target/release/your_app

# Memory profiling with Allocations
xcrun xctrace record --template 'Allocations' \
    --launch -- target/release/your_app

# Open results in Instruments.app
open *.trace
```

### samply (Cross-Platform)

Modern sampling profiler with Firefox Profiler visualization:

```bash
# Install
cargo install samply

# Profile and open in browser automatically
samply record target/release/your_app

# Opens Firefox Profiler UI showing:
# - Call tree with timing breakdown
# - Flame graph
# - Timeline markers
# - Thread activity

# Best features vs perf:
# - Cross-platform (Linux, macOS, Windows)
# - Beautiful interactive UI in browser
# - No root access needed on macOS
# - Automatic symbol resolution
```

### tracing Crate Instrumentation

Add structured timing and span tracking to your code:

```rust
use tracing::{info, instrument, span, Level};
use tracing_subscriber;

#[instrument]
fn heavy_computation(data: &str) -> String {
    let process_span = span!(Level::INFO, "processing", data_len = data.len());
    let _enter = process_span.enter();

    // Simulated work
    std::thread::sleep(std::time::Duration::from_millis(50));

    format!("Processed: {}", data.to_uppercase())
}

#[instrument]
fn main_task() {
    info!("Starting main task");

    let result1 = heavy_computation("example");
    info!(%result1);

    let result2 = heavy_computation("another");
    info!(%result2);

    info!("Main task finished");
}

fn setup_tracing() {
    tracing_subscriber::fmt()
        .with_max_level(Level::DEBUG)
        .with_span_events(tracing_subscriber::fmt::format::FmtSpan::CLOSE)
        .init();
    // Output includes span entry/exit timing:
    // INFO main_task: Starting main task
    // INFO heavy_computation{data="example"}: processing...
    // INFO heavy_computation{data="example"}: close time.busy=51ms
}
```

For production, integrate with distributed tracing systems (Jaeger, Zipkin) via `tracing-opentelemetry`.

## Best Practices

### Benchmarking Guidelines

1. **Always benchmark in release mode** — `cargo bench` does this automatically
2. **Use `black_box`** to prevent dead code elimination and constant folding
3. **Warm up the cache** — criterion handles this automatically with configurable warm-up time
4. **Run benchmarks on quiet systems** — minimize background processes, disable turbo boost for consistency
5. **Use statistical analysis** — criterion provides confidence intervals and regression detection
6. **Track regressions over time** — save baseline results: `cargo bench -- --save-baseline before_change`
7. **Separate setup from measurement** — use `iter_with_setup` for expensive test data creation
8. **Test with realistic data sizes** — don't benchmark with 10 elements when production uses 1M

### Profiling Guidelines

1. **Profile representative workloads** — use realistic data sizes, patterns, and concurrency levels
2. **Profile before optimizing** — identify actual bottlenecks; don't guess
3. **Verify optimizations with benchmarks** — re-profile after changes to confirm improvement
4. **Consider the whole system** — I/O, network, external services, and OS overhead all matter
5. **Use the right tool** — perf/flamegraph for CPU, cachegrind for memory access patterns, heaptrack for allocations, tracing for async

### Optimization Strategies by Bottleneck Type

| Bottleneck | Symptoms | Strategies |
|------------|----------|------------|
| CPU-bound | High CPU%, slow execution | Algorithm improvements, SIMD, parallelization (rayon) |
| I/O-bound | Low CPU%, high wait time | Buffering (BufReader/BufWriter), async I/O, batching, caching |
| Memory-bound | High allocation rate | Pre-allocate (with_capacity), reuse buffers, arena allocation, Cow<T> |
| Cache misses | Slow despite low CPU% | SoA layout, sequential access, smaller structs, avoid pointer chasing |
| Lock contention | Poor thread scaling | Finer-grained locks, lock-free (atomics), message passing, sharding |
| String copies | Hot clone/to_string | Use `&str`, `Cow<str>`, `Arc<str>`, `String::with_capacity` |

### Common Performance Patterns

```rust
// Pre-allocate collections
let mut v = Vec::with_capacity(expected_size);  // Avoid reallocations
let mut s = String::with_capacity(expected_len);
let mut map = HashMap::with_capacity(expected_entries);

// Reuse buffers instead of allocating in loops
let mut buf = String::new();
for item in items {
    buf.clear();  // Reuse allocation
    write!(buf, "{item}").unwrap();
    process(&buf);
}

// Use Cow for conditional ownership
use std::borrow::Cow;
fn process_name(name: &str) -> Cow<str> {
    if name.contains(' ') {
        Cow::Owned(name.replace(' ', "_"))
    } else {
        Cow::Borrowed(name) // No allocation needed
    }
}

// Avoid unnecessary cloning
// BAD: cloning just to iterate
for item in collection.clone() { ... }
// GOOD: borrow
for item in &collection { ... }

// Use iterators instead of indexed loops
// BAD: bounds-checked on each access
for i in 0..v.len() { sum += v[i]; }
// GOOD: bounds check eliminated by iterator
for x in &v { sum += x; }
// Also: iterator enables SIMD auto-vectorization more reliably
```

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: collections overview, iterators, ownership patterns
- **[language-patterns.md](language-patterns.md)** — Iterator composition, entry API, `Cow<T>`, zero-copy patterns
- **[async-concurrency.md](async-concurrency.md)** — Rayon parallel iterators, `DashMap`, atomic operations
- **[architecture.md](architecture.md)** — Performance profiling in production, tracing integration
- **[testing.md](testing.md)** — Benchmarking with criterion, property-based testing for data structures
