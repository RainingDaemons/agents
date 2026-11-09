# Benchmarking

## Description

How to answer "did this change actually make it faster?" with numbers you can trust. A benchmark needs a realistic workload, an optimized build, a metric with acceptable variance, and enough repetitions to separate the change from the noise. Mediocre benchmarking is far better than none, but a benchmark that measures a dev build or one noisy run is worse than useless.

## When to use

- Before optimizing anything: establish a baseline.
- After every change: confirm the improvement and catch regressions.
- Choosing between implementations, allocators, hashers, or build settings.
- Adding a performance regression test to CI.

Do not benchmark code that is not on the measured hot path, and do not compare runs on different machines, toolchains, or profiles.

## Choose the metric

| Metric | Good for | Caveat |
|---|---|---|
| Wall time | What users perceive; end-to-end latency | High variance from layout, frequency, OS noise |
| CPU time | CPU-bound work | Ignores waiting and I/O |
| Cycles | Low-variance comparison | Needs hardware counters |
| Instructions | Very stable, portable comparisons | Ignores cache misses and branch mispredictions |
| Allocations / bytes | Memory regressions | Requires DHAT/`dhat-rs` instrumentation |
| Throughput | Batch and server workloads | Must state concurrency and input size |

Report which metric you used. For small changes, prefer instruction counts (`iai-callgrind`) or cycles over wall time.

## Checks

```bash
cargo build --release                 # never benchmark target/debug
cargo bench                           # nightly `test::Bencher`, or Criterion
cargo bench --bench my_bench
hyperfine --warmup 3 --runs 20 './old' './new'
perf stat -e cycles,instructions,cache-misses,branch-misses ./target/release/app
```

Probes:

```bash
rg -n 'Instant::now|SystemTime::now|std::time' benches src
rg -n 'cargo run( |$)|target/debug' .github scripts Makefile justfile 2>/dev/null
rg -n '#\[bench\]|criterion|criterion_group|divan' .
rg -n 'black_box' benches
rg -n 'iter_batched|iter_with_setup' benches
```

## Code examples

### Criterion: the default harness

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

`black_box` prevents the optimizer from deleting the call because the result is unused. The closure must do the work on every iteration.

### Keep setup out of the measured region

```rust
// SLOW measurement: the clone is included in every iteration
b.iter(|| {
    let data = big_input.clone();
    process(black_box(&data))
});
```

```rust
// FAST measurement: inputs are prepared per batch, only `process` is timed
b.iter_batched(
    || big_input.clone(),
    |data| process(black_box(&data)),
    criterion::BatchSize::SmallInput,
);
```

If constructing the input dominates, you are benchmarking the constructor.

### Several sizes, not one

```rust
fn bench_sizes(c: &mut Criterion) {
    let mut group = c.benchmark_group("parse");
    for n in [1_000usize, 10_000, 100_000] {
        let input = make_input(n);
        group.bench_with_input(format!("n={n}"), &input, |b, input| {
            b.iter(|| parse(black_box(input)))
        });
    }
    group.finish();
}
```

Vary `n` across orders of magnitude. A flat `time / n` suggests `O(n)`; a rising ratio suggests a hidden worse term (rehashing, reallocation, cache misses) or a different code path.

### Whole-program comparison with hyperfine

```bash
hyperfine --warmup 3 --runs 20 \
  './target/release/app --input big.txt' \
  './target/release/app-new --input big.txt'
```

Hyperfine reports mean, standard deviation, min/max, and relative speedup. Do not trust a single run.

### Low-variance instruction counts

```toml
[dev-dependencies]
iai-callgrind = "0.14"
```

```rust
use iai_callgrind::lib_benchmark;

fn bench_parse() {
    lib_benchmark!(parse, "sample", include_str!("../fixtures/sample.txt"));
}

iai_callgrind::main!(bench_parse);
```

Instruction counts are stable across runs, which makes them good for CI regression checks. They do not capture memory stalls; pair them with wall-time benchmarks when locality matters.

### Preventing dead-code elimination

```rust
// BAD: the compiler may remove the computation
let _ = expensive();

// GOOD: force the result to be observed
std::hint::black_box(expensive());
```

For manual harnesses, accumulate results into a checksum and print it, or pass inputs and outputs through `black_box`.

## Harnesses

| Tool | Use for |
|---|---|
| Criterion | Statistical Rust microbenchmarks and benches with plots |
| Divan | Fast, simple benchmark functions |
| `test::Bencher` | Built-in, nightly only |
| `hyperfine` | Whole-program CLI timing |
| `iai-callgrind` | Instruction counts, low variance, CI-friendly |
| Bencher | Continuous benchmarking in CI |
| Custom harness | Specialized workloads (e.g. compiler benchmarks) |

## Anti-patterns

1. Benchmarking a dev build; it is 10–100x slower.
2. One run, no warmup, no repetitions.
3. Timing a microbenchmark that does not resemble real input sizes or distributions.
4. Comparing runs on different machines, toolchains, or flags.
5. Leaving setup or allocation inside the measured closure.
6. Letting the optimizer delete the work (`black_box` missing).
7. Reporting a difference smaller than the run-to-run variance.
8. Benchmarking only the happy path with tiny inputs that fit in cache.
9. Mixing latency and throughput numbers.
10. Changing several things at once, so the cause of the improvement is unknown.

## Checklist

- Is the build `--release` (and the same profile as production)?
- Is the workload representative, with at least two orders of magnitude of `n`?
- Is the metric stated (wall time, cycles, instructions, allocations)?
- Are warmup and repetitions sufficient, and is the spread reported?
- Is the setup outside the measured region?
- Are inputs and outputs passed through `black_box`?
- Are both versions built and run under identical conditions?
- Is the machine idle, with frequency scaling and turbo under control?
- Does the benchmark run in CI to catch regressions?

## Related

- `profiling.md` for finding what to benchmark.
- `buildconfig.md` for the release and profiling profiles.
- `heap.md` for allocation-count comparisons.
- `machinecode.md` for explaining a result at the instruction level.
