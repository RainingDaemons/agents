# Inlining

## Description

How to get the compiler to inline the functions that matter, and how to stop it from inlining the ones that should stay out. Inlining removes call/return overhead and enables further optimization, but it also increases code size, compile time, and instruction-cache pressure. The compiler already inlines small functions and single-call-site functions; attributes are for the cases it gets wrong.

## When to use

- A profile shows call overhead or a hot function that is not inlined.
- A small cross-crate function is called in a hot loop.
- One call site is hot while others are cold.
- Code bloat or compile time increased after adding inline attributes.
- An error/panic path is inlined into a hot function and hurting code layout.

Do not add inline attributes "just in case". Measure with a profiler or by inspecting the generated code.

## Attributes

| Attribute | Meaning | Use when |
|---|---|---|
| none | Compiler decides | Default; usually right |
| `#[inline]` | Suggest inlining | Small cross-crate functions; generics are already monomorphized |
| `#[inline(always)]` | Strongly suggest inlining | Rare; a measured hot call site |
| `#[inline(never)]` | Strongly suggest not inlining | Cold wrappers, error paths, large rarely-called functions |
| `#[cold]` | Mark a function as rarely called | Error/panic paths; helps layout and inlining decisions |

Inlining is **non-transitive**: if `f` calls `g` and both should be inlined into `f`'s caller, both may need attributes.

## Checks

- Cachegrind tells you whether a function was inlined: an inlined function's first and last lines are not marked with event counts.
- `cargo asm`/`cargo-show-asm` or Compiler Explorer shows the call instruction (or its absence) in the generated code.
- Compare benchmarks and binary size before and after each attribute.

```bash
cargo asm --rust my_crate::hot_fn | rg 'call|jmp'
cargo bloat --release -n 20
```

## Code examples

### Small cross-crate function

```rust
// In a library crate: without `#[inline]`, cross-crate inlining depends on LTO
#[inline]
pub fn clamp_u8(x: u8, lo: u8, hi: u8) -> u8 {
    x.max(lo).min(hi)
}
```

For a function this small the compiler usually inlines it anyway; `#[inline]` makes cross-crate inlining reliable without full LTO. Do not add it to large functions.

### Hot and cold call sites of the same function

```rust
// One function, one hot call site and several cold ones
fn my_function() {
    one();
    two();
    three();
}
```

```rust
// Split: inline the hot body, keep the cold wrapper out of line
#[inline(always)]
fn inlined_my_function() {
    one();
    two();
    three();
}

#[inline(never)]
fn uninlined_my_function() {
    inlined_my_function();
}
```

Call `inlined_my_function` from the hot site and `uninlined_my_function` from cold sites. This gives the hot path the inline win without duplicating the body at every cold site.

### Outlining cold paths

```rust
// SLOW layout: the error path is inlined into the hot function
fn process(input: &[u8]) -> Result<Output, Error> {
    if input.is_empty() {
        return Err(Error::Empty);      // rarely taken
    }
    // ... hot work ...
}
```

```rust
// FAST layout: move the rare path out and mark it cold
#[cold]
#[inline(never)]
fn empty_error() -> Error {
    Error::Empty
}

fn process(input: &[u8]) -> Result<Output, Error> {
    if input.is_empty() {
        return Err(empty_error());
    }
    // ... hot work ...
}
```

`#[cold]` tells the compiler the path is unlikely, which helps branch layout and keeps the hot function small.

### Verify, do not assume

```bash
# Look for a call to the function in the hot loop; absence means it was inlined
cargo asm --rust my_crate::process | rg 'call.*transform'
```

Cachegrind's source annotation is another reliable check.

## Anti-patterns

1. `#[inline(always)]` on everything; it bloats code and can prevent better inlining elsewhere.
2. Adding `#[inline]` to large functions without a measurement.
3. Assuming `#[inline]` guarantees inlining; only `#[inline(always)]` is close, and still not absolute.
4. Expecting one attribute to propagate: inlining is non-transitive.
5. Inlining error/panic paths into hot functions instead of outlining them.
6. Measuring inline changes without also checking compile time and binary size.
7. Ignoring cross-crate boundaries, where inlining depends on LTO or attributes.
8. Optimizing inlining in a dev build, where nothing is representative.

## Checklist

- Did a profile or the generated code show a real call-overhead problem?
- Is the function small, or does it have a single hot call site?
- Are hot and cold call sites split where needed?
- Are cold/error paths outlined with `#[cold]`/`#[inline(never)]`?
- Was the effect verified in the machine code (Cachegrind or `cargo asm`)?
- Did you re-run the benchmark and check binary size and compile time?
- Is `#[inline(always)]` limited to the few places where it measured a win?

## Related

- `machinecode.md` for inspecting inlining and vectorization.
- `compiling.md` for compile-time and binary-size effects.
- `benchmarking.md` for measuring the change.
- `buildconfig.md` for LTO, which changes cross-crate inlining.
