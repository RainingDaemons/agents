# Type Sizes

## Description

Why hot types should be small, how to measure their size and layout, and how to shrink them. Rust types larger than 128 bytes are copied with `memcpy` instead of inline code; large structs increase memory traffic, cache pressure, and allocation size. Shrinking a hot type often improves speed without changing the algorithm.

## When to use

- A profile shows `memcpy` or `clone` in non-trivial amounts.
- Memory use is high, or a hot `Vec`/`HashMap` holds large values.
- An enum has one outsized variant.
- Indices use `usize` everywhere on 64-bit targets.
- A `Vec` field is stored in many instances but is usually empty.
- You want to protect a hot layout from regressions.

## Measure

```rust
use std::mem::{align_of, size_of, size_of_val};

println!("{} {}", size_of::<HotType>(), align_of::<HotType>());
println!("{}", size_of_val(&instance));
```

```bash
# Full layout: size, alignment, discriminant, field order, padding
RUSTFLAGS=-Zprint-type-sizes cargo +nightly build --release
rustc +nightly -Zprint-type-sizes input.rs

# Compact view
cargo install top-type-sizes
top-type-sizes target/release/app
```

`-Zprint-type-sizes` output shows the size and alignment, the discriminant, per-variant sizes (largest first), field order (the compiler reorders under `repr(Rust)`), and padding.

## Code examples

### Measure before and after

```rust
use std::mem::size_of;

struct Hot {
    a: u64,
    b: u8,
    c: u64,
    d: u8,
}

fn main() {
    println!("{}", size_of::<Hot>());   // 24: compiler reorders, 6 bytes padding
}
```

Do not reorder fields by hand under `repr(Rust)`; the compiler already minimizes size. Under `repr(C)`, it does not.

### Box a large enum variant

```rust
// SLOW: every value is as large as the biggest variant
type Large = [u8; 100];

enum Event {
    Tick,
    Value(i32),
    Payload(i32, Large),
}
```

```rust
// FAST: 16 bytes or so; the rare large variant pays one allocation
enum Event {
    Tick,
    Value(i32),
    Payload(Box<(i32, Large)>),
}
```

This is a net win when the large variant is rare. Clippy's `large_enum_variant` flags it.

### Smaller integers for indices

```rust
// Larger values, more memory traffic
struct Node {
    parent: usize,
    first_child: usize,
    next_sibling: usize,
}
```

```rust
// Half the size on 64-bit; coerce to usize at use points
struct Node {
    parent: u32,
    first_child: u32,
    next_sibling: u32,
}
```

Use the smallest integer that safely fits the data. Conversion cost is usually negligible compared with the cache savings; validate the bound.

### `Vec` vs `Box<[T]>` vs `ThinVec`

```rust
use std::mem::size_of_val;

let v: Vec<u32> = vec![1, 2, 3];
assert_eq!(size_of_val(&v), 3 * size_of::<usize>());    // ptr, len, cap

let bs: Box<[u32]> = v.into_boxed_slice();
assert_eq!(size_of_val(&bs), 2 * size_of::<usize>());   // ptr, len
```

```rust
// ThinVec stores len/cap next to the elements: one word
use thin_vec::ThinVec;

struct Node {
    children: ThinVec<u32>,
}
```

`ThinVec` is useful when a hot struct contains many vectors that are usually empty. It is not a drop-in for every `Vec`; measure ergonomics and performance.

### Niche optimization

```rust
use std::mem::size_of;

assert_eq!(size_of::<Option<&u8>>(), size_of::<&u8>());          // no tag
assert_eq!(size_of::<Option<Box<u8>>>(), size_of::<Box<u8>>());  // no tag
assert_eq!(size_of::<Option<std::num::NonZeroU32>>(), 4);        // no tag
assert_eq!(size_of::<Option<u32>>(), 8);                         // tag needed
```

Using `Option<NonZeroU32>` instead of `Option<u32>` can halve a field. `Option<Box<T>>` and `Option<&T>` are free.

### Struct-of-arrays for partial access

```rust
// SLOW: a loop touching only `x` streams the whole particle
struct Particle {
    x: f32,
    y: f32,
    z: f32,
    vx: f32,
    vy: f32,
    vz: f32,
}
```

```rust
// FAST: the `x` update reads one contiguous array
struct Particles {
    x: Vec<f32>,
    y: Vec<f32>,
    z: Vec<f32>,
    vx: Vec<f32>,
    vy: Vec<f32>,
    vz: Vec<f32>,
}
```

Use struct-of-arrays when hot loops touch a subset of fields; keep array-of-structs when they touch all fields together.

---

### Protect hot types with static assertions

```rust
#[cfg(target_arch = "x86_64")]
static_assertions::assert_eq_size!(HotType, [u8; 64]);
```

This catches accidental growth in CI. Scope it to the platform where the size is guaranteed.

## Anti-patterns

1. Guessing a type's size instead of calling `size_of`.
2. Hand-reordering fields under `repr(Rust)`; it is unnecessary.
3. Keeping a huge variant in an enum that is mostly small.
4. Using `usize` for every index when `u32` fits.
5. `Option<u32>` where `Option<NonZeroU32>` is possible.
6. A `Vec` field in every instance when most are empty (`ThinVec`).
7. Array-of-structs for loops that touch one field.
8. Boxing everything to shrink types; allocation can cost more than the size saved.
9. Shrinking cold types; only hot, oft-instantiated types matter.
10. Changing layout without re-benchmarking.

## Checklist

- Is the hot type's size measured, not assumed?
- Did `-Zprint-type-sizes` show unexpected padding or a large variant?
- Is the largest enum variant boxed if it is rare?
- Are indices and flags using the smallest safe integer?
- Could niche optimization remove an `Option` tag?
- Is a frequently empty `Vec` worth replacing with `ThinVec`?
- Does the loop access pattern suggest AoS or SoA?
- Is there a static assertion protecting the layout?
- Did the change improve the benchmark and memory profile?

## Related

- `heap.md` for allocation cost and capacity.
- `std.md` for `Vec`/`Box<[T]>` methods.
- `rustonomicon.md` for layout rules and `repr`.
- `machinecode.md` for verifying `memcpy` removal.
