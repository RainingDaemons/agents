# Dynamic Programming

## Description

How to recognize and analyze dynamic programming (DP): problems with **overlapping subproblems** and **optimal substructure**. Instead of recomputing subproblems, store each answer once and reuse it.

Two equivalent formulations:

| Form | Shape | Notes |
|---|---|---|
| Top-down (memoization) | Recursion + cache | Only visits reachable states; keeps recursion overhead and stack depth |
| Bottom-up (tabulation) | Loop over states in topological order | No recursion overhead; can often roll the array to save space |

Cost model:

- **Time = number of states × cost per transition.**
- **Space = number of states**, or the frontier actually kept.

If there are no overlapping subproblems, DP adds overhead for nothing; use plain recursion or divide and conquer.

## Use cases

- Counting paths, ways, or sequences (staircases, coin change, paths in a grid).
- Optimization with a choice at each step (knapsack, LCS, edit distance, interval scheduling).
- Sequence alignment, diffing, approximate string matching.
- Subset/bitmask state over small `n` (TSP with `n ≤ 20`, assignment).
- Interval DP (matrix-chain multiplication, optimal BST, burst balloons).
- Tree DP (independent set on a tree, subtree sums).
- When a greedy rule fails but the problem still has optimal substructure.

## How to define a DP

1. **State.** Name `dp[...]` precisely: "the best value using the first `i` items with capacity `j`". The state must contain everything needed to decide the future and nothing more.
2. **Transition.** Which earlier states does it depend on?
3. **Order.** A topological order of the state graph: every dependency is computed first.
4. **Base cases.**
5. **Answer.** Which state or aggregate of states is the result?
6. **Space.** Which states does the transition actually read? Keep only those.

Common bounds: knapsack `O(nW)`, LCS/edit distance `O(nm)`, TSP `O(n²2ⁿ)`, interval DP `O(n³)`, tree DP `O(n)`.

## Code examples

### Fibonacci: memoization vs rolling array

```python
# top-down: O(n) time, O(n) cache + O(n) stack
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)
```

```python
# bottom-up, rolling: O(n) time, O(1) space
def fib(n):
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a
```

The naive recursive version without the cache is `O(2ⁿ)`: the same subproblems are solved exponentially many times.

### 0/1 knapsack: rolling array direction matters

```python
def knapsack(weights, values, capacity):
    dp = [0] * (capacity + 1)
    for w, v in zip(weights, values):
        for cap in range(capacity, w - 1, -1):   # descending: each item once
            dp[cap] = max(dp[cap], dp[cap - w] + v)
    return dp[capacity]
```

`O(nW)` time, `O(W)` space. Iterating **upward** turns this into the unbounded knapsack (each item reusable). Getting the direction wrong is the classic bug.

Full 2-D version (needed to reconstruct the chosen items):

```python
def knapsack_items(weights, values, capacity):
    n = len(weights)
    dp = [[0] * (capacity + 1) for _ in range(n + 1)]
    for i in range(1, n + 1):
        w, v = weights[i - 1], values[i - 1]
        for cap in range(capacity + 1):
            dp[i][cap] = dp[i - 1][cap]
            if w <= cap:
                dp[i][cap] = max(dp[i][cap], dp[i - 1][cap - w] + v)
    return dp[n][capacity]
```

`O(nW)` time and space.

### Longest common subsequence

```python
def lcs(a, b):
    m, n = len(a), len(b)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if a[i - 1] == b[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])
    return dp[m][n]
```

`O(mn)` time and space. Only the length needs `O(min(m, n))` space; reconstruction needs the table or a divide-and-conquer variant (Hirschberg).

### Edit distance

```python
def edit_distance(a, b):
    m, n = len(a), len(b)
    prev = list(range(n + 1))
    for i in range(1, m + 1):
        cur = [i] + [0] * n
        for j in range(1, n + 1):
            cost = 0 if a[i - 1] == b[j - 1] else 1
            cur[j] = min(prev[j] + 1,        # delete
                         cur[j - 1] + 1,      # insert
                         prev[j - 1] + cost)  # substitute
        prev = cur
    return prev[n]
```

`O(mn)` time, `O(n)` space. The rolling array is correct because row `i` depends only on rows `i` and `i-1`.

### Bitmask DP: TSP (Held–Karp)

```python
def tsp(dist):
    n = len(dist)
    if n == 1:
        return 0
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

`O(2ⁿ · n²)` time and `O(2ⁿ · n)` space. Feasible for roughly `n ≤ 20`; beyond that the problem is NP-hard in practice (see `np-hardness.md`).

### Interval DP: matrix-chain multiplication

```python
def matrix_chain(dims):
    n = len(dims) - 1
    dp = [[0] * n for _ in range(n)]
    for length in range(2, n + 1):
        for i in range(n - length + 1):
            j = i + length - 1
            dp[i][j] = min(
                dp[i][k] + dp[k + 1][j] + dims[i] * dims[k + 1] * dims[j + 1]
                for k in range(i, j)
            )
    return dp[0][n - 1]
```

`O(n³)` time, `O(n²)` space.

## Anti-patterns

1. A state that omits information the future needs, or includes information it does not need.
2. Exponential state space chosen without checking `n`; `O(2ⁿ)` is not better than backtracking when `n` is 30.
3. Iterating in the wrong order, so a transition reads a value that is not final yet.
4. Knapsack with an ascending loop when items are supposed to be used once.
5. Keeping the full table when only the previous row is read.
6. Memoizing a function with mutable or unhashable arguments.
7. Recomputing `dp` for every query instead of precomputing once.

## Checklist

- Are subproblems actually overlapping? If not, are you paying for a cache you do not need?
- Is the state minimal and sufficient?
- Is every dependency computed before it is read?
- Do the base cases cover the empty input and the smallest valid input?
- Did you verify the direction of each 1-D rolling loop?
- Is the space bound the frontier you actually keep, not the whole table?
- Did you differential-test against brute force on small random inputs?

## Related

- `recursion.md` for writing the recurrence first.
- `greedy.md` for when a local choice is provably enough.
- `backtracking.md` for the search version of the same problem.
- `np-hardness.md` for why `O(2ⁿ)` may be the best exact option.

