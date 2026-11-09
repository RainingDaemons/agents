# SIMD and Vectorization

## Description

How to make one instruction operate on several values at once, and when it is worth it. Covers auto-vectorization, intrinsics, reductions, masking, shuffles, and portability.

SIMD does not change the asymptotic complexity. It divides the constant factor by up to the vector width (4 floats with SSE, 8 with AVX2, 16 with AVX-512), and only when the data and control flow allow it.

## Use cases

- Element-wise arithmetic on large arrays: scaling, clamping, activation functions.
- Reductions: sum, min/max, dot product, argmin.
- Comparisons and masks: filtering, thresholding, counting.
- Image, audio, and signal processing.
- String scanning and parsing (SIMD JSON, base64, memchr-like work).
- Numerical kernels (matrix multiplication, convolutions) combined with blocking.

Do **not** reach for SIMD first: if the loop is memory-bound, vectorizing the arithmetic does not help. Fix locality first (`memory.md`).

## What enables vectorization

A loop can be autovectorized when:

- Iterations are independent (no loop-carried dependency).
- Data is contiguous (or has a constant stride).
- There is no aliasing the compiler cannot rule out (`__restrict` in C/C++, `&mut` in Rust helps).
- Control flow is simple; branches become masks or are hoisted.
- The trip count and the operation are worth vectorizing (a very short loop may not be).

Check whether it happened:

```bash
g++ -O3 -march=native -fopt-info-vec-optimized -c loop.cpp
clang++ -O3 -march=native -Rpass=loop-vectorize -c loop.cpp
```

## Code examples

### The one-line version

```c++
const int n = 1e5;
int a[n], s = 0;

for (int i = 0; i < n; i++)
    s += a[i];
```

With `g++ -O3` the compiler may vectorize this; with `g++ -O3 -march=native` it can use wider registers. Do not assume — inspect the output or the optimization report. The same loop shows roughly a 2× difference between the default target and `-mavx2` on a sum loop.

### Explicit intrinsics (AVX2, 8 int32 per instruction)

```c++
#include <immintrin.h>

int sum_avx2(const int* a, int n) {
    __m256i acc = _mm256_setzero_si256();
    int i = 0;
    for (; i + 7 < n; i += 8) {
        __m256i v = _mm256_loadu_si256(reinterpret_cast<const __m256i*>(a + i));
        acc = _mm256_add_epi32(acc, v);
    }
    __m128i lo = _mm256_castsi256_si128(acc);
    __m128i hi = _mm256_extracti128_si256(acc, 1);
    __m128i s = _mm_add_epi32(lo, hi);
    s = _mm_hadd_epi32(s, s);
    s = _mm_hadd_epi32(s, s);
    int total = _mm_cvtsi128_si32(s);
    for (; i < n; i++) total += a[i];   // scalar tail
    return total;
}
```

Still `O(n)`, but 8 additions per instruction in the main loop. The horizontal reduction at the end costs a few cycles and runs once. The scalar tail handles `n % 8`. `_mm256_loadu` tolerates unaligned data; aligned loads (`_mm256_load_si256`) can be marginally faster but require 32-byte alignment.

### Multiple accumulators to hide latency

A single `acc` creates a dependency chain across iterations. Use independent accumulators and combine at the end:

```c++
__m256i a0 = _mm256_setzero_si256();
__m256i a1 = _mm256_setzero_si256();
for (int i = 0; i + 15 < n; i += 16) {
    a0 = _mm256_add_epi32(a0, _mm256_loadu_si256((const __m256i*)(a + i)));
    a1 = _mm256_add_epi32(a1, _mm256_loadu_si256((const __m256i*)(a + i + 8)));
}
```

Same number of adds, twice the instruction-level parallelism. This is the SIMD analogue of the scalar trick in `codecomp.md`.

### Masking instead of branching

```c++
// branchy: mispredicts on random signs
for (int i = 0; i < n; i++)
    if (a[i] > 0) s += a[i];
```

```c++
// vectorized with a mask: no branch, no misprediction
__m256i zero = _mm256_setzero_si256();
__m256i acc = zero;
for (int i = 0; i + 7 < n; i += 8) {
    __m256i v = _mm256_loadu_si256((const __m256i*)(a + i));
    __m256i mask = _mm256_cmpgt_epi32(v, zero);
    acc = _mm256_add_epi32(acc, _mm256_and_si256(v, mask));
}
```

Masking turns control flow into data, which is what SIMD needs. It wins when the branch is unpredictable; it can lose when it is predictable.

### Portability: runtime dispatch

A binary compiled with `-march=native` crashes on older CPUs. Either ship per-target variants or dispatch at runtime:

```c++
if (__builtin_cpu_supports("avx2")) {
    result = sum_avx2(a, n);
} else {
    result = sum_scalar(a, n);
}
```

GCC and Clang also support function multiversioning (`__attribute__((target_clones("avx2","default")))`). The cost is code size and build complexity; do it only for confirmed hot kernels.

### Python: let the library do it

```python
import numpy as np

a = np.arange(1_000_000, dtype=np.int64)
total = a.sum()          # NumPy uses SIMD and multiple accumulators internally
clipped = np.clip(a, 0, 1000)
```

NumPy, `numpy.dot` (BLAS), PyTorch, and similar libraries are already vectorized. A pure-Python loop over a million elements cannot compete; the win is using the library, not writing intrinsics.

## When SIMD does not help

- **Memory-bound loops:** the arithmetic is already hidden behind DRAM latency/bandwidth. Vectorizing it changes little.
- **Branchy, data-dependent work:** masks handle some cases, not arbitrary control flow.
- **Gather-heavy access:** scattered indices require gather instructions, which are slow.
- **Tiny or variable-length inputs:** setup and tail handling dominate.
- **Portability costs outweigh the gain:** a 10% gain for a per-CPU build matrix is often not worth it.

## Anti-patterns

1. Vectorizing a loop that is bandwidth-bound.
2. Using intrinsics without a scalar fallback or a runtime check.
3. Assuming the compiler vectorized; check the report or the assembly.
4. Forgetting the scalar tail, so `n % width` elements are skipped.
5. One accumulator with SIMD, recreating the dependency chain.
6. Misaligned loads with aligned intrinsics (`_mm256_load_si256` on unaligned data).
7. Hand-writing intrinsics for a loop the compiler already handles well.
8. Ignoring denormals, NaNs, and floating-point reassociation rules.

## Checklist

- Is the loop independent across iterations?
- Is the data contiguous and the stride constant?
- Is the compiler allowed to assume no aliasing?
- Did the optimization report confirm vectorization?
- Is there a scalar tail for the non-multiple case?
- Is the bottleneck memory rather than arithmetic?
- Are you keeping a portable fallback?
- Did you measure the actual speedup rather than assume the vector width?

## Related

- `memory.md` for locality and bandwidth limits.
- `codecomp.md` for inspecting the generated vector instructions.
- `parallelism.md` for scaling across cores, which composes with SIMD.

