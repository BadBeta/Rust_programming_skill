# Rust Quick Reference — Most-Used Functions & Methods

Comprehensive reference organized by natural themes. Covers std library, async runtime, serialization, CLI, HTTP, and common crate patterns. Within each section, methods are ordered from most to least commonly used.

---

## Iterator Methods

The most-used trait in Rust. All methods are on `Iterator<Item = T>` unless noted.

```rust
// --- Transforming ---

// map — transform each element (used in nearly every Rust file)
let names: Vec<String> = users.iter().map(|u| u.name.clone()).collect();
let lengths: Vec<usize> = words.iter().map(|w| w.len()).collect();
let parsed: Vec<i32> = strings.iter().map(|s| s.parse().unwrap()).collect();

// filter — keep elements matching predicate
let active: Vec<_> = users.iter().filter(|u| u.is_active).collect();
let evens: Vec<_> = numbers.iter().filter(|n| *n % 2 == 0).collect();

// filter_map — filter + map in one step (removes None results)
let valid_ports: Vec<u16> = lines.iter()
    .filter_map(|line| line.parse::<u16>().ok())
    .collect();
// Equivalent to .map(...).filter(|x| x.is_some()).map(|x| x.unwrap())

let values: Vec<_> = entries.iter()
    .filter_map(|e| e.value.as_ref())  // Keep only Some values
    .collect();

// flat_map — map + flatten (each element produces an iterator)
let all_words: Vec<&str> = lines.iter()
    .flat_map(|line| line.split_whitespace())
    .collect();

let all_children: Vec<_> = nodes.iter()
    .flat_map(|node| &node.children)
    .collect();

// flatten — remove one level of nesting
let flat: Vec<i32> = vec![vec![1, 2], vec![3, 4]].into_iter().flatten().collect();
// [1, 2, 3, 4]

let values: Vec<_> = options.into_iter().flatten().collect();
// Flattens Option: Some(v) → v, None → skipped

// enumerate — add index to each element
for (i, item) in items.iter().enumerate() {
    println!("{i}: {item}");
}
let indexed: Vec<_> = items.iter().enumerate().collect(); // Vec<(usize, &T)>

// zip — pair elements from two iterators
let pairs: Vec<_> = keys.iter().zip(values.iter()).collect();
for (key, value) in keys.iter().zip(values.iter()) {
    map.insert(key, value);
}
// Stops at shorter iterator

// chain — concatenate two iterators
let combined: Vec<_> = first.iter().chain(second.iter()).collect();
let all = defaults.iter().chain(overrides.iter());

// take — first N elements
let top5: Vec<_> = sorted.iter().take(5).collect();
let preview: String = content.chars().take(100).collect();

// skip — skip first N elements
let rest: Vec<_> = items.iter().skip(1).collect();  // Skip header
let page: Vec<_> = items.iter().skip(offset).take(limit).collect(); // Pagination

// take_while / skip_while — conditional take/skip
let header: Vec<_> = lines.iter().take_while(|l| !l.is_empty()).collect();
let body: Vec<_> = lines.iter().skip_while(|l| !l.is_empty()).skip(1).collect();

// peekable — look ahead without consuming
let mut iter = tokens.iter().peekable();
while let Some(token) = iter.next() {
    if iter.peek() == Some(&&Token::Comma) {
        iter.next(); // consume comma
    }
}

// cloned / copied — convert &T to T
let owned: Vec<String> = borrowed_strings.iter().cloned().collect();
let vals: Vec<i32> = int_refs.iter().copied().collect(); // For Copy types

// inspect — debug without consuming (like tap)
let result: Vec<_> = items.iter()
    .inspect(|x| tracing::debug!(?x, "processing"))
    .map(|x| x * 2)
    .collect();

// map_while — map + take_while combined (stable 1.57+)
let parsed: Vec<i32> = strings.iter()
    .map_while(|s| s.parse().ok())  // Stop at first parse failure
    .collect();

// scan — stateful map (carries accumulator)
let running_sum: Vec<i32> = numbers.iter()
    .scan(0, |acc, &x| { *acc += x; Some(*acc) })
    .collect();
// [1, 3, 6, 10, ...]

// step_by — take every Nth element
let every_other: Vec<_> = items.iter().step_by(2).collect();

// intersperse — insert separator between elements (nightly, use itertools)
// use itertools::Itertools;
// let csv = fields.iter().intersperse(&",").collect::<String>();

// --- Consuming / Reducing ---

// collect — consume iterator into collection (most-used consumer)
let vec: Vec<_> = iter.collect();
let string: String = chars.collect();
let set: HashSet<_> = iter.collect();
let map: HashMap<_, _> = pairs.collect();
let result: Result<Vec<_>, _> = fallible_iter.collect(); // Short-circuits on Err

// for_each — apply side effect to each element
entries.iter().for_each(|e| cache.insert(e.key, e.value));
// Prefer `for` loop when you need control flow (break, continue, ?)

// fold — reduce to single value with initial accumulator
let sum: i32 = numbers.iter().fold(0, |acc, &x| acc + x);
let product: i64 = factors.iter().fold(1, |acc, &x| acc * x);
let csv = fields.iter().fold(String::new(), |mut acc, f| {
    if !acc.is_empty() { acc.push(','); }
    acc.push_str(f);
    acc
});

// reduce — fold without initial value (returns Option)
let max = numbers.iter().copied().reduce(|a, b| a.max(b)); // Option<i32>

// sum / product — numeric reduction
let total: i32 = prices.iter().sum();
let total: f64 = weights.iter().sum();
let factorial: u64 = (1..=n).product();

// count — number of elements
let active_count = users.iter().filter(|u| u.is_active).count();
let line_count = content.lines().count();

// min / max — smallest/largest element (returns Option)
let smallest = numbers.iter().min();           // Option<&i32>
let largest = numbers.iter().max();             // Option<&i32>

// min_by_key / max_by_key — by derived key
let cheapest = items.iter().min_by_key(|i| i.price);
let longest = strings.iter().max_by_key(|s| s.len());

// min_by / max_by — with custom comparator
let closest = points.iter().min_by(|a, b| {
    a.distance(origin).partial_cmp(&b.distance(origin)).unwrap()
});

// --- Searching ---

// find — first element matching predicate
let admin = users.iter().find(|u| u.role == Role::Admin); // Option<&User>

// find_map — find + map (return first Some)
let first_valid: Option<Config> = paths.iter()
    .find_map(|p| Config::load(p).ok());

// position — index of first match
let idx = items.iter().position(|x| x.id == target_id); // Option<usize>

// rposition — index of last match (requires ExactSizeIterator + DoubleEndedIterator)
let last = items.iter().rposition(|x| x.is_valid());

// any — true if any element matches
let has_errors = results.iter().any(|r| r.is_err());
if args.iter().any(|a| a == "--verbose") { /* ... */ }

// all — true if every element matches
let all_valid = entries.iter().all(|e| e.is_valid());
let sorted = windows.iter().all(|w| w[0] <= w[1]);

// --- Partitioning ---

// partition — split into two collections by predicate
let (valid, invalid): (Vec<_>, Vec<_>) = items.iter()
    .partition(|i| i.is_valid());

// unzip — split pairs into two collections
let (keys, values): (Vec<_>, Vec<_>) = entries.iter()
    .map(|e| (e.key.clone(), e.value.clone()))
    .unzip();

// chunks / windows (on slices, not Iterator)
for chunk in data.chunks(64) {     // Non-overlapping groups of 64
    process_batch(chunk);
}
for window in data.windows(3) {    // Sliding window of size 3
    let [a, b, c] = window else { unreachable!() };
}

// --- Adapters for DoubleEndedIterator ---

// rev — reverse iteration
for item in items.iter().rev() { /* last to first */ }
let last3: Vec<_> = items.iter().rev().take(3).collect();

// --- Chaining pattern (real production example from ripgrep-style code) ---
let matches: Vec<Match> = lines.iter()
    .enumerate()
    .filter(|(_, line)| !line.is_empty())
    .filter_map(|(num, line)| {
        regex.find(line).map(|m| Match {
            line_number: num + 1,
            offset: m.start(),
            text: m.as_str().to_string(),
        })
    })
    .take(max_results)
    .collect();
```

## String & str Methods

```rust
let s: &str = "Hello, World! 🌍";
let mut owned = String::from("hello");

// --- Creation ---
String::new()                              // Empty string
String::with_capacity(1024)                // Pre-allocated
String::from("hello")                      // From &str
"hello".to_string()                        // From &str (Display)
"hello".to_owned()                         // From &str (ToOwned)
format!("{}-{}", a, b)                     // Formatted creation
["a", "b", "c"].join(", ")                // Join slice: "a, b, c"
["a", "b", "c"].concat()                  // Concat: "abc"
"x".repeat(5)                              // "xxxxx"
String::from_utf8(bytes)?                  // Vec<u8> → String (fallible)
String::from_utf8_lossy(&bytes)            // &[u8] → Cow<str> (replaces invalid)

// --- Searching ---
s.contains("World")                        // true — substring check
s.contains('!')                            // true — char check
s.starts_with("Hello")                     // true
s.ends_with('!')                           // false (ends with 🌍)
s.find("World")                            // Some(7) — first occurrence byte index
s.rfind(',')                               // Some(5) — last occurrence
s.matches("l").count()                     // 3 — count occurrences
s.match_indices("l")                       // Iterator of (usize, &str)

// --- Splitting ---
s.split(',')                               // Iterator<Item = &str>
s.split(',').collect::<Vec<_>>()          // ["Hello", " World! 🌍"]
s.splitn(2, ',')                           // At most 2 parts
s.rsplit('.')                              // Split from right
s.rsplitn(2, '.')                          // At most 2 parts from right
s.split_whitespace()                       // Split on any whitespace
s.split_ascii_whitespace()                 // ASCII whitespace only
s.split_terminator('\n')                   // Like split but no trailing empty
s.lines()                                  // Split by \n or \r\n
s.split_at(5)                              // (&str, &str) at byte index
s.split_once(',')                          // Option<(&str, &str)> first only
s.rsplit_once(',')                         // Option<(&str, &str)> last only

// --- Trimming ---
" hello ".trim()                           // "hello" — both ends
" hello ".trim_start()                     // "hello " — left only
" hello ".trim_end()                       // " hello" — right only
"##hello##".trim_matches('#')              // "hello"
"hello\n\r".trim_end_matches(|c: char| c.is_ascii_whitespace())

// --- Replacing ---
s.replace("World", "Rust")                // New string with replacements
s.replacen("l", "L", 2)                   // Replace first N occurrences

// --- Case ---
s.to_uppercase()                           // "HELLO, WORLD! 🌍" (Unicode)
s.to_lowercase()                           // "hello, world! 🌍" (Unicode)
s.to_ascii_uppercase()                     // ASCII only
s.to_ascii_lowercase()                     // ASCII only
s.eq_ignore_ascii_case("HELLO, WORLD! 🌍") // Case-insensitive compare

// --- Inspection ---
s.len()                                    // Byte length (NOT char count!)
s.is_empty()                               // true if len() == 0
s.is_ascii()                               // false (contains 🌍)
s.chars().count()                          // Character count (Unicode)
s.as_bytes()                               // &[u8]
s.as_bytes()[0]                            // b'H' (72u8)
s.is_char_boundary(5)                      // Safe to split here?

// --- Character iteration ---
s.chars()                                  // Iterator<Item = char>
s.char_indices()                           // Iterator<Item = (usize, char)>
s.bytes()                                  // Iterator<Item = u8>
s.encode_utf16()                           // Iterator<Item = u16>

// --- String (owned) mutation ---
owned.push('!');                           // Append char
owned.push_str(" world");                  // Append &str
owned.insert(0, 'H');                      // Insert char at byte index
owned.insert_str(0, "Hello ");             // Insert &str at byte index
owned.truncate(5);                         // Keep first 5 bytes
owned.pop();                               // Remove last char → Option<char>
owned.remove(0);                           // Remove char at byte index → char
owned.clear();                             // Empty the string
owned.retain(|c| c.is_alphanumeric());     // Keep matching chars
owned.drain(..5);                          // Remove range, returns iterator

// --- Conversion ---
owned.as_str()                             // &str
owned.as_mut_str()                         // &mut str
owned.into_bytes()                         // Vec<u8>
&owned[1..5]                               // &str slice (panics if not char boundary!)
owned.get(1..5)                            // Option<&str> (safe slicing)

// --- Building strings efficiently ---
use std::fmt::Write;
let mut buf = String::with_capacity(estimate);
write!(buf, "key={}, value={}", k, v).unwrap();
writeln!(buf, "line {i}").unwrap();

// Join with separator (production pattern)
let csv: String = fields.iter()
    .map(|f| f.to_string())
    .collect::<Vec<_>>()
    .join(",");
```

## Vec & Slice Methods

```rust
// --- Creation ---
Vec::new()                                 // Empty
Vec::with_capacity(1000)                   // Pre-allocated
vec![0; 100]                               // 100 zeros
vec![1, 2, 3]                              // From values
Vec::from([1, 2, 3])                       // From array
(0..10).collect::<Vec<_>>()               // From iterator

// --- Adding ---
v.push(item);                              // Append to end
v.insert(idx, item);                       // Insert at index (shifts right, O(n))
v.extend([4, 5, 6]);                       // Append from IntoIterator
v.extend_from_slice(&[7, 8]);             // Append from slice (Clone required)
v.append(&mut other);                      // Move all from other vec
v.resize(10, 0);                           // Grow/shrink to len, fill with value
v.resize_with(10, Default::default);       // Grow/shrink with closure

// --- Removing ---
v.pop()                                    // Remove last → Option<T>
v.remove(idx)                              // Remove at index → T (shifts, O(n))
v.swap_remove(idx)                         // Remove at index → T (swaps with last, O(1))
v.truncate(5)                              // Keep only first 5
v.retain(|x| x.is_valid())               // Keep elements matching predicate
v.retain_mut(|x| { x.clean(); x.valid }) // Mutable access during retain
v.drain(1..3)                              // Remove range → iterator of removed
v.drain(..)                                // Remove all → iterator (like into_iter but reuses alloc)
v.clear()                                  // Remove all elements
v.dedup()                                  // Remove consecutive duplicates
v.dedup_by_key(|e| e.id)                  // Dedup by derived key
v.dedup_by(|a, b| a.name == b.name)       // Dedup with custom equality

// --- Access ---
v.first()                                  // Option<&T>
v.last()                                   // Option<&T>
v.first_mut()                              // Option<&mut T>
v.last_mut()                               // Option<&mut T>
v.get(idx)                                 // Option<&T> — bounds-checked
v.get_mut(idx)                             // Option<&mut T>
v[idx]                                     // &T — panics if out of bounds
v.get(1..3)                                // Option<&[T]>

// --- Searching ---
v.contains(&42)                            // bool (linear scan)
v.iter().position(|x| x.id == target)     // Option<usize> — first index
v.iter().rposition(|x| x.id == target)    // Option<usize> — last index
v.binary_search(&42)                       // Result<usize, usize> — must be sorted
v.binary_search_by(|x| x.cmp(&target))   // Custom comparator
v.binary_search_by_key(&target, |x| x.key) // By derived key
v.partition_point(|x| x < &target)        // Binary search for partition point

// --- Sorting ---
v.sort()                                   // Stable sort (requires Ord)
v.sort_unstable()                          // Faster, not stable
v.sort_by(|a, b| b.cmp(a))               // Custom comparator (reverse)
v.sort_by(|a, b| a.name.cmp(&b.name))    // Sort by field
v.sort_by_key(|item| item.priority)       // Sort by derived key
v.sort_unstable_by_key(|item| std::cmp::Reverse(item.score)) // Reverse sort by key
v.is_sorted()                              // bool (stable 1.82+)

// --- Slicing ---
let slice: &[T] = &v[1..3];              // Borrowed slice
let (left, right) = v.split_at(mid);     // Split into two slices
let (left, right) = v.split_at_mut(mid); // Mutable split
v.chunks(64)                               // Iterator of &[T] chunks (last may be shorter)
v.chunks_exact(64)                         // Iterator (panics if not divisible, remainder via .remainder())
v.rchunks(64)                              // Chunks from the end
v.windows(3)                               // Sliding window iterator
v.split(|x| *x == 0)                     // Split at elements matching predicate
v.splitn(3, |x| *x == 0)                 // Split at most N times
v.group_by(|a, b| a == b)                // Group consecutive equal elements (nightly)

// --- Transforming ---
v.iter()                                   // Iterator<Item = &T>
v.iter_mut()                               // Iterator<Item = &mut T>
v.into_iter()                              // Iterator<Item = T> (consumes vec)
v.as_slice()                               // &[T]
v.as_mut_slice()                           // &mut [T]

// --- Capacity ---
v.len()                                    // Number of elements
v.is_empty()                               // true if len() == 0
v.capacity()                               // Allocated capacity
v.reserve(100)                             // Ensure space for 100 more
v.shrink_to_fit()                          // Release excess capacity

// --- Conversion ---
v.into_boxed_slice()                       // Box<[T]>
let arr: [T; 3] = v.try_into().unwrap();  // Vec → array (panics if wrong size)

// --- Slice methods (also available on Vec via Deref) ---
slice.iter().copied()                      // Iterator<Item = T> for Copy types
slice.to_vec()                             // Clone into Vec
slice.repeat(3)                            // Repeat N times into Vec
slice.concat()                             // Flatten &[Vec<T>] → Vec<T>
slice.join(&separator)                     // Join &[String] with separator
slice.fill(0)                              // Fill with value
slice.fill_with(Default::default)          // Fill with closure
slice.rotate_left(2)                       // Rotate elements left
slice.rotate_right(2)                      // Rotate elements right
slice.reverse()                            // Reverse in place
slice.swap(0, 3)                           // Swap two elements
slice.copy_from_slice(&src)               // Copy from another slice (same length)
```

## HashMap & HashSet

```rust
use std::collections::{HashMap, HashSet};

// --- HashMap creation ---
HashMap::new()
HashMap::with_capacity(100)
HashMap::from([("a", 1), ("b", 2)])        // From array of tuples
let map: HashMap<_, _> = vec.into_iter().collect(); // From iterator of pairs

// --- Insert / Update ---
map.insert(key, value)                     // Returns Option<V> (previous value)
map.insert(key, value);                    // Overwrites existing

// Entry API — the most idiomatic insert-or-update pattern
map.entry(key).or_insert(default)          // Insert if absent, return &mut V
map.entry(key).or_insert_with(|| expensive()) // Lazy default
map.entry(key).or_default()                // Uses Default::default()
*map.entry(key).or_insert(0) += 1;        // Counter pattern
map.entry(key)
    .and_modify(|v| *v += 1)              // Modify if present
    .or_insert(1);                         // Insert if absent

// --- Access ---
map.get(&key)                              // Option<&V>
map.get_mut(&key)                          // Option<&mut V>
map[&key]                                  // &V — panics if missing
map.get_key_value(&key)                    // Option<(&K, &V)>
map.contains_key(&key)                     // bool
map.len()                                  // Number of entries
map.is_empty()                             // bool

// --- Removing ---
map.remove(&key)                           // Option<V>
map.remove_entry(&key)                     // Option<(K, V)>
map.retain(|k, v| v.is_valid())           // Keep matching entries
map.clear()                                // Remove all

// --- Iteration ---
map.keys()                                 // Iterator<Item = &K>
map.values()                               // Iterator<Item = &V>
map.values_mut()                           // Iterator<Item = &mut V>
map.iter()                                 // Iterator<Item = (&K, &V)>
map.iter_mut()                             // Iterator<Item = (&K, &mut V)>
map.into_keys()                            // Iterator<Item = K> (consumes map)
map.into_values()                          // Iterator<Item = V> (consumes map)
for (key, value) in &map { /* ... */ }    // Idiomatic iteration

// --- Merging ---
map.extend(other_map)                      // Merge (overwrites on conflict)
map.extend(vec.into_iter())               // From iterator of pairs

// --- Production patterns ---
// Word frequency counter
let counts: HashMap<&str, usize> = text.split_whitespace()
    .fold(HashMap::new(), |mut m, w| { *m.entry(w).or_insert(0) += 1; m });

// Group by key
let by_category: HashMap<Category, Vec<Item>> = items.into_iter()
    .fold(HashMap::new(), |mut m, item| {
        m.entry(item.category).or_default().push(item); m
    });

// Invert map
let inverted: HashMap<V, K> = map.into_iter().map(|(k, v)| (v, k)).collect();

// --- HashSet ---
HashSet::new()
HashSet::with_capacity(100)
HashSet::from([1, 2, 3])
let set: HashSet<_> = vec.into_iter().collect();

set.insert(value)                          // bool — true if new
set.remove(&value)                         // bool — true if existed
set.contains(&value)                       // bool
set.get(&value)                            // Option<&T>
set.len()                                  // Number of elements
set.is_empty()                             // bool
set.retain(|x| x.is_valid())             // Keep matching

// Set operations
set.intersection(&other)                   // Iterator of common elements
set.union(&other)                          // Iterator of all elements
set.difference(&other)                     // Iterator of elements in self but not other
set.symmetric_difference(&other)           // Iterator of elements in either but not both
set.is_subset(&other)                      // bool
set.is_superset(&other)                    // bool
set.is_disjoint(&other)                    // bool — no common elements

// Dedup a Vec using HashSet
let unique: Vec<_> = items.into_iter().collect::<HashSet<_>>().into_iter().collect();
// Order-preserving dedup:
let mut seen = HashSet::new();
items.retain(|x| seen.insert(x.clone()));
```

## BTreeMap, BTreeSet, VecDeque, BinaryHeap

```rust
use std::collections::{BTreeMap, BTreeSet, VecDeque, BinaryHeap};

// --- BTreeMap (sorted by key, O(log n) ops) ---
let mut btree = BTreeMap::new();
btree.insert(key, value);
btree.get(&key)                            // Option<&V>
btree.range(start..end)                    // Iterator over key range
btree.range(..&upper)                      // Iterator up to upper bound
btree.range(&lower..)                      // Iterator from lower bound
btree.first_key_value()                    // Option<(&K, &V)> — smallest key
btree.last_key_value()                     // Option<(&K, &V)> — largest key
btree.pop_first()                          // Option<(K, V)> — remove smallest
btree.pop_last()                           // Option<(K, V)> — remove largest
btree.entry(key).or_insert(default)       // Same Entry API as HashMap
btree.keys()                               // Iterator in sorted order
btree.values()                             // Iterator in key order
btree.split_off(&key)                     // Split at key → new BTreeMap

// When to use BTreeMap over HashMap:
// - Need sorted iteration
// - Need range queries
// - Key doesn't implement Hash
// - Need deterministic iteration order

// --- BTreeSet (sorted set) ---
let mut bset = BTreeSet::new();
bset.insert(value);
bset.contains(&value);
bset.range(1..=5)                          // Iterator over value range
bset.first()                               // Option<&T> — smallest
bset.last()                                // Option<&T> — largest
// Same set operations as HashSet (intersection, union, etc.)

// --- VecDeque (double-ended queue) ---
let mut deque = VecDeque::new();
let mut deque = VecDeque::with_capacity(100);

deque.push_back(item)                      // Add to back
deque.push_front(item)                     // Add to front
deque.pop_back()                           // Remove from back → Option<T>
deque.pop_front()                          // Remove from front → Option<T>
deque.front()                              // Peek front → Option<&T>
deque.back()                               // Peek back → Option<&T>
deque.get(idx)                             // Index access → Option<&T>
deque.len()                                // Number of elements
deque.is_empty()                           // bool
deque.contains(&value)                     // bool
deque.iter()                               // Front-to-back iterator
deque.drain(..)                            // Remove all → iterator
deque.rotate_left(n)                       // Rotate elements
deque.make_contiguous()                    // Rearrange into contiguous slice → &mut [T]
deque.as_slices()                          // (&[T], &[T]) — two internal slices

// When to use VecDeque:
// - FIFO queue (push_back + pop_front)
// - Sliding window / ring buffer
// - Need efficient insert/remove at both ends

// --- BinaryHeap (max-heap / priority queue) ---
let mut heap = BinaryHeap::new();
let mut heap = BinaryHeap::with_capacity(100);

heap.push(item)                            // Insert element
heap.peek()                                // Largest element → Option<&T>
heap.pop()                                 // Remove largest → Option<T>
heap.len()                                 // Number of elements
heap.is_empty()                            // bool
heap.into_sorted_vec()                     // Consume → sorted Vec (ascending)
heap.drain()                               // Remove all → iterator (arbitrary order)

// Min-heap using Reverse
use std::cmp::Reverse;
let mut min_heap = BinaryHeap::new();
min_heap.push(Reverse(5));
min_heap.push(Reverse(1));
let Reverse(smallest) = min_heap.pop().unwrap(); // 1

// Priority queue with custom priority
#[derive(Eq, PartialEq)]
struct Task { priority: u32, name: String }
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
```

## Option Methods

```rust
let opt: Option<String> = Some("hello".into());

// --- Transforming ---
opt.map(|s| s.len())                      // Option<usize> — Some(5)
opt.and_then(|s| s.parse::<i32>().ok())   // Option<i32> — flat-map
opt.filter(|s| s.len() > 3)              // None if predicate false
opt.or(Some("default".into()))            // Use alternative if None
opt.or_else(|| compute_default())         // Lazy alternative
opt.xor(other)                             // Some if exactly one is Some
opt.zip(other)                             // Option<(A, B)> — both must be Some
opt.unzip()                                // (Option<A>, Option<B>) from Option<(A,B)>
opt.flatten()                              // Option<Option<T>> → Option<T>

// --- Unwrapping ---
opt.unwrap()                               // T — panics if None
opt.unwrap_or("default".into())           // T — value or default
opt.unwrap_or_default()                    // T — value or Default::default()
opt.unwrap_or_else(|| compute())          // T — lazy default
opt.expect("should have value")           // T — panic with message
opt?                                       // Return None from function

// --- Inspecting ---
opt.is_some()                              // bool
opt.is_none()                              // bool
opt.is_some_and(|s| s.len() > 3)          // bool (stable 1.70+)
opt.is_none_or(|s| s.len() > 3)           // bool (stable 1.82+)
opt.inspect(|s| println!("{s}"))          // Peek at value (stable 1.76+)

// --- Converting ---
opt.as_ref()                               // Option<&T> — borrow inner
opt.as_mut()                               // Option<&mut T>
opt.as_deref()                             // Option<&str> for Option<String>
opt.as_deref_mut()                         // Option<&mut str>
opt.ok_or(Error::Missing)?               // Result<T, E> — None becomes Err
opt.ok_or_else(|| Error::new("missing"))? // Lazy error
opt.iter()                                 // Iterator of 0 or 1 elements
opt.into_iter()                            // Consuming iterator
opt.take()                                 // Take value, leave None (&mut self)
opt.replace(new_value)                     // Replace value → Option<T> (old)
opt.get_or_insert(default)                // Insert if None, return &mut T
opt.get_or_insert_with(|| compute())      // Lazy insert

// --- Combining ---
opt.and(other)                             // other if self is Some, else None
opt.or(other)                              // self if Some, else other

// --- Production patterns ---
// Chaining fallible lookups
let result = cache.get(&key)
    .or_else(|| db.find(&key).ok().as_ref())
    .map(|v| v.clone());

// Optional field extraction
let display_name = user.nickname
    .as_deref()
    .unwrap_or(&user.username);

// Convert Vec<Option<T>> to Vec<T> (remove Nones)
let values: Vec<T> = options.into_iter().flatten().collect();
```

## Result Methods

```rust
let result: Result<String, io::Error> = Ok("42".into());

// --- Transforming ---
result.map(|s| s.len())                   // Result<usize, E>
result.map_err(|e| MyError::from(e))      // Result<T, MyError>
result.and_then(|s| s.parse::<i32>().map_err(Into::into)) // Flat-map Ok
result.or_else(|e| recover(e))            // Try to recover from error
result.inspect(|v| tracing::info!(%v))    // Peek at Ok value (stable 1.76+)
result.inspect_err(|e| tracing::error!(%e)) // Peek at Err value

// --- Unwrapping ---
result.unwrap()                            // T — panics on Err
result.unwrap_err()                        // E — panics on Ok
result.unwrap_or("default".into())        // T — value or default
result.unwrap_or_default()                 // T — value or Default
result.unwrap_or_else(|e| fallback(e))    // T — lazy default
result.expect("should succeed")           // T — panic with message
result?                                    // Return Err from function

// --- Inspecting ---
result.is_ok()                             // bool
result.is_err()                            // bool
result.is_ok_and(|v| v.len() > 0)         // bool
result.is_err_and(|e| e.kind() == NotFound) // bool

// --- Converting ---
result.ok()                                // Option<T> — discards Err
result.err()                               // Option<E> — discards Ok
result.as_ref()                            // Result<&T, &E>
result.as_mut()                            // Result<&mut T, &mut E>
result.as_deref()                          // Result<&str, &E> for Result<String, E>
result.iter()                              // Iterator of 0 or 1 elements
result.into_iter()                         // Consuming iterator

// --- Collecting Results ---
// Collect stops at first Err (short-circuit)
let all: Result<Vec<i32>, _> = strings.iter()
    .map(|s| s.parse::<i32>())
    .collect();

// Partition into successes and failures
let (oks, errs): (Vec<_>, Vec<_>) = results.into_iter()
    .partition(Result::is_ok);
let oks: Vec<T> = oks.into_iter().map(Result::unwrap).collect();
let errs: Vec<E> = errs.into_iter().map(Result::unwrap_err).collect();

// Transpose: Option<Result<T, E>> ↔ Result<Option<T>, E>
let opt_result: Option<Result<i32, Error>> = Some(Ok(42));
let result_opt: Result<Option<i32>, Error> = opt_result.transpose(); // Ok(Some(42))
```

## Smart Pointers — Arc, Rc, Box, Cow

```rust
// --- Box<T> (heap allocation) ---
let boxed: Box<i32> = Box::new(42);
let large = Box::new([0u8; 1_000_000]);    // Put large data on heap
let trait_obj: Box<dyn Display> = Box::new(42); // Trait object
Box::into_inner(boxed)                     // Unwrap → T (stable 1.80+)
// Box auto-derefs: boxed.method() calls T::method()

// --- Rc<T> (single-threaded reference counting) ---
use std::rc::Rc;
let rc = Rc::new(vec![1, 2, 3]);
let clone = Rc::clone(&rc);               // Increment refcount (NOT deep clone)
Rc::strong_count(&rc)                      // Number of Rc references
Rc::weak_count(&rc)                        // Number of Weak references
Rc::try_unwrap(rc)                         // Result<T, Rc<T>> — unwrap if last ref
Rc::make_mut(&mut rc)                      // Clone inner if shared, return &mut T
Rc::downgrade(&rc)                         // Create Weak reference
// weak.upgrade() → Option<Rc<T>>

// --- Arc<T> (thread-safe reference counting) ---
use std::sync::Arc;
let arc = Arc::new(data);
let clone = Arc::clone(&arc);             // Atomic increment (NOT deep clone)
Arc::strong_count(&arc)                    // Number of Arc references
Arc::try_unwrap(arc)                       // Result<T, Arc<T>>
Arc::make_mut(&mut arc)                    // Clone inner if shared
Arc::downgrade(&arc)                       // Create Weak reference
// Arc::new(Mutex::new(value))  — shared mutable state pattern

// --- Cow<'a, T> (clone-on-write) ---
use std::borrow::Cow;

// Avoid cloning when modification isn't needed
fn process(input: &str) -> Cow<str> {
    if input.contains("bad") {
        Cow::Owned(input.replace("bad", "good"))  // Only allocates when needed
    } else {
        Cow::Borrowed(input)                        // Zero-cost borrow
    }
}

Cow::Borrowed(&data)                       // Wrap a reference
Cow::Owned(data)                           // Wrap owned data
cow.into_owned()                           // Force ownership (clones if borrowed)
cow.to_mut()                               // Get &mut T (clones if borrowed)
cow.is_borrowed()                          // bool
cow.is_owned()                             // bool
// Cow<str> auto-derefs to &str
// Cow<[T]> auto-derefs to &[T]

// --- Cell<T> / RefCell<T> (interior mutability) ---
use std::cell::{Cell, RefCell};

// Cell — for Copy types, no runtime borrow checking
let cell = Cell::new(42);
cell.get()                                 // Copy the value out
cell.set(99)                               // Replace value
cell.replace(100)                          // Replace and return old value
cell.take()                                // Take value, leave Default

// RefCell — for non-Copy types, runtime borrow checking
let refcell = RefCell::new(vec![1, 2, 3]);
let borrowed = refcell.borrow();           // Ref<Vec<i32>> — panics if mut borrowed
let mut_borrowed = refcell.borrow_mut();   // RefMut<Vec<i32>> — panics if borrowed
refcell.try_borrow()                       // Result<Ref<T>, BorrowError>
refcell.try_borrow_mut()                   // Result<RefMut<T>, BorrowMutError>
refcell.replace(new_value)                // Replace and return old
refcell.take()                             // Take value, leave Default
```

## Mutex, RwLock, Atomics

```rust
use std::sync::{Mutex, RwLock, Arc};
use std::sync::atomic::{AtomicBool, AtomicUsize, AtomicI64, Ordering};

// --- Mutex<T> ---
let mutex = Mutex::new(0);
let mut guard = mutex.lock().unwrap();     // MutexGuard<T> — blocks until available
*guard += 1;                                // Access through deref
drop(guard);                                // Release lock (also released when guard drops)
mutex.try_lock()                           // Result<MutexGuard, TryLockError> — non-blocking
mutex.is_poisoned()                        // true if a thread panicked while holding
mutex.into_inner().unwrap()                // Consume mutex → T
mutex.get_mut().unwrap()                   // &mut T — no locking needed when &mut self

// Shared mutex pattern
let counter = Arc::new(Mutex::new(0));
let counter_clone = Arc::clone(&counter);
tokio::spawn(async move {
    *counter_clone.lock().unwrap() += 1;
});

// --- RwLock<T> (multiple readers OR one writer) ---
let rwlock = RwLock::new(data);
let read_guard = rwlock.read().unwrap();   // RwLockReadGuard — multiple concurrent readers
let write_guard = rwlock.write().unwrap(); // RwLockWriteGuard — exclusive access
rwlock.try_read()                          // Non-blocking read
rwlock.try_write()                         // Non-blocking write

// --- parking_lot alternatives (faster, no poisoning) ---
// use parking_lot::{Mutex, RwLock};
// let mutex = Mutex::new(0);
// let guard = mutex.lock();               // No .unwrap() needed — no poisoning
// mutex.try_lock()                        // Option<MutexGuard>

// --- Atomic types (lock-free) ---
let flag = AtomicBool::new(false);
flag.store(true, Ordering::Release);       // Set value
flag.load(Ordering::Acquire)               // Get value → bool
flag.swap(false, Ordering::AcqRel)        // Swap and return old value
flag.compare_exchange(false, true, Ordering::AcqRel, Ordering::Relaxed)
    // CAS: if current == false, set to true → Result<bool, bool>

let counter = AtomicUsize::new(0);
counter.fetch_add(1, Ordering::Relaxed)    // Increment, return old value
counter.fetch_sub(1, Ordering::Relaxed)    // Decrement
counter.fetch_max(new, Ordering::Relaxed)  // Set to max(current, new)
counter.fetch_min(new, Ordering::Relaxed)  // Set to min(current, new)
counter.fetch_or(mask, Ordering::Relaxed)  // Bitwise OR
counter.fetch_and(mask, Ordering::Relaxed) // Bitwise AND

// Ordering guide:
// Relaxed  — no synchronization, just atomicity (counters, statistics)
// Acquire  — reads see writes before corresponding Release
// Release  — writes are visible to subsequent Acquire reads
// AcqRel   — both Acquire and Release (for read-modify-write)
// SeqCst   — total ordering (safest, default if unsure)

// Production pattern: shutdown flag
let shutdown = Arc::new(AtomicBool::new(false));
let shutdown_clone = Arc::clone(&shutdown);
ctrlc::set_handler(move || shutdown_clone.store(true, Ordering::Release)).unwrap();
while !shutdown.load(Ordering::Acquire) {
    do_work();
}
```

## std::io — Read, Write, BufRead

```rust
use std::io::{self, Read, Write, BufRead, BufReader, BufWriter, Seek, SeekFrom, Cursor};

// --- Read trait ---
reader.read(&mut buf)?                     // Read into buffer → usize bytes read
reader.read_exact(&mut buf)?              // Fill buffer exactly (error if EOF)
reader.read_to_end(&mut vec)?             // Read all to Vec<u8> → usize
reader.read_to_string(&mut string)?       // Read all to String → usize
reader.bytes()                             // Iterator<Item = Result<u8>>
reader.chain(other_reader)                // Concatenate two readers
reader.take(limit)                         // Read at most N bytes

// --- Write trait ---
writer.write(&buf)?                        // Write buffer → usize bytes written
writer.write_all(&buf)?                   // Write entire buffer (error if can't)
writer.flush()?                            // Flush buffered data
writeln!(writer, "line {}", n)?           // Write formatted line
write!(writer, "no newline")?             // Write formatted (no newline)

// --- BufRead trait ---
let reader = BufReader::new(file);
reader.lines()                             // Iterator<Item = Result<String>>
reader.read_line(&mut string)?            // Read one line → usize
reader.split(b'\n')                       // Split on byte delimiter

// --- BufWriter ---
let writer = BufWriter::new(file);
// Writes are buffered; call .flush() or let it drop

// --- BufReader + BufWriter with capacity ---
BufReader::with_capacity(64 * 1024, file)  // 64KB buffer
BufWriter::with_capacity(64 * 1024, file)  // 64KB buffer

// --- Cursor (in-memory reader/writer) ---
let cursor = Cursor::new(vec![0u8; 1024]);
let cursor = Cursor::new(b"hello world");
cursor.position()                          // Current read position
cursor.set_position(0)                     // Seek to position
cursor.into_inner()                        // Get inner buffer
cursor.get_ref()                           // Borrow inner buffer

// --- Seek ---
file.seek(SeekFrom::Start(0))?            // Seek to absolute position
file.seek(SeekFrom::Current(-10))?        // Seek relative to current
file.seek(SeekFrom::End(-100))?           // Seek relative to end
file.rewind()?                             // Seek to start (stable 1.55+)
file.stream_position()?                    // Current position without seeking

// --- stdin/stdout/stderr ---
let stdin = io::stdin();
let mut line = String::new();
stdin.read_line(&mut line)?;              // Read one line from stdin
stdin.lock()                               // BufRead over stdin (faster in loops)

let stdout = io::stdout();
let mut out = stdout.lock();              // Locked handle for performance
writeln!(out, "fast output")?;

io::stderr()                               // Stderr handle
io::copy(&mut reader, &mut writer)?       // Copy all bytes between streams → u64
```

## File System — std::fs

```rust
use std::fs::{self, File, OpenOptions};
use std::path::{Path, PathBuf};

// --- Simple read/write (entire file) ---
fs::read_to_string("file.txt")?           // → String
fs::read("file.bin")?                      // → Vec<u8>
fs::write("file.txt", "content")?         // Write &[u8] or &str
fs::write("file.bin", &bytes)?            // Write binary

// --- File handles ---
File::open("file.txt")?                    // Open for reading
File::create("file.txt")?                 // Create/truncate for writing
File::create_new("file.txt")?             // Create only if doesn't exist (stable 1.77+)
OpenOptions::new()
    .read(true)
    .write(true)
    .append(true)
    .create(true)
    .truncate(false)
    .open("file.txt")?

// --- Metadata ---
fs::metadata("file.txt")?                  // fs::Metadata
metadata.len()                             // File size in bytes
metadata.is_file()                         // bool
metadata.is_dir()                          // bool
metadata.is_symlink()                      // bool
metadata.modified()?                       // SystemTime
metadata.created()?                        // SystemTime (not all platforms)
metadata.permissions()                     // fs::Permissions
fs::symlink_metadata("link")?            // Metadata of symlink itself

// --- Directory operations ---
fs::create_dir("dir")?                     // Create single directory
fs::create_dir_all("path/to/dir")?        // mkdir -p
fs::remove_dir("empty_dir")?             // Remove empty directory
fs::remove_dir_all("dir")?               // rm -rf
fs::remove_file("file.txt")?             // Delete file
fs::rename("old", "new")?                // Move/rename
fs::copy("src", "dst")?                   // Copy file → u64 bytes copied
fs::hard_link("target", "link")?         // Create hard link
#[cfg(unix)]
std::os::unix::fs::symlink("target", "link")?

// --- Directory listing ---
for entry in fs::read_dir(".")? {
    let entry = entry?;                    // DirEntry
    entry.path()                           // PathBuf
    entry.file_name()                      // OsString
    entry.file_type()?                     // FileType
    entry.metadata()?                      // Metadata
}

// Recursive directory walk (use walkdir crate for production)
fn walk(dir: &Path) -> io::Result<Vec<PathBuf>> {
    let mut files = Vec::new();
    for entry in fs::read_dir(dir)? {
        let entry = entry?;
        let path = entry.path();
        if path.is_dir() {
            files.extend(walk(&path)?);
        } else {
            files.push(path);
        }
    }
    Ok(files)
}

// --- Path methods ---
Path::new("/home/user/file.tar.gz")
path.parent()                              // Option<&Path>
path.file_name()                           // Option<&OsStr>
path.file_stem()                           // Option<&OsStr> — "file.tar"
path.extension()                           // Option<&OsStr> — "gz"
path.exists()                              // bool
path.is_file()                             // bool
path.is_dir()                              // bool
path.is_absolute()                         // bool
path.is_relative()                         // bool
path.is_symlink()                          // bool
path.components()                          // Iterator of Component
path.ancestors()                           // Iterator: self, parent, grandparent...
path.strip_prefix("/home")?              // Relative path from prefix
path.starts_with("/home")                  // bool
path.ends_with("file.txt")                // bool
path.join("subdir")                        // PathBuf
path.with_file_name("other.txt")          // PathBuf
path.with_extension("md")                 // PathBuf
path.display()                             // Display impl for printing
path.to_str()                              // Option<&str>
path.to_string_lossy()                     // Cow<str>
path.canonicalize()?                       // Resolve symlinks → PathBuf

// PathBuf mutation
let mut buf = PathBuf::from("/home");
buf.push("user");                          // /home/user
buf.push("file.txt");                      // /home/user/file.txt
buf.pop();                                 // /home/user
buf.set_file_name("other.txt");           // /home/user/other.txt
buf.set_extension("md");                   // /home/user/other.md
```

## std::mem & std::ptr

```rust
use std::mem;

// --- std::mem ---
mem::size_of::<u64>()                      // 8 — size of type in bytes
mem::size_of_val(&value)                   // Size of value's type
mem::align_of::<u64>()                     // 8 — alignment of type
mem::swap(&mut a, &mut b)                 // Swap two values
mem::replace(&mut slot, new_value)        // Replace and return old
mem::take(&mut slot)                       // Take value, leave Default
mem::drop(value)                           // Explicit drop (rarely needed)
mem::forget(value)                         // Don't run destructor (unsafe territory)
mem::discriminant(&enum_val)               // Enum variant discriminant (for comparison)
mem::needs_drop::<T>()                     // true if T has a destructor
mem::zeroed::<T>()                         // All-zero bytes (UNSAFE — usually wrong)
mem::transmute::<A, B>(val)               // Reinterpret bits (VERY UNSAFE)

// MaybeUninit — safe uninitialized memory
use std::mem::MaybeUninit;
let mut uninit: MaybeUninit<u64> = MaybeUninit::uninit();
uninit.write(42);                          // Initialize
let value = unsafe { uninit.assume_init() }; // Assert initialized

// Production patterns
let old = mem::replace(&mut self.state, State::Processing);
// Process old state while self.state is already updated

let owned = mem::take(&mut self.buffer);
// Take buffer for processing, leave empty buffer in place

// --- std::ptr ---
use std::ptr;
ptr::null::<T>()                           // *const T null pointer
ptr::null_mut::<T>()                       // *mut T null pointer
ptr.is_null()                              // bool
unsafe { ptr::read(src) }                 // Read value from pointer
unsafe { ptr::write(dst, value) }         // Write value to pointer
unsafe { ptr::copy(src, dst, count) }     // memmove (handles overlap)
unsafe { ptr::copy_nonoverlapping(src, dst, count) } // memcpy (no overlap)
unsafe { ptr::write_bytes(dst, 0, count) } // memset

// Pointer arithmetic
unsafe { ptr.add(offset) }                // Advance by offset elements
unsafe { ptr.sub(offset) }                // Go back by offset elements
unsafe { ptr.offset(isize_offset) }       // Signed offset
ptr.wrapping_add(offset)                   // Won't panic on overflow (for comparisons)
ptr.cast::<U>()                            // Cast pointer type
```

## std::cmp & std::convert

```rust
use std::cmp::{self, Ordering, Reverse};

// --- Comparison ---
cmp::min(a, b)                             // Smaller of two values
cmp::max(a, b)                             // Larger of two values
cmp::min_by_key(a, b, |x| x.len())       // Min by derived key
cmp::max_by_key(a, b, |x| x.len())       // Max by derived key
cmp::min_by(a, b, |x, y| x.partial_cmp(y).unwrap()) // Custom comparator
value.clamp(min, max)                      // Clamp to range

// --- Ordering ---
a.cmp(&b)                                 // Ordering (Ord trait)
a.partial_cmp(&b)                          // Option<Ordering> (PartialOrd)
Ordering::Less | Ordering::Equal | Ordering::Greater
ordering.then(other)                       // Tiebreaker
ordering.then_with(|| fallback_cmp())     // Lazy tiebreaker
ordering.reverse()                         // Flip ordering

// Multi-field sort
items.sort_by(|a, b| {
    a.priority.cmp(&b.priority)
        .then(a.name.cmp(&b.name))
        .then(b.created.cmp(&a.created))  // Reverse created
});

// Reverse wrapper for reversed ordering
let mut sorted = vec![3, 1, 2];
sorted.sort_by_key(|&x| Reverse(x));     // [3, 2, 1]

// --- Type conversions ---
// From / Into (infallible)
let s: String = String::from("hello");     // From<&str>
let s: String = "hello".into();            // Into<String>
let n: i64 = i32_val.into();              // Widening is From

// TryFrom / TryInto (fallible)
let n: u8 = u8::try_from(256i32)?;        // Err — out of range
let n: u8 = large_int.try_into()?;        // Same via TryInto
let arr: [u8; 4] = slice.try_into()?;     // Slice → array

// AsRef / AsMut (cheap reference conversion)
fn read_file(path: impl AsRef<Path>) -> io::Result<String> {
    fs::read_to_string(path.as_ref())      // Accepts &str, String, Path, PathBuf
}

// Borrow (like AsRef but with hash/eq consistency guarantee)
use std::borrow::Borrow;
fn find<Q: ?Sized>(map: &HashMap<String, V>, key: &Q) -> Option<&V>
where String: Borrow<Q>, Q: Hash + Eq {
    map.get(key)                           // Can pass &str to look up String keys
}
```

## Threading — std::thread

```rust
use std::thread;
use std::time::Duration;

// --- Spawning threads ---
let handle = thread::spawn(|| {
    // Runs in new thread
    42
});
let result = handle.join().unwrap();       // Wait for thread → T

// With captured data (must be 'static + Send)
let data = Arc::new(vec![1, 2, 3]);
let data_clone = Arc::clone(&data);
let handle = thread::spawn(move || {
    data_clone.iter().sum::<i32>()
});

// Named threads (shows in debugger/tracing)
thread::Builder::new()
    .name("worker-1".into())
    .stack_size(4 * 1024 * 1024)           // 4MB stack
    .spawn(|| { /* ... */ })?;

// --- Thread utilities ---
thread::sleep(Duration::from_millis(100)); // Sleep current thread
thread::yield_now();                       // Hint to scheduler
thread::current().name()                   // Option<&str> — current thread name
thread::current().id()                     // ThreadId
thread::available_parallelism()?           // NonZeroUsize — logical CPU count

// --- Scoped threads (stable 1.63+) — borrow local data without Arc ---
let mut data = vec![1, 2, 3, 4, 5];
thread::scope(|s| {
    let (left, right) = data.split_at_mut(3);
    s.spawn(|| { left.iter_mut().for_each(|x| *x *= 2); });
    s.spawn(|| { right.iter_mut().for_each(|x| *x *= 3); });
});
// All scoped threads joined before scope exits
// data is now [2, 4, 6, 12, 15]

// --- Thread-local storage ---
thread_local! {
    static BUFFER: RefCell<Vec<u8>> = RefCell::new(Vec::with_capacity(4096));
}
BUFFER.with(|buf| {
    buf.borrow_mut().clear();
    buf.borrow_mut().extend_from_slice(&data);
});

// --- Barrier (synchronization point) ---
use std::sync::Barrier;
let barrier = Arc::new(Barrier::new(num_threads));
// In each thread:
barrier.wait(); // All threads block until all arrive
```

## Tokio Async Runtime

```rust
use tokio::time::{sleep, timeout, interval, Duration, Instant};
use tokio::sync::{mpsc, oneshot, broadcast, watch, Semaphore, Notify};
use tokio::task::{self, JoinHandle, JoinSet};

// --- Spawning tasks ---
let handle: JoinHandle<i32> = tokio::spawn(async {
    do_work().await;
    42
});
let result = handle.await?;               // Wait for task

// spawn_blocking — run CPU-heavy or sync code off the async runtime
let result = task::spawn_blocking(|| {
    heavy_computation()                     // Runs on blocking thread pool
}).await?;

// JoinSet — manage multiple concurrent tasks (stable tokio 1.21+)
let mut set = JoinSet::new();
for url in urls {
    set.spawn(async move { fetch(url).await });
}
while let Some(result) = set.join_next().await {
    let response = result??;
    process(response);
}

// --- Timing ---
sleep(Duration::from_secs(1)).await;      // Async sleep (NEVER use std::thread::sleep)

let result = timeout(Duration::from_secs(5), async_operation()).await;
match result {
    Ok(value) => { /* completed in time */ }
    Err(_) => { /* timed out */ }
}

let mut interval = interval(Duration::from_secs(10));
loop {
    interval.tick().await;                 // Fires every 10 seconds
    do_periodic_work().await;
}

let deadline = Instant::now() + Duration::from_secs(30);
tokio::time::sleep_until(deadline).await;

// --- Channels ---
// mpsc — bounded multi-producer single-consumer
let (tx, mut rx) = mpsc::channel::<Message>(100);
tx.send(msg).await?;                      // Blocks if full
tx.try_send(msg)?;                        // Non-blocking
rx.recv().await                            // Option<T> — None when all senders dropped

// oneshot — single value transfer
let (tx, rx) = oneshot::channel::<Response>();
tx.send(response).unwrap();               // Send (no .await)
let value = rx.await?;                    // Receive

// broadcast — all receivers get every message
let (tx, _) = broadcast::channel::<Event>(100);
let mut rx1 = tx.subscribe();
let mut rx2 = tx.subscribe();
tx.send(event)?;

// watch — latest-value channel
let (tx, mut rx) = watch::channel(initial_config);
tx.send(new_config)?;                     // Update value
tx.send_modify(|cfg| cfg.debug = true);  // Modify in place
rx.changed().await?;                      // Wait for change
let current = rx.borrow().clone();        // Read current value

// --- Synchronization ---
let semaphore = Arc::new(Semaphore::new(10)); // Max 10 concurrent
let permit = semaphore.acquire().await?;  // Wait for permit
drop(permit);                              // Release

let notify = Arc::new(Notify::new());
notify.notify_one();                       // Wake one waiter
notify.notify_waiters();                   // Wake all waiters
notify.notified().await;                   // Wait for notification

// --- select! — wait for first of multiple futures ---
tokio::select! {
    msg = rx.recv() => { handle_message(msg); }
    _ = sleep(Duration::from_secs(30)) => { handle_timeout(); }
    _ = shutdown.notified() => { return; }
}
// biased; prefix makes it check branches in order (useful for prioritization)

// --- I/O ---
use tokio::fs;
use tokio::io::{AsyncReadExt, AsyncWriteExt, AsyncBufReadExt, BufReader};

let content = fs::read_to_string("file.txt").await?;
fs::write("file.txt", content).await?;

let file = fs::File::open("file.txt").await?;
let reader = BufReader::new(file);
let mut lines = reader.lines();
while let Some(line) = lines.next_line().await? {
    process(line);
}

// --- TCP ---
use tokio::net::{TcpListener, TcpStream};
let listener = TcpListener::bind("0.0.0.0:8080").await?;
loop {
    let (stream, addr) = listener.accept().await?;
    tokio::spawn(async move { handle(stream).await });
}

let stream = TcpStream::connect("127.0.0.1:8080").await?;
let (reader, writer) = stream.into_split();
```

## serde & serde_json

```rust
use serde::{Serialize, Deserialize};

// --- JSON ---
use serde_json::{self, Value, json};

// Serialize
let json_string = serde_json::to_string(&value)?;            // Compact
let json_pretty = serde_json::to_string_pretty(&value)?;    // Indented
let json_bytes = serde_json::to_vec(&value)?;                // Vec<u8>
serde_json::to_writer(file, &value)?;                        // Write to io::Write

// Deserialize
let parsed: MyStruct = serde_json::from_str(&json_string)?;
let parsed: MyStruct = serde_json::from_slice(&bytes)?;
let parsed: MyStruct = serde_json::from_reader(reader)?;

// Dynamic JSON (serde_json::Value)
let v: Value = serde_json::from_str(raw_json)?;
v["key"]                                   // &Value (returns Null if missing)
v["key"].as_str()                          // Option<&str>
v["key"].as_i64()                          // Option<i64>
v["key"].as_f64()                          // Option<f64>
v["key"].as_bool()                         // Option<bool>
v["key"].as_array()                        // Option<&Vec<Value>>
v["key"].as_object()                       // Option<&Map<String, Value>>
v["key"].is_null()                         // bool
v.get("key")                               // Option<&Value>
v.pointer("/nested/path/0")               // JSON pointer → Option<&Value>

// json! macro for constructing Value
let payload = json!({
    "name": name,
    "age": 30,
    "tags": ["rust", "serde"],
    "address": null,
});

// --- Common serde attributes ---
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]         // Field naming convention
#[serde(deny_unknown_fields)]              // Strict deserialization
struct Config {
    #[serde(default)]                      // Use Default if missing
    debug_mode: bool,
    #[serde(rename = "type")]              // Rename this field
    kind: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    description: Option<String>,
    #[serde(skip)]                         // Never serialize/deserialize
    internal_cache: Vec<u8>,
    #[serde(alias = "colour")]             // Accept alternative name
    color: String,
    #[serde(flatten)]                      // Inline nested struct fields
    metadata: Metadata,
    #[serde(with = "chrono::serde::ts_seconds")] // Custom ser/de module
    created_at: DateTime<Utc>,
    #[serde(default = "default_port")]     // Custom default function
    port: u16,
    #[serde(deserialize_with = "deserialize_bool_from_int")]
    enabled: bool,
}

// --- TOML ---
// let config: Config = toml::from_str(&content)?;
// let toml_string = toml::to_string_pretty(&config)?;

// --- Other formats ---
// bincode::serialize(&value)?             // Binary, compact
// bincode::deserialize::<T>(&bytes)?
// csv::ReaderBuilder::new().from_reader(reader)
```

## Formatting — format!, write!, Display, Debug

```rust
// --- Positional & named ---
format!("{} {}", a, b)                     // Sequential
format!("{0} {1} {0}", a, b)              // Positional reuse
format!("{name}: {val}", name = "x", val = 42) // Named
format!("{a} + {b} = {}", a + b, a = 1, b = 2) // Mixed

// --- Width & alignment ---
format!("{:>10}", "right")                 // "     right" — right-align
format!("{:<10}", "left")                  // "left      " — left-align
format!("{:^10}", "center")                // "  center  " — center
format!("{:_>10}", "pad")                  // "_______pad" — custom fill char
format!("{:10}", 42)                       // "        42" — numbers right by default

// --- Numbers ---
format!("{:05}", 42)                       // "00042" — zero-padded
format!("{:+}", 42)                        // "+42" — sign
format!("{:+}", -42)                       // "-42" — sign
format!("{:e}", 1234.0)                    // "1.234e3" — scientific
format!("{:E}", 1234.0)                    // "1.234E3" — uppercase scientific

// --- Floats ---
format!("{:.2}", 3.14159)                  // "3.14" — precision
format!("{:8.2}", 3.14159)                // "    3.14" — width + precision
format!("{:08.2}", 3.14159)               // "00003.14" — zero-padded float

// --- Hex, octal, binary ---
format!("{:x}", 255)                       // "ff"
format!("{:X}", 255)                       // "FF"
format!("{:#x}", 255)                      // "0xff"
format!("{:#010x}", 255)                   // "0x000000ff"
format!("{:o}", 255)                       // "377"
format!("{:b}", 255)                       // "11111111"
format!("{:#010b}", 42)                    // "0b00101010"
format!("{:p}", &value)                    // "0x7ffd5e8e5a4c" — pointer

// --- Debug & Display ---
format!("{:?}", value)                     // Debug format
format!("{:#?}", value)                    // Pretty-printed Debug
format!("{}", value)                       // Display format

// --- Implementing Display ---
use std::fmt;
impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

// --- Implementing Debug (custom) ---
impl fmt::Debug for SecretKey {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "SecretKey(***)")         // Redact sensitive data
    }
}

// --- write! to String ---
use std::fmt::Write;
let mut buf = String::new();
write!(buf, "count: {}", n)?;
writeln!(buf, " ({}%)", pct)?;
```

## Environment, Process, Networking

```rust
use std::env;
use std::process::{Command, Stdio};
use std::net::{TcpListener, TcpStream, SocketAddr, IpAddr, Ipv4Addr};

// --- Environment ---
env::var("HOME")                           // Result<String, VarError>
env::var("PORT").unwrap_or_else(|_| "8080".into())
env::var_os("PATH")                        // Option<OsString> — no UTF-8 requirement
env::vars()                                // Iterator<Item = (String, String)>
env::args()                                // Iterator<Item = String>
env::args_os()                             // Iterator<Item = OsString>
env::current_dir()?                        // PathBuf
env::current_exe()?                        // PathBuf of running binary
env::temp_dir()                            // System temp directory
env::consts::OS                            // "linux", "macos", "windows"
env::consts::ARCH                          // "x86_64", "aarch64"

// --- Process / Command ---
let output = Command::new("git")
    .args(["log", "--oneline", "-5"])
    .current_dir("/repo")
    .env("GIT_PAGER", "cat")
    .env_clear()                           // Clear all env vars
    .env_remove("DEBUG")                   // Remove specific var
    .stdin(Stdio::null())                  // Redirect stdin
    .stdout(Stdio::piped())                // Capture stdout
    .stderr(Stdio::piped())                // Capture stderr
    .output()?;                            // Run and capture output

output.status.success()                    // bool
output.status.code()                       // Option<i32>
String::from_utf8_lossy(&output.stdout)   // stdout as string
String::from_utf8_lossy(&output.stderr)   // stderr as string

// Status only (inherits stdio)
let status = Command::new("make").status()?;

// Spawn and interact
let mut child = Command::new("cat")
    .stdin(Stdio::piped())
    .stdout(Stdio::piped())
    .spawn()?;
child.stdin.as_mut().unwrap().write_all(b"hello")?;
let output = child.wait_with_output()?;

// --- Networking (std) ---
// TCP server
let listener = TcpListener::bind("0.0.0.0:8080")?;
for stream in listener.incoming() {
    let stream = stream?;
    handle_connection(stream);
}

// TCP client
let mut stream = TcpStream::connect("127.0.0.1:8080")?;
stream.set_read_timeout(Some(Duration::from_secs(5)))?;
stream.set_write_timeout(Some(Duration::from_secs(5)))?;
stream.set_nodelay(true)?;                // Disable Nagle's algorithm

// Address parsing
let addr: SocketAddr = "127.0.0.1:8080".parse()?;
let addr = SocketAddr::new(IpAddr::V4(Ipv4Addr::LOCALHOST), 8080);
```

## Time & Duration

```rust
use std::time::{Duration, Instant, SystemTime, UNIX_EPOCH};

// --- Duration ---
Duration::from_secs(60)
Duration::from_millis(500)
Duration::from_micros(100)
Duration::from_nanos(1000)
Duration::from_secs_f64(1.5)              // 1.5 seconds
Duration::ZERO                             // Zero duration
Duration::MAX                              // Maximum duration

dur.as_secs()                              // u64 — whole seconds
dur.as_millis()                            // u128 — whole milliseconds
dur.as_micros()                            // u128
dur.as_nanos()                             // u128
dur.as_secs_f64()                          // f64 — fractional seconds
dur.subsec_millis()                        // u32 — fractional part as ms
dur.subsec_nanos()                         // u32 — fractional part as ns
dur.is_zero()                              // bool
dur.checked_add(other)                     // Option<Duration>
dur.saturating_add(other)                  // Duration (caps at MAX)
dur.checked_sub(other)                     // Option<Duration>
dur.mul_f64(1.5)                           // Scale duration
dur.div_f64(2.0)                           // Scale duration

// --- Instant (monotonic clock — for measuring elapsed time) ---
let start = Instant::now();
do_work();
let elapsed: Duration = start.elapsed();
println!("Took {elapsed:?}");             // "Took 1.234567s"
println!("Took {elapsed:.2?}");           // "Took 1.23s"
start.checked_duration_since(earlier)      // Option<Duration>
Instant::now().duration_since(start)       // Duration (panics if backwards)

// --- SystemTime (wall clock — for timestamps) ---
let now = SystemTime::now();
let ts = now.duration_since(UNIX_EPOCH)?.as_secs(); // Unix timestamp
SystemTime::UNIX_EPOCH + Duration::from_secs(ts)    // From timestamp

// For real date/time formatting, use chrono or time crate
// use chrono::{Utc, Local, DateTime, NaiveDate, NaiveDateTime};
// let now: DateTime<Utc> = Utc::now();
// now.format("%Y-%m-%d %H:%M:%S").to_string()
// NaiveDate::from_ymd_opt(2024, 6, 15).unwrap()
// let tomorrow = now + chrono::Duration::days(1);
```

## Regular Expressions (regex crate)

```rust
use regex::Regex;
use std::sync::LazyLock;

// Always compile once and reuse (regex compilation is expensive)
static RE: LazyLock<Regex> = LazyLock::new(|| {
    Regex::new(r"pattern").unwrap()
});

let re = Regex::new(r"^\d{4}-\d{2}-\d{2}$")?;

// --- Matching ---
re.is_match("2024-01-15")                  // bool
re.find("text 2024-01-15 here")           // Option<Match> — first match
re.find_iter("a=1 b=2 c=3")              // Iterator<Item = Match>

match_obj.as_str()                         // &str — matched text
match_obj.start()                          // usize — start byte offset
match_obj.end()                            // usize — end byte offset
match_obj.range()                          // Range<usize>

// --- Captures ---
let re = Regex::new(r"(\d{4})-(\d{2})-(\d{2})")?;
if let Some(caps) = re.captures("Date: 2024-01-15") {
    &caps[0]                                // "2024-01-15" — full match
    &caps[1]                                // "2024" — group 1
    caps.get(2).map(|m| m.as_str())       // Some("01") — optional access
}

// Named captures
let re = Regex::new(r"(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})")?;
if let Some(caps) = re.captures(text) {
    &caps["year"]                          // "2024"
}

// All capture groups
for caps in re.captures_iter(text) {
    println!("{}", &caps[0]);
}

// --- Replacing ---
re.replace("input", "replacement")        // Cow<str> — first match
re.replace_all("input", "replacement")    // Cow<str> — all matches
re.replace_all("2024-01-15", "$3/$2/$1") // Backreference: "15/01/2024"
re.replacen("a a a", 2, "b")             // Replace first N: "b b a"

// Dynamic replacement
re.replace_all(text, |caps: &regex::Captures| {
    format!("[{}]", &caps[0].to_uppercase())
});

// --- RegexSet (match against multiple patterns) ---
use regex::RegexSet;
let set = RegexSet::new(&[r"\d+", r"[a-z]+", r"[A-Z]+"])?;
let matches: Vec<_> = set.matches("Hello123").into_iter().collect();
// matches = [0, 1, 2] — indices of matching patterns
set.is_match("Hello123")                   // true if any pattern matches
```

## Common Crate Patterns

```rust
// --- reqwest (HTTP client) ---
let client = reqwest::Client::new();       // Reuse client for connection pooling
let client = reqwest::Client::builder()
    .timeout(Duration::from_secs(30))
    .default_headers(headers)
    .user_agent("myapp/1.0")
    .build()?;

let resp = client.get("https://api.example.com/data")
    .header("Authorization", format!("Bearer {token}"))
    .query(&[("page", "1"), ("limit", "10")])
    .send().await?
    .error_for_status()?;                  // Err on 4xx/5xx

resp.status()                              // StatusCode
resp.headers()                             // &HeaderMap
resp.text().await?                         // String body
resp.json::<T>().await?                    // Deserialize JSON body
resp.bytes().await?                        // Bytes body

// POST with JSON body
client.post(url)
    .json(&payload)                        // Serialize as JSON
    .send().await?;

// POST with form body
client.post(url)
    .form(&[("key", "value")])
    .send().await?;

// --- tracing ---
use tracing::{info, warn, error, debug, trace, instrument, span, Level};

info!("server started on port {port}");
warn!(retry_count = 3, "request failed");
error!(?err, "database connection lost");  // ?err uses Debug format
debug!(%user_id, "processing request");    // %user_id uses Display format

#[instrument(skip(db), fields(user_id))]   // Auto-create span
async fn get_user(db: &Pool, id: UserId) -> Result<User> {
    tracing::Span::current().record("user_id", id.0);
    // ...
}

let span = span!(Level::INFO, "processing", batch_id = %id);
let _guard = span.enter();                // Sync span entry
// async: use .instrument(span) instead

// Subscriber setup
use tracing_subscriber::{fmt, EnvFilter};
tracing_subscriber::fmt()
    .with_env_filter(EnvFilter::from_default_env()) // RUST_LOG
    .json()                                // JSON output
    .init();

// --- clap (CLI parsing) ---
use clap::Parser;
#[derive(Parser)]
#[command(name = "myapp", about = "Description")]
struct Cli {
    #[arg(short, long)]
    verbose: bool,
    #[arg(short, long, default_value = "8080")]
    port: u16,
    #[arg(value_name = "FILE")]
    input: PathBuf,
    #[command(subcommand)]
    command: Option<Commands>,
}

#[derive(clap::Subcommand)]
enum Commands {
    Init { name: String },
    Run { #[arg(long)] release: bool },
}

let cli = Cli::parse();

// --- uuid ---
use uuid::Uuid;
Uuid::new_v4()                             // Random UUID
Uuid::new_v7(uuid::Timestamp::now(uuid::NoContext)) // Time-ordered (sortable)
Uuid::parse_str("550e8400-...")?          // Parse from string
uuid.to_string()                           // "550e8400-..."
uuid.as_bytes()                            // &[u8; 16]
uuid.is_nil()                              // true if all zeros

// --- chrono ---
use chrono::{Utc, Local, NaiveDate, DateTime, Duration as CDuration};
Utc::now()                                 // DateTime<Utc>
Local::now()                               // DateTime<Local>
dt.format("%Y-%m-%d %H:%M:%S").to_string()
dt.timestamp()                             // i64 Unix timestamp
dt.timestamp_millis()                      // i64
NaiveDate::from_ymd_opt(2024, 6, 15)     // Option<NaiveDate>
dt + CDuration::days(1)                    // Add duration
DateTime::parse_from_rfc3339("2024-01-15T10:30:00Z")? // Parse ISO 8601

// --- rand ---
use rand::Rng;
let mut rng = rand::thread_rng();
rng.gen::<bool>()                          // Random bool
rng.gen::<f64>()                           // Random f64 in [0, 1)
rng.gen_range(1..=100)                    // Random i32 in range
rng.gen_range(0.0..1.0)                   // Random f64 in range

use rand::seq::SliceRandom;
items.shuffle(&mut rng);                   // Shuffle in place
items.choose(&mut rng)                     // Random element → Option<&T>
items.choose_multiple(&mut rng, 3)        // Random N elements → iterator

// --- base64 ---
use base64::{Engine, engine::general_purpose::{STANDARD, URL_SAFE}};
STANDARD.encode(b"hello")                 // "aGVsbG8="
STANDARD.decode("aGVsbG8=")?             // Vec<u8>
URL_SAFE.encode(b"hello")                 // URL-safe variant

// --- anyhow (application error handling) ---
use anyhow::{Context, Result, bail, ensure, anyhow};
fn process() -> Result<()> {              // anyhow::Result
    let f = File::open("cfg.toml").context("opening config")?;
    ensure!(count > 0, "count must be positive, got {count}");
    bail!("unsupported format: {fmt}");   // Return Err immediately
    Err(anyhow!("custom error: {detail}"))
}

// --- thiserror (library error types) ---
use thiserror::Error;
#[derive(Debug, Error)]
enum AppError {
    #[error("not found: {entity} #{id}")]
    NotFound { entity: &'static str, id: u64 },
    #[error("database error")]
    Database(#[from] sqlx::Error),        // Auto From impl
    #[error("validation failed: {0}")]
    Validation(String),
    #[error(transparent)]                  // Delegate Display to inner
    Other(#[from] anyhow::Error),
}
```

## Common Trait Implementations

```rust
use std::fmt;

// Display — user-facing output (used by {} and .to_string())
impl fmt::Display for Money {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "${:.2}", self.cents as f64 / 100.0)
    }
}

// FromStr — parsing from strings (used by .parse())
impl std::str::FromStr for Color {
    type Err = ParseColorError;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        match s.to_lowercase().as_str() {
            "red" => Ok(Color::Red),
            "blue" => Ok(Color::Blue),
            _ => Err(ParseColorError(s.to_string())),
        }
    }
}
// let color: Color = "red".parse()?;

// From / Into — type conversions
impl From<i32> for Temperature {
    fn from(celsius: i32) -> Self {
        Temperature { celsius }
    }
}
// let temp: Temperature = 100.into();
// let temp = Temperature::from(100);

// AsRef — cheap reference conversions
impl AsRef<str> for Username {
    fn as_ref(&self) -> &str { &self.0 }
}
// fn greet(name: impl AsRef<str>) { println!("Hi, {}!", name.as_ref()); }

// Deref — smart pointer / newtype delegation
impl std::ops::Deref for Username {
    type Target = str;
    fn deref(&self) -> &str { &self.0 }
}
// username.len() works — delegates to str::len()

// Index — custom indexing
impl std::ops::Index<usize> for Grid {
    type Output = Cell;
    fn index(&self, idx: usize) -> &Cell { &self.cells[idx] }
}

// Iterator — make types iterable
impl IntoIterator for Grid {
    type Item = Cell;
    type IntoIter = std::vec::IntoIter<Cell>;
    fn into_iter(self) -> Self::IntoIter { self.cells.into_iter() }
}

// Drop — custom cleanup
impl Drop for TempFile {
    fn drop(&mut self) {
        let _ = std::fs::remove_file(&self.path);
    }
}
```

## Macros Quick Reference

```rust
// --- Standard library macros ---
vec![1, 2, 3]                              // Vec from values
vec![0u8; 1024]                            // Vec of N copies
format!("hello {name}")                    // String formatting
println!() / eprintln!()                   // stdout / stderr
write!(buf, "{}", x)? / writeln!()?       // Write to impl Write/fmt::Write
dbg!(&value)                               // Debug print with file:line → &T
dbg!(a + b)                                // Prints expression + value → T
todo!("implement this")                   // Panic with TODO message
unimplemented!()                           // Panic — intentionally not implemented
unreachable!()                             // Panic — should never reach here
panic!("message")                          // Explicit panic
assert!(cond) / assert_eq!(a, b)          // Assertion (always)
debug_assert!(cond)                        // Assertion (debug builds only)
matches!(val, Pattern)                     // Bool pattern match
cfg!(target_os = "linux")                  // Runtime conditional
env!("CARGO_PKG_VERSION")                 // Compile-time env var (fails if missing)
option_env!("DB_URL")                      // Compile-time env var → Option<&str>
include_str!("file.txt")                  // Embed file as &str at compile time
include_bytes!("data.bin")                // Embed file as &[u8] at compile time
concat!("a", "b", "c")                    // Compile-time string concatenation
stringify!(expr)                           // Expression → &str at compile time
file!() / line!() / column!()            // Source location
module_path!()                             // Current module path
type_name::<T>()                           // Type name as &str (std::any)

// --- cfg / conditional compilation ---
#[cfg(test)]                               // Test-only code
#[cfg(target_os = "linux")]               // OS-specific
#[cfg(feature = "serde")]                 // Feature-gated
#[cfg(debug_assertions)]                  // Debug builds only
#[cfg(not(feature = "std"))]              // Negation
#[cfg(any(target_os = "linux", target_os = "macos"))]
#[cfg(all(feature = "a", feature = "b"))]

// --- Common derive macros ---
#[derive(Debug)]                           // {:?} format
#[derive(Clone, Copy)]                    // Value semantics
#[derive(PartialEq, Eq)]                 // Equality
#[derive(PartialOrd, Ord)]               // Ordering
#[derive(Hash)]                            // HashMap/HashSet key
#[derive(Default)]                         // Default::default()
#[derive(Serialize, Deserialize)]         // serde
#[derive(clap::Parser)]                   // CLI parsing
#[derive(thiserror::Error)]              // Error types

// --- Attribute macros ---
#[allow(unused)]                           // Suppress warning
#[must_use]                                // Warn if return value ignored
#[inline]                                  // Hint to inline
#[inline(always)]                          // Force inline
#[cold]                                    // Unlikely code path
#[non_exhaustive]                          // Prevent external exhaustive matching
#[deprecated(since = "1.0", note = "use X instead")]
#[doc(hidden)]                             // Hide from docs
#[repr(C)]                                // C-compatible layout
#[repr(transparent)]                      // Same layout as single field
#[repr(u8)]                                // Enum discriminant type
```

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: ownership, traits, error handling, iterators, serde, async
- **[language-patterns.md](language-patterns.md)** — Everyday idioms: pattern matching, closures, RAII, conversions
- **[type-system.md](type-system.md)** — Trait patterns, type state, GATs, const generics, Pin/Unpin
