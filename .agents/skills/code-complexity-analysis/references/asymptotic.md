# Asymptotic Complexity

## Description

What Big-O, Big-Omega, and Big-Theta actually claim, how to compute them, and why they frequently disagree with measured performance. Covers growth orders, amortized and expected analysis, space accounting, and the machine-model caveats.

Asymptotic analysis counts **elementary operations as a function of input size** and drops constant factors and lower-order terms. It is a model, not the truth: it assumes uniform memory access and ignores caches, instruction-level parallelism, branches, and I/O.

## Use cases

- Comparing two algorithms for the same problem before writing either.
- Estimating whether a solution can scale from `n = 10³` to `n = 10⁶`.
- Reviewing code for nested loops, repeated searches, and hidden library costs.
- Communicating a bound in a design review or a code comment.
- Deciding when to stop optimizing: is the dominant term already optimal?

## Definitions

| Notation | Claim | Intuition |
|---|---|---|
| `f(n) = O(g(n))` | `f` grows at most like `g` | Upper bound (for large `n`) |
| `f(n) = Ω(g(n))` | `f` grows at least like `g` | Lower bound |
| `f(n) = Θ(g(n))` | `f` grows exactly like `g` | Tight bound |
| `f(n) = o(g(n))` | `f/g → 0` | Strictly slower growth |
| `f(n) = ω(g(n))` | `f/g → ∞` | Strictly faster growth |

Formally, `f(n) = O(g(n))` means there exist constants `c > 0` and `n₀` such that `f(n) ≤ c·g(n)` for all `n ≥ n₀`. The bound is about **growth**, not about a specific input or machine.

Rules:

- Sequential blocks **add**; nested blocks **multiply**.
- Keep the dominant term; drop constants and lower-order terms.
- Different variables stay separate: `O(V + E)`, `O(n·m)`, `O(n·W)`, `O(n log k)`.
- Report best, average, and worst case when they differ.
- Report **expected** cost for randomized algorithms, separately from worst case.
- Distinguish **auxiliary** space from total space (including input and output).

## Growth orders

| Complexity | `n = 10³` | `n = 10⁶` | Practical up to approximately |
|---|---|---|---|
| `O(1)`, `O(log n)` | trivial | trivial | any size |
| `O(n)` | ~10³ ops | ~10⁶ ops | 10⁸–10⁹ |
| `O(n log n)` | ~10⁴ | ~2·10⁷ | 10⁶–10⁷ |
| `O(n²)` | 10⁶ | 10¹² (infeasible) | ~10⁴ |
| `O(n³)` | 10⁹ | infeasible | ~500 |
| `O(2ⁿ)` | infeasible | — | ~25 |
| `O(n!)` | — | — | ~10–11 |

Orientation values: about `10⁸` simple operations per second in compiled languages, `10⁶–10⁷` in interpreted ones. Memory access and I/O can lower these by orders of magnitude.

## Amortized, expected, and space

- **Amortized:** average over a sequence of operations, worst-case per operation is allowed to be expensive. Dynamic-array `append` is `O(1)` amortized; a single resize is `O(n)`. Union-find is `O(α(n))` amortized.
- **Expected:** average over the algorithm's own randomness. Quicksort is `O(n log n)` expected, `O(n²)` worst. Hashing is `O(1)` expected, `O(n)` worst.
- **Space:** count auxiliary arrays, copies, the recursion stack, memoization tables, and output. An in-place algorithm can still use `O(log n)` stack.

## Code examples

### Nested loops: `O(n²)` → `O(n)`

```python
def has_duplicates(nums):
    for i in range(len(nums)):
        for j in range(i + 1, len(nums)):
            if nums[i] == nums[j]:
                return True
    return False
```

Worst case `O(n²)` (no duplicates), best `O(1)`, `O(1)` space.

```python
def has_duplicates(nums):
    seen = set()
    for x in nums:
        if x in seen:
            return True
        seen.add(x)
    return False
```

Expected `O(n)` time, `O(n)` space. The trade is memory for time.

### String concatenation: hidden quadratic cost

```python
# SLOW: immutable strings, O(n²) total
out = ""
for word in words:
    out += word + " "
```

```python
# FAST: join allocates once, O(n) total
out = " ".join(words) + " "
```

The loop looks linear but each `+=` copies the accumulated string. This is the most common hidden quadratic in Python and JavaScript.

### Binary search vs linear scan: the constant-factor crossover

```python
def linear_search(a, target):
    for i, x in enumerate(a):
        if x == target:
            return i
    return -1
```

`O(n)`, but with a tiny constant and perfect locality. For small arrays (roughly `n < 32–64`), it is often faster than binary search `O(log n)` because binary search pays branch mispredictions and cache misses on each step. Report the crossover, not only the asymptotic winner.

### Amortized `append` vs `pop(0)`

```python
# amortized O(1) per append
stack = []
for x in data:
    stack.append(x)
```

```python
# O(n) per pop(0): the whole list shifts
queue = list(data)
while queue:
    process(queue.pop(0))
```

Use `collections.deque` for a FIFO queue: `O(1)` per operation. The asymptotic difference is what matters; the constant follows.

### Why asymptotics disagree with reality

The claim that an asymptotically faster algorithm is always faster for sufficiently large `n` relied on hardware getting faster and on uniform memory. In practice:

- A cache miss costs roughly 100 ns, while an L1 hit costs a few cycles. Two `O(n)` algorithms can differ by 10× on layout alone.
- Branch mispredictions and dependency chains can make a "fewer operations" algorithm slower.
- Memory bandwidth saturates: a well-vectorized `O(n)` loop becomes bandwidth-bound and no longer scales with instruction count.
- Python and JavaScript add per-operation costs (`list.pop(0)`, boxed numbers, GC) that dominate the asymptotic term for realistic `n`.

Therefore: use asymptotics to eliminate bad algorithms, then measure and reason about constants for the remaining candidates.

## Anti-patterns

1. Quoting Big-O without saying what `n` is, or collapsing independent variables.
2. Ignoring hidden library costs (`sort`, `slice`, `concat`, `in`, regex compilation).
3. Treating a single benchmark as a complexity measurement.
4. Using `O(1)` to mean "fast" without considering the constant (a `O(1)` hash lookup can be 100 ns; a `O(1)` array access is a cycle).
5. Forgetting that string concatenation and list insertion in the middle are linear.
6. Reporting only the worst case when the input distribution is benign (or only the average when adversarial input is possible).
7. Claiming an optimization is asymptotically better when it only changes constants — or the reverse.

## Checklist

- What exactly is `n`? What are `m`, `k`, `V`, `E`, `W`?
- What does one "operation" cost in this language/runtime?
- Did you include the cost of built-in functions and allocations?
- Best, average, worst, expected: which are you reporting?
- Is the space bound auxiliary or total?
- Is the dominant term correct, and what is the next term?
- Did you measure to confirm the predicted growth across orders of magnitude?

## Related

- `codecomp.md` for the generated code and instruction-level view.
- `memory.md` for the cache-aware model that explains the disagreements.
- `profiling-benchmarking.md` for measuring instead of guessing.
- `parallelism.md` for work/span and Amdahl's law.
- `recursion.md` for recurrences and the recursion tree.
