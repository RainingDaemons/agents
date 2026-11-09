# Parallelism

## Description

How to scale Rust code across cores without losing the gains to scheduling overhead, contention, or false sharing. Rust makes data parallelism safe and ergonomic (`rayon`, `crossbeam`), but parallelism only helps when the work per item is large enough, the threads are not fighting over a lock, and the workload is not already limited by memory bandwidth. Parallelism reduces wall-clock time, not total work.

## When to use

- A CPU-bound loop runs long enough to matter and the items are independent.
- A service underutilizes cores during heavy requests.
- Batch jobs: many files, records, or independent tasks.
- A hot computation can be split (`par_iter`, `par_chunks`, `join`).
- A lock or shared counter is the measured bottleneck.

Do not parallelize tiny tasks, inherently sequential algorithms, or code that is already memory-bandwidth-bound.

## Checks

```bash
perf stat -e cycles,instructions,cache-misses,branch-misses ./target/release/app
```

| Symptom | Likely cause | Fix |
|---|---|---|
| Speedup plateaus at 2–4× on many cores | Sequential fraction or memory bandwidth | Amdahl analysis; reduce data volume |
| Workers idle in profiles | Tasks too small / scheduling overhead | Coarser chunks, `with_min_len` |
| Time in `Mutex::lock`/futex | Contention | Per-thread accumulators, sharding, channels |
| Cache misses rise with threads | False sharing | Padding, per-thread data |
| Scaling is flat | Bandwidth-bound or a single shared resource | Improve locality; do not add threads |

Probes:

```bash
rg -n 'rayon|par_iter|par_chunks|join\(' src
rg -n 'Mutex|RwLock|Atomic|lock\(\)' src
rg -n 'Arc::clone|Arc<Mutex' src
rg -n 'thread::spawn|scope\(' src
rg -n 'fold\(|reduce\(' src
```

## Code examples

### Data parallelism with rayon

```rust
// SLOW: one core
let sum: u64 = data.iter().map(|x| x.cost).sum();
```

```rust
// FAST: split the work, combine the results
use rayon::prelude::*;

let sum: u64 = data.par_iter().map(|x| x.cost).sum();
```

Rayon splits the range recursively and reduces the partial results. Gains need enough work per item: a closure that does a few nanoseconds is dominated by scheduling. Tune with `with_min_len`:

```rust
let sum: u64 = data
    .par_iter()
    .with_min_len(10_000)
    .map(|x| x.cost)
    .sum();
```

### Chunks for cache-friendly parallel work

```rust
use rayon::prelude::*;

// Each thread works on a contiguous chunk, reducing cache traffic
let totals: Vec<u64> = data
    .par_chunks(4_096)
    .map(|chunk| chunk.iter().map(|x| x.cost).sum())
    .collect();
let total: u64 = totals.iter().sum();
```

### Avoid a shared counter

```rust
// SLOW: every increment takes the same lock
let total = std::sync::Mutex::new(0u64);
data.par_iter().for_each(|x| {
    *total.lock().unwrap() += x.cost;
});
```

```rust
// FAST: per-thread partials, merged once
let total: u64 = data
    .par_iter()
    .map(|x| x.cost)
    .reduce(|| 0, |a, b| a + b);
```

`reduce`/`fold` keep the accumulation local and avoid contention. The same idea applies to maps: shard them or collect per-thread results.

### False sharing

```rust
// SLOW: independent counters share a cache line; writes ping-pong between cores
struct Counters {
    a: std::sync::atomic::AtomicU64,
    b: std::sync::atomic::AtomicU64,
}
```

```rust
// FAST: pad each counter to its own cache line
use crossbeam_utils::CachePadded;

struct Counters {
    a: CachePadded<std::sync::atomic::AtomicU64>,
    b: CachePadded<std::sync::atomic::AtomicU64>,
}
```

False sharing shows up as high cache-miss counts and poor scaling even though the threads touch different variables. Padding, per-thread counters, or thread-local state fixes it.

### `Arc::clone` in a hot loop

```rust
// SLOW: an atomic refcount increment per item
for item in items {
    let shared = Arc::clone(&config);
    spawn(move || work(shared, item));
}
```

```rust
// FAST: clone once and share the clone, or use references/scoped threads
let shared = Arc::clone(&config);
for item in items {
    let shared = Arc::clone(&shared);
    spawn(move || work(shared, item));
}
```

The second still clones per task, but from a local; if the tasks are truly hot, restructure to avoid the per-item clone (scoped threads, `rayon`, or passing the config by reference).

### Atomics and ordering

```rust
use std::sync::atomic::{AtomicU64, Ordering};

// Counters: relaxed is enough when only the final value matters
static HITS: AtomicU64 = AtomicU64::new(0);
HITS.fetch_add(1, Ordering::Relaxed);

// Publishing data: release on the writer, acquire on the reader
static READY: AtomicBool = AtomicBool::new(false);
READY.store(true, Ordering::Release);
if READY.load(Ordering::Acquire) {
    // safe to read the published data
}
```

| Ordering | Use |
|---|---|
| `Relaxed` | Counters and statistics with no ordering requirement |
| `Release` (store) / `Acquire` (load) | Publishing/reading shared data |
| `AcqRel` | Read-modify-write that both acquires and releases |
| `SeqCst` | When a total order is genuinely required; the safest, slowest default |

Wrong ordering is a correctness bug, not a performance knob. Use `loom` to test lock-free code.

### Locks

- Keep critical sections short; never hold a lock across I/O or `.await`.
- `parking_lot` can be faster and smaller than `std`, but std has improved; measure.
- `RwLock` helps only when reads dominate heavily; otherwise it can be slower than a short `Mutex`.
- Prefer message passing (`std::sync::mpsc`, `crossbeam-channel`, `flume`) over shared mutable state.
- Do not oversubscribe: `rayon` inside another thread pool, or blocking inside a rayon worker, causes stalls.

## Anti-patterns

1. Parallelizing before reducing work or fixing the algorithm.
2. Parallelizing tiny tasks; overhead dominates.
3. A shared `Mutex` counter in the inner loop.
4. False sharing between per-thread counters.
5. Parallelizing a memory-bandwidth-bound loop.
6. Holding a lock across I/O or `.await`.
7. Cloning an `Arc` per item when one clone or a reference would do.
8. Using `SeqCst` everywhere "to be safe" instead of reasoning about ordering.
9. Oversubscribing cores or nesting thread pools.
10. Assuming "more threads" always means "faster".

## Checklist

- Is the work independent and large enough per item?
- What is the speedup at 1, 2, 4, and 8 threads? Where does it plateau?
- Is the workload CPU-bound, memory-bound, lock-bound, or I/O-bound?
- Are partial results accumulated per thread and merged once?
- Is there false sharing between hot counters?
- Are atomics using the weakest correct ordering?
- Are critical sections short and free of I/O?
- Is the thread pool sized to the machine and not nested?
- Did you Re-run the benchmark and compare against the sequential version?

## Related

- `async.md` for I/O concurrency and blocking on the executor.
- `profiling.md` for diagnosing contention and idle workers.
- `type-sizes.md` for false sharing and padding.
- `buildconfig.md` for release builds and target features.
