# Hashing

## Description

How to choose and use hashers in `HashMap`/`HashSet`. The default hasher is SipHash 1-3: high quality and resistant to HashDoS, but relatively slow, especially for short integer keys. Faster hashers trade collision resistance for speed and must never be used with untrusted input. The other half of the problem is using the map correctly: pre-size it, use `entry`, and avoid hashing large keys repeatedly.

## When to use

- A profile shows time in `core::hash`, `SipHash`, or `HashMap` operations.
- Maps are keyed by integers, IDs, or short trusted strings.
- A hot loop does `contains_key` followed by `insert`.
- A map is built per request or per iteration without capacity.
- Keys are large and only part of the key is needed for equality.

Do not switch hashers for untrusted keys (user input, request parameters, network data) without a HashDoS mitigation.

## Choosing a hasher

| Need | Prefer | Notes |
|---|---|---|
| Untrusted input, default safety | `std::collections::HashMap` | SipHash, slower, DoS-resistant |
| Fast hash for trusted keys | `rustc_hash::FxHashMap`/`FxHashSet` | Low-quality, very fast for integers |
| Slightly higher quality than Fx | `fnv::FnvHashMap`/`FnvHashSet` | A little slower than Fx |
| AES-assisted | `ahash::AHashMap`/`AHashSet` | Uses AES instructions where available |
| Random integer newtype | `nohash_hasher` | Skips hashing entirely |
| Ordered iteration or ranges | `BTreeMap`/`BTreeSet` | Not a hasher; often better for small/ordered data |
| Small map (≲ 16 entries) | `Vec<(K, V)>` or sorted vec | Linear search beats hashing |

## Checks

```bash
cargo clippy -- -W clippy::perf
perf record -g ./target/release/app    # look for hash frames
```

Probes:

```bash
rg -n 'HashMap|HashSet' src
rg -n 'contains_key\(|\.entry\(|with_capacity|reserve\(' src
rg -n 'std::collections::Hash(Map|Set)' src
rg -n 'FxHash|AHash|FnvHash|nohash' src Cargo.toml
rg -n 'HashDoS|untrusted|user input' src docs 2>/dev/null
```

## Code examples

### Default hasher on hot integer keys

```rust
use std::collections::HashMap;

// SLOW: SipHash on small integer keys in a hot loop
let mut counts: HashMap<u64, u64> = HashMap::new();
for id in ids {
    *counts.entry(id).or_insert(0) += 1;
}
```

```rust
use rustc_hash::FxHashMap;

// FAST: FxHash for trusted integer keys
let mut counts: FxHashMap<u64, u64> = FxHashMap::default();
for id in ids {
    *counts.entry(id).or_insert(0) += 1;
}
```

Only do this when the keys are trusted. For request-derived keys, keep the default hasher or use a keyed hash with a secret.

### Pre-size the map

```rust
// SLOW: repeated rehashing while the map grows
let mut map = HashMap::new();
for (k, v) in pairs {
    map.insert(k, v);
}
```

```rust
// FAST: one allocation, no rehashing
let mut map = HashMap::with_capacity(pairs.len());
for (k, v) in pairs {
    map.insert(k, v);
}
```

### `entry` instead of two lookups

```rust
// SLOW: hashes and probes twice
if !map.contains_key(&key) {
    map.insert(key, value);
}
```

```rust
// FAST: one lookup
map.entry(key).or_insert(value);
```

For updates, `entry` also avoids recomputing the key or value:

```rust
*map.entry(key).or_insert_with(Vec::new).push(item);
```

### No hash at all for random integer keys

```rust
use nohash_hasher::BuildNoHashHasher;

// Keys are dense/random integers; hashing adds nothing
let mut map: HashMap<u64, V, BuildNoHashHasher<u64>> = HashMap::default();
```

Use only when key values are already well distributed. Clustered IDs (sequential, multiples of a page size) degrade badly.

### Byte-wise hashing

`#[derive(Hash)]` hashes each field separately. For some hash functions it is faster to hash the raw bytes of the whole value, but only for types with no padding and a stable layout.

```rust
// Advanced: requires a type with no padding, e.g. #[repr(C)] + all fields sized
#[derive(zerocopy::ByteHash)]
#[repr(C)]
struct Key {
    a: u32,
    b: u32,
}
```

This is highly dependent on the hash function and layout; measure. Never hash padding bytes: they are uninitialized and leak memory contents or produce unstable hashes.

### Clippy: prevent accidental use of the default map

```toml
# clippy.toml
disallowed-types = ["std::collections::HashMap", "std::collections::HashSet"]
```

Use this only after deciding that all maps in the crate should use an alternative hasher.

## Anti-patterns

1. Using `FxHashMap`/`ahash` for untrusted keys, enabling HashDoS.
2. Switching hashers without a profile showing hashing is hot.
3. Using a hash map for a handful of entries where a `Vec` scan is faster.
4. Forgetting `with_capacity` when the size is known.
5. `contains_key` followed by `insert` instead of `entry`.
6. Hashing a large struct when a small derived key (ID, prefix) is sufficient.
7. Hashing padding bytes via `unsafe` casts.
8. Assuming a switch to `ahash` is always faster; in rustc it was slower than `fxhash`.
9. Iterating a `HashMap` and expecting a stable order.

## Checklist

- Is the input trusted? If not, is HashDoS handled?
- Did a profile show hashing as the bottleneck?
- Is the map pre-sized where the size is known?
- Is `entry` used instead of two lookups?
- Are keys as small as possible?
- Is the collection the right one (map vs sorted vec vs `BTreeMap`)?
- Did you benchmark the hasher change on the real workload?
- Is `clippy.toml` used to prevent accidental regressions (if standardized)?

## Related

- `heap.md` for map allocation and growth.
- `iterators-cost.md` for collection operation costs.
- `linting.md` for `disallowed-types` and performance lints.
- `profiling.md` for confirming a hashing hot spot.
