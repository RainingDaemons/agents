# Build Configuration

## Description

How Cargo and rustc settings change runtime speed, memory use, binary size, compile time, debugability, and portability — often without touching a line of code. The single most important setting is using a release build; the rest are trade-offs to measure one at a time.

## When to use

- A program is slow and you have not verified it is a release build.
- Shipping a binary and choosing between size and speed.
- Enabling LTO, `codegen-units = 1`, `target-cpu`, PGO, or a different allocator.
- Compile times or binary size are a problem.
- Profiling a release binary and needing symbols.
- Deciding whether a deployment target allows CPU-specific instructions.

## Settings

| Setting | Effect | Cost |
|---|---|---|
| `opt-level = 3` | Maximum runtime optimization | Compile time |
| `opt-level = "s"` / `"z"` | Smaller binary | Some runtime speed |
| `lto = "thin"` / `"fat"` | Whole-program optimization, often 10–20% | Much slower builds |
| `codegen-units = 1` | Better optimization, smaller binary | Slower, less parallel compilation |
| `panic = "abort"` | Smaller/faster, no unwinding | No `catch_unwind`, different behavior |
| `strip = "symbols"` | Smaller binary | Poor profiles and backtraces |
| `debug = "line-tables-only"` | Source lines in profiles | Larger binary than stripped |
| `-C target-cpu=native` | Newer instructions, more vectorization | Binary not portable |
| PGO / BOLT | 10%+ on large branchy binaries | Training workload and build complexity |
| Alternative allocator | Can cut allocation cost significantly | Binary size, compile time, platform variance |

## Checks

```bash
cargo build --release          # confirm target/release, not target/debug
cargo build --timings          # where compile time goes
cargo bloat --release           # what takes space
cargo bloat --release -n 20     # top symbols
rustc --print cfg
rustc --print cfg -C target-cpu=native
```

Probes:

```bash
rg -n '\[profile\.|opt-level|lto|codegen-units|panic|strip|debug' Cargo.toml
rg -n 'cargo (build|run)( |$)' .github scripts Makefile justfile 2>/dev/null
rg -n 'global_allocator|mimalloc|jemalloc' src Cargo.toml
rg -n 'target-cpu|target-feature' .cargo Cargo.toml 2>/dev/null
```

## Code examples

### Release and profiling profiles

```toml
# Cargo.toml
[profile.release]
opt-level = 3
lto = "thin"
codegen-units = 1
panic = "abort"
strip = "symbols"        # do NOT use when profiling

# Profiling without changing release behavior
[profile.profiling]
inherits = "release"
debug = "line-tables-only"
strip = "none"
```

```bash
cargo build --profile profiling
perf record -g ./target/profiling/app
```

`lto = false` is not "no LTO": it leaves thin local LTO enabled. Use `lto = "off"` to disable it entirely. Profile settings in dependencies are ignored; only the workspace root matters.

### Faster dependencies, fast local builds

```toml
[profile.dev.package."*"]
opt-level = 3
```

Dependencies are compiled optimized while your crate stays fast to rebuild. This is a good default for development.

### CPU-specific instructions

```bash
RUSTFLAGS="-C target-cpu=native" cargo build --release
```

```toml
# .cargo/config.toml
[build]
rustflags = ["-C", "target-cpu=native"]
```

Verify the features are detected:

```bash
rustc --print cfg -C target-cpu=native | rg 'target_feature'
```

If deployment hardware differs from the build machine, do not use `native`. Target a baseline (`x86-64-v2`, `x86-64-v3`) or ship runtime-dispatched code with `is_x86_feature_detected!`.

### Alternative allocators

```toml
[dependencies]
mimalloc = "0.1"
# or: tikv-jemallocator = "0.5"
```

```rust
#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

```rust
// jemalloc variant
#[global_allocator]
static GLOBAL: tikv_jemallocator::Jemalloc = tikv_jemallocator::Jemalloc;
```

Measure: gains depend on the workload and platform, and binary size and compile time grow. On Linux, jemalloc can use transparent huge pages via `MALLOC_CONF`.

### Profile-guided optimization

```bash
cargo install cargo-pgo
cargo pgo build
./target/x86_64-unknown-linux-gnu/release/app --benchmark-input workloads/
cargo pgo optimize
```

PGO improves branch layout using real workloads; keep the training input representative. BOLT can be combined with `cargo-pgo`.

### Minimizing binary size

```toml
[profile.release]
opt-level = "z"
lto = "fat"
codegen-units = 1
panic = "abort"
strip = "symbols"
```

Check the result with `cargo bloat` and `ls -lh`. Smaller is not automatically faster: measure if runtime matters.

## Anti-patterns

1. Benchmarking or profiling a dev build.
2. Enabling every aggressive setting at once, then not knowing which one helped.
3. Using `-C target-cpu=native` for binaries shipped to unknown CPUs.
4. Stripping symbols and then wondering why profiles are unreadable.
5. Using `panic = "abort"` without checking that no code relies on unwinding.
6. Adding an alternative allocator without measuring; it can be neutral or worse.
7. Assuming `lto = false` disables LTO.
8. Tuning profiles in a dependency's `Cargo.toml`; they are ignored.
9. Using PGO with a training workload that does not resemble production.

## Checklist

- Is the measured/shipped build a release (or dedicated profiling) build?
- Is each setting justified by a measurement or a clear requirement?
- Is `target-cpu` compatible with the deployment hardware?
- Are symbols available when profiling and stripped when shipping?
- Are `panic` settings compatible with the error-handling strategy?
- Was the allocator benchmarked on the target platform?
- Are compile time and binary size acceptable after the changes?
- Is the build reproducible and documented?

## Related

- `profiling.md` for debug info, frame pointers, and symbol demangling.
- `benchmarking.md` for validating each change.
- `compiling.md` for reducing compile time.
- `heap.md` for deciding whether an alternative allocator is worth it.
