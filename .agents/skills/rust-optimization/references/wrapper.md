# Wrapper Types

## Description

`RefCell`, `Cell`, `Mutex`, `RwLock`, `Rc`, `Arc`, and similar wrappers provide runtime behavior for a value, and that behavior costs something: borrow checks, refcount increments, lock/unlock, atomics, and poisoning. When several wrapped values are usually accessed together, one wrapper around a tuple can be cheaper than several wrappers; when their access patterns are independent, merging increases contention.

## When to use

- A profile shows time in `RefCell::borrow`, `Mutex::lock`, atomic operations, or refcount updates.
- A hot struct has multiple `Arc<Mutex<T>>` fields read together.
- A wrapper exists only to satisfy the borrow checker, not for real sharing.
- Graphs or trees use `Rc<RefCell<Node>>` and suffer from pointer chasing and runtime panics.
- Deciding between `Rc` and `Arc`, or between `Mutex` and `RwLock`.

## Costs

| Wrapper | What it does per access | Cost |
|---|---|---|
| `RefCell<T>` | Runtime borrow flag check | A branch and a counter; panics on conflict |
| `Cell<T>` | Copy in/out | Cheap for `Copy`; no borrows |
| `Mutex<T>` | Lock/unlock, may park | Atomic + possible syscall under contention |
| `RwLock<T>` | Read/write lock | Atomics; writer starvation possible |
| `Rc<T>` | Non-atomic refcount | Increment/decrement, not thread-safe |
| `Arc<T>` | Atomic refcount | Atomic RMW per clone/drop |
| `OnceLock`/`LazyLock` | One-time init check | Cheap after initialization |

## Checks

```bash
perf record -g ./target/release/app | rg 'RefCell|Mutex|futex|atomic'
cargo asm --rust my_crate::hot_fn | rg 'lock|cmpxchg|xadd'
```

Probes:

```bash
rg -n 'RefCell|Cell<' src
rg -n 'Arc<Mutex|Rc<RefCell|Arc<RwLock' src
rg -n 'Mutex<|RwLock<' src
rg -n 'borrow\(\)|borrow_mut\(\)' src
rg -n '\.clone\(\)' src | rg -i 'arc|rc'
```

## Code examples

### Multiple wrappers accessed together

```rust
use std::sync::{Arc, Mutex};

// SLOW: two locks, two refcounts, two critical sections for one logical update
struct State {
    x: Arc<Mutex<u32>>,
    y: Arc<Mutex<u32>>,
}
```

```rust
// FAST: one lock protects the values that are used together
struct State {
    xy: Arc<Mutex<(u32, u32)>>,
}
```

Only merge values that are actually accessed together. If `x` is read by one thread and `y` by another, a shared lock serializes them and can be worse.

### Do not merge independent values

```rust
// SLOW: unrelated counters contend on one lock
struct Metrics {
    requests: Mutex<u64>,
    errors: Mutex<u64>,
    bytes: Mutex<u64>,
}
```

If each counter is updated on a different path, keep them separate or use atomics (`AtomicU64`) instead of locking.

### `RefCell` used only to satisfy the borrow checker

```rust
// SLOW: runtime borrow checks in a hot traversal
struct Tree {
    children: RefCell<Vec<Box<Tree>>>,
}
```

```rust
// FAST: restructure so ownership and borrowing are known at compile time
struct Tree {
    children: Vec<Tree>,
}

fn walk(tree: &Tree) {
    for child in &tree.children {
        walk(child);
    }
}
```

Use `RefCell` for genuine interior mutability with short borrows, not as a workaround for a design that can use `&mut`.

### `Rc<RefCell<Node>>` graphs

```rust
// SLOW: pointer chasing, refcount updates, runtime borrow panics
type Link = Option<Rc<RefCell<Node>>>;
```

```rust
// FAST: contiguous nodes, index links, no refcounts
struct Graph {
    nodes: Vec<Node>,
}

#[derive(Clone, Copy)]
struct NodeId(u32);

struct Node {
    next: Option<NodeId>,
    value: u64,
}
```

Index-based graphs and trees have better locality, no refcount overhead, and no runtime borrow panics. Use them unless the graph is small or needs shared ownership across independent owners.

### `Rc` vs `Arc`

```rust
// Single-threaded: cheaper non-atomic refcount
use std::rc::Rc;

// Shared across threads: atomic refcount
use std::sync::Arc;
```

Use `Arc` only when the value crosses threads. `Rc` avoids atomic operations and is smaller/faster.

### Atomic counters instead of locks

```rust
use std::sync::atomic::{AtomicU64, Ordering};

// FAST: no lock, one atomic increment
struct Metrics {
    hits: AtomicU64,
}

impl Metrics {
    fn hit(&self) {
        self.hits.fetch_add(1, Ordering::Relaxed);
    }
}
```

For counters and statistics, atomics are usually better than `Mutex<u64>`. For multi-field invariants, a lock is still the right tool.

### `OnceLock` instead of an always-locked lazy value

```rust
use std::sync::OnceLock;

static CONFIG: OnceLock<Config> = OnceLock::new();

fn config() -> &'static Config {
    CONFIG.get_or_init(load_config)
}
```

After initialization, reads are a cheap check, not a lock.

## Anti-patterns

1. `Arc<Mutex<T>>` for values that never cross a thread; use `Rc`/`RefCell` or plain ownership.
2. Two wrappers where one would do, or one where two independent ones would contend.
3. `RefCell` in a hot loop for values that could be borrowed statically.
4. `Rc<RefCell<Node>>` graphs with pointer chasing and runtime panics.
5. Cloning `Arc` per item instead of once.
6. `RwLock` when writes are frequent.
7. `Mutex<u64>` counters instead of atomics.
8. Holding a lock while doing I/O, formatting, or awaiting.
9. Poisoning (`std::sync::Mutex`) ignored with `unwrap` on a path that can panic.
10. Adding a wrapper without measuring its access cost.

## Checklist

- Is the wrapper required for sharing or interior mutability, or only to appease the borrow checker?
- Are co-accessed values under one wrapper?
- Are independently accessed values kept independent (or atomic)?
- Is `Rc` used where `Arc` is not needed, and vice versa?
- Is `RefCell` limited to short borrows outside hot loops?
- Could an index-based arena replace `Rc<RefCell<T>>`?
- Are counters using atomics instead of locks?
- Is any lock held across I/O or `.await`?
- Was the change measured with a profile/benchmark?

## Related

- `std.md` for `parking_lot` and standard synchronization types.
- `parallelism.md` for contention, false sharing, and atomics.
- `rustonomicon.md` for `Send`/`Sync` and atomic ordering rules.
- `type-sizes.md` for reducing the size of wrapped values.
