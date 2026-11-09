---
name: rust-optimization
description: Rust performance work - benchmarking (Criterion/divan/hyperfine), profiling (perf, samply, flamegraph, DHAT, heaptrack), release profile and LTO/PGO, linting, inlining, hashing, heap allocations, type sizes, allocation and clone reduction, data layout, iterators and bounds checks, strings, I/O, wrapper types, concurrency (rayon, atomics, locks), SIMD, async/Tokio pitfalls, and compile time/binary size. Use when Rust code is slow, uses too much memory, allocates heavily, or when choosing data structures or algorithms.
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
  source: Generated using Rust Optimization recommendations.
---

# Rust Optimization

How to make Rust code fast. This skill owns **runtime performance, memory behavior, and the build/operational causes behind slow binaries**. The compiler and hardware explain *why* code is slow: allocations, copies, cache misses, branch mispredictions, bounds checks, lock contention, blocking in async tasks, missed inlining or vectorization, and suboptimal build settings.

## When to activate

- A function, endpoint, or job is slow, or latency/throughput targets are missed.
- Memory use is high, grows over time, or allocation count is excessive.
- Choosing a data structure, collection, algorithm, or concurrency model.
- A hot loop needs review (iterators, bounds checks, inlining, vectorization).
- An async service stalls, has high tail latency, or poor CPU utilization.
- Compile times or binary size are a problem.

Do not optimize code that is not on a measured hot path. A clear, correct solution that meets its target is the right solution.

## Where to start

The guides in `references/` are the source material for this skill. Each is a short, task-oriented note with checks, before/after code, and a checklist; consult only the guide that matches the question. Verify version-specific behavior against the toolchain in `rust-toolchain.toml` or `Cargo.toml` (`rust-version`).

| Question | Read |
|---|---|
| How do I benchmark and choose metrics? | [benchmarking.md](references/benchmarking.md) |
| How do I configure release, LTO, PGO, allocators, binary size, or compile times? | [buildconfig.md](references/buildconfig.md) |
| Which Clippy lints find performance problems? | [linting.md](references/linting.md) |
| How do I profile and get usable symbols? | [profiling.md](references/profiling.md) |
| Is inlining helping or hurting? | [inlining.md](references/inlining.md) |
| Which hasher should I use? | [hashing.md](references/hashing.md) |
| Why is allocation or cloning hot? | [heap.md](references/heap.md) |
| Is my hot type too large? | [type-sizes.md](references/type-sizes.md) |
| Which standard library method is cheaper? | [std.md](references/std.md) |
| Are iterators, `collect`, or bounds checks the problem? | [iterators.md](references/iterators.md) |
| What does this collection/iterator cost? | [iterators-cost.md](references/iterators-cost.md) |
| Is I/O, logging, or assertion code slow? | [logging-debugging.md](references/logging-debugging.md) |
| Are wrapper types adding overhead? | [wrapper.md](references/wrapper.md) |
| What does the generated machine code look like? | [machinecode.md](references/machinecode.md) |
| How do I parallelize safely? | [parallelism.md](references/parallelism.md) |
| What are the general optimization principles? | [generaltips.md](references/generaltips.md) |
| How do I reduce compile time or binary size? | [compiling.md](references/compiling.md) |
| Why is this allocation or layout costly? | [rustonomicon.md](references/rustonomicon.md) |
| What is this unsafe/atomic ordering rule? | [rustonomicon.md](references/rustonomicon.md) |
| Why is the async task blocked? | [async.md](references/async.md) |

## Workflow

1. **Reproduce and measure.** Capture the workload, input sizes, hardware, toolchain, and build profile. Establish a baseline with a benchmark and a profile. Always measure with `--release`.
2. **Find where the time or memory goes.** Read the flamegraph or profile top-down: the widest frames are the candidates. For memory, count allocations and peak bytes, not only RSS.
3. **Explain the cause.** Classify it: algorithmic complexity, allocation/copy, data layout/cache, bounds checks or missed vectorization, inlining, hashing, lock contention, blocking in async, I/O or syscalls, logging, serialization, or build configuration.
4. **Change one thing.** Prefer an algorithmic or data-structure fix before micro-optimizations; prefer safe code before `unsafe`; prefer build configuration before code changes when it is sufficient; use Clippy for obvious waste, but verify with measurement.
5. **Verify.** Re-run the same benchmark on the same machine and compare. Confirm correctness with the test suite. Do not accept a win that is not measured, and do not trust a single noisy run.

## Measure first

### Build profile

Benchmarks and profiling must use an optimized build, with debug symbols for readable profiles.

```toml
# Cargo.toml
[profile.release]
opt-level = 3
lto = "thin"         # "fat" is more aggressive; false keeps thin local LTO; "off" disables
codegen-units = 1    # better optimization, slower compile
panic = "abort"      # smaller/faster, but no unwinding: verify it is acceptable
strip = "symbols"    # smaller binary; do NOT use when profiling

# Profile with symbols without changing release behavior
[profile.profiling]
inherits = "release"
debug = "line-tables-only"
strip = "none"
```

- **Release vs dev is the first check.** Dev builds are unoptimized and 10–100x slower; `cargo build` and `cargo run` use the dev profile.
- Cargo profile settings at the workspace root apply; profile settings in dependencies are ignored.
- LTO forms: `lto = false` keeps thin local LTO, `lto = "thin"` is more aggressive, `lto = "fat"` is most aggressive, and `lto = "off"` disables it. Fat LTO sometimes does not improve speed; measure.
- `codegen-units = 1`, LTO, and `panic = "abort"` trade compile time (and unwinding behavior) for runtime speed or size; benchmark each separately.
- `strip = "symbols"` may reduce debuggability and profile quality; use the profiling profile when investigating.
- `debug = "line-tables-only"` gives source-line info for profilers without full debug info. Shipped standard libraries do not include debug info, so standard-library frames may remain opaque.
- Frame pointers may be optimized away. Use `RUSTFLAGS="-C force-frame-pointers=yes"` when stack traces are poor.
- If profiler symbols look mangled (`_ZN`, `_R`), try `-C symbol-mangling-version=v0` and `rustfilt`.
- `-C target-cpu=native` enables CPU-specific instructions and vectorization but makes the binary non-portable. Use it only when deployment hardware is known; verify detection with `rustc --print cfg` and `rustc --print cfg -C target-cpu=native`.
- Profile-guided optimization (`cargo-pgo`) and BOLT can give 10%+ gains on large, branchy binaries; measure and keep the training workload representative.
- `cargo-wizard` can generate a reasonable profile if you do not want to tune every setting.
- Check whether the default allocator is the bottleneck: `mimalloc` or `jemalloc` (`tikv-jemallocator`) can help allocation-heavy or multi-threaded workloads, at the cost of binary size and compile time. Benchmark before adopting; behavior varies by platform. On Linux, jemalloc can also use transparent huge pages via `MALLOC_CONF`.

```toml
[dependencies]
mimalloc = "0.1"
# or: tikv-jemallocator = "0.5"
```

```rust
#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

### Benchmarks

Use a statistics-aware harness; `cargo bench` with plain timing loops is misleading.

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_parse(c: &mut Criterion) {
    let input = include_str!("../fixtures/sample.txt");
    c.bench_function("parse_sample", |b| {
        b.iter(|| parse(black_box(input)))
    });
}

criterion_group!(benches, bench_parse);
criterion_main!(benches);
```

- Use `black_box` on inputs and results so the optimizer cannot delete the work.
- Benchmark with realistic input sizes and several sizes (for example 10³, 10⁴, 10⁵) to see the growth curve and confirm the complexity.
- Rust's built-in benchmark tests use unstable features and require nightly. Criterion and divan are more flexible; hyperfine is good for whole-program CLI timing; Bencher supports continuous benchmarking; Gungraun provides high-precision `cargo bench` integration; `iai-callgrind` gives stable instruction counts.
- Use real workloads where possible; microbenchmarks and stress tests are useful in moderation.
- Wall time matches user perception but has high variance; cycles or instruction counts are lower-variance alternatives. Report the metric you used.
- Keep setup out of the measured closure (`iter_batched` for per-iteration inputs).
- Run on an idle machine; pin frequency/CPU if possible; compare medians and confidence intervals, not single runs.
- Mediocre benchmarking is better than none; improve the harness as you learn the workload.

### Profilers

| Tool | Use for |
|---|---|
| `perf`, Hotspot, Firefox Profiler, `cargo flamegraph` | CPU hot spots on Linux |
| Instruments | General profiling on macOS |
| Intel VTune, AMD uProf | General profiling on Windows/Linux/macOS |
| `samply` | Cross-platform sampling profiler with a web UI |
| `perf stat` | Cycles, instructions, cache misses, branch misses |
| Cachegrind / Callgrind | Instruction counts, simulated cache and branch data, source-line attribution |
| DHAT / `dhat-rs` / `heaptrack` / `bytehound` | Allocation counts, sizes, lifetimes, peak memory, hot `memcpy` |
| `counts` | Ad hoc profiling via `eprintln!` plus frequency post-processing |
| Coz / `coz-rs` | Causal profiling and optimization potential |
| `valgrind --tool=callgrind` | Instruction-level profiles |
| `cargo-llvm-lines` | Which generics produce the most LLVM IR (compile time, binary size) |
| `cargo-bloat` | What takes space in the binary |
| `cargo asm` / `cargo-show-asm` / Compiler Explorer | Check inlining, vectorization, bounds checks |
| `tokio-console` / `tracing` | Async task behavior, poll times, stalls |
| `cargo build --timings` | Where compile time goes |

### Reading a profile

| Symptom | Likely cause | First checks |
|---|---|---|
| Wide frame in `malloc`/`free`/`realloc` | Allocation in a hot loop; growing `Vec`/`String` | `with_capacity`, reuse buffers, avoid `collect` per iteration |
| Wide frame in `memcpy`/`clone` | Unnecessary copies or deep clones; hot type over 128 bytes | Borrow instead of clone; `Cow`; `Arc`/`Rc`; shrink types |
| Time in `core::hash`/`SipHash` | Default hasher on hot lookups | `FxHashMap`/`ahash`/`fnv` when HashDoS is not a concern |
| Time in `Mutex::lock`/futex | Lock contention | Narrow critical sections, sharding, atomics, channels |
| High cache-miss rate (`perf stat`) | Pointer chasing, poor layout, large structs | Contiguous storage, struct-of-arrays, smaller types |
| High branch-miss rate | Unpredictable branches in a hot loop | Branchless forms, sorting input, lookup tables |
| `panic_bounds_check` frames | Index bounds checks in hot loops | Iterators, slices with known length, `get_unchecked` only as a last resort |
| Time in `fmt`/`format!`/`to_string` | Formatting in hot paths or logging | Avoid formatting on the fast path; lazy logging |
| Time in `serde`/parsing | Serialization or text parsing cost | Zero-copy borrowing, binary formats, `simd-json` |
| Executor threads idle while tasks wait | Blocking call inside async code | `spawn_blocking`, async I/O, `tokio-console` |
| Many tiny syscalls | Unbuffered I/O or `print!` per line | `BufReader`/`BufWriter`, lock stdout once, batching |
| Iterator chain slower than expected | `chain` or layered adapters prevent optimization | Simplify; `filter_map`; try `iter().copied()` for small `Copy` types |
| Hot `assert!` or debug-only work | Checks or logging run in release | `debug_assert!`; guard logging and data collection |

## Algorithms first

The largest wins are asymptotic. Before tuning constants, check the complexity of the dominant loop.

- Replace nested loops over the same data with a `HashSet`/`HashMap` lookup (O(n²) → O(n)), sorting plus two pointers, or binary search (`slice::binary_search`, `partition_point`) on sorted data.
- Use `BinaryHeap` for repeated min/max, top-k, or scheduling; use `select_nth_unstable` when only the k-th element is needed.
- Use `VecDeque` for queues; `Vec::remove(0)` and `Vec::insert(0, _)` are O(n).
- Use `BTreeMap`/`BTreeSet` for ordered iteration and range queries; they are often faster than hash maps for small or ordered workloads because of cache behavior.
- Avoid `Vec::contains`/`iter().position()` inside loops; use a set or index.
- Memoize or use dynamic programming for overlapping subproblems; precompute prefix sums or lookup tables for repeated range queries.
- Use `sort_unstable*` when stability is not required; it is generally faster and does not allocate. Use `sort_by_cached_key` when the key is expensive.
- Prefer `retain`, `drain`, `swap_remove`, and `dedup` over rebuilding collections when order permits.
- Handle common cases first: special-case empty, one-element, and two-element collections when those dominate; use compact representations for repetitive data; put a small cache in front of a high-locality lookup.
- Avoid computing things until needed; lazy and on-demand computation is often a win.

For complexity analysis and algorithm selection in depth, see the `code-complexity-analysis` skill.

## Linting

Clippy finds many common performance mistakes; its performance lints are enabled by default.

```bash
cargo clippy
cargo clippy -- -W clippy::perf   # explicit, when you want perf lints highlighted
```

- Performance lint suggestions usually make code simpler and more idiomatic, so follow them even outside hot paths unless there is a measured reason not to.
- Some non-performance lints improve performance too. For example, `ptr_arg` suggests `&mut [T]` instead of `&mut Vec<T>`, which can reduce indirection and help optimization.
- If you adopt alternative types (faster hashers, `parking_lot` locks), use `clippy.toml` to prevent accidental use of the standard types:

```toml
disallowed-types = ["std::collections::HashMap", "std::collections::HashSet"]
```

- Clippy is not a profiler. Fixing every lint does not replace measuring the hot path.

## Memory and allocation

Allocation is often the dominant hidden cost in Rust. Count allocations in hot paths.

### Reduce allocations

```rust
// BAD: grows repeatedly, reallocating
let mut out = Vec::new();
for x in items { out.push(transform(x)); }

// GOOD: reserve once
let mut out = Vec::with_capacity(items.len());
for x in items { out.push(transform(x)); }

// GOOD: reuse a buffer across iterations
let mut buf = String::new();
for line in lines {
    buf.clear();
    write!(&mut buf, "{}:{}", a, b)?;
    sink(&buf);
}
```

- Use `Vec::with_capacity`, `String::with_capacity`, `HashMap::with_capacity`, and `reserve`/`reserve_exact` when the size is known or estimable. `Vec` growth is quasi-doubling (0, 4, 8, 16, ...), so a known length can avoid several reallocations.
- `clone_from` may reuse an existing allocation: `dst.clone_from(&src)` is not just `dst = src.clone()`.
- Reuse buffers with `clear()` instead of reallocating in loops; keep scratch buffers in a struct. `Vec::shrink_to_fit` can reduce wasted capacity but may reallocate.
- `collect::<Vec<_>>()` inside a hot loop allocates every time; iterate lazily or collect into a reused buffer (`extend`).
- Return iterators (`impl Iterator`) instead of `Vec` when the caller only iterates. If you write a custom iterator, implement `size_hint` or `ExactSizeIterator::len` when possible so `collect` can allocate once.
- Small, short-lived collections can avoid the heap with `smallvec`, `arrayvec`, or `tinyvec`; `SmallString`/`smartstring` can avoid small string allocations. Measure: they add branching and larger stack frames.
- `Box<dyn Trait>` per item allocates; prefer enums or generics when the set of types is closed or the call is hot.
- Avoid `Box`, `Rc`, `Arc` wrappers on small values that fit inline; `Rc`/`Arc` on rarely shared values can increase allocations.
- Use arenas (`bumpalo`, `typed-arena`) for many short-lived nodes with the same lifetime (ASTs, graphs), freeing them together.
- `BufRead::lines()` allocates a `String` per line. Reuse a line buffer instead:

```rust
let mut line = String::new();
while reader.read_line(&mut line)? != 0 {
    process(&line);
    line.clear();
}
```

- `format!` allocates; `format_args!` or `lazy_format` can defer formatting when the result may not be used.
- `dhat-rs` supports heap-usage tests that assert allocation counts/bytes for a code snippet, catching regressions.

### Avoid unnecessary copies and clones

- Accept `&str`, `&[T]`, and `&Path` rather than `String`, `Vec<T>`, and `PathBuf` when you do not need ownership.
- Take `impl AsRef<str>`/`impl Into<String>` only when the API genuinely benefits; otherwise generics can bloat code.
- Use `Cow<'a, str>` when most paths borrow and only some need to own.
- Share immutable data with `Arc<T>`/`Arc<[T]>`/`Arc<str>` instead of cloning large structures; clone the `Arc` (a refcount increment), not the data.
- `Rc::make_mut`/`Arc::make_mut` provide clone-on-write: they clone the inner value only when the refcount is greater than one.
- Pass large structs by reference; moving large values is a `memcpy`. Rust types larger than 128 bytes are copied with `memcpy` rather than inline code, so shrinking hot types is worthwhile.
- Box huge enum variants (Clippy: `large_enum_variant`); the extra allocation is usually worth it when the large variant is rare.
- Avoid `.clone()` to satisfy the borrow checker without thought; restructure borrows, split structs, use `mem::take`/`mem::replace`/`Option::take`, `clone_from`, or use indices instead of references.
- `to_vec()`, `to_owned()`, `to_string()`, and `format!` allocate; check that each is needed.
- Clippy lints that find waste: `redundant_clone`, `needless_collect`, `unnecessary_to_owned`, `large_enum_variant`, `inefficient_to_string`, `manual_memcpy`. Run `cargo clippy -- -W clippy::perf`.

### Strings

- `String` concatenation with `+` in loops reallocates; use `push_str` on a pre-sized `String`, `write!`, or `join`.
- Prefer `&str` slicing over building new strings; `split`, `lines`, and `trim` return borrowed slices without allocating.
- `chars().count()`, `chars().nth(n)`, and indexing by character are O(n) because UTF-8 is variable width. Work with bytes (`as_bytes()`, `b"..."`) when the data is ASCII and you have validated that.
- `to_lowercase()`/`to_uppercase()` allocate; use `eq_ignore_ascii_case` for ASCII comparisons.
- Parse numbers with `str::parse` or specialized crates (`atoi`, `lexical`); avoid regex for simple parsing. Compile `Regex` once (`std::sync::LazyLock`/`OnceLock`/`once_cell`), never inside a loop.
- Use interning (`string-interner`, `lasso`) when many duplicated strings are compared often.

### Standard library type tips

- `vec![0; n]` is the best way to create a zero-filled `Vec`; it can use OS assistance and is usually as fast as or faster than `resize`, `extend`, or unsafe alternatives.
- `Vec::remove(i)` is O(n) because it shifts elements; `Vec::swap_remove(i)` is O(1) when order does not matter. `retain` removes many items efficiently.
- `Vec::into_boxed_slice()` reduces capacity waste and a `Box<[T]>` uses two words instead of `Vec`'s three. `ThinVec` stores length/capacity with the elements and uses one word, which is useful for many empty vectors in hot types.
- `Option::ok_or`/`Option::map_or`/`unwrap_or` evaluate their fallback eagerly; use `ok_or_else`, `map_or_else`, and `unwrap_or_else` when the fallback is expensive.
- `Rc::make_mut`/`Arc::make_mut` are useful for clone-on-write, but only when sharing is rare enough.
- Multiple `Arc<Mutex<T>>` or `RefCell<T>` fields that are usually accessed together may be cheaper as one wrapper around a tuple; measure access patterns.

## Type sizes and data layout

- `repr(Rust)` automatically reorders struct and enum fields to minimize size; you do not need to order fields manually unless you use `#[repr(C)]`, which disables reordering.
- Check sizes with `std::mem::size_of::<T>()` and `size_of_val`. Use `-Zprint-type-sizes` on nightly to see field order, padding, discriminants, and per-variant sizes; `top-type-sizes` gives a compact view.
- `Vec<T>` (contiguous) beats `LinkedList` and pointer-heavy trees for nearly all workloads because of cache locality. `LinkedList` is almost never the right choice.
- Prefer arrays of structs of small, hot fields; use **struct-of-arrays** when a loop touches only one or two fields of a large struct.
- Use the smallest integer type that fits when memory bandwidth matters (`u32` indices instead of `usize`, `u8` flags); keep arithmetic types consistent to avoid conversion cost.
- `Option<Box<T>>`, `Option<&T>`, `Option<NonZeroU32>` use niche optimization and take no extra space; plain `Option<u32>` does.
- Box rare large enum variants to shrink the enum; consider `ThinVec` when an oversized variant contains a `Vec`.
- Use index-based graphs/trees (`Vec<Node>` with `usize`/`u32` indices) instead of `Rc<RefCell<Node>>` graphs: better locality, no refcount overhead, no borrow-check panics at runtime.
- Avoid false sharing in concurrent code: pad hot atomics/counters per thread with `crossbeam_utils::CachePadded`.
- Large lookup tables and hot loops should fit in L1/L2 cache when possible; block (tile) algorithms that stride through large arrays.
- Protect hot type sizes with static assertions, scoped to a platform:

```rust
#[cfg(target_arch = "x86_64")]
static_assertions::assert_eq_size!(HotType, [u8; 64]);
```

## Hashing

`HashMap`/`HashSet` are common hot spots. The default hasher is not specified, but at the time of writing it is SipHash 1-3: high quality and DoS-resistant, but relatively slow, especially for short integer keys.

| Need | Prefer | Notes |
|---|---|---|
| Resistant hashing for untrusted input | Default `HashMap`/`HashSet` | SipHash, slower but safer |
| Fast hash for trusted keys | `rustc_hash::FxHashMap`/`FxHashSet` | Very fast, low-quality; not HashDoS-resistant |
| Slightly higher quality than Fx | `fnv::FnvHashMap`/`FnvHashSet` | A little slower than Fx |
| AES-assisted hashing | `ahash::AHashMap`/`AHashSet` | Can use AES instructions; measure |
| Integer newtype with random/low-collision values | `nohash_hasher` | Avoids hashing entirely |
| Ordered iteration or ranges | `BTreeMap`/`BTreeSet` | Cache-friendly for small/ordered data |
| Small maps (≲ 16 entries) | `Vec<(K, V)>`, sorted vec, or `smallvec` | Linear search often beats hashing |

- Use alternative hashers only when profiling shows hashing is hot and HashDoS is not a concern.
- `HashMap::entry` avoids double lookups (`contains_key` then `insert`). Pre-size maps with `with_capacity` to avoid rehashing.
- Hashing large keys is expensive; hash a derived small key (ID, prefix) when valid.
- Byte-wise hashing can be faster for `#[derive(Hash)]` types with no padding, using `zerocopy`/`bytemuck` `ByteHash` or `derive_hash_fast`; this is advanced and depends on the hash function and layout. Measure.
- Do not iterate `HashMap` expecting stable order; use `BTreeMap` or `IndexMap` when order matters.
- Use `clippy.toml` `disallowed-types` if you standardize on an alternative hasher.

## Collections

| Need | Prefer | Notes |
|---|---|---|
| Growable sequence | `Vec<T>` | Reserve capacity; amortized O(1) push |
| Queue / deque | `VecDeque<T>` | O(1) at both ends |
| Hash lookup | `HashMap`/`HashSet` | Default hasher is slow for trusted short keys |
| Fast hash for trusted keys | `FxHashMap`, `ahash::AHashMap`, `fnv` | Not HashDoS-resistant |
| Ordered map/set, range queries | `BTreeMap`/`BTreeSet` | Cache-friendly; good for small/ordered data |
| Small maps (≲ 16 entries) | `Vec<(K, V)>` linear search, `smallvec`, or a sorted vec | Often faster than hashing |
| Priority queue | `BinaryHeap<T>` | Use `Reverse` for a min-heap |
| Dense integer keys | `Vec<Option<T>>` or `slotmap`/`slab` | Index lookup, no hashing |
| Bit sets | `bitvec`, `fixedbitset` | Compact membership |
| Concurrent map | `dashmap`, `papaya`, or a sharded `Mutex<HashMap>` | Benchmark; contention dominates |

## Iterators, loops, and bounds checks

Iterator chains compile to tight loops and usually match hand-written loops. They also let the optimizer remove bounds checks and vectorize.

```rust
// Usually preferable: no bounds checks, easy to vectorize
let sum: u64 = data.iter().map(|&x| u64::from(x)).sum();

// Index loop: bounds check per access unless the compiler proves safety
let mut sum = 0u64;
for i in 0..data.len() { sum += u64::from(data[i]); }
```

- Prefer `iter()`, `iter_mut()`, `zip`, `windows`, `chunks`, and `chunks_exact` over indexing.
- `collect` allocates; do not collect into a `Vec` only to iterate it again. Return `impl Iterator` or use `extend`.
- `chain` can be slower than a single iterator; `filter_map` is often faster than `filter` followed by `map`.
- Use `chunks_exact` when the chunk size divides the length; handle `remainder()` explicitly. Related: `rchunks_exact`, `chunks_exact_mut`, `rchunks_exact_mut`.
- For small `Copy` types, `iter().copied()` can let LLVM generate better code than `iter()`; verify in assembly if it matters.
- When indexing is unavoidable, hoist a slice so the compiler knows the length: `let data = &data[..n];`, or assert lengths once (`assert!(a.len() == b.len())`) before the loop.
- `get_unchecked` removes checks but is `unsafe`; use it only after profiling shows the check matters, with a documented safety invariant, tests, and ideally Miri. See `references/iterators.md` for safe techniques.
- `iter().rev()`, `skip`, and `step_by` are cheap; `nth(n)` on non-random-access iterators is O(n).
- Overflow checks are on in debug builds and off in release by default; use `wrapping_*`, `checked_*`, or `saturating_*` explicitly where semantics matter, and never rely on release-mode wraparound.
- Floating-point reductions are not auto-vectorized in the same way as integer sums because reordering changes results; manual unrolling, multiple accumulators, or SIMD crates can help.

### SIMD

- Start with auto-vectorization: simple iterator loops over contiguous slices with `chunks_exact`.
- For explicit control use `std::simd` (nightly), `wide`, `pulp`, or `std::arch` with runtime feature detection (`is_x86_feature_detected!`) and a scalar fallback.
- Prefer well-tested crates for hot primitives (`memchr`, `simd-json`, `bytecount`, `aho-corasick`) over hand-written intrinsics.
- Confirm vectorization in the generated machine code; do not assume it.

## Inlining and machine code

Inlining removes call overhead and enables further optimization, but it can also bloat code and increase compile time.

| Attribute | Meaning |
|---|---|
| none | Compiler decides |
| `#[inline]` | Suggests inlining |
| `#[inline(always)]` | Strongly suggests inlining (not an absolute guarantee) |
| `#[inline(never)]` | Strongly suggests not inlining |

- Best candidates: very small functions and functions with a single call site. The compiler often handles these without help.
- Inlining is non-transitive: if `f` calls `g` and both should be inlined at a call site of `f`, both may need attributes.
- Use `#[inline]` for small cross-crate functions that are not generic; generics are monomorphized, but cross-crate inlining still depends on the compiler.
- `#[inline(always)]` is rarely justified. Measure: it can bloat code, increase compile time, and sometimes slow things down by preventing other inlining.
- When one call site is hot and others are cold, split into an always-inlined hot variant and a never-inlined cold wrapper. Mark rarely executed helpers `#[cold]` (outlining) to keep the hot path small.
- Use `#[inline(never)]` on cold/error paths, not on hot functions.
- Cachegrind can show whether a function was inlined: inlined functions lack event counts on their first and last lines.
- Inspect generated code with `cargo asm`/`cargo-show-asm` or Compiler Explorer; check for bounds checks, vectorization, and unexpected calls. `core::arch` exposes architecture-specific intrinsics.

## Concurrency and parallelism

- Data parallelism: `rayon` (`par_iter`, `par_chunks`, `join`) is the default choice for CPU-bound loops. Gains need enough work per item; tiny closures are dominated by scheduling overhead. Tune with `with_min_len`.
- Reduce shared mutable state. Prefer message passing (`std::sync::mpsc`, `crossbeam-channel`, `flume`) or per-thread accumulators merged at the end (`fold`/`reduce`).
- Locks: keep critical sections short; never hold a lock across I/O or `.await` (use `tokio::sync::Mutex` only when you must hold across an await, and prefer restructuring).
- `parking_lot` mutexes can be faster and smaller than `std` ones, but std has improved; benchmark before switching.
- Use `RwLock` only when reads dominate heavily; it can be slower than a `Mutex` with short sections and can starve writers.
- Atomics: `Ordering::Relaxed` suffices for counters; use `Acquire`/`Release` for publishing data; `SeqCst` only when required. Wrong ordering is a correctness bug, not a performance setting. Use `loom` to test lock-free code.
- Shard hot shared structures (per-thread or per-shard counters/maps) to avoid contention and false sharing.
- Thread count: do not oversubscribe cores; mixing `rayon` with other thread pools or blocking in the pool causes stalls.
- `Arc::clone` in hot loops costs atomic increments; clone once outside the loop or pass references.

## Async and Tokio

- **Never block the executor.** CPU-heavy work, blocking I/O, `std::thread::sleep`, `std::fs`, and synchronous locks held for long periods stall every task on that worker. Use `tokio::task::spawn_blocking`, a dedicated rayon pool, or async equivalents (`tokio::fs`, `tokio::time::sleep`).
- A task that runs for a long time without `.await` starves others; insert `tokio::task::yield_now().await` or move the work to a blocking thread.
- Use `tokio-console` and `tracing` spans to find tasks with long poll times.
- Avoid spawning a task per tiny unit of work; batch or use bounded concurrency (`futures::stream::buffer_unordered`, `tokio::sync::Semaphore`).
- Use bounded channels for backpressure; unbounded channels hide overload and grow memory.
- Avoid holding `std::sync::MutexGuard` across `.await` (it makes the future `!Send` and risks deadlock).
- Large futures are copied/moved; `Box::pin` big futures, and avoid very deep `async` recursion.
- Prefer `Arc<str>`/`Bytes` (`bytes` crate) for cheap cloning of network buffers.
- Choose the runtime flavor deliberately: `current_thread` for simple/low-latency tasks, multi-threaded for throughput. Tune `worker_threads`.
- Database and HTTP clients: reuse connections through pools (`deadpool`, `bb8`, `sqlx::Pool`, `reqwest::Client`); creating a client per request is a common, severe cost.

## I/O, logging, and serialization

- `print!`/`println!` lock stdout on every call. For repeated output, lock once and use `writeln!`; combine with `BufWriter` for many writes.
- stdin and stderr can likewise be locked for repeated operations.
- Rust file I/O is unbuffered by default. Wrap files and sockets in `BufReader`/`BufWriter`; unbuffered per-line or per-byte I/O causes syscalls. Flush explicitly where errors matter (Drop ignores flush errors).
- `read_to_string`/`read_to_end` on large inputs load everything into memory; stream with `BufRead::read_line` into a reused `String`, or memory-map (`memmap2`, `unsafe` with documented invariants) for large read-only files.
- `BufRead::lines()` allocates per line; use `read_line` with a reusable buffer when the body only needs `&str`.
- For ASCII/byte-oriented input, `BufRead::read_until` avoids UTF-8 validation overhead; crates like `bstr` or byte-line readers help.
- Serialization: `serde` with `#[serde(borrow)]` and `&str`/`Cow<str>` fields gives zero-copy deserialization when the input outlives the data. Binary formats (`bincode`, `postcard`, `rkyv`, `prost`) are smaller and faster than JSON; use `simd-json` or `sonic-rs` when JSON is required and hot.
- Avoid deserializing into `serde_json::Value` when a typed struct works; `Value` allocates per node.
- Avoid repeated `format!`/`println!` in hot loops; use `write!` into a reused buffer.
- Logging and debugging code can be surprisingly expensive. Ensure data collection and formatting do not run when logging is disabled; use `tracing` filters and macros rather than pre-formatted strings.
- `assert!` always runs; `debug_assert!` only runs in dev builds. If a hot assertion is not needed for safety, use `debug_assert!`.

## Wrapper types

`RefCell`, `Mutex`, `RwLock`, and similar wrappers add non-trivial access cost, including borrow checks, refcounts, and locking.

- If several wrapped values are usually accessed together, put them in one wrapper: `Arc<Mutex<(u32, u32)>>` instead of two `Arc<Mutex<u32>>` fields. This reduces lock/unlock and refcount overhead.
- Conversely, do not merge values with independent access patterns; that increases contention and critical-section size.
- Prefer plain ownership and borrowing when the runtime check or lock is unnecessary.

## Errors, dispatch, and generics

- `Result` and `Option` are cheap; `Box<dyn Error>`/`anyhow::Error` allocate on the error path only, which is usually fine, but avoid constructing errors with `format!` in hot success-adjacent paths.
- Use lazy combinators (`ok_or_else`, `map_or_else`, `unwrap_or_else`, `or_else`) when the fallback is expensive.
- Static dispatch (generics, `impl Trait`) enables inlining; dynamic dispatch (`dyn Trait`) costs an indirect call and prevents inlining. Use `dyn` where the flexibility matters or to reduce code bloat, and measure when it sits in a hot loop.
- Heavy generics increase compile time and binary size (monomorphization). Use `cargo-llvm-lines`, and extract a non-generic inner function (`fn inner(&str)`) from a generic outer one to limit duplication.
- `panic!`/`unwrap` paths are cold; mark rarely taken paths `#[cold]` and keep hot code small to help inlining.
- `Mutex<T>` poisoning, `RefCell` borrow checks, and `Rc` refcounts have runtime cost; prefer ownership and borrowing at compile time where possible.

## Unsafe code

- `unsafe` is not a default speed tool. Safe iterators, `chunks_exact`, and slice patterns often reach the same machine code.
- Allowed only when: a profile shows the safe version is the bottleneck, the invariant is stated in a `// SAFETY:` comment, the unsafe block is as small as possible, and tests plus Miri (`cargo +nightly miri test`) cover it.
- Prefer a vetted crate (`memchr`, `bytes`, `crossbeam`) over writing unsafe primitives yourself.
- `MaybeUninit`, `get_unchecked`, `from_utf8_unchecked`, and `transmute` each need explicit justification and a benchmark showing the benefit.

## Compile time and binary size

- `cargo build --timings` shows the critical path and crate-graph parallelism; `cargo-llvm-lines` shows generic bloat.
- Reduce dependencies and disable unused features (`default-features = false`); inspect with `cargo tree -d` and `cargo-udeps`.
- Use a faster linker (`mold`, `lld`), disable debug info outside profiling builds, and use `cargo check` during development; split large crates into a workspace so incremental compilation can reuse work.
- `-Zmacro-stats` (nightly) shows how much code macros generate; `cargo-expand` shows the expanded code. Large proc macros can dominate compile time.
- LLVM IR size affects backend compile time; reduce unnecessary monomorphization and generic instantiations.
- Use `opt-level = "s"`/`"z"`, LTO, strip, and `panic = "abort"` for small binaries; check with `cargo-bloat`.
- Dev-profile dependencies can be optimized while your crate stays fast to compile: `[profile.dev.package."*"] opt-level = 3`.
- Release builds with `codegen-units = 1` and fat LTO are slower to build; keep a separate CI/profile for quick checks.
- The experimental parallel front-end and Cranelift backend may improve compile times; evaluate them against your toolchain.
- `cargo-wizard` can generate sensible profiles; `cargo-pgo` helps with PGO/BOLT.

## General tips

- Optimize only hot code; optimized code is often more complex and harder to maintain.
- The biggest wins usually come from algorithms and data structures, not low-level tweaks.
- Eliminate silly slowdowns before introducing clever speedups.
- Avoid computing things unless necessary; lazy/on-demand work often wins.
- Special-case common inputs (0, 1, 2 elements; common enum variants; frequent values) and handle the most frequent cases first.
- For repetitive data, a compact representation for common values plus a fallback table can save memory and time.
- A small cache in front of a high-locality lookup can help.
- Most optimizations are small; many small measured wins add up.
- Use more than one profiler: they have different strengths.
- When a function is hot, either make it faster or call it less.
- Keep comments that explain non-obvious optimized code, especially those citing profile evidence (for example, "99% of the time this vector has 0 or 1 elements, so handle those first").

## Common anti-patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Benchmarking a dev build | 10–100x slower, misleading | `--release` / `profile.profiling` |
| `.clone()` to quiet the borrow checker | Hidden allocation/copy | Restructure borrows, `Arc`, `Cow`, `mem::take`, `clone_from` |
| `Vec::new()` + push in a loop with known size | Repeated reallocation | `with_capacity`/`reserve_exact` |
| `for i in 0..v.len() { v[i] }` | Bounds checks, missed vectorization | Iterators, slice hoisting |
| `String` building with `+`/`format!` in a loop | Many allocations | `push_str`/`write!` into a reused buffer; `format_args!` |
| `collect()` only to iterate again | Needless allocation | Return `impl Iterator` or use `extend` |
| `Regex::new` inside a loop or request handler | Recompiles every call | Compile once with `LazyLock`/`OnceLock` |
| `HashMap` with default SipHash for trusted hot keys | Slow hashing | `FxHashMap`/`ahash`/`fnv` (only for trusted keys) |
| `Vec::contains` in a loop | O(n²) | `HashSet`, sort + binary search |
| `Vec::remove(0)` / `insert(0, _)` | O(n) per op | `VecDeque`, `swap_remove` |
| Hot type larger than 128 bytes | `memcpy` copies, cache pressure | Shrink, box rare variants, `ThinVec`, boxed slice |
| Many `Arc<Mutex<T>>` fields accessed together | Extra locks/refcounts | One wrapper around a tuple |
| `BufRead::lines()` in a hot loop | Allocation per line | `read_line` with reused `String` |
| `print!`/`println!` per line | Locks stdout every call | Lock once and use `writeln!`; buffer |
| Unbuffered file/socket I/O | Syscall per small write/read | `BufReader`/`BufWriter` |
| Hot `assert!` or expensive debug data | Checks/formatting in release | `debug_assert!`; guard logging |
| `serde_json::Value` for known schemas | Allocation per node | Typed structs, borrowed fields |
| Blocking call inside `async fn` | Stalls the executor | `spawn_blocking`, async APIs |
| Lock held across `.await` or I/O | Contention, deadlocks | Narrow scope; copy data out before awaiting |
| `Rc<RefCell<T>>` graphs | Pointer chasing, runtime borrow panics | Index-based arena/`slotmap` |
| New HTTP/DB client per request | Connection setup cost | Shared pooled client |
| `#[inline(always)]` everywhere | Code bloat, compile time, worse inlining elsewhere | Measure; use `#[inline]` selectively |
| Premature `unsafe` | Risk without proven gain | Safe version first; benchmark; Miri |

## Output format

```
## Rust Performance Analysis

**Target:** [function/endpoint/job and its goal: latency, throughput, memory]
**Environment:** Rust version, profile, CPU/OS, input sizes, workload
**Baseline:** [benchmark numbers, profile summary, allocation counts]
**Evidence:** [hot frames, allocation sites, cache/branch stats, lock or poll-time data, type sizes, generated code]
**Root cause:** [algorithm | allocation/copy | layout/cache | bounds checks/vectorization | inlining | hashing | contention | blocking async | I/O/logging/serialization | build config]

**Changes:**
1. Algorithm/data structure: [code and why, complexity before → after]
2. Allocation/copy/type size: [code and why]
3. Hashing/linting: [type/hasher/lint and threat model]
4. Inlining/layout: [attribute or layout change and evidence]
5. Concurrency/async: [code and why]
6. Build configuration: [only if justified]

**Before/after:**
- Benchmark time (median ± CI): ...
- Allocations / peak memory: ...
- Instructions, cache misses, branch misses (if relevant): ...
- Binary size / compile time (if relevant): ...

**Trade-offs:** [memory vs speed, readability, portability, unsafe surface, compile time]
**Verification:** [commands to re-run: cargo bench, cargo test, cargo clippy, Miri; references consulted]
```

## Checklist

- [ ] Baseline benchmark and profile captured with `--release`
- [ ] Hot path identified from a profile, not intuition
- [ ] Algorithmic complexity of the dominant loop checked and improved if needed
- [ ] `cargo clippy` and `-W clippy::perf` run; findings justified
- [ ] Allocations in hot paths counted; `with_capacity`, `reserve`, and buffer reuse applied
- [ ] Unnecessary `.clone()`, `to_string()`, `to_vec()`, and `collect()` removed
- [ ] APIs take borrowed types (`&str`, `&[T]`) where ownership is not needed
- [ ] Hasher choice matches the workload and trust model (HashDoS considered)
- [ ] Collection choice matches the workload; maps pre-sized
- [ ] Hot loops use iterators or hoisted slices; bounds checks verified in assembly if critical
- [ ] Data layout reviewed (contiguous storage, field sizes, `repr(Rust)` behavior, struct-of-arrays)
- [ ] Hot type sizes checked with `size_of`/`-Zprint-type-sizes`; >128-byte types shrunk or justified
- [ ] Inlining attributes measured; hot/cold paths split where useful
- [ ] Locks held briefly; no lock held across `.await` or I/O
- [ ] Wrapper types consolidated where access patterns justify it
- [ ] No blocking calls on the async executor; blocking work moved to `spawn_blocking` or a pool
- [ ] I/O buffered; stdout locked once for repeated output; line reading reuses buffers
- [ ] Serialization is typed/zero-copy where hot
- [ ] Regexes, clients, and pools created once and reused
- [ ] Logging/debug code does no work when disabled; hot assertions use `debug_assert!`
- [ ] Any `unsafe` justified by a profile, documented with `// SAFETY:`, and tested (Miri)
- [ ] Build settings (LTO, codegen-units, panic, allocator, target-cpu) evaluated individually
- [ ] Compile time and binary size considered when relevant
- [ ] Correctness preserved: tests pass; edge cases (empty, single, large, duplicates) covered
- [ ] Improvement confirmed by re-running the same benchmark; noise and variance considered
- [ ] Trade-offs documented; profile evidence recorded in comments where non-obvious
