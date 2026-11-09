# Standard Library Types

## Description

Cheaper methods and better alternatives in the standard library: `Vec` capacity and removal, lazy `Option`/`Result` combinators, clone-on-write for `Rc`/`Arc`, boxed slices, and synchronization primitives. These are small, local changes that often remove an allocation or a copy without changing the design.

## When to use

- You are about to write a loop that grows, removes, or rebuilds a collection.
- An `Option`/`Result` fallback is expensive but computed eagerly.
- Sharing immutable data and cloning large values.
- Choosing between `Mutex`, `RwLock`, and an alternative from `parking_lot`.
- A `Vec` is stored in a field that is never mutated.

## Quick reference

| Need | Use | Avoid |
|---|---|---|
| Zero-filled `Vec` of length `n` | `vec![0; n]` | `resize`, `extend`, unsafe tricks |
| Remove by index, order irrelevant | `Vec::swap_remove(i)` | `Vec::remove(i)` (`O(n)`) |
| Remove many items | `Vec::retain` | Rebuilding a new `Vec` |
| Shrink a `Vec` to its length | `Vec::into_boxed_slice()` | Keeping excess capacity forever |
| Empty `Vec` in a hot struct | `ThinVec` | A full three-word `Vec` per instance |
| Lazy fallback | `ok_or_else`, `map_or_else`, `unwrap_or_else` | `ok_or`, `map_or`, `unwrap_or` with expensive args |
| Clone-on-write shared data | `Rc::make_mut`/`Arc::make_mut` | Cloning the whole value every time |
| Build a value once | `OnceLock`/`LazyLock` | Recomputing per call |
| Short critical sections | `Mutex` | `RwLock` when writes are common |

## Code examples

### Zero-filled `Vec`

```rust
// FAST: simple, can use OS assistance, usually as fast or faster
let zeros = vec![0u8; 1_000_000];
```

```rust
// Usually slower or not better: resize, extend, or unsafe alternatives
let mut zeros = Vec::new();
zeros.resize(1_000_000, 0);
```

`vec![0; n]` is the recommended way to create a zero-filled `Vec`.

### `remove` vs `swap_remove`

```rust
// SLOW: O(n) shift
let item = v.remove(i);
```

```rust
// FAST: O(1), order not preserved
let item = v.swap_remove(i);
```

Use `swap_remove` when order does not matter; use `retain` to remove many items in one pass.

### `retain` instead of rebuilding

```rust
// SLOW: allocates a second Vec
let kept: Vec<_> = v.into_iter().filter(|x| x.active).collect();
```

```rust
// FAST: in place
let mut v = v;
v.retain(|x| x.active);
```

### `Vec` → `Box<[T]>`

```rust
// A field that is never mutated: drop the capacity word
let values: Box<[u32]> = values.into_boxed_slice();
```

`Box<[T]>` is two words (pointer, length) versus `Vec`'s three (pointer, length, capacity), and it may reallocate to drop excess capacity. `slice::into_vec` converts back without cloning.

### Lazy `Option`/`Result` fallbacks

```rust
// SLOW: `expensive()` runs even when `o` is `Some`
let r = o.ok_or(expensive());
let v = o.unwrap_or(expensive());
```

```rust
// FAST: computed only when needed
let r = o.ok_or_else(expensive);
let v = o.unwrap_or_else(expensive);
```

The same applies to `map_or`/`map_or_else`, `or`/`or_else`, and `Result` equivalents.

### `Rc`/`Arc` clone-on-write

```rust
use std::sync::Arc;

// Clone only when shared; mutate in place when uniquely owned
let mut data = Arc::new(vec![1, 2, 3]);
Arc::make_mut(&mut data).push(4);
```

`make_mut` clones the inner value only when the refcount is greater than one. Cloning an `Arc` itself is just a refcount increment; clone the handle, not the data.

### `OnceLock` for one-time initialization

```rust
use std::sync::OnceLock;

static RE: OnceLock<regex::Regex> = OnceLock::new();

fn is_match(s: &str) -> bool {
    RE.get_or_init(|| regex::Regex::new(r"^\d+$").unwrap())
        .is_match(s)
}
```

Never compile a regex, build a client, or open a pool per call.

### Synchronization primitives

- `Mutex`: default for short critical sections.
- `RwLock`: only when reads dominate heavily; otherwise it can be slower and can starve writers.
- `Condvar`, `Once`: use the standard versions unless a profile says otherwise.
- `parking_lot`: alternative implementations with similar APIs; historically smaller and faster, but std has improved. Measure before switching.

```toml
[dependencies]
parking_lot = "0.12"
```

```rust
use parking_lot::Mutex;

let lock = Mutex::new(0);
*lock.lock() += 1;          // no poisoning, no unwrap
```

## Anti-patterns

1. `Vec::remove(i)` in a loop when order does not matter.
2. Rebuilding a collection to remove a few items instead of `retain`.
3. `ok_or(expensive())` / `unwrap_or(expensive())` on a hot path.
4. Cloning an `Arc`'s contents instead of the `Arc`.
5. Recomputing a regex, client, or pool per call.
6. `RwLock` for a workload with frequent writes.
7. Growing a `Vec` without capacity when the size is known.
8. Switching to `parking_lot` without a measurement.
9. Keeping `Vec` capacity that will never be used (consider `into_boxed_slice`).

## Checklist

- Is the zero-fill done with `vec![0; n]`?
- Is removal using `swap_remove` or `retain` where order is irrelevant?
- Are `Option`/`Result` fallbacks lazy?
- Is shared immutable data cloned as a handle, not as the value?
- Are one-time constructions hoisted into `OnceLock`/`LazyLock`?
- Is the synchronization primitive matched to the read/write mix?
- Was an alternative primitive benchmarked before adoption?
- Are collection capacities sized for the expected workload?

## Related

- `heap.md` for allocation and capacity details.
- `type-sizes.md` for `Vec` vs `Box<[T]>` vs `ThinVec`.
- `wrapper.md` for `Mutex`/`RefCell` access costs.
- `hashing.md` for map types and hashers.
