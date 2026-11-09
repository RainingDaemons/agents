# Memory Hierarchy and Cache-Aware Analysis

## Description

Why the asymptotic model is not enough for memory-bound code, and how to restructure data and loops for locality. Covers the cache hierarchy, latency vs bandwidth, cache lines, prefetching, tiling/blocking, AoS vs SoA, pointer chasing, alignment, and the external-memory model.

Key idea: an L1 hit costs a few cycles, a DRAM access costs hundreds. Two algorithms with the same operation count can differ by an order of magnitude because of **where the data lives**.

## Use cases

- Large arrays, matrices, joins, scans, and sorting that do not fit in cache.
- Code that is "memory-bound": more time waiting for data than computing.
- Choosing between an array and a linked structure, or between AoS and SoA.
- Matrix traversal and multiplication order.
- Database, serialization, and I/O-heavy code where round trips dominate.
- Any profiling result with a high cache-miss rate.

## Approximate latencies

Orders of magnitude, not guarantees:

| Level | Typical size | Latency | Bandwidth |
|---|---|---|---|
| L1 | ~32 KB/core | a few cycles (~1–2 ns) | very high |
| L2 | ~256 KB–1 MB/core | ~10 cycles (~4–10 ns) | high |
| L3 | several MB, shared | tens of cycles (~20–50 ns) | moderate |
| DRAM | GB | ~100 ns | ~10 GB/s per core |
| SSD | TB | ~100 µs | ~GB/s |
| Network (datacenter) | — | ~0.5 ms | — |
| Disk seek | — | ~ms | — |

Two separate properties:

- **Latency:** time until the requested data arrives.
- **Bandwidth:** how many bytes per second can be moved in total.

A loop that streams data is usually **bandwidth-bound**; a loop that chases pointers is usually **latency-bound**. They need different fixes.

## The cache line

Memory moves in **cache lines**, typically 64 bytes. Loading one byte loads the whole line. Consequences:

- Sequential access uses the entire line; random access wastes most of it.
- Structs smaller than a line share it; false sharing across threads is possible.
- Array-of-structs can waste bandwidth when only one field is used.

## Code examples

### Row-major vs column-major traversal

```python
# FAST for row-major storage: contiguous, prefetcher-friendly
total = 0
for i in range(n):
    for j in range(m):
        total += grid[i][j]
```

```python
# SLOW for row-major storage: jumps a full row per step
total = 0
for j in range(m):
    for i in range(n):
        total += grid[i][j]
```

Both are `O(n·m)`. If a row is larger than the cache, the second version can miss on nearly every access; the first streams. The fix is to match the traversal to the storage order, or to block.

### Blocking / tiling: keep a tile in cache

```python
def matmul_blocked(a, b, n, block=64):
    c = [[0.0] * n for _ in range(n)]
    for ii in range(0, n, block):
        for jj in range(0, n, block):
            for kk in range(0, n, block):
                for i in range(ii, min(ii + block, n)):
                    for k in range(kk, min(kk + block, n)):
                        aik = a[i][k]
                        for j in range(jj, min(jj + block, n)):
                            c[i][j] += aik * b[k][j]
    return c
```

Same `O(n³)` arithmetic as the naive triple loop, but each tile is reused while it is still in cache. Block size should be chosen so the working set fits in L1/L2, not blindly.

### Array-of-structs vs struct-of-arrays

```python
# AoS: each record is a contiguous block; good when you use all fields
particles = [{"x": 0.0, "y": 0.0, "vx": 0.0, "vy": 0.0} for _ in range(n)]

# SoA: each field is a contiguous array; good when you use one field at a time
xs = [0.0] * n
ys = [0.0] * n
vxs = [0.0] * n
vys = [0.0] * n
```

A position update over SoA streams one or two arrays. Over AoS it streams records that are `4×` larger than needed, wasting bandwidth and cache capacity. AoS wins when the access pattern touches all fields of one record (few records).

### Pointer chasing vs contiguous scan

```python
# linked list: each step depends on the previous load, cache miss per node
total = 0
node = head
while node:
    total += node.value
    node = node.next
```

```python
# array: independent loads, prefetcher hides latency
total = 0
for x in values:
    total += x
```

Both are `O(n)`, but the list has a **dependent load chain**: the next address is unknown until the previous load returns, so latency is not hidden. The array enables prefetching and multiple outstanding loads. This is the single largest constant-factor difference in many "same complexity" comparisons.

### Prefetching

```c++
for (int i = 0; i < n; i++) {
    __builtin_prefetch(&a[i + 16]);   // hint: fetch ahead
    total += a[i];
}
```

Hardware prefetchers already handle simple sequential and strided patterns. Manual prefetching helps with irregular but predictable access (indirect indices, multiple streams) and can hurt when the distance is wrong. Measure.

### Alignment and packing

```c++
struct Bad  { char a; long b; char c; };    // 24 bytes: 7 bytes of padding after 'a'
struct Good { long b; char a; char c; };    // 16 bytes: the long is aligned, no internal gap
```

A `struct` is padded to its alignment. Ordering fields large-to-small can reduce `sizeof`, which means fewer cache lines per element. `alignas(64)` can align hot structures to a cache line, but also increases footprint.

### External-memory model

When data does not fit in RAM, count **block transfers** instead of operations:

- Scan of `N` records: `O(N/B)` transfers.
- Sequential sort: `O((N/B) log_{M/B}(N/B))` transfers, where `M` is memory size and `B` is block size.
- The model says: make accesses sequential, keep the working set in `M`, and avoid one transfer per element.

This is why reading a file line by line is fine, but `seek`-ing per record is not.

## Anti-patterns

1. Column-major traversal of row-major data (or the reverse).
2. Walking a linked list or tree over a large working set in performance-critical code.
3. AoS when only one field is read per pass; or SoA when all fields of one record are read together.
4. Random access into a large array (hash tables, sparse matrices) without accounting for misses.
5. Pointer-heavy structures (`list` of small objects, `vector<unique_ptr>`) where a contiguous vector would do.
6. Per-element I/O or database round trips instead of batching.
7. Parallelizing a memory-bandwidth-bound loop: more cores do not add bandwidth.
8. Ignoring false sharing between threads (see `parallelism.md`).
9. Assuming "it fits in RAM" means "it is fast".

## Checklist

- How big is the working set at each level of the hierarchy?
- Is the access sequential, strided, or random?
- Is the loop latency-bound (dependent loads) or bandwidth-bound (streaming)?
- Does the traversal match the storage order?
- Would tiling make a reused tile fit in cache?
- Is the layout AoS or SoA, and does it match the access pattern?
- Are there pointer indirections in the hot path that a contiguous array could remove?
- If the data exceeds RAM, are accesses block-sequential?
- Did the profiler show cache misses, or are you guessing from the code?

## Related

- `profiling-benchmarking.md` for measuring misses and bandwidth.
- `simd.md` for what to do once data is in registers.
- `parallelism.md` for false sharing and bandwidth saturation.
- `asymptotic.md` for why the RAM model disagrees here.

