# Parallelism

## Description

How to scale work across cores, why the speedup is usually below the core count, and when not to parallelize. Covers Amdahl's law, work/span analysis, threads vs processes, chunking, contention, false sharing, and bandwidth saturation.

Parallelism reduces **wall-clock time**, not the total instruction count. It is the last lever, after removing work, choosing a better algorithm, and improving constants. Never parallelize an `O(n²)` algorithm when an `O(n log n)` one exists.

## Use cases

- Independent work over a large dataset: map/filter/aggregate.
- CPU-bound computation that already has a good sequential algorithm.
- Batch jobs: many independent requests, files, or records.
- Numerical kernels on large matrices (combined with SIMD and blocking).
- Serving many concurrent clients (I/O concurrency, a different problem from CPU parallelism).
- Background work that should not block the main thread.

Do **not** use it for tiny tasks (thread overhead dominates), for inherently sequential algorithms (prefix sums with strict dependencies, most graph traversals), or when the bottleneck is a single shared resource (one disk, one database, one lock).

## Two models

| Model | Measures | Use |
|---|---|---|
| **Amdahl's law** | Speedup from parallelizing a fraction `p` | Upper bound from the sequential part |
| **Work/span** | `T₁` = total work, `T∞` = critical-path length | Upper bound from dependencies |

Amdahl: `speedup(N) = 1 / ((1 - p) + p/N)`. Even with infinite cores, the ceiling is `1 / (1 - p)`.

| Sequential fraction | Max speedup (∞ cores) |
|---|---|
| 50% | 2× |
| 10% | 10× |
| 5% | 20× |
| 1% | 100× |

Work/span: `parallelism = T₁ / T∞` is the maximum speedup on any number of cores; speedup is also `≤ N`. A long dependency chain (large `T∞`) caps the gain regardless of cores.

## Code examples

### Amdahl in practice

```python
def speedup(p, n):
    return 1.0 / ((1.0 - p) + p / n)

print(speedup(0.95, 8))    # ~5.9x, not 8x
print(speedup(0.50, 8))    # ~1.8x
```

The sequential 5% limits an 8-core machine to under 6×. Before adding threads, reduce the sequential portion.

### Parallel sum with chunks (processes, not threads)

```python
import os
from concurrent.futures import ProcessPoolExecutor

_DATA = None

def _init(data):
    global _DATA
    _DATA = data

def _partial_sum(bounds):
    start, end = bounds
    return sum(_DATA[start:end])

def parallel_sum(data, workers=None):
    workers = workers or os.cpu_count()
    n = len(data)
    chunk = (n + workers - 1) // workers
    ranges = [(i, min(i + chunk, n)) for i in range(0, n, chunk)]
    with ProcessPoolExecutor(
        max_workers=workers, initializer=_init, initargs=(data,)
    ) as pool:
        return sum(pool.map(_partial_sum, ranges))
```

`O(n)` work and `O(n / workers)` wall-clock time if memory bandwidth allows. `ProcessPoolExecutor` sidesteps the GIL, but each worker still gets its own copy of `data`; for large arrays use `multiprocessing.shared_memory` or a native library that parallelizes internally. Chunking matters: too many tiny tasks means overhead dominates.

### Why Python threads do not speed up CPU work

```python
from concurrent.futures import ThreadPoolExecutor

def cpu_heavy(_):
    return sum(i * i for i in range(10_000_000))

with ThreadPoolExecutor(max_workers=4) as pool:
    results = list(pool.map(cpu_heavy, range(4)))
```

Under CPython's GIL this runs **serially** for pure-Python bytecode, often slower than sequential because of switching. Threads are fine for I/O (releasing the GIL during blocking calls) and for libraries that release the GIL (NumPy, I/O, database drivers). For CPU work use `multiprocessing`/`ProcessPoolExecutor`, or a native library that parallelizes internally.

### Rust / C++: data parallelism

```rust
// Cargo.toml: rayon = "1"
use rayon::prelude::*;

fn total(data: &[i64]) -> i64 {
    data.par_iter().sum()
}
```

The work is split recursively and reduced; the scheduler handles chunking. This is the recommended path over hand-rolled threads for data parallelism.

### False sharing

```c++
// SLOW: two independent counters land on the same 64-byte cache line
struct Counters { long a; long b; };
Counters c;

// FIX: align each counter to its own cache line
struct alignas(64) Padded { long value; };
Padded c[2];
```

When independent variables share a cache line, writes ping-pong between cores. Padding, per-thread accumulators combined at the end, or thread-local state removes it. Python does not expose this layout, so in Python use per-thread/per-process partial results instead.

### Contention: the lock is the bottleneck

```python
import threading

lock = threading.Lock()
total = 0

def add(x):
    global total
    with lock:
        total += x
```

This is correct but serializes on `lock`; with many small increments, the parallel version can be slower than the sequential one. Use per-thread partial results and combine once. Amdahl applies to the locked section as if it were sequential.

### Bandwidth saturation

A memory-bound loop does not scale with cores: one core can already saturate a large fraction of the memory bandwidth. Adding cores can even slow it down through contention for the memory controller. Check with `perf stat` or a scaling test (1, 2, 4, 8 workers). If the time is flat, you are bandwidth-bound; optimize locality or reduce data volume first (`memory.md`).

## Anti-patterns

1. Parallelizing before removing work and choosing a good algorithm.
2. Spawning a thread per item instead of using a pool and chunks.
3. Assuming threads speed up CPU-bound Python (the GIL).
4. Parallelizing a memory-bandwidth-bound or lock-bound loop.
5. Sharing a mutable counter without per-thread accumulation.
6. False sharing between per-thread variables.
7. Ignoring the sequential fraction (Amdahl) or the critical path (work/span).
8. Using more threads than cores for CPU-bound work (oversubscription).
9. Treating I/O concurrency and CPU parallelism as the same tool.

## Checklist

- Is the work genuinely independent, or is there a dependency chain?
- What is the sequential fraction, and what ceiling does Amdahl give?
- What is the critical path (`T∞`), and what is `T₁/T∞`?
- Are you CPU-bound, memory-bound, lock-bound, or I/O-bound?
- Threads, processes, or async: which matches the bottleneck and the runtime?
- Is the task granularity large enough to amortize scheduling overhead?
- Are partial results combined without a shared hot variable?
- Is there false sharing or padding to check?
- Did you measure scaling at 1, 2, 4, and 8 workers?

## Related

- `asymptotic.md` for work and the sequential baseline.
- `memory.md` for bandwidth saturation and false sharing mechanics.
- `simd.md` for the other axis of parallelism (within a core).
- `profiling-benchmarking.md` for measuring scaling.

