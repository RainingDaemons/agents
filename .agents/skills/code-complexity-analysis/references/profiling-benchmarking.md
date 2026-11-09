# Profiling and Benchmarking

## Description

How to find out where time actually goes, and how to measure a change without fooling yourself. Covers instrumentation, statistical sampling, simulation, machine-code analyzers, benchmark harnesses, and the practices that make results reproducible.

The performance loop is: **measure → find the bottleneck → remove it → measure again**. Skipping the first or last step produces optimizations that do not help.

## Use cases

- "This is slow" with no known hot spot.
- Comparing two implementations of the same function.
- Verifying that a change actually improved performance.
- Checking whether a loop is bound by cycles, branches, cache, or bandwidth.
- Establishing a baseline before refactoring.
- Deciding whether an optimization is worth its complexity.

## Choose the method

| Question | Method | Tools |
|---|---|---|
| Where does time go in a large program? | Statistical sampling | `perf record`, `pprof`, `async-profiler`, Chrome DevTools |
| How many cycles, instructions, cache misses? | Hardware counters | `perf stat`, VTune, `perf stat -e ...` |
| Why is this one loop slow? | Simulation / machine model | Cachegrind, `llvm-mca` |
| Did my change make it faster? | Benchmark harness | `hyperfine`, `criterion`, `pyperf`, `timeit`, JMH |
| Which function calls which? | Call graph | `perf report --call-graph`, flame graphs |
| How much memory, where allocated? | Heap profiling | `heaptrack`, `massif`, DevTools Memory |

Measure **before** optimizing. If the function is not hot, or `n` is small, the readable version is usually already right.

## Code examples

### Instrumentation: manual timing around a region

```python
import time

def timeit(fn, *args, repeat=5):
    best = float("inf")
    for _ in range(repeat):
        start = time.perf_counter()
        result = fn(*args)
        best = min(best, time.perf_counter() - start)
    return best, result
```

`perf_counter` is monotonic and high-resolution. Use the **minimum** of several runs for short code: noise only adds time. Consume `result` so the work is not optimized away.

In C++:

```c++
#include <chrono>

auto start = std::chrono::steady_clock::now();
long checksum = run();
auto elapsed = std::chrono::steady_clock::now() - start;
auto ns = std::chrono::duration_cast<std::chrono::nanoseconds>(elapsed).count();
printf("%lld ns, checksum=%ld\n", (long long)ns, checksum);  // checksum prevents dead-code elimination
```

### Statistical profiling: `perf`

```bash
perf record -g ./app            # sample stacks at a fixed frequency
perf report                     # interactive, sorted by self time
perf stat -e cycles,instructions,cache-misses,branch-misses ./app
```

`perf stat` gives counts and ratios (IPC, miss rate). `perf record -g` gives a call graph. Sampling has low overhead and works on unmodified binaries when symbols are available. Do not profile a debug build and report production numbers.

### Cache and branch simulation: Cachegrind

```bash
valgrind --tool=cachegrind --cache-sim=yes ./app
cg_annotate cachegrind.out.*
```

Cachegrind simulates the cache hierarchy and branch predictor, giving per-line miss counts. It is 10–100× slower than native execution, so use it on small representative inputs, not on the full workload.

### Machine-code analysis: `llvm-mca`

```bash
clang++ -O3 -march=native -S -o loop.s loop.cpp
llvm-mca -mcpu=native loop.s
```

Reports cycles per iteration, resource pressure, and whether the loop is latency- or throughput-bound. Use it on the isolated hot loop.

### Benchmark harness: compare implementations

```python
import timeit

setup = "from mymod import fast, slow; data = list(range(1000))"
print(timeit.timeit("slow(data)", setup=setup, number=1000))
print(timeit.timeit("fast(data)", setup=setup, number=1000))
```

For CLI-level comparisons, `hyperfine` handles warmup, repetitions, and statistics:

```bash
hyperfine --warmup 3 --runs 20 './old --input big.txt' './new --input big.txt'
```

It reports mean, standard deviation, min/max, and a relative speedup. Do not trust a single run.

### Vary `n` across orders of magnitude

```python
for n in (10**3, 10**4, 10**5, 10**6):
    t = timeit.timeit(lambda: algorithm(make_input(n)), number=1)
    print(f"n={n:>9}  t={t:.6f}s  t/n={t/n:.3e}")
```

Check that the measured growth fits the predicted one. A flat `t/n` means `O(n)`; a rising `t/n` means a worse term is appearing (cache misses, rehashing, GC).

## Getting accurate results

| Practice | Why |
|---|---|
| Warm up before measuring | JIT compilation, caches, and lazy initialization |
| Repeat and take the median/min | Noise is one-sided; outliers are common |
| Report spread, not just a mean | A 2× difference may be inside the noise |
| Pin the CPU / disable turbo | Frequency scaling changes cycles per second |
| Close background programs | Other processes steal cache and bandwidth |
| Use representative input sizes | Small inputs fit in cache and hide the real cost |
| Consume the result | Otherwise the compiler deletes the work |
| Compare on the same machine and flags | Different hardware invalidates the comparison |
| Measure latency **and** throughput when both matter | A batch of one is not the same as many concurrent |
| Separate hot-loop time from end-to-end time | Profiling overhead and I/O distort end-to-end |

The biggest source of wrong conclusions is comparing numbers produced under different conditions. Record the environment with the result.

## Anti-patterns

1. Optimizing without a profile.
2. Benchmarking a debug build or an interpreted development server.
3. One run, no warmup, no repetition.
4. Timing code that the compiler optimized away (result unused).
5. Measuring a microbenchmark that does not resemble the real input distribution.
6. Mixing latency and throughput numbers.
7. Profiling a whole program when the question is about one loop.
8. Changing several variables at once, so the cause of the improvement is unknown.
9. Reporting a speedup smaller than the run-to-run variance.
10. Trusting `time` on a single run under an active desktop.

## Checklist

- Is there a baseline, measured on representative input?
- Did a profiler identify the hot spot, or are you guessing?
- Is the bottleneck CPU, memory, branch, lock, or I/O?
- Are warmup, repetitions, and noise handled?
- Is the result consumed so the work is not eliminated?
- Are both versions compiled and run under identical conditions?
- Did you measure across at least two orders of magnitude of `n`?
- Did you report the median and the spread?
- Did you re-measure after the change and compare against the baseline?

## Related

- `asymptotic.md` for the predicted growth to compare against.
- `codecomp.md` for reading the generated code once the hot loop is known.
- `memory.md` for cache-miss interpretation.
- `simd.md` and `parallelism.md` for the last two levers after work is reduced.

