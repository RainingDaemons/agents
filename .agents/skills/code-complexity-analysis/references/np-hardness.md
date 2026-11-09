# NP-Hardness

## Description

How to recognize that a problem is probably not solvable in polynomial time, and what to do about it. Covers the decision/optimization distinction, reductions, the P/NP/NP-complete/NP-hard vocabulary, and the practical alternatives: exact exponential algorithms, fixed-parameter algorithms, approximation, solvers, and heuristics.

Do not claim a polynomial algorithm for a problem unless you have a correct algorithm **and** a proof. "It runs fast on my inputs" is not a complexity bound.

## Use cases

- A new feature looks like a known hard problem: routing, scheduling, packing, covering, partitioning.
- Choosing between exact, approximated, or heuristic solutions under a deadline.
- Reviewing a claim of an `O(n log n)` solution to a combinatorial optimization problem.
- Deciding whether to use a MIP/SAT/CP solver instead of hand-rolled code.
- Estimating whether an exact exponential algorithm is feasible for the actual `n`.

## The vocabulary

| Term | Meaning |
|---|---|
| **P** | Solvable in polynomial time by a deterministic algorithm. |
| **NP** | A proposed solution can be **verified** in polynomial time. |
| **NP-hard** | At least as hard as every problem in NP (under polynomial reductions). |
| **NP-complete** | In NP and NP-hard. |
| **Reduction** `X ≤p Y` | A polynomial-time algorithm that turns an instance of `X` into an instance of `Y` with the same answer. If `Y` is easy, `X` is easy; if `X` is hard, `Y` is hard. |

The standard way to prove a problem `Y` is NP-hard: pick a known NP-hard `X` and show `X ≤p Y`.

Classic NP-hard problems to recognize: SAT / 3-SAT, independent set, vertex cover, clique, set cover, Hamiltonian path and cycle, TSP, subset sum, partition, bin packing, graph coloring, job scheduling with deadlines, longest path, integer programming, maximum cut.

## Decision vs optimization

Complexity theory talks about **decision** problems ("is there a tour of length ≤ k?"). The optimization version ("find the shortest tour") is at least as hard: a polynomial optimizer would decide the decision problem by comparing its answer to `k`. Approximation algorithms usually target the optimization version and state a ratio.

## What to do when the problem is NP-hard

Choose deliberately and **state the guarantee**:

| Option | Use when | Guarantee |
|---|---|---|
| **Exact, exponential** | `n` is small (roughly `n ≤ 20–25` for `2ⁿ`, `≤ 12` for `n!`) | Optimal, worst-case exponential |
| **Dynamic programming** | The hardness comes from a large value range, not from `n` | Pseudo-polynomial; exact |
| **Fixed-parameter (FPT)** | A small parameter `k` captures the difficulty | `f(k) · poly(n)`; exact for bounded `k` |
| **Approximation** | A near-optimal answer is acceptable | Ratio `ρ(n)`, proven |
| **MIP / SAT / CP solver** | Structured constraints, medium size | Exact if it finishes; no worst-case bound |
| **Heuristics** | Large instances, no guarantee needed | None; measure quality empirically |
| **Reformulate** | Extra structure is available (treewidth, planarity, intervals, DAG, matroid) | Depends on the structure |

A **pseudo-polynomial** algorithm such as `O(nW)` for subset sum is polynomial in `W`, not in the number of bits of `W`. Problems with such algorithms are **weakly NP-hard**; bin packing and TSP are **strongly NP-hard**, and no pseudo-polynomial algorithm exists unless P = NP.

## Code examples

### Exact exponential: TSP with Held–Karp

```python
def tsp(dist):
    n = len(dist)
    INF = float("inf")
    dp = [[INF] * n for _ in range(1 << n)]
    dp[1][0] = 0
    for mask in range(1 << n):
        for u in range(n):
            if dp[mask][u] == INF:
                continue
            for v in range(n):
                if mask & (1 << v):
                    continue
                nmask = mask | (1 << v)
                dp[nmask][v] = min(dp[nmask][v], dp[mask][u] + dist[u][v])
    full = (1 << n) - 1
    return min(dp[full][u] + dist[u][0] for u in range(1, n))
```

`O(2ⁿ n²)` time, `O(2ⁿ n)` space. Better than `O(n!)` brute force, but still infeasible beyond about `n = 20–22`.

### Brute force: only for tiny `n`

```python
from itertools import permutations

def tsp_bruteforce(dist):
    n = len(dist)
    if n == 1:
        return 0
    best = float("inf")
    for perm in permutations(range(1, n)):
        cost = dist[0][perm[0]]
        for a, b in zip(perm, perm[1:]):
            cost += dist[a][b]
        cost += dist[perm[-1]][0]
        best = min(best, cost)
    return best
```

`O(n!)`. For `n = 10` this is about 3.6 million tours; for `n = 15`, about 87 billion. Use it only as a reference implementation for differential testing.

### Fixed-parameter: vertex cover with parameter `k`

```python
def has_vertex_cover(edges, k):
    if not edges:
        return True
    if k == 0:
        return False
    u, v = edges[0]
    # any cover must contain u or v
    without_u = [e for e in edges if u not in e]
    without_v = [e for e in edges if v not in e]
    return has_vertex_cover(without_u, k - 1) or has_vertex_cover(without_v, k - 1)
```

`O(2ᵏ · E)` time, `O(k)` stack. Exact for small `k`; exponential in `k`, polynomial in `n` for fixed `k`. This is the branching rule behind many FPT algorithms.

### Approximation: 2-approximate vertex cover

```python
def approx_vertex_cover(edges):
    cover, used = set(), set()
    for u, v in edges:
        if u in used or v in used:
            continue
        cover.add(u)
        cover.add(v)
        used.add(u)
        used.add(v)
    return cover
```

`O(E)` time. Any vertex cover must contain at least one endpoint of each disjoint edge we pick, and we add both, so the result is at most twice the optimum. Better constant ratios are possible, but improving on `2 - ε` for the general problem is a long-standing hard problem; do not claim a better bound without a proof.

### Weakly NP-hard: subset sum in pseudo-polynomial time

```python
def subset_sum(nums, target):
    reachable = 1                       # bitset: bit i set means sum i is reachable
    mask = (1 << (target + 1)) - 1
    for x in nums:
        reachable = (reachable | (reachable << x)) & mask
    return (reachable >> target) & 1 == 1
```

`O(n · target / word_size)` time, `O(target)` space. Assumes non-negative integers; negatives break the bitset. Polynomial in the **value** of `target`, exponential in its bit length. If `target` is huge, this is not a practical algorithm.

### Reporting a heuristic honestly

```python
def nearest_neighbor_tsp(dist, start=0):
    n = len(dist)
    unvisited = set(range(n)) - {start}
    tour, current, total = [start], start, 0
    while unvisited:
        nxt = min(unvisited, key=lambda v: dist[current][v])
        total += dist[current][nxt]
        tour.append(nxt)
        unvisited.remove(nxt)
        current = nxt
    return total + dist[current][start]
```

`O(n²)`. The tour can be up to a logarithmic factor worse than optimal; there is no constant-ratio guarantee. Label it a heuristic, and measure it against a solver or the exact algorithm on small instances.

## Anti-patterns

1. Claiming a polynomial algorithm for an NP-hard problem without a proof.
2. Using an exact exponential algorithm on `n = 1000` because the tests used `n = 10`.
3. Calling an `O(nW)` pseudo-polynomial algorithm "polynomial" without qualifying `W`.
4. Reporting "fast in practice" as a complexity bound.
5. Approximating without stating the ratio, or with an unproven ratio.
6. Reimplementing a MIP/SAT/CP solver when a mature one exists.
7. Forgetting that the decision version is a lower bound on the optimization version.

## Checklist

- Which known NP-hard problem does this generalize?
- Is the hardness weak or strong? Is a pseudo-polynomial bound available?
- What is the actual `n` and the actual value range?
- Which option did you choose, and what guarantee does it give?
- Did you measure quality against exact solutions on small instances?
- Is the approximation ratio proven or only hoped for?
- Did you document the complexity with the parameters that matter (`n`, `k`, `W`)?

## Related

- `dynamicprog.md` for exact `O(2ⁿ)` and pseudo-polynomial DPs.
- `backtracking.md` for exact search with pruning and branch and bound.
- `graphs.md` for the polynomial graph algorithms that are often confused with the hard ones.
- `asymptotic.md` for what "polynomial" and "exponential" mean precisely.

