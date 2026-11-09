# Profiling

## Description

How to find the code that actually matters. Profiling identifies the hot functions and, with the right tool, the hot allocation sites, cache misses, branch mispredictions, and lock contention. Profile a release build with symbols; read the result top-down; and never optimize from intuition when a profiler can tell you where the time goes.

## When to use

- Before any optimization effort: establish where time or memory goes.
- After a change: confirm the expected effect and find the next bottleneck.
- A service is slow, memory grows, or CPU utilization is low.
- Allocation, copying, hashing, or locking is suspected.
- Tail latency is high and the cause is unclear.

## Tool selection

| Question | Tool |
|---|---|
| Where is CPU time spent? | `perf record -g`, `cargo flamegraph`, `samply`, Instruments, VTune |
| Cycles, instructions, cache/branch misses | `perf stat` |
| Which allocations, how many, peak memory | DHAT, `dhat-rs`, heaptrack, bytehound |
| Instruction counts per line, simulated caches | Cachegrind, Callgrind |
| Which function was inlined? | Cachegrind (inlined functions lack counts on first/last line) |
| Ad hoc, domain-specific counts | `counts` with `eprintln!` |
| Optimization potential / causal profiling | Coz, `coz-rs` |
| Async task poll times and stalls | `tokio-console`, `tracing` |
| Compile-time hot spots | `cargo build --timings`, `cargo llvm-lines` |
| Binary composition | `cargo bloat` |
| Generated code | `cargo asm`, Compiler Explorer |

## Checks

```bash
# Linux CPU profile
perf record -g ./target/release/app
perf report
perf stat -e cycles,instructions,cache-misses,branch-misses ./target/release/app

# Flamegraph
cargo install flamegraph
cargo flamegraph --bin app

# Cross-platform sampling
samply record ./target/release/app

# Heap
valgrind --tool=dhat ./target/release/app
heaptrack ./target/release/app
```

Probes:

```bash
rg -n '\[profile\.release\]|strip|debug|force-frame-pointers' Cargo.toml
rg -n 'Instant::now|eprintln!\(".*count' src
rg -n 'dhat|heaptrack|criterion' Cargo.toml
```

## Making profiles readable

- Build a release profile with debug info:

```toml
[profile.profiling]
inherits = "release"
debug = "line-tables-only"
strip = "none"
```

```bash
cargo build --profile profiling
```

- Keep frame pointers when stack traces are poor:

```bash
RUSTFLAGS="-C force-frame-pointers=yes" cargo build --profile profiling
```

- If symbols are mangled (`_ZN`, `_R`), try the v0 mangling and `rustfilt`:

```bash
RUSTFLAGS="-C symbol-mangling-version=v0" cargo build --profile profiling
cargo install rustfilt
perf report | rustfilt
```

- Standard-library frames may remain opaque because shipped std does not include debug info; build std with debug info (`-Z build-std`) only if you really need those frames.
- Do not strip symbols in a binary you intend to profile.

## Reading a profile

Start from the widest frames and work down. A wide frame is a candidate, not a conclusion.

| Wide frame / counter | Likely cause | Next step |
|---|---|---|
| `malloc`/`free`/`realloc` | Allocation rate | DHAT, `with_capacity`, reuse buffers (`heap.md`) |
| `memcpy`/`clone` | Large copies or deep clones | Borrow, `Arc`, `Cow`, shrink types (`type-sizes.md`) |
| `core::hash`/SipHash | Default hasher on hot lookups | Faster hasher for trusted keys (`hashing.md`) |
| `Mutex::lock`/futex | Lock contention | Narrow sections, shard, per-thread state (`parallelism.md`) |
| High cache-miss rate | Pointer chasing, poor layout, large structs | Contiguous storage, SoA, smaller types (`type-sizes.md`) |
| High branch-miss rate | Unpredictable branches | Branchless forms, sorting, lookup tables |
| `panic_bounds_check` | Bounds checks in a hot loop | Iterators, slices, assertions (`iterators.md`) |
| `fmt`/`format!` | Formatting on the hot path | Guard logging, defer formatting (`logging-debugging.md`) |
| `serde`/parsing | Serialization cost | Borrowed fields, binary format, `simd-json` |
| Worker threads idle while tasks wait | Blocking in async | `spawn_blocking`, async I/O (`async.md`) |
| Many tiny syscalls | Unbuffered I/O | `BufReader`/`BufWriter`, lock once (`std.md`) |

## Ad hoc profiling with `counts`

When a general profiler does not answer a domain question, instrument the code and post-process frequencies:

```rust
eprintln!("vector length: {}", v.len());
```

```bash
cargo run --release 2> log.txt
counts log.txt | head
```

This is crude but often reveals distributions (most vectors have 0 or 1 elements, one branch dominates) that guide special-casing and let you delete the instrumentation afterward.

## Profiling memory

- DHAT: hot allocation sites, allocation rates, peak memory, and hot `memcpy` copies.
- `dhat-rs`: cross-platform, small code changes, can assert allocation counts in tests.
- heaptrack/bytehound: allocation lifetimes and flame graphs.
- Count **allocations and bytes**, not only RSS; RSS hides fragmentation and allocator caching.

## Anti-patterns

1. Optimizing without a profile.
2. Profiling a dev build or a stripped binary.
3. Profiling one tiny microbenchmark and generalizing to the whole program.
4. Reading only the top frame; the cause is often a caller.
5. Trusting a single profiler; different tools see different bottlenecks.
6. Profiling with debug logging or assertions enabled in a release build.
7. Profiling on a noisy machine or under a different workload.
8. Reporting a profile percentage without saying what the total includes.
9. Forgetting to remove ad hoc `eprintln!` instrumentation.
10. Ignoring allocation and cache profiles because the CPU profile "looks fine".

## Checklist

- Is the binary a release (or profiling) build with symbols?
- Did the profile run on the real workload and input sizes?
- Are the top frames explained, not just listed?
- For allocation problems, are allocation counts and bytes measured?
- For memory-bound problems, are cache misses and layout checked?
- For async, are poll times and blocking calls visible?
- Did you use more than one tool where the first was inconclusive?
- Was the profile taken again after the change to confirm the improvement?

## Related

- `benchmarking.md` for measuring the effect of the change.
- `heap.md`, `hashing.md`, `type-sizes.md`, `iterators.md` for the common causes.
- `async.md` for executor and task profiling.
- `buildconfig.md` for debug info, frame pointers, and stripping.
