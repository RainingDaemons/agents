# Compiling: Compile Time, Generics, Macros, and LLVM IR

## Description

How to find and reduce compile time without hurting runtime performance. Rust compile time is dominated by a few things: generic monomorphization (the same function compiled many times), large macro expansions, LLVM optimizing a lot of IR, linking, and debug info. The tools below show where the time goes, and the fixes are usually small, local changes.

## When to use

- `cargo build` or `cargo check` is slow enough to hurt iteration.
- CI build time or cache size is a problem.
- A generic function or trait-heavy API is instantiated in many places.
- A procedural macro generates a large amount of code.
- Link time dominates (large binaries, incremental builds).
- You need to reduce binary size as well as compile time.

## Checks

```bash
cargo build --timings                 # Gantt chart of the crate graph
cargo check                           # fast feedback during development
cargo llvm-lines | head -30           # functions producing the most LLVM IR
cargo llvm-lines --crates | head      # per-crate totals
RUSTFLAGS="-Zmacro-stats" cargo +nightly build   # macro-generated code
cargo +nightly rustc -- -Zmacro-stats            # leaf crate only
cargo expand path::to::item           # expanded macro output
cargo tree -d                         # duplicate dependencies
cargo udeps                           # unused dependencies
```

Probes:

```bash
rg -n '^\[profile\.' Cargo.toml
rg -n 'debug\s*=\s*(true|2|"full")|incremental' Cargo.toml .cargo 2>/dev/null
rg -n 'fn .*<[A-Z]|impl<.*>|where ' src | head -50
rg -n '#\[derive\(|macro_rules!|proc_macro' src | head -50
rg -n 'default-features\s*=\s*false|features\s*=' Cargo.toml
```

## Code examples

### Generic bloat: move the non-generic part out

```rust
// SLOW to compile: the whole body is instantiated for every `P`
pub fn read<P: AsRef<Path>>(path: P) -> std::io::Result<Vec<u8>> {
    let mut file = File::open(path.as_ref())?;
    let mut bytes = Vec::new();
    std::io::Read::read_to_end(&mut file, &mut bytes)?;
    Ok(bytes)
}
```

```rust
// FAST to compile: one non-generic instantiation
pub fn read<P: AsRef<Path>>(path: P) -> std::io::Result<Vec<u8>> {
    fn inner(path: &Path) -> std::io::Result<Vec<u8>> {
        let mut file = File::open(path)?;
        let mut bytes = Vec::new();
        std::io::Read::read_to_end(&mut file, &mut bytes)?;
        Ok(bytes)
    }
    inner(path.as_ref())
}
```

The thin generic wrapper is still instantiated, but the expensive body is compiled once. This is the pattern used by `std::fs::read`.

### Generic combinators repeated everywhere

```rust
// Many instantiations of `Option::map` across a large codebase
let value = maybe.map(|x| x + 1);
```

```rust
// Sometimes cheaper for compile time (and binary size)
let value = match maybe {
    Some(x) => Some(x + 1),
    None => None,
};
```

This is a micro-optimization for compile time; use it where `cargo llvm-lines` shows the combinator is a top contributor, not everywhere.

### Proc macros generating a lot of code

```bash
RUSTFLAGS="-Zmacro-stats" cargo +nightly build
```

Look for a macro whose generated code is comparable to the hand-written code of the crate. Options: remove the macro, replace it with a cheaper one, or change it to generate less code. `cargo expand` shows the actual output.

### Link time

```toml
# .cargo/config.toml
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "link-arg=-fuse-ld=mold"]
```

`mold` (Linux) and `lld` are faster linkers than the default. They are especially noticeable on large binaries and incremental builds.

### Disable debug info outside profiling

```toml
[profile.dev]
debug = false          # faster builds, worse debugging

[profile.release]
debug = false
```

Keep a separate `profiling` profile with `debug = "line-tables-only"` when you need symbols.

### Workspace and incremental compilation

- Split a large crate into smaller workspace crates so changes do not rebuild everything and so compilation can reuse work.
- Use `cargo check` while iterating; it skips codegen.
- Keep dependencies lean: disable default features, remove unused crates, deduplicate versions.
- Avoid rebuilding dependencies by pinning features and toolchains.

### Experimental options

- The parallel front-end (`-Z threads=N` on nightly) can reduce front-end time.
- The Cranelift backend compiles much faster but produces slower runtime code; useful for dev/test, not release.
- Evaluate them against the pinned toolchain; do not adopt unstable flags in production builds without a reason.

## Anti-patterns

1. Measuring compile time on a cold cache and comparing it with a warm one.
2. Optimizing build config before removing unused dependencies and generics.
3. Sprinkling `#[inline(always)]` or heavy generics to "help" the compiler; both increase compile time.
4. Keeping `debug = 2` in release builds you do not profile.
5. Denying incremental compilation in development.
6. Using a slow linker and blaming rustc.
7. Ignoring macro expansion size; one proc macro can dominate the crate.
8. Compiling all workspace crates when only one changed.

## Checklist

- Did `cargo build --timings` identify the critical path?
- Did `cargo llvm-lines` identify the biggest generic contributors?
- Are large generic bodies split into non-generic inner functions?
- Did `-Zmacro-stats` check for oversized macro expansion?
- Are unused dependencies and features removed?
- Is a faster linker configured where supported?
- Is debug info disabled outside profiling builds?
- Is the workspace split so incremental builds reuse work?
- Did runtime performance and binary size stay acceptable after the changes?

## Related

- `buildconfig.md` for profiles, LTO, and binary size.
- `inlining.md` for the runtime side of inlining.
- `machinecode.md` for inspecting what the compiler produced.
- `linting.md` for automated checks during the build.
