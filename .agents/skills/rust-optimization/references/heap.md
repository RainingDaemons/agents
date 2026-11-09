# Heap Allocations

## Description

How to find and reduce heap allocation, which is often the dominant hidden cost in Rust. Each allocation and deallocation may take a lock, manipulate allocator metadata, and occasionally make a system call. Small allocations are not necessarily cheaper than large ones, so the goal is fewer allocations, not smaller ones. For `Vec` and `String`, the growth policy is the usual cause: 0, 4, 8, 16, 32, ... with copying on each growth.

## When to use

- A profile shows `malloc`, `free`, `realloc`, or `memcpy` as hot.
- Memory use is high, grows over time, or peak memory matters.
- A hot loop builds `Vec`, `String`, `HashMap`, or `format!` output.
- `BufRead::lines()` allocates a `String` per line.
- Cloning large values or calling `to_owned`/`to_vec`/`to_string` in hot paths.
- A service allocates per request or per item.

## Checks

```bash
cargo install dhat
valgrind --tool=dhat ./target/release/app
heaptrack ./target/release/app
```

```rust
// dhat-rs works on all platforms and can assert allocation counts in tests
#[cfg(test)]
mod tests {
    #[test]
    fn allocates_once() {
        let _profiler = dhat::Profiler::builder().testing().build();
        let before = dhat::HeapStats::get();
        let v: Vec<u32> = (0..100).collect();
        dhat::assert_eq!(before.total_blocks + 1, dhat::HeapStats::get().total_blocks);
        drop(v);
    }
}
```

Probes:

```bash
rg -n 'Vec::new\(\)|String::new\(\)' src | head -50
rg -n 'format!\(|to_string\(\)|to_owned\(\)|to_vec\(\)' src | head -50
rg -n '\.clone\(\)' src | head -50
rg -n 'collect::<Vec' src
rg -n '\.lines\(\)|read_line|read_to_string' src
rg -n 'HashMap::new\(\)|HashSet::new\(\)' src
```

## Code examples

### Reserve capacity for a known size

```rust
// SLOW: grows 0 -> 4 -> 8 -> 16 -> 32, copying each time
let mut out = Vec::new();
for x in items {
    out.push(transform(x));
}
```

```rust
// FAST: one allocation
let mut out = Vec::with_capacity(items.len());
for x in items {
    out.push(transform(x));
}
```

For 20 elements, pushing one at a time causes four allocations (capacities 4, 8, 16, 32). `with_capacity(20)` causes one. `reserve` and `reserve_exact` do the same when the size is only known later.

### Reuse a buffer instead of reallocating

```rust
// SLOW: a new String per line
for line in reader.lines() {
    let line = line?;
    process(&line);
}
```

```rust
// FAST: one String reused
let mut line = String::new();
while reader.read_line(&mut line)? != 0 {
    process(&line);
    line.clear();
}
```

`clear` keeps the capacity, so subsequent lines do not allocate. The same pattern applies to `Vec`, `HashMap`, and scratch buffers stored in a struct.

### Avoid `collect` only to iterate again

```rust
// SLOW: allocates a Vec that is immediately consumed
let sum: u64 = items.iter().map(|x| x.cost).collect::<Vec<_>>().iter().sum();

// FAST: no intermediate allocation
let sum: u64 = items.iter().map(|x| x.cost).sum();
```

If a function only needs to iterate, return `impl Iterator<Item = T>` instead of `Vec<T>`.

### Extend a reused collection

```rust
// SLOW: a new Vec per iteration
for batch in batches {
    let parsed: Vec<_> = batch.iter().map(parse).collect();
    out.extend(&parsed);
}
```

```rust
// FAST: extend directly into the destination
for batch in batches {
    out.extend(batch.iter().map(parse));
}
```

### Short-lived small collections: `SmallVec` / `ArrayVec`

```rust
use smallvec::SmallVec;

// Most vectors hold 0-4 elements; avoid the heap until they grow
let mut v: SmallVec<[u32; 4]> = SmallVec::new();
v.push(1);
```

`SmallVec<[T; N]>` stores up to `N` elements inline and falls back to the heap. `ArrayVec<T, N>` never allocates but has a fixed maximum. Both add branching and can increase the size of the containing value; measure. `SmallString`/`smartstring` do the same for strings.

### `clone` vs `clone_from` vs borrow

```rust
// SLOW: allocates a new buffer and drops the old one
dst = src.clone();

// FASTER: reuses dst's existing allocation when possible
dst.clone_from(&src);
```

`clone_from` is not just `dst = src.clone()`. Still prefer borrowing, `Cow`, or `Arc` when ownership is not required.

### Share instead of clone

```rust
// SLOW: deep-copies a large structure on every share
let copy = config.clone();

// FAST: refcount increment
let shared = Arc::clone(&config);
```

`Arc::clone` is cheap, but still an atomic increment; clone once outside a loop rather than per iteration.

### Cloning into `Cow`

```rust
use std::borrow::Cow;

fn normalize(input: &str) -> Cow<'_, str> {
    if input.chars().all(|c| c.is_ascii_lowercase()) {
        Cow::Borrowed(input)          // no allocation
    } else {
        Cow::Owned(input.to_lowercase())
    }
}
```

Use `Cow` when most paths borrow and only some need to own.

### Alternative allocator

If allocation is hot even after reducing the rate, an alternative allocator may help:

```toml
[dependencies]
mimalloc = "0.1"
```

```rust
#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

Benchmark on the target platform. `mimalloc` and `jemalloc` can improve allocation-heavy or multi-threaded workloads but increase binary size and compile time. See `buildconfig.md`.

## Anti-patterns

1. `Vec::new()` + `push` in a loop when the size is known.
2. `clone()` to satisfy the borrow checker without restructuring.
3. `format!`/`to_string()` in a hot loop for diagnostics.
4. `collect()` only to iterate or `extend` again.
5. `BufRead::lines()` per line in a hot path.
6. `HashMap::new()` in a loop when the size is known (`with_capacity`).
7. `Box<dyn Trait>` per item when an enum or generic would do.
8. `Rc`/`Arc` on small values that are rarely shared.
9. A new `Vec`/`String` scratch buffer on every call instead of reusing one.
10. Adopting an allocator without measuring.

## Checklist

- Did a profiler show allocation/copy as hot, or are you guessing?
- Is every `Vec`/`String`/map with a known size pre-allocated?
- Are hot-path buffers reused with `clear()`?
- Is `collect` avoided when the collection is only iterated?
- Are clones, `to_owned`, `to_string`, and `format!` justified?
- Are small collections using inline storage where it pays?
- Is line reading reusing a buffer?
- Were allocation counts verified before and after (DHAT/`dhat-rs`)?
- Was an alternative allocator benchmarked, if used?

## Related

- `type-sizes.md` for shrinking the values being allocated.
- `std.md` for cheaper standard-library methods.
- `iterators.md` for avoiding intermediate collections.
- `buildconfig.md` for alternative allocators and profile settings.
- `profiling.md` for reading allocation profiles.
