# Greedy Algorithms

## Description

How to decide whether a locally optimal choice is globally optimal, and how to prove it. A greedy algorithm commits to the best-looking option at each step and never reconsiders. Correctness is not free: it needs an **exchange argument**, a **matroid** property, or a **stays-ahead** argument.

If you cannot prove the greedy choice correct, the default fallback is dynamic programming (`dynamicprog.md`) or backtracking (`backtracking.md`).

## Use cases

- Scheduling with deadlines or fixed profits (earliest deadline, earliest finish).
- Interval problems: maximum non-overlapping intervals, minimum rooms, merge intervals.
- Sorting for a cost function: tape storage, shortest processing time.
- Huffman coding and optimal merge patterns.
- Minimum spanning trees (Kruskal, Prim) and some shortest-path cases.
- Fractional knapsack (greedy is correct) as opposed to 0/1 knapsack (greedy is wrong).
- Cache eviction, compression, and other online decisions with a stated policy.

## How to prove a greedy choice

| Argument | Shape |
|---|---|
| **Exchange** | Take an optimal solution that differs from the greedy one; swap the first differing element. Show the result is still optimal and closer to greedy. Induct. |
| **Greedy stays ahead** | Show that after each step the greedy partial solution is at least as good as any other partial solution of the same size. |
| **Matroid / structure** | Show the feasible sets form a matroid (hereditary + exchange); then greedy by weight is optimal. |
| **Lower bound** | Show no solution can beat what greedy achieves, and greedy achieves it. |

A counterexample is enough to disprove a greedy rule. Always try small instances with ties and unusual values before trusting intuition.

## Code examples

### Greedy fails: coin change

```python
def greedy_coins(coins, amount):
    coins = sorted(coins, reverse=True)
    used = []
    for c in coins:
        while amount >= c:
            used.append(c)
            amount -= c
    return used if amount == 0 else None
```

For `coins = [1, 3, 4]` and `amount = 6`, greedy returns `[4, 1, 1]` (3 coins) but the optimum is `[3, 3]` (2 coins). The greedy rule is not correct for arbitrary coin systems. (It is correct for canonical systems such as euro coins, but that must be proven, not assumed.)

**Fix:** DP over amounts.

```python
def min_coins(coins, amount):
    INF = float("inf")
    dp = [0] + [INF] * amount
    for a in range(1, amount + 1):
        dp[a] = min((dp[a - c] + 1 for c in coins if c <= a), default=INF)
    return dp[amount] if dp[amount] < INF else -1
```

`O(amount · len(coins))` time, `O(amount)` space.

### Interval scheduling: earliest finish is optimal

```python
def max_non_overlapping(intervals):
    intervals.sort(key=lambda x: x[1])       # by finish time
    count, last_end = 0, float("-inf")
    for start, end in intervals:
        if start >= last_end:
            count += 1
            last_end = end
    return count
```

`O(n log n)` time, `O(1)` extra space. Exchange argument: if an optimal solution picks an interval that finishes later than the greedy choice, replacing it with the greedy interval frees at least as much room for the rest.

Sorting by start time or by length does **not** give the optimum; both have small counterexamples.

### Tape storage: shortest first

```python
def tape_order(lengths):
    return sorted(lengths)
```

With equal access probability, the expected read cost is minimized by increasing length. Exchange argument: swapping an out-of-order pair `L[i] > L[i+1]` changes the expected cost by `(L[i+1] - L[i]) / n < 0`, so the sorted order is better. `O(n log n)`.

### Huffman coding: repeatedly merge the two smallest

```python
import heapq

def huffman_lengths(freqs):
    heap = freqs[:]
    heapq.heapify(heap)
    total = 0
    while len(heap) > 1:
        a = heapq.heappop(heap)
        b = heapq.heappop(heap)
        total += a + b
        heapq.heappush(heap, a + b)
    return total
```

`O(n log n)` time, `O(n)` space. Correctness follows from an exchange argument on the two least frequent symbols; the result is the minimum total encoded length.

### Fractional knapsack: greedy by value density

```python
def fractional_knapsack(items, capacity):
    items = sorted(items, key=lambda x: x[1] / x[0], reverse=True)
    total = 0.0
    for weight, value in items:
        if capacity == 0:
            break
        take = min(weight, capacity)
        total += value * take / weight
        capacity -= take
    return total
```

`O(n log n)`. Correct because items are divisible: taking the best density first is always safe.

The same rule is **wrong** for 0/1 knapsack; use the DP in `dynamicprog.md`.

### Minimum spanning tree: Kruskal with union-find (greedy + matroid)

```python
def kruskal(n, edges):
    parent = list(range(n))
    rank = [0] * n

    def find(x):
        while parent[x] != x:
            parent[x] = parent[parent[x]]
            x = parent[x]
        return x

    def union(a, b):
        a, b = find(a), find(b)
        if a == b:
            return False
        if rank[a] < rank[b]:
            a, b = b, a
        parent[b] = a
        if rank[a] == rank[b]:
            rank[a] += 1
        return True

    total = 0
    for w, u, v in sorted(edges):
        if union(u, v):
            total += w
    return total
```

`O(E log E)` for the sort plus near-`O(E α(V))` for the union-find. This is the matroid case: the acyclic edge sets form a graphic matroid, so greedy by weight is optimal.

## Anti-patterns

1. Assuming a greedy rule works because it matches intuition; no proof and no counterexample search.
2. Sorting by the wrong key (start instead of finish, value instead of density).
3. Using a greedy for 0/1 knapsack, set cover, or TSP where it has no guarantee.
4. Re-sorting inside a loop when one sort is enough.
5. Forgetting ties, zero, negative, or duplicate values in the exchange argument.
6. Confusing "optimal for the relaxed (fractional) problem" with "optimal for the integer problem".

## Checklist

- Can you state the greedy choice precisely?
- Can you prove it with an exchange or stays-ahead argument?
- Did you try to break it with small adversarial instances?
- Is the sort key the one the proof requires?
- Is the data structure for the greedy step appropriate (heap, union-find, sorted list)?
- If the proof fails, did you switch to DP or an approximation with a stated ratio?

## Related

- `dynamicprog.md` for the fallback when greedy is not provably correct.
- `graphs.md` for MST and shortest-path special cases.
- `np-hardness.md` for approximation guarantees when no exact polynomial algorithm is known.

