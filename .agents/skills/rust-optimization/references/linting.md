# Linting

## Description

How to use Clippy to find performance problems, and where its limits are. Clippy's performance lints catch common patterns that produce suboptimal code: needless `collect`, redundant clones, large enum variants, manual `memcpy`, and lazy-evaluation mistakes. Some non-performance lints (`ptr_arg`) also improve code generation. Clippy is not a profiler: fixing every lint does not replace measuring the hot path.

## When to use

- On every change, as part of the normal build/test cycle.
- Before a performance review, to catch obvious waste cheaply.
- Standardizing on alternative types (faster hashers, `parking_lot`) without accidental regressions.
- Reviewing code that allocates, clones, or builds collections in loops.

## Commands

```bash
cargo clippy
cargo clippy --all-targets --all-features
cargo clippy -- -W clippy::perf          # highlight performance lints
cargo clippy -- -D warnings              # in CI, if the project enforces it
```

## Checks

| Lint | Finds | Typical fix |
|---|---|---|
| `redundant_clone` | Clone of a value that is not used afterward | Remove the clone or reuse the value |
| `needless_collect` | `collect` followed by iteration | Keep the iterator lazy |
| `unnecessary_to_owned` | Redundant `to_owned`/`to_string` | Borrow instead |
| `large_enum_variant` | One variant much larger than the others | `Box` the large variant |
| `manual_memcpy` | Index loop copying a slice | `copy_from_slice` |
| `inefficient_to_string` | `to_string` on a value with a cheaper conversion | Use the cheaper path |
| `ptr_arg` | `&Vec<T>`/`&String` arguments | `&[T]`/`&str` |
| `map_entry` | `contains_key` then `insert` | `entry` |
| `or_fun_call` | Eager fallback in `unwrap_or`/`ok_or` | `unwrap_or_else`/`ok_or_else` |
| `single_char_pattern` | `split("x")` with a one-char pattern | `split('x')` |
| `needless_range_loop` | Index loop over a range | Iterator |
| `manual_let_else` | Nested match for early return | `let ... else` |
| `unnecessary_sort_by` | `sort_by` on a simple key | `sort_by_key` |
| `redundant_pattern_matching` | `if let Some(_)` | `is_some()` |

Run `cargo clippy -- -W clippy::perf` to surface the performance group explicitly. The full list is in the Clippy lint documentation, filtered to the "Perf" group.

## Code examples

### `ptr_arg`: slices instead of containers

```rust
// SLOW API: forces callers to pass a Vec and adds indirection
fn process(items: &Vec<Item>) {
    for item in items {
        handle(item);
    }
}
```

```rust
// FAST API: accepts arrays, slices, and Vecs; clearer length information
fn process(items: &[Item]) {
    for item in items {
        handle(item);
    }
}
```

`&mut Vec<T>` → `&mut [T]` is the same idea. The lint's primary motivation is API flexibility; the performance benefit is less indirection and better optimization.

### `needless_collect`

```rust
// SLOW: allocates only to iterate again
let names: Vec<_> = users.iter().map(|u| &u.name).collect();
for name in &names {
    print(name);
}
```

```rust
// FAST: lazy
for name in users.iter().map(|u| &u.name) {
    print(name);
}
```

### `map_entry`

```rust
// SLOW: two lookups
if !map.contains_key(&key) {
    map.insert(key, Vec::new());
}
map.get_mut(&key).unwrap().push(item);
```

```rust
// FAST: one lookup
map.entry(key).or_default().push(item);
```

### `or_fun_call`

```rust
// SLOW: `expensive()` runs even when the value is present
let value = maybe.unwrap_or(expensive());

// FAST: computed only when needed
let value = maybe.unwrap_or_else(expensive);
```

### `large_enum_variant`

```rust
// SLOW: every value is as large as the biggest variant
enum Message {
    Ping,
    Data([u8; 4096]),
}
```

```rust
// FAST: box the rare large variant
enum Message {
    Ping,
    Data(Box<[u8; 4096]>),
}
```

### Disallowing standard types

```toml
# clippy.toml
disallowed-types = [
    "std::collections::HashMap",
    "std::collections::HashSet",
    "std::sync::Mutex",
]
```

Use after deciding the project standardizes on `FxHashMap`, `parking_lot::Mutex`, or similar. Without this, it is easy to reintroduce the default type by accident.

## Anti-patterns

1. Treating Clippy as a profiler; lint-free code can still be slow.
2. Blanket `#[allow(...)]` to silence a lint instead of understanding it.
3. Applying a lint fix that makes hot code slower (measure).
4. Ignoring `-W clippy::perf` because "the code is not hot"; perf lints usually also simplify the code.
5. Disallowing types without documenting the reason, confusing new contributors.
6. Running Clippy only on the library and not on tests/benches.
7. Denying all warnings in CI without a pinned Clippy version; new lints break builds.
8. Not formatting after applying lint fixes (`cargo fmt`).

## Checklist

- Does `cargo clippy` pass, and are warnings addressed or justified?
- Was `-W clippy::perf` reviewed explicitly?
- Are hot-path lint fixes verified with a benchmark?
- Is `clippy.toml` used consistently if alternative types are standardized?
- Are `#[allow]` attributes scoped and explained?
- Is Clippy run on all targets (`--all-targets`)?
- Is the Clippy version pinned in CI so new lints do not break the build unexpectedly?

## Related

- `hashing.md` for `disallowed-types` on maps.
- `std.md` for `parking_lot` and standard type choices.
- `heap.md` for the allocation patterns behind several lints.
- `benchmarking.md` for verifying that a lint fix actually helps.
