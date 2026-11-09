# Collection and Iterator Costs

## Description

What each common standard-library operation costs, so you can choose the right collection and spot hidden quadratic behavior. The standard library documents complexity per method; this guide collects the numbers that matter most and the places where the cost is easy to miss. Costs are asymptotic: constants and cache behavior can reverse the ranking for small or ordered data.

Notation: `n` = number of elements, `k` = keys/items touched, `c` = capacity, `B` = branching factor.

## When to use

- Choosing between `Vec`, `VecDeque`, `HashMap`, `BTreeMap`, `BinaryHeap`, or `LinkedList`.
- A loop contains `contains`, `position`, `remove(0)`, `insert(0, _)`, or `nth`.
- A collection is rebuilt when an in-place method would do.
- Iterating a map and expecting order.
- Deciding whether to sort once or use a heap for top-k.

## Operation costs

| Operation | Cost | Notes |
|---|---|---|
| `Vec` index / push / pop | `O(1)` (push amortized) | Growth copies on reallocation |
| `Vec::insert(i, _)` / `remove(i)` | `O(n)` | Shifts all later elements |
| `Vec::swap_remove(i)` | `O(1)` | Does not preserve order |
| `Vec::contains` / `iter().position` | `O(n)` | Linear scan |
| `Vec::retain` | `O(n)` | In-place, preserves order |
| `Vec::sort` / `sort_unstable` | `O(n log n)` | `sort_unstable` is usually faster, no allocation |
| `Vec::binary_search` | `O(log n)` | Requires sorted data |
| `VecDeque` push/pop both ends | `O(1)` amortized | Use for queues |
| `VecDeque` index | `O(1)` | |
| `VecDeque` insert/remove middle | `O(n)` | |
| `HashMap`/`HashSet` get/insert/remove | `O(1)` expected, `O(n)` worst | Hashing cost dominates for short keys |
| `BTreeMap`/`BTreeSet` get/insert/remove | `O(log n)` | Ordered; cache-friendly for small/ordered data |
| `BTreeMap` range | `O(log n + k)` | Output size matters |
| `BinaryHeap` push/pop | `O(log n)` | `peek` is `O(1)` |
| `BinaryHeap::from(vec)` | `O(n)` | Heapify in place |
| `LinkedList` push/pop ends | `O(1)` | Index/get `O(n)`; poor locality |
| `String::push_str` | Amortized `O(len)` | Reallocates like `Vec` |
| `str::chars().nth(i)` | `O(i)` | UTF-8 is variable width |
| `str::to_lowercase` / `to_uppercase` | `O(n)` + allocation | |
| `slice::sort` on a linked structure | — | Not applicable; collect into a `Vec` first |

Iterators:

| Adapter | Cost | Notes |
|---|---|---|
| `map`, `filter`, `filter_map`, `zip` | `O(n)` | Lazy; usually optimized into one loop |
| `chain` | `O(n)` | Can be slower than a single iterator in hot loops |
| `skip` | `O(k)` | |
| `step_by` | `O(n)` | |
| `nth(k)` | `O(k)` on non-random-access iterators | `O(1)` for slices/`Vec` |
| `rev` | `O(1)` setup | Requires a double-ended iterator |
| `collect::<Vec<_>>` | `O(n)` + allocation | Avoid if only iterating again |
| `extend` | `O(n)` + growth | Reuses existing capacity |
| `sum`, `product`, `count` | `O(n)` | Consumes the iterator |
| `windows(k)`, `chunks(k)` | `O(n)` | `chunks_exact` is faster |

## Code examples

### `contains` in a loop: `O(n²)`

```rust
// SLOW: linear scan inside a loop
let mut found = Vec::new();
for id in ids {
    if ids.contains(&id) && !found.contains(&id) {
        found.push(id);
    }
}
```

```rust
// FAST: hash membership
use std::collections::HashSet;

let unique: HashSet<_> = ids.iter().copied().collect();
let found: Vec<_> = ids.iter().filter(|id| unique.contains(id)).copied().collect();
```

`O(n)` expected instead of `O(n²)`. If the data is sorted, `binary_search` or a two-pointer scan avoids hashing.

### `Vec::remove(0)` as a queue: `O(n)` per pop

```rust
// SLOW: shifts every element on each pop
let mut queue = Vec::new();
while !queue.is_empty() {
    let item = queue.remove(0);
    process(item);
}
```

```rust
// FAST: VecDeque is O(1) at both ends
use std::collections::VecDeque;

let mut queue = VecDeque::new();
while let Some(item) = queue.pop_front() {
    process(item);
}
```

### `LinkedList` vs `Vec`

```rust
// SLOW for almost every workload: pointer chasing, allocation per node
let mut list = std::collections::LinkedList::new();
list.push_back(1);
let first = *list.front().unwrap();
```

```rust
// FAST: contiguous storage, better locality, less allocation
let mut vec = Vec::new();
vec.push(1);
let first = vec[0];
```

`LinkedList` is rarely the right choice, even when inserting in the middle: finding the position is `O(n)` and the cache behavior is poor. Prefer `Vec` or `VecDeque`.

### HashMap vs BTreeMap vs sorted Vec

```rust
// Small, ordered, read-mostly data: a sorted Vec + binary search is often fastest
let mut entries: Vec<(u32, String)> = load();
entries.sort_unstable_by_key(|(k, _)| *k);

match entries.binary_search_by_key(&key, |(k, _)| *k) {
    Ok(i) => Some(&entries[i].1),
    Err(_) => None,
}
```

- `HashMap`: expected `O(1)`, no order, hashing cost.
- `BTreeMap`: `O(log n)`, ordered iteration and ranges, good cache behavior for small/ordered data.
- Sorted `Vec`: `O(log n)` lookup, `O(n)` insertion; excellent for static/small data.

### Top-k: heap instead of full sort

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

// O(n log k): keep only k elements
let mut heap = BinaryHeap::with_capacity(k + 1);
for item in items {
    heap.push(Reverse(item.cost));
    if heap.len() > k {
        heap.pop();
    }
}
```

A full sort is `O(n log n)`; when `k << n`, the bounded heap is better.

### k-th element without full sort

```rust
// O(n) average instead of O(n log n)
let (_, kth, _) = data.select_nth_unstable(k);
```

### `chars().nth(i)` in a loop

```rust
// SLOW: O(n) per call inside the loop
for i in 0..s.len() {
    let c = s.chars().nth(i);
    // ...
}
```

```rust
// FAST: iterate once
for c in s.chars() {
    // ...
}
```

If the data is known ASCII, work with `as_bytes()` and index bytes, after validating that assumption.

## Anti-patterns

1. `Vec::contains` or `iter().position` inside a loop over the same data.
2. `Vec::remove(0)`/`insert(0, _)` as a queue or stack substitute.
3. `LinkedList` for anything that needs indexing or iteration performance.
4. Hashing for a handful of entries where a linear scan is faster.
5. Re-sorting inside a loop instead of sorting once.
6. Full sort when only top-k or the k-th element is needed.
7. `collect` into a `Vec` just to take `.len()` or iterate again.
8. Indexing by character (`chars().nth`) in a loop.
9. Assuming `HashMap` is always faster than `BTreeMap`; for small or ordered data it often is not.
10. Ignoring cache behavior: two `O(n)` algorithms can differ by an order of magnitude.

## Checklist

- Is the collection matched to the access pattern (index, ends, lookup, order, priority)?
- Is membership tested with a set/map rather than a linear scan?
- Is the queue a `VecDeque`, not a `Vec` with `remove(0)`?
- Are inserts/removes in the middle of a `Vec` deliberate and rare?
- Is the data sorted once, not per query?
- Is top-k using a bounded heap and k-th using `select_nth_unstable`?
- Are string operations byte-oriented where the data allows it?
- Did you check the standard library's documented complexity for the exact method?
- Did you measure, since constants and cache effects can beat the asymptotics?

## Related

- `iterators.md` for iterator-specific costs and bounds checks.
- `hashing.md` for map lookup costs and hasher choice.
- `std.md` for cheaper standard-library methods.
- `heap.md` for allocation costs of collections.
- The `code-complexity-analysis` skill for a deeper asymptotic analysis.
