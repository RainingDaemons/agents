# Machine Code

## Description

When a small piece of code is very hot, inspect what the compiler actually generated. The source can look efficient while the assembly still contains bounds checks, unvectorized loops, unexpected calls, or `memcpy` for large values. Machine-code inspection answers questions that profiles cannot: was this inlined, was this vectorized, is this check still there, and what instructions dominate the loop?

## When to use

- A hot loop is slower than its operation count suggests.
- You suspect bounds checks, a missed inlining opportunity, or failed vectorization.
- Choosing between two implementations with the same asymptotic complexity.
- Writing or reviewing SIMD/intrinsics code (`core::arch`).
- Confirming that an optimization attribute or `unsafe` block changed the output.

Do not inspect the assembly before a profile identifies the hot function. Optimizing cold code at the instruction level is wasted effort.

## Tools

| Tool | Use |
|---|---|
| Compiler Explorer (godbolt.org) | Small snippets, side-by-side with source |
| `cargo asm` / `cargo-show-asm` | Assembly for a function in a full crate |
| `objdump -d` | Disassembly of a binary |
| `rustc --emit asm` | Assembly file from a crate |
| `llvm-mca` | Throughput/latency model of a loop |
| `perf stat` | Cycles, instructions, cache and branch misses |

```bash
cargo install cargo-show-asm
cargo asm --rust my_crate::hot_fn
cargo asm --rust --simplify my_crate::hot_fn
objdump -d --no-show-raw-insn target/release/app | rg -A30 'hot_fn'
rustc --emit asm -O src/lib.rs
```

## What to look for

| Pattern | Meaning |
|---|---|
| `call` to a small function | Not inlined |
| `panic_bounds_check` / `cmp` + `ja` before every load | Bounds check remains |
| `memcpy` / `rep movsb` | Large value copy or `Vec` clone |
| `vpaddd`, `vfmadd`, `vmovdqu` | SIMD instructions |
| `mov` in a tight loop | Memory-bound; arithmetic is not the limit |
| `div` / `idiv` | Expensive division in the loop |
| `lock cmpxchg`, `lock xadd` | Atomic contention |
| `cmov` | Branchless conditional move |

## Code examples

### Check inlining

```bash
cargo asm --rust my_crate::process | rg 'call'
```

If `transform` appears as a `call` inside the hot loop, it was not inlined. Add `#[inline]` and confirm; see `inlining.md`.

### Check bounds checks

```bash
cargo asm --rust my_crate::sum_indexed | rg 'panic_bounds_check|cmp|jae|ja'
```

```rust
// Candidate for a remaining check
pub fn sum_indexed(data: &[u8]) -> u32 {
    let mut s = 0;
    for i in 0..data.len() {
        s += data[i] as u32;
    }
    s
}
```

```rust
// Usually compiles to the same loop without a check
pub fn sum_iter(data: &[u8]) -> u32 {
    data.iter().map(|&x| x as u32).sum()
}
```

Inspect both. The iterator version usually removes the check because the length is known; the indexed version may still have one.

### Check vectorization

```rust
pub fn add_arrays(a: &[f32], b: &[f32], out: &mut [f32]) {
    for ((o, &x), &y) in out.iter_mut().zip(a).zip(b) {
        *o = x + y;
    }
}
```

```bash
cargo asm --rust my_crate::add_arrays | rg 'vfmadd|vaddps|vmulps|addps'
```

If you see scalar `addss` per element instead of `vaddps`, the loop was not vectorized. Causes: aliasing, a loop-carried dependency, a branch, or a target without the needed feature. `-C target-cpu=native` can enable wider instructions (at the cost of portability).

### Explicit intrinsics with a fallback

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2")]
unsafe fn sum_avx2(data: &[i32]) -> i32 {
    let mut acc = _mm256_setzero_si256();
    let mut i = 0;
    while i + 8 <= data.len() {
        let v = _mm256_loadu_si256(data.as_ptr().add(i) as *const __m256i);
        acc = _mm256_add_epi32(acc, v);
        i += 8;
    }
    let mut total = 0;
    let mut lanes = [0i32; 8];
    _mm256_storeu_si256(lanes.as_mut_ptr() as *mut __m256i, acc);
    for lane in lanes {
        total += lane;
    }
    for &x in &data[i..] {
        total += x;
    }
    total
}

fn sum(data: &[i32]) -> i32 {
    #[cfg(target_arch = "x86_64")]
    {
        if std::is_x86_feature_detected!("avx2") {
            // SAFETY: the feature check above guarantees AVX2 is available.
            return unsafe { sum_avx2(data) };
        }
    }
    data.iter().sum()
}
```

`#[target_feature]` functions are `unsafe` to call unless the feature is guaranteed. Provide a scalar fallback and a runtime check; do not ship a binary that requires AVX2 without knowing the target hardware.

### Understand the loop with `llvm-mca`

```bash
rustc --emit asm -O -C target-cpu=native loop.rs
llvm-mca -mcpu=native loop.s
```

`llvm-mca` reports cycles per iteration, which port is the bottleneck, and whether the loop is latency- or throughput-bound. Use it on the isolated hot loop.

## Anti-patterns

1. Inspecting assembly for code that is not hot.
2. Assuming a bounds check exists (or does not) without looking.
3. Assuming vectorization happened; check the instructions.
4. Hand-writing intrinsics without a scalar fallback and runtime detection.
5. Using `-C target-cpu=native` in binaries shipped to unknown hardware.
6. Ignoring `memcpy`/`call` in the hot loop because the source looks fine.
7. Chasing instruction-level wins before fixing the algorithm or allocation.
8. Reading unoptimized (`dev`) assembly and drawing conclusions.

## Checklist

- Did a profile identify this function as hot?
- Is the build `--release` with representative flags?
- Was the function inlined? Are there unexpected `call`s?
- Are bounds checks gone? Is there a `panic_bounds_check` path?
- Is the loop vectorized? Which width?
- Is the loop latency-bound, throughput-bound, branch-bound, or memory-bound?
- Is every `unsafe` intrinsic justified, documented, and covered by a fallback?
- Did the change improve the benchmark, not only the assembly?

## Related

- `inlining.md` for attributes and hot/cold splitting.
- `iterators.md` for removing bounds checks safely.
- `profiling.md` for finding the hot function.
- `buildconfig.md` for `target-cpu`, LTO, and release settings.
