# General Optimization Tips

## Description

The principles that apply before and after the Rust-specific techniques: optimize only hot code, prefer algorithmic and data-structure changes, remove silly slowdowns, avoid unnecessary work, special-case common inputs, and accumulate many small measured wins. Rust is already fast and memory-efficient when you avoid the obvious pitfalls; most remaining wins are deliberate, not clever.

## When to use

- Starting an optimization effort and deciding where to spend effort.
- Reviewing a proposed optimization for whether it is worth the complexity.
- A change is "clever" but unmeasured.
- Looking for cheap wins before deeper work.
- Writing comments and benchmarks that will justify an optimization later.

## Principles

| Principle | Consequence |
|---|---|
| Optimize only hot code | Optimized code is more complex; cold code should stay simple |
| Algorithms and data structures first | The biggest wins are asymptotic, not instruction-level |
| Remove silly slowdowns before clever speedups | Dev builds, logging, unbounded allocation, N+1 calls |
| Avoid unnecessary work | Lazy/on-demand computation often wins |
| Special-case common inputs | 0, 1, 2 elements; common enum variants; frequent values |
| Handle frequent cases first | Measure case frequencies, then order the branches |
| Use compact representations for repetitive data | Common value inline plus a fallback table |
| Put a small cache in front of high-locality lookups | Cheap memoization for repeated queries |
| Most improvements are small | Many measured small wins add up |
| Use more than one profiler | Different tools see different bottlenecks |
| When a function is hot, make it faster or call it less | Both are valid |

## Checks

```bash
cargo build --release
cargo clippy -- -W clippy::perf
cargo bench
```

Probes:

```bash
rg -n 'TODO|FIXME|HACK|slow|workaround' src
rg -n '\.clone\(\)|to_string\(\)|to_vec\(\)|format!\(' src | head -50
rg -n 'Vec::new\(\)|String::new\(\)' src | head -50
rg -n 'for .* in 0\.\.|\.len\(\)' src | head -50
rg -n 'unwrap_or\(|ok_or\(|map_or\(' src
```

## Code examples

### Special-case the common sizes

```rust
// SLOW: the general path runs even when the collection is empty or tiny
fn total(items: &[Item]) -> u64 {
    items.iter().map(|i| i.value).sum()
}
```

```rust
// FAST: avoid setup for the overwhelmingly common small cases
fn total(items: &[Item]) -> u64 {
    match items {
        [] => 0,
        [one] => one.value,
        [a, b] => a.value + b.value,
        _ => items.iter().map(|i| i.value).sum(),
    }
}
```

This only helps when small sizes dominate. Measure the size distribution first (for example with `counts`).

### Avoid work until it is needed

```rust
// SLOW: computes the expensive value even when it is not used
let report = build_report(&data);
if verbose {
    println!("{report}");
}
```

```rust
// FAST: compute only on the path that needs it
if verbose {
    let report = build_report(&data);
    println!("{report}");
}
```

The same idea applies to logging, error messages, and optional diagnostics (see `logging-debugging.md`).

### Compact representation for repetitive values

```rust
// SLOW: every value is a full String, even when most are one of a few constants
struct Token {
    kind: String,
}

// FAST: small enum for common kinds, boxed/string fallback for the rest
enum TokenKind {
    Identifier,
    Number,
    Keyword(u8),
    Other(Box<str>),
}
```

The compact representation saves memory and improves cache behavior. Measure the actual value distribution before committing to it.

### Small cache in front of a lookup

```rust
// Repeated lookups of the same few keys
let mut cache: std::collections::HashMap<Key, Value> = std::collections::HashMap::new();
fn get(key: Key, cache: &mut HashMap<Key, Value>, source: &Source) -> &Value {
    cache.entry(key).or_insert_with(|| source.lookup(key))
}
```

A cache only helps when there is locality. A cache with no reuse adds overhead; measure the hit rate.

### Make it faster or call it less

When a function is hot, there are two independent levers:

1. **Make it faster** — better algorithm, fewer allocations, better layout.
2. **Call it less** — memoize, batch, precompute, skip redundant calls.

The second is often easier and larger. A `format!` that runs on every request is a formatting problem; a `format!` that runs on every *field* is a "call it less" problem.

## Anti-patterns

1. Optimizing cold code because it "looks slow".
2. Micro-optimizing before profiling.
3. Adding complexity for an unmeasured gain.
4. Assuming the biggest win is low-level; check algorithms and data structures first.
5. Keeping an optimization whose benchmark regressed.
6. Special-casing without knowing the input distribution.
7. Adding a cache without measuring the reuse rate.
8. Replacing clear code with a clever trick and no comment explaining why.
9. Trusting a single profiler or a single benchmark.
10. Optimizing a dev build.

## Checklist

- Is the code on a measured hot path?
- Did you consider an algorithmic or data-structure change first?
- Is there an obvious silly slowdown (dev build, logging, allocation, I/O) to remove?
- Can the work be avoided, deferred, batched, or cached?
- Is the common case handled first, based on measured frequencies?
- Is the representation as compact as the data justifies?
- Is the change measured against a baseline on the same machine and profile?
- Is the complexity worth the win, and is the reason documented in a comment?
- Do the tests still pass, including edge cases (empty, single, large, duplicates)?

## Related

- `benchmarking.md` for measuring each change.
- `profiling.md` for finding the hot path.
- `heap.md` and `std.md` for common allocation and standard-library wins.
- `buildconfig.md` for the release build that makes any of this meaningful.
