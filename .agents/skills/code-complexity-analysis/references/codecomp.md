# Compiler Output, Instruction-Level Parallelism, and Code Comparison

## Description

How to inspect what the compiler actually generated, how optimization flags change it, and how to compare two implementations at the instruction level. Covers assembly reading, inlining, unrolling, aliasing, undefined behavior, profile-guided optimization, instruction tables, dependency chains, and multiple accumulators.

Same algorithm, same Big-O, different code: this file is about the **constant factor** that the asymptotic model hides.

## Use cases

- A hot loop is slower than expected and the source looks fine.
- Choosing between two implementations with the same complexity.
- Checking whether a loop was vectorized, inlined, or unrolled.
- Deciding whether a language/compiler can optimize a construct at all.
- Reviewing performance-critical code in Rust, C, C++, or a JIT runtime.
- Confirming that a "clever" rewrite actually produces better code.

## Reading generated code

| Tool | Command | What it shows |
|---|---|---|
| GCC / Clang assembly | `g++ -O2 -S -masm=intel file.cpp` | Human-readable assembly |
| Object dump | `objdump -d --no-show-raw-insn ./a.out` | Machine code with symbols |
| Compiler Explorer | godbolt.org | Side-by-side source and assembly |
| Rust | `cargo asm --rust` or `rustc --emit asm` | Rust-generated assembly |
| LLVM machine analyzer | `llvm-mca -mcpu=native -iterations=100` | Throughput, latency, port pressure |
| Compiler optimization report | `-fopt-info-vec-optimized` (GCC) | Which loops vectorized |

What to look for:

- The loop body: is the array load inside or outside? Was it unrolled?
- Registers: are values kept in registers or spilled to the stack?
- Calls: was the hot function inlined? A `call` in the inner loop is expensive.
- Vector instructions (`vmovdqu`, `vpaddd`, `vfmadd`): is the loop vectorized?
- `cmp` / `jne` / `cmov`: branches or conditional moves?
- `lea`, `imul`, `div`: address arithmetic and expensive divisions.

## Flags and what they change

| Flag | Effect | Risk |
|---|---|---|
| `-O0` | No optimization, easy debugging | 5–50× slower |
| `-O2` | Standard optimization, inlining, some vectorization | Safe default |
| `-O3` | More inlining, loop unrolling, aggressive vectorization | Larger code, possible regressions |
| `-march=native` | Use the local CPU's extensions (AVX2/AVX-512) | Binary not portable |
| `-ffast-math` | Reassociate floating point | Changes results; not IEEE-compliant |
| `-fno-omit-frame-pointer` | Keep frame pointers | Better profiling, slightly slower |
| LTO / `-flto` | Optimize across translation units | Longer builds |
| PGO | Optimize using real profiling data | Needs a training run |

Python, JavaScript, and Java runtimes make these decisions at runtime (JIT); you cannot read a single static assembly listing. Use the runtime's own tooling (`perf`, `--print-opt-code`, `V8 --trace-opt`) instead.

## Code examples

### Without and with target-specific instructions

```c++
// compiled with g++ -O2
int sum(const int* a, int n) {
    int s = 0;
    for (int i = 0; i < n; i++) s += a[i];
    return s;
}
```

```bash
g++ -O2 -S -masm=intel sum.cpp     # scalar adds, maybe SSE2
g++ -O3 -march=native -S -masm=intel sum.cpp
```

With `-march=native`, the compiler may use 256-bit (`ymm`) or 512-bit (`zmm`) registers and add 8–16 integers per instruction. The source did not change; the generated code did. Confirm with `objdump` or `-fopt-info-vec-optimized` instead of assuming.

### Aliasing blocks optimization

```c++
void scale(float* a, float* b, float* out, int n) {
    for (int i = 0; i < n; i++) out[i] = a[i] * b[i];
}
```

The compiler must assume `out` may alias `a` or `b`, so it may not vectorize. Tell it they do not overlap:

```c++
void scale(float* __restrict a, float* __restrict b, float* __restrict out, int n) {
    for (int i = 0; i < n; i++) out[i] = a[i] * b[i];
}
```

`__restrict` is a promise. If the pointers do overlap, the result is undefined behavior. In Rust, `&mut` and `&` already encode non-aliasing, which is one reason Rust vectorizes more reliably.

### Inlining and call overhead

```c++
int square(int x) { return x * x; }          // tiny: almost always inlined

int sum_squares(const int* a, int n) {
    int s = 0;
    for (int i = 0; i < n; i++) s += square(a[i]);
    return s;
}
```

At `-O2`, `square` disappears. At `-O0`, there is a `call` per element: easily 5–10× slower. Do not "optimize" by hand-inlining before checking whether the compiler already did it.

### Unrolling

```c++
// compiler-unrolled (conceptually)
for (int i = 0; i + 3 < n; i += 4) {
    s0 += a[i];
    s1 += a[i + 1];
    s2 += a[i + 2];
    s3 += a[i + 3];
}
```

Unrolling reduces loop-control overhead and exposes more independent work, but it can hurt the instruction cache and does not help if the loop is memory-bound. Measure; do not unroll by hand without evidence.

### Instruction-level parallelism: one accumulator vs four

```c++
// SLOW: one loop-carried dependency chain
int s = 0;
for (int i = 0; i < n; i++) s += a[i];
```

Each addition depends on the previous one. The result is a chain of `n` adds, each with a few cycles of latency.

```c++
// FAST: four independent chains, combined at the end
int s0 = 0, s1 = 0, s2 = 0, s3 = 0;
int i = 0;
for (; i + 3 < n; i += 4) {
    s0 += a[i];
    s1 += a[i + 1];
    s2 += a[i + 2];
    s3 += a[i + 3];
}
for (; i < n; i++) s0 += a[i];
int s = s0 + s1 + s2 + s3;
```

Same `O(n)` and same number of additions, but four independent dependency chains can execute in parallel on a superscalar CPU. The speedup is bounded by the number of execution ports and by memory bandwidth, not by `n`.

This is also why a compiler cannot always do it for you: floating-point reassociation changes results, so it needs `-ffast-math` (or an explicit fast-math block) to reassociate.

### Branchless code

```c++
// branchy: mispredicts when the sign is random
int s = 0;
for (int i = 0; i < n; i++)
    if (a[i] > 0) s += a[i];

// branchless: conditional move or mask, no misprediction
int s = 0;
for (int i = 0; i < n; i++)
    s += (a[i] > 0) ? a[i] : 0;
```

The second form is a candidate for `cmov` or a mask, but the compiler decides. Branchless code wins only when the branch is **unpredictable**; predictable branches are cheap and can even be faster. Measure with branch-miss counters (`perf stat -e branch-misses`).

### Inspecting throughput and latency with `llvm-mca`

```bash
clang++ -O3 -march=native -S -o loop.s loop.cpp
llvm-mca -mcpu=native -iterations=100 loop.s
```

`llvm-mca` reports the expected cycles per iteration, the bottleneck resource (a port, a dependency), and the effect of unrolling. Use it on the isolated hot loop, not on a whole program.

## Anti-patterns

1. Hand-optimizing before checking the generated code.
2. Assuming `-O3` is always faster than `-O2`; sometimes it is slower.
3. Using `-ffast-math` in code that needs IEEE semantics, NaN handling, or reproducibility.
4. Breaking `__restrict` promises; the result is undefined behavior.
5. Relying on undefined behavior for speed (signed overflow, out-of-bounds reads) — the compiler may exploit it arbitrarily.
6. Hand-inlining or hand-unrolling something the compiler already does better.
7. Optimizing a cold path because the assembly "looks bad".
8. Comparing two implementations compiled with different flags or runtimes.
9. Ignoring the memory-bound reality: ILP and unrolling cannot beat a bandwidth limit.

## Checklist

- Did you compile both implementations with the same compiler, flags, and target?
- Did you inspect the loop body, not just the source?
- Was the hot function inlined? Was the loop vectorized or unrolled?
- Are there hidden calls (allocation, bounds checks, virtual dispatch)?
- Is the loop bound by latency, throughput, branches, or memory?
- Did you confirm the bottleneck with `perf stat` or `llvm-mca` instead of guessing?
- Does the optimization preserve the language's defined behavior?

## Related

- `profiling-benchmarking.md` for measuring the effect.
- `memory.md` for the bandwidth and cache limits that cap ILP gains.
- `simd.md` for vectorization in detail.
- `asymptotic.md` for the model that hides these constants.

