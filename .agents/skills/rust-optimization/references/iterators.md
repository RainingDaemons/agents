# Iterators and Bounds Checks

## Description

Whether iterators, `collect`, or bounds checks are the problem. Iterator chains compile to tight loops and usually match — or beat — hand-written indexing because they let the optimizer see lengths, remove bounds checks, and vectorize. The real problems are unnecessary `collect`, iterator adapters that block optimization, and index loops where the compiler cannot prove safety.

## When to use

- A hot loop indexes slices or vectors.
- A chain has `collect` in the middle, or collects only to iterate again.
- `chain`, `filter().map()`, or layered adapters appear in a hot path.
- `cargo asm` shows `panic_bounds_check` or unexpected `memcpy`.
- A function allocates a `Vec` only to return data that the caller iterates.
- Chunking does not divide evenly and the tail is handled awkwardly.

## Checks

```bash
cargo asm --rust my_crate::hot_fn | rg 'panic_bounds_check|call|memcpy'
cargo clippy -- -W clippy::perf
RUSTFLAGS="-C target-cpu=native" cargo asm --rust my_crate::sum
```

Probes:

```bash
rg -n 'for .* in 0\.\..*\.len\(\)' src
rg -n '\[[a-z_]+\]' src | head -50
rg -n '\.collect::<Vec' src
rg -n '\.chain\(|\.filter\(.*\)\.map\(' src
rg -n '\.chunks\(|\.windows\(' src
rg -n 'get_unchecked' src
```

## Code examples

### Index loop vs iterator

```rust
// SLOW: a bounds check per access unless the compiler proves it safe
let mut sum = 0u64;
for i in 0..data.len() {
    sum += u64::from(data[i]);
}
```

```rust
// FAST: no bounds checks, easy to vectorize
let sum: u64 = data.iter().map(|&x| u64::from(x)).sum();
```

Both are `O(n)`, but the iterator form gives LLVM the length and the absence of aliasing. Check the assembly if it matters.

### Avoid `collect` when the result is only iterated

```rust
// SLOW: allocates a Vec that is immediately consumed
let total: u64 = items
    .iter()
    .map(|x| x.cost)
    .collect::<Vec<_>>()
    .iter()
    .sum();
```

```rust
// FAST: no intermediate allocation
let total: u64 = items.iter().map(|x| x.cost).sum();
```

Return `impl Iterator<Item = T>` from a function when the caller only iterates:

```rust
fn active(items: &[Item]) -> impl Iterator<Item = &Item> {
    items.iter().filter(|x| x.active)
}
```

Implement `size_hint` or `ExactSizeIterator::len` on custom iterators so `collect` can allocate once.

### `chain` in a hot iterator

```rust
// Can be slower than a single iterator; fine when cold
for x in first.iter().chain(second.iter()) {
    process(x);
}
```

```rust
// Often faster when both are the same type and hot
first.iter().for_each(process);
second.iter().for_each(process);
```

Measure; `chain` is convenient and usually fine. Avoid it in a tight inner loop only if the profile or assembly says so.

### `filter` then `map` vs `filter_map`

```rust
// SLOW: two adapters, one extra branch
let values: Vec<_> = items.iter().filter(|x| x.ok).map(|x| x.value).collect();
```

```rust
// FAST: one adapter
let values: Vec<_> = items.iter().filter_map(|x| x.ok.then_some(x.value)).collect();
```

### Chunking: `chunks_exact` plus the remainder

```rust
// SLOW: `chunks` carries a variable-length last chunk in every iteration
for chunk in data.chunks(4) {
    process(chunk);
}
```

```rust
// FAST: fixed-size chunks, remainder handled once
let mut chunks = data.chunks_exact(4);
for chunk in &mut chunks {
    process(chunk);           // chunk.len() == 4, known to the optimizer
}
let rest = chunks.remainder();
if !rest.is_empty() {
    process(rest);
}
```

Related: `rchunks_exact`, `chunks_exact_mut`, `rchunks_exact_mut`.

### `iter()` vs `iter().copied()`

```rust
// For small Copy types, consuming by value can produce better code
let sum: u32 = values.iter().copied().sum();
```

This is advanced; verify in the generated code. For large types, `copied` would copy and is the wrong choice.

### Bounds checks: safe techniques first

```rust
// SLOW: repeated indexing with a check
for i in 0..a.len() {
    out[i] = a[i] + b[i];
}
```

```rust
// FAST: zip pairs the slices, no checks
for (o, (x, y)) in out.iter_mut().zip(a.iter().zip(b)) {
    *o = x + y;
}
```

Other safe options:

1. Iterate instead of indexing.
2. Slice once before the loop: `let a = &a[..n];`
3. Assert lengths once: `assert_eq!(a.len(), b.len());`
4. Use `windows`, `chunks_exact`, or `zip` to express the invariant.

### `get_unchecked` as a last resort

```rust
// SAFETY: `i < data.len()` is guaranteed by the loop bound and `data.len() == n`
unsafe {
    *data.get_unchecked(i)
}
```

Only after a profile shows the check matters, with a `// SAFETY:` comment, tests, and ideally Miri. Prefer the safe techniques; they usually remove the check too.

## Anti-patterns

1. Index loops over slices in hot code when an iterator expresses the same thing.
2. `collect()` in the middle of a chain or only to iterate again.
3. Returning `Vec<T>` when the caller only iterates.
4. Rebuilding an iterator with `chain` in the innermost loop.
5. `chunks` when `chunks_exact` plus `remainder` is possible.
6. `get_unchecked` without a proof, comment, and tests.
7. Assuming an iterator is slower than a `for` loop without measuring.
8. Custom iterators without `size_hint`, causing repeated `collect` reallocations.
9. Manual indexing when `zip`/`windows` would encode the invariant.
10. Optimizing iterator code in a dev build.

## Checklist

- Did the profile or assembly actually identify the iterator/check as hot?
- Is `collect` necessary, or can the chain stay lazy?
- Are lengths expressed with slices, `zip`, `chunks_exact`, or assertions?
- Is `size_hint` implemented for custom iterators?
- Is `chain` or a layered adapter in the hot path, and is there a faster equivalent?
- Is `copied` appropriate for the element size?
- Is every `get_unchecked` justified and documented?
- Did you verify the generated code and re-run the benchmark?

## Related

- `iterators-cost.md` for the complexity of adapters and collections.
- `machinecode.md` for checking bounds checks and vectorization in assembly.
- `std.md` for standard-library methods such as `retain` and `swap_remove`.
- `profiling.md` for finding the hot loop in the first place.
