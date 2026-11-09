# Rustonomicon Notes: Layout, Allocation, Unsafe, and Atomics

## Description

The rules behind the two questions machine-level performance keeps raising: **why is this allocation or layout costly?** and **what is this unsafe/atomic ordering rule?** Covers type layout and representation, where values live, the invariants of `unsafe`, and the memory model for atomics. Unsafe is a correctness boundary first and a performance tool second.

## When to use

- A hot type is larger than expected or copies via `memcpy`.
- Choosing `repr(C)` versus the default `repr(Rust)`.
- Writing or reviewing `unsafe` (`get_unchecked`, `MaybeUninit`, `transmute`, FFI).
- Reasoning about `Send`/`Sync`, atomics, or lock-free code.
- Deciding whether `Box`, `Rc`, `Arc`, or an arena is appropriate.
- A bug appears only under optimization or across threads (aliasing or ordering).

## Layout

| Concept | Rule |
|---|---|
| Size | Bytes occupied, including padding |
| Alignment | Address must be a multiple of this; `size` is a multiple of `align` |
| `repr(Rust)` | Compiler may reorder fields to minimize size and padding |
| `repr(C)` | Field order preserved; required for FFI and some layouts |
| `repr(transparent)` | Single non-zero-sized field; same layout as that field |
| Niche optimization | `Option<&T>`, `Option<Box<T>>`, `Option<NonNull<T>>`, `Option<NonZeroU32>` use a reserved bit and add no size |
| Enum | Discriminant plus the largest variant; `Option` of a non-nullable pointer is one word |

```rust
use std::mem::{align_of, size_of};

#[repr(C)]
struct CBad {
    a: u8,
    b: u64,
    c: u8,
} // 24 bytes: padding after a and c

struct RustGood {
    a: u8,
    b: u64,
    c: u8,
} // compiler reorders: 16 bytes

assert_eq!(size_of::<CBad>(), 24);
assert_eq!(size_of::<RustGood>(), 16);
```

Do not hand-reorder fields under `repr(Rust)`; the compiler already does it. Under `repr(C)`, ordering and padding are your responsibility.

## Allocation

- `Box<T>` is one pointer plus a heap allocation; use it for indirection or to shrink an enum.
- `Vec<T>` is three words (ptr, len, cap); `Box<[T]>` is two (ptr, len); `ThinVec` is one.
- `Rc<T>`/`Arc<T>` add a strong (and weak) count to the allocation; `clone` is a refcount increment, not a data copy.
- `Rc`/`Arc` on small, rarely shared values usually costs more than it saves.
- Arena allocation (`bumpalo`, `typed-arena`) frees many short-lived nodes together and avoids per-node deallocation.
- Stack allocation is cheap but bounded; huge stack values risk overflow (especially in async, where futures are moved).

## Unsafe rules

| Operation | Requirement |
|---|---|
| `get_unchecked(i)` | `i < len` must hold; document why |
| `MaybeUninit<T>` | Never read before initialization; track initialization state |
| `transmute` | Same size and a valid value for the target type; almost always avoidable |
| `from_utf8_unchecked` | Bytes must be valid UTF-8 |
| Raw pointer deref | Non-null, aligned, valid for reads/writes, no aliasing violations |
| `Send`/`Sync` impl | The type must genuinely be safe to move/share across threads |
| FFI | Match the ABI, lifetimes, and ownership contract exactly |

Every `unsafe` block needs a `// SAFETY:` comment stating the invariant and why the surrounding code upholds it. Keep the block as small as possible, test it, and run Miri (`cargo +nightly miri test`) and `loom` for concurrency.

```rust
// SAFETY: the loop guarantees `i < data.len()`.
let x = unsafe { *data.get_unchecked(i) };
```

## Atomics and ordering

A data race is undefined behavior. Atomics prevent races on a location, but **ordering** determines what other memory operations may be reordered around them.

| Ordering | Store | Load | Use |
|---|---|---|---|
| `Relaxed` | yes | yes | Counters, statistics; no synchronization |
| `Release` | yes | — | Publish data written before the store |
| `Acquire` | — | yes | Observe data written before the matching release |
| `AcqRel` | yes | yes | Read-modify-write that both publishes and observes |
| `SeqCst` | yes | yes | A single total order; safest, strongest, slowest |

```rust
use std::sync::atomic::{AtomicBool, AtomicU64, Ordering};

static HITS: AtomicU64 = AtomicU64::new(0);
HITS.fetch_add(1, Ordering::Relaxed);          // no ordering needed

static READY: AtomicBool = AtomicBool::new(false);
static mut DATA: u64 = 0;

// Writer
unsafe { DATA = 42; }
READY.store(true, Ordering::Release);          // publish

// Reader
if READY.load(Ordering::Acquire) {             // observe
    let value = unsafe { DATA };                // safe: release/acquire pairs
    assert_eq!(value, 42);
}
```

Do not use `static mut` in real code; the example is only to show the pairing. Use `OnceLock`, `Mutex`, channels, or an atomic pointer instead.

`SeqCst` is not a "performance setting"; choosing a weaker ordering requires that the code remains correct. If you cannot prove the weaker ordering, use `SeqCst` and revisit with `loom`.

## Code examples

### Shrink a large enum variant

```rust
// 4 KB per value, even for `Ping`
enum Message {
    Ping,
    Data([u8; 4096]),
}
```

```rust
// 16 bytes per value; the large variant pays one allocation when used
enum Message {
    Ping,
    Data(Box<[u8; 4096]>),
}
```

### Boxed slice instead of Vec

```rust
use std::mem::size_of_val;

let v: Vec<u32> = vec![1, 2, 3];
assert_eq!(size_of_val(&v), 3 * size_of::<usize>());   // ptr, len, cap

let bs: Box<[u32]> = v.into_boxed_slice();
assert_eq!(size_of_val(&bs), 2 * size_of::<usize>());  // ptr, len
```

### `MaybeUninit` without UB

```rust
use std::mem::MaybeUninit;

let mut slot: MaybeUninit<u64> = MaybeUninit::uninit();
slot.write(7);
// SAFETY: we initialized `slot` with `write` immediately above.
let value = unsafe { slot.assume_init() };
assert_eq!(value, 7);
```

Never `assume_init` a value that may not be initialized, and never read padding bytes.

## Anti-patterns

1. `transmute` between types that differ in layout or validity.
2. `get_unchecked` without a proven bound and a `// SAFETY:` comment.
3. Reading `MaybeUninit` before initialization.
4. Implementing `Send`/`Sync` for a type that is not actually safe to share.
5. Using `static mut`; use `OnceLock`/`Mutex`/atomics with a proper protocol.
6. Assuming `Relaxed` is enough for publishing data.
7. Using `SeqCst` without understanding, or a weaker ordering without proof.
8. Hashing or comparing padding bytes.
9. Relying on `repr(Rust)` field order for FFI or serialization.
10. Using `unsafe` for speed when safe code already produces the same machine code.

## Checklist

- Do you know the size and alignment of the hot type (`size_of`, `align_of`)?
- Is `repr(C)` required (FFI, serialization), and is its padding handled?
- Could niche optimization remove an `Option` tag?
- Is the allocation strategy (`Box`, `Vec`, `Rc`, `Arc`, arena) justified by ownership and sharing?
- Does every `unsafe` block have a `// SAFETY:` invariant?
- Are `Send`/`Sync` impls justified, not asserted?
- Are atomics using the weakest correct ordering, with the pairing documented?
- Is the code tested with Miri and, if concurrent, `loom`?

## Related

- `type-sizes.md` for measuring and shrinking types.
- `heap.md` for allocation strategies.
- `parallelism.md` for atomics in practice.
- `machinecode.md` for verifying that `unsafe` changed the generated code.
