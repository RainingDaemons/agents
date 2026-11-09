---
name: code-complexity-analysis
description: Analyzes source code to assess performance using algorithmic complexity (time and space) plus a machine-aware cost model (memory hierarchy, branches, ILP, SIMD, parallelism). Identifies bottlenecks and recommends algorithmic, data-structure, and constant-factor optimizations. Use when the user asks to review performance, compute Big-O, optimize a function, reduce complexity, profile a hot path, or compare algorithmic approaches.
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
---

# Algorithmic Complexity Analysis and Code Optimization

## Goal

Given a code fragment, module, or repository, this skill:

1. Establishes an explicit cost model and determines time and space complexity (best, average, worst).
2. Identifies the critical work that dominates cost, both asymptotically and on real hardware.
3. Proposes concrete improvements: fewer operations, better data structures, better algorithms, better memory behavior, fewer per-operation costs.
4. Delivers a verifiable result: optimized code, new complexity, measured or theoretical evidence, and trade-offs.

## When to use it

- "What is the complexity of this function?"
- "This is slow, how do I optimize it?"
- "Review the performance of this module."
- "Is there a better algorithm for this?"
- Code review where nested loops, repeated queries, heavy allocation, cache misses, or unsuitable structures are suspected.
- Choosing between approaches with different asymptotic and constant-factor behavior.

Do not use it for micro-optimizations without evidence that the code is on a critical path. Do not use pure asymptotic analysis for I/O, network, or database problems where round trips or bandwidth dominate; analyze those as I/O cost and suggest profiling.

## References

Twelve local guides in `references/` are the source material for this skill. Consult only the relevant guide instead of reading everything.

Each guide contains a description, use cases, code examples with before/after complexity, anti-patterns, and a checklist.

| Topic | Guide | Use it for |
|---|---|---|
| Recursion and recurrences | [recursion.md](references/recursion.md) | Base cases, divide and conquer, solving `T(n)`, stack depth |
| Backtracking | [backtracking.md](references/backtracking.md) | Search trees, pruning, branch and bound, symmetry breaking |
| Dynamic programming | [dynamicprog.md](references/dynamicprog.md) | State definition, transitions, memo vs tabulation, rolling arrays |
| Greedy algorithms | [greedy.md](references/greedy.md) | Greedy correctness, exchange arguments, when greedy fails |
| Graphs | [graphs.md](references/graphs.md) | Representations, BFS/DFS, shortest paths, MST, SCC, union-find |
| NP-hardness | [np-hardness.md](references/np-hardness.md) | Recognizing hard problems and choosing exact/FPT/approximation/solver |
| Asymptotic complexity | [asymptotic.md](references/asymptotic.md) | Big-O/Θ/Ω, amortized and expected cost, why constants matter |
| Compiler output and ILP | [codecomp.md](references/codecomp.md) | Reading assembly, flags, inlining, unrolling, aliasing, dependency chains |
| Profiling and benchmarking | [profiling-benchmarking.md](references/profiling-benchmarking.md) | Sampling, counters, harnesses, noise control, accurate comparisons |
| Memory hierarchy | [memory.md](references/memory.md) | Caches, latency vs bandwidth, locality, tiling, AoS/SoA, external memory |
| SIMD | [simd.md](references/simd.md) | Auto-vectorization, intrinsics, reductions, masking, portability |
| Parallelism | [parallelism.md](references/parallelism.md) | Amdahl, work/span, chunking, false sharing, bandwidth saturation |

### Where to look

| Question | Guide |
|---|---|
| How do I set up and solve a recurrence? | [recursion.md](references/recursion.md) |
| How do I prune a search? | [backtracking.md](references/backtracking.md) |
| How do I define states and transitions? | [dynamicprog.md](references/dynamicprog.md) |
| Is this greedy choice correct? | [greedy.md](references/greedy.md) |
| Which graph algorithm or representation? | [graphs.md](references/graphs.md) |
| Is this problem NP-hard, and what are my options? | [np-hardness.md](references/np-hardness.md) |
| Why do asymptotics disagree with reality? | [asymptotic.md](references/asymptotic.md) |
| How do I inspect cycles and generated code? | [codecomp.md](references/codecomp.md) |
| How do I profile and benchmark correctly? | [profiling-benchmarking.md](references/profiling-benchmarking.md) |
| Why is this memory-bound? | [memory.md](references/memory.md) |
| Can this loop vectorize? | [simd.md](references/simd.md) |
| How do I scale across cores? | [parallelism.md](references/parallelism.md) |

## Workflow

### 0. Choose the cost model

State which model you are using and why:

| Model | Counts | Good for |
|---|---|---|
| Asymptotic (RAM) | Primitive operations as a function of input size | Comparing algorithms, growth, scalability |
| Cache-aware / external memory | Cache lines or block transfers, not operations | Arrays, matrices, joins, scans, large data |
| Instruction-level | Cycles, latencies, throughput, dependencies, branch misses | Tight loops, numeric kernels, latency-critical code |
| Parallel | Work, span, cores, bandwidth, synchronization | Threads, SIMD, GPUs, distributed systems |

Default to the asymptotic model first, then add machine-level analysis when constants, locality, or hardware behavior can change the answer. Report the model and assumptions explicitly.

### 1. Understand the context

Read the code before analyzing. Identify, asking only when necessary:

- Language and runtime: the real cost of operations varies (`list.pop(0)` is O(n) in Python, `shift()` can be O(n) in JS, boxed values and GC change constants).
- What `n` represents, plus every other variable (`m`, `k`, `V`, `E`, `W`, `B`, `M`). Define them.
- Expected input size and call frequency: once, per request, or inside a hot loop.
- Constraints: memory, latency, throughput, concurrency, sortedness, duplicate density, value ranges, sparsity.
- Correctness contract: the optimization must preserve semantics, error handling, and side effects.
- Whether the workload is CPU-bound, memory-bound, branch-bound, lock-bound, or I/O-bound.

### 2. Measure or gather evidence

Measure before optimizing when a runtime is available:

- Establish a baseline with representative inputs and realistic sizes.
- Use a profiler before guessing: `perf`, `Cachegrind`, `llvm-mca`, `pprof`, `async-profiler`, Chrome DevTools, `pyperf`, JMH, `criterion`, `hyperfine`.
- Distinguish end-to-end latency from hot-loop time, and latency from throughput.
- If measurement is impossible, say so and present the analysis as theoretical with explicit assumptions.

### 3. Compute the asymptotic complexity

Analyze inside out, block by block:

| Code pattern | Typical cost |
|---|---|
| Simple statement, index access, hash lookup | O(1) average |
| Loop from 1 to n | O(n) |
| Nested loops depending on n | O(n²), O(n³)... |
| Loop that halves the space | O(log n) |
| Divide in halves + linear work | O(n log n) |
| Recursion with two calls that do not reduce well | O(2ⁿ) |
| Permutations / subsets | O(n!) / O(2ⁿ) |
| Comparison sort | O(n log n) |
| String concatenation in a loop (immutable strings) | O(n²) |
| `in` / `contains` on a list or array | O(n) |
| `in` / `contains` on a set or hash map | O(1) average |

Rules:

- **Sequential adds, nested multiplies.** Keep the dominant term and drop constants in the asymptotic model only.
- **Include hidden library costs** (`sort`, `sum`, `slice`, `copy`, `join`, `index`, `remove`, `insert`, spread/concat, regex compilation).
- **Amortized analysis** where applicable: `append` on a dynamic array is O(1) amortized; hash table rehashing is amortized; union-find is ~O(α(n)).
- **Randomized algorithms:** report expected cost and worst case separately (for example hashing and quicksort).
- **Space:** count auxiliary structures, copies, memoization, recursion stack, and output. Distinguish auxiliary from total space.
- **Cases:** report best, average, and worst when they differ.
- **Multiple variables:** keep them separate: O(n·m), O(V + E), O(n log k), O(n·W).

#### Recurrences

For recursive code, write T(n) and solve it. Typical forms:

| Recurrence | Solution | Example |
|---|---|---|
| T(n) = T(n/2) + O(1) | O(log n) | Binary search |
| T(n) = T(n-1) + O(1) | O(n) | Linear recursion |
| T(n) = 2T(n/2) + O(1) | O(n) | Tree traversal |
| T(n) = 2T(n/2) + O(n) | O(n log n) | Merge sort |
| T(n) = T(n-1) + O(n) | O(n²) | Naive selection sort recurrence |
| T(n) = 2T(n-1) + O(1) | O(2ⁿ) | Naive subset enumeration |
| T(n) = aT(n/b) + f(n) | Master theorem | Divide and conquer |

Use the recursion tree or substitution method when the Master theorem does not apply. Count the call stack for space, and check whether tail-call elimination is guaranteed (it often is not).

#### Dynamic programming

Analyze a DP as a state graph:

- **Time = number of states × cost per transition.**
- **Space = number of states** (or the frontier actually kept).
- Name the state precisely: `dp[i][j]` means what and why is it sufficient?
- Check for overlapping subproblems; without them, recursion plus memoization adds overhead for nothing.
- Common bounds: knapsack O(nW) time and space, O(W) with a rolling array; LCS and edit distance O(nm); TSP O(n²·2ⁿ); interval DP O(n³); tree DP O(n).

#### Search and backtracking

- Worst case is the size of the search tree: O(bᵈ) for branching factor b and depth d.
- Pruning changes the practical cost, not necessarily the worst-case bound. Report the effective branching factor when measured.
- Bound the search with branch and bound, constraint propagation, ordering heuristics, symmetry breaking, and bitsets.

#### NP-hard problems

- Recognize when the problem generalizes a known NP-hard problem (TSP, 3-SAT, vertex cover, set cover, bin packing, scheduling with deadlines).
- Do not claim a polynomial algorithm without a correct argument or proof.
- Look for structure that makes it tractable: bounded parameter, treewidth, intervals, planarity, small n, small value range, DAG, matroid.
- Otherwise choose deliberately: exact exponential or DP for small n, approximation with a stated ratio, fixed-parameter algorithm, SAT/ILP/MIP solver, or heuristic with no guarantee.
- State the guarantee you are giving. "Fast in practice" is not a complexity bound.

### 4. Add the machine-aware cost model

Asymptotics hide constants, and on modern hardware constants can dominate for realistic n.

#### Latency is not uniform

Approximate orders of magnitude, not exact values:

| Operation | Order of magnitude |
|---|---|
| L1 cache hit | a few cycles |
| L2 cache hit | ~10 cycles |
| L3 cache hit | tens of cycles |
| DRAM access | ~100 ns, hundreds of cycles |
| SSD read | ~100 µs |
| Network round trip in a datacenter | ~0.5 ms |
| Disk seek | ~ms |

Check the code for:

- **Memory hierarchy and locality:** sequential access, prefetching, cache lines (typically 64 bytes), working-set size, temporal and spatial locality, cache misses, TLB misses, pointer chasing.
- **Data layout:** contiguous arrays versus linked structures; array-of-structs versus struct-of-arrays; hot/cold splitting; structure size, alignment, padding, false sharing.
- **Branches:** unpredictable branches cost mispredictions; sorting data, branchless arithmetic, predication, or lookup tables can help; measure with branch-miss counters.
- **Instruction-level parallelism:** dependency chains, loop-carried dependencies, unrolling, multiple accumulators, division and modulo in hot loops, strength reduction, function call overhead, inlining.
- **Autovectorization:** contiguous data, no aliasing, no loop-carried dependencies, simple control flow. Where it matters, consider SIMD intrinsics, masking, and vectorized reductions.
- **Allocation and runtime overhead:** avoid allocation, boxing, deep copies, and hash overhead in hot loops; reuse buffers; prefer slices and views; watch GC pressure.
- **I/O and round trips:** batch requests, paginate by cursor, stream, compress, cache with invalidation, avoid N+1 queries. Count round trips as part of the complexity.
- **Parallelism:** apply only after reducing work. Consider Amdahl's law, load balance, synchronization, contention, false sharing, memory bandwidth saturation, and work/span. Parallelism reduces wall-clock time, not asymptotic instruction count.

#### Constants can beat asymptotics

For realistic input sizes, a simpler algorithm with better constants can win:

- Insertion sort beats merge sort for small arrays; use a cutoff in hybrid sorts.
- Naive matrix multiplication can beat Strassen below large n.
- A linear scan beats a balanced tree for tiny collections.
- A hash map with poor locality can lose to a sorted array with binary search for medium n.

Report the crossover range rather than only the asymptotic winner. When in doubt, benchmark both.

### 5. Identify bottlenecks

Prioritize algorithmic causes first, then machine-level causes. Confirm with measurement when possible.

Algorithmic anti-patterns:

1. Nested loops over the same collection to search, compare, or match.
2. Repeated linear search (`in list`, `.find`, `.indexOf`, `.includes`) inside a loop.
3. Sorting repeatedly inside a loop, or sorting only to get min/max/top-k.
4. Recomputing the same values: repeated subproblems, loop-invariant expressions, regexes compiled per iteration.
5. Unnecessary copies: slicing, spread, `concat` in a loop, `deepcopy`.
6. Unsuitable data structures: list as queue, list as set, array as dictionary.
7. Recursion without memoization on overlapping subproblems.
8. N+1 queries or calls instead of one batch.
9. String concatenation in a loop instead of a buffer or `join`.
10. Loading an entire dataset when streaming or generators would suffice.

Machine-level anti-patterns:

1. Pointer chasing and random access over large working sets.
2. Column-major traversal of row-major data or the reverse (cache-hostile order).
3. Oversized structs and hot/cold fields interleaved.
4. Unpredictable branches in a hot loop.
5. Long dependency chains that prevent ILP.
6. Division or modulo inside the innermost loop.
7. Allocation and boxing per element.
8. Excessive synchronization or false sharing across threads.
9. Chatty I/O, per-element network calls, missing pagination.
10. Parallelizing code that is already memory-bandwidth-bound or lock-bound.

### 6. Choose the fix

Apply strategies in this order; stop at the first one that fully solves the problem.

1. **Remove work.** Memoize, precompute, cache, incrementally update, prune the search, exit early, avoid redundant queries. Deletion beats optimization.
2. **Change the data structure.** Use the structure whose operations match the access pattern.
3. **Change the algorithm or paradigm.** Pick the technique that fits the problem shape.
4. **Exploit structure.** Sortedness, monotonicity, sparsity, bounded range, small alphabet, symmetry, algebraic identities, DAG structure, divide and conquer.
5. **Restructure the computation.** Batch, stream, tile/block for locality, interchange loops, reduce state, use prefix sums or difference arrays.
6. **Trade resources deliberately.** Memory for time or time for memory, precomputation versus query time, exactness for approximation only if the contract allows it.
7. **Approximate when acceptable.** Sampling, sketching, Bloom filters, approximation algorithms with stated ratios, heuristics. State the loss of guarantee.
8. **Use parallelism or vectorization.** Threads, SIMD, GPUs, distributed execution. State the expected speedup and its limits.
9. **Improve constants.** Cache locality, layout, allocation reuse, branchless code, unrolling, avoiding expensive instructions, compiler flags, profiling-guided optimization.

What not to do:

- Do not micro-optimize before profiling a cold path.
- Do not parallelize an O(n²) algorithm when an O(n log n) algorithm exists.
- Do not add a complex data structure for tiny n.
- Do not trade correctness, error handling, security, or accessibility for speed.
- Do not claim a win without measurement or a stated theoretical argument.

### 7. Validate

Correctness:

- Check edge cases: empty input, single element, duplicates, negatives, already sorted, very large input, malformed input.
- Differential-test against a brute-force implementation on random small inputs when possible.
- Verify invariants and state the assumptions that must hold.

Performance:

- Benchmark the baseline and the new version on the same machine, compiler, flags, and input distribution.
- Vary n across orders of magnitude (for example 10³, 10⁴, 10⁵, 10⁶) and check that the measured growth fits the predicted one.
- Warm up, repeat, and report median and spread; do not trust a single run.
- Consume results so the compiler cannot eliminate the work; prevent constant folding.
- Measure the right thing: latency versus throughput, hot loop versus end-to-end, CPU time versus wall-clock time.
- Use hardware counters when available: cycles, instructions, IPC, cache misses, branch misses.
- If a claim is theoretical only, label it as such.

## Optimization catalog

### By data structure

| Situation | Change to | Effect |
|---|---|---|
| Frequent search or membership | Hash set / hash map | O(n) → O(1) average |
| Frequency counting | Hash map / `Counter` | O(n²) → O(n) |
| Repeated min/max, top-k, scheduling | Heap / priority queue | O(n) per query → O(log n) |
| FIFO queue | Deque / linked list / circular buffer | `pop(0)` O(n) → O(1) |
| Search in sorted data, ranges, successor/predecessor | Balanced tree / `bisect` / BST | O(n) → O(log n) |
| Range queries with updates | Fenwick tree (BIT) / segment tree | O(n) → O(log n) |
| Range sums without updates | Prefix sums | O(n) per query → O(1) |
| String prefixes, autocomplete | Trie | O(n·L) → O(L) |
| Dynamic connectivity, components | Union-Find (DSU) | ~O(α(n)) amortized |
| Pattern search in text | KMP / Rabin-Karp / Z-algorithm | O(n·m) → O(n + m) |
| Approximate membership, minimal memory | Bloom filter | O(1) space per element, with false positives |
| Min/max in a sliding window | Monotonic deque | O(n·k) → O(n) |
| Large workload, poor locality | Contiguous arrays / blocked layout | Fewer cache misses, better prefetching |

### By algorithmic pattern

| Pattern | When to apply it | Typical improvement |
|---|---|---|
| **Two pointers** | Sorted array, pairs with a target sum, partitions | O(n²) → O(n) |
| **Sliding window** | Contiguous subarrays/substrings with a condition | O(n·k) or O(n²) → O(n) |
| **Prefix sums / difference arrays** | Range sum queries, range updates | O(n) per query → O(1) |
| **Hashing (complement)** | Two-sum, duplicates, anagrams | O(n²) → O(n) |
| **Binary search** | Monotonic domain, "minimum value that satisfies X" | O(n) → O(log n) |
| **Divide and conquer** | Independent subproblems | O(n²) → O(n log n) |
| **Dynamic programming / memoization** | Overlapping subproblems + optimal substructure | O(2ⁿ) → O(n²) or O(n·W) |
| **Greedy** | Local choice is provably globally optimal | Usually O(n log n) |
| **Backtracking with pruning / branch and bound** | Combinatorial search | Reduces the explored space in practice |
| **Monotonic stack/deque** | Next greater element, histograms, windows | O(n²) → O(n) |
| **Sort first** | Sorting enables linear or logarithmic steps | O(n²) → O(n log n) |
| **Quickselect / heap of size k** | Only the k-th element or top-k is needed | O(n log n) → O(n) / O(n log k) |
| **Bit manipulation / bitmask DP** | Small subsets, compact states | Fewer operations and less memory |
| **Precomputation / lookup tables** | Repeated queries over static data | One-time O(n), queries O(1) |
| **Batching** | N+1 calls, repeated round trips | N calls → 1 call |
| **Lazy evaluation / generators / streaming** | Large data, not all needed at once | Space O(n) → O(1) |
| **Parallelism / vectorization** | Independent work, CPU-bound | Lower wall-clock time; state the limit |

### By paradigm

- **Recursion:** convert to iteration or DP when subproblems overlap or the stack can overflow; use memoization; check tail-call support; solve the recurrence instead of guessing.
- **Backtracking:** order candidates by constraint, propagate constraints early, bound with an optimistic estimate, break symmetries, use bitsets for membership and visited sets.
- **Dynamic programming:** define the state minimally, choose a topological order, use rolling arrays to cut space, reconstruct the solution only if needed, and prefer bottom-up when recursion overhead matters.
- **Greedy:** prove the exchange argument or a matroid property. If the proof fails, use DP.
- **Graphs:** use adjacency lists for sparse graphs and matrices for small dense graphs; BFS for unweighted shortest paths; Dijkstra for non-negative weights; Bellman-Ford for negative weights; topological order for DAGs; flows for matching and assignment; SCC for condensation; union-find for connectivity; A* with an admissible heuristic when applicable.

### By domain

- **Graphs:** BFS/DFS O(V+E); Dijkstra with a binary heap O((V+E) log V); Bellman-Ford O(V·E); Floyd-Warshall O(V³) only for small or dense graphs; topological sort O(V+E); Kruskal O(E log E); Prim O(E log V); Dinic O(V²E) or better; avoid adjacency matrices on sparse graphs.
- **Strings:** avoid concatenation in a loop; use builders or `join`; use rolling hash for substring comparison; suffix arrays or automata for advanced queries.
- **Matrices and numerics:** vectorize (NumPy, BLAS); block for cache locality; use fast exponentiation O(log n); Strassen only for very large sizes; consider sparse representations; watch floating-point rounding and operation order.
- **Sorting:** counting or radix sort O(n + k) for bounded integer keys; insertion sort for nearly sorted or small inputs; use the standard library sort unless there is a strong reason not to.
- **Databases and backends:** indexes, avoid N+1, cursor pagination, caching with an expiration policy, push joins and aggregation into the database, batch writes, measure round trips.
- **Memory-bound kernels:** sequential access, tiling/blocking, struct-of-arrays, prefetching, alignment, avoid false sharing, reduce working-set size.
- **Cell-level kernels:** branchless code, multiple accumulators, unrolling, strength reduction, SIMD, inspect generated assembly (`llvm-mca`).

## Quick reference

### Growth orders

| Complexity | n = 10³ | n = 10⁶ | Practical up to approximately |
|---|---|---|---|
| O(1), O(log n) | trivial | trivial | any size |
| O(n) | ~10³ ops | ~10⁶ ops | 10⁸–10⁹ |
| O(n log n) | ~10⁴ | ~2·10⁷ | 10⁶–10⁷ |
| O(n²) | 10⁶ | 10¹² (infeasible) | ~10⁴ |
| O(n³) | 10⁹ | infeasible | ~500 |
| O(2ⁿ) | infeasible | — | ~25 |
| O(n!) | — | — | ~10–11 |

Rule of thumb: ~10⁸ simple operations per second in compiled languages; 10⁶–10⁷ in interpreted languages. These are orientation values, not guarantees; memory and I/O can lower them significantly.

### Profiling and benchmarking tools

| Need | Tools |
|---|---|
| CPU sampling | `perf record` / `perf report`, `pprof`, `async-profiler` |
| Counters and cache behavior | `perf stat`, Cachegrind |
| Generated code quality | Compiler assembly output, `llvm-mca`, Compiler Explorer |
| Wall-clock comparison | `hyperfine`, `time`, custom harness |
| Language benchmarks | `pyperf`, `timeit`, JMH, `criterion`, Go testing benchmarks |
| Browser/frontend | Chrome DevTools Performance and Memory panels |
| Database | `EXPLAIN (ANALYZE, BUFFERS)` and query statistics |

## Response format

Always structure the output like this, adjusting length to the request:

```
## Summary
One or two sentences: current complexity, main bottleneck (algorithmic and/or machine-level), and proposed improvement.

## Cost model and assumptions
Model used (asymptotic, cache-aware, instruction-level, parallel), n definition, input sizes, hardware/runtime assumptions, and whether measurements were possible.

## Complexity analysis
- Variables (n = ..., m = ...)
- Time: best / average / worst; expected for randomized algorithms
- Space: auxiliary / total
- Recurrence or DP state/transition argument, if applicable
- Breakdown by block showing where the dominant term comes from

## Bottlenecks
1. [location] — algorithmic cause — cost
2. [location] — machine-level cause — cost

## Recommendations
For each: problem → solution → complexity before/after → expected constant-factor effect → trade-off.
Order by impact: largest asymptotic reduction first, then largest constant-factor reduction.

## Optimized code
Complete, runnable code with brief comments where the logic changes.

## Comparison
| Version | Time | Space | Notes |
|---|---|---|---|

## Validation
Correctness edge cases, benchmark method and environment, measured or theoretical evidence, and what remains unmeasured.
```

For a short function, Summary + Complexity + Code may be enough. For a large module, use the full format and prioritize the 3–5 highest-impact findings.

## Principles

- **Complexity is a model, not the truth.** It abstracts hardware. State what the model ignores and when that matters.
- **Measure before optimizing.** Profile first. If the code is not hot or n is small, the simple readable version may already be right.
- **Optimize the dominant term first, then constants.** Going from O(n²) to O(n log n) usually matters more than shaving cycles, but locality and branch behavior can reverse the ranking for realistic n.
- **Make trade-offs explicit.** Time versus memory, precomputation versus query, readability versus speed, average versus worst case, exact versus approximate, sequential versus parallel.
- **Do not invent complexities.** If a cost depends on a library or runtime, state the assumption. Separate proven bounds from measurements and from guesses.
- **Preserve correctness.** An optimization that changes results, error handling, or side effects is not an optimization.
- **Respect the contract.** Never sacrifice security, data integrity, accessibility, or required behavior for performance.
- **Prefer deletion and simplification.** Fewer operations, allocations, and branches usually beat clever tricks.
- **Document the shortcut.** If an optimization has a known limit, leave a one-line comment: `shortcut: <limit>, <when to revisit>`.

## Short examples

### Algorithmic: duplicate detection

**Input (Python):**

```python
def has_duplicates(nums):
    for i in range(len(nums)):
        for j in range(i + 1, len(nums)):
            if nums[i] == nums[j]:
                return True
    return False
```

**Analysis:** two nested loops over `n` → O(n²) worst case (no duplicates), O(1) best; O(1) space.

**Recommendation:** use a `set` for O(1) average membership. Time O(n); space O(n).

```python
def has_duplicates(nums):
    seen = set()
    for x in nums:
        if x in seen:
            return True
        seen.add(x)
    return False
```

**Trade-off:** O(n) memory for a quadratic-to-linear reduction. If memory is critical, sort first (O(n log n) time, O(1) extra space in-place) and compare adjacent elements.

### Machine-aware: two traversals of the same matrix

Both snippets are O(n²) asymptotically, but not equally fast:

```python
for i in range(n):
    for j in range(m):
        total += grid[i][j]

for j in range(m):
    for i in range(n):
        total += grid[i][j]
```

If `grid` is row-major, the first version streams contiguous memory and benefits from prefetching; the second jumps by a full row per step and can miss cache on every access when a row exceeds the cache. Same complexity, different constants.

**Recommendation:** traverse in storage order, or block the loops so each block fits in cache. Report this as a constant-factor improvement, not an asymptotic one, and measure it: the crossover depends on `n`, `m`, element size, and the cache hierarchy.

**Trade-off:** loop reordering and blocking can make code less obvious; keep the simple order unless the matrix is large enough or the profile shows misses.
