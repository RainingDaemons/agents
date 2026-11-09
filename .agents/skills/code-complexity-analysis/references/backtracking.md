# Backtracking

## Description

How to explore a combinatorial search space without enumerating it blindly. A backtracking algorithm builds a solution one piece at a time, recursively tries every alternative for the next piece, and abandons a partial solution as soon as it cannot lead to a valid or optimal one.

The worst case is the size of the search tree, `O(bᵈ)` for branching factor `b` and depth `d`; permutations are `O(n!)`, subsets `O(2ⁿ)`. Pruning changes the practical cost, not the worst-case bound. Always state which one you are reporting.

## Use cases

- Constraint satisfaction: n-queens, Sudoku, graph coloring, timetabling.
- Enumeration: permutations, combinations, subsets, partitions, word breaks.
- Optimization over discrete choices: subset sum, knapsack, assignment, TSP for small `n`.
- Game trees: minimax with alpha-beta pruning.
- Any problem where a greedy choice is not provably correct and a DP state is not obvious.

Do **not** use backtracking when the subproblems overlap and a polynomial DP exists, or when the instance is large enough that only an approximation is feasible (see `np-hardness.md`).

## The pattern

```
solve(state):
    if state is complete: record solution; return
    for candidate in candidates(state):
        if not feasible(candidate, state): continue     # prune
        apply(candidate, state)
        solve(state)
        undo(candidate, state)                          # backtrack
```

Three knobs control the cost:

| Knob | Effect |
|---|---|
| **Ordering** | Try the most constrained / most promising candidate first |
| **Pruning** | Reject partial states that violate constraints or cannot beat the best |
| **Symmetry breaking** | Do not explore states equivalent by symmetry |

Branch and bound adds an optimistic bound: prune when `current + optimistic_remaining` cannot beat the best known solution. The bound must be **admissible** (never underestimate the remaining cost, never overestimate the remaining value).

## Code examples

### n-Queens: incremental constraints

```python
def solve_n_queens(n):
    solutions = []
    cols, diag1, diag2 = set(), set(), set()

    def place(row, board):
        if row == n:
            solutions.append(board[:])
            return
        for col in range(n):
            if col in cols or (row - col) in diag1 or (row + col) in diag2:
                continue
            cols.add(col); diag1.add(row - col); diag2.add(row + col)
            board.append(col)
            place(row + 1, board)
            board.pop()
            cols.remove(col); diag1.remove(row - col); diag2.remove(row + col)

    place(0, [])
    return solutions
```

`O(n!)` candidates in the worst case, `O(n)` extra space plus the output. The three sets make the feasibility check `O(1)`; the naive version scans the previous rows and adds a factor of `n`.

Symmetry: the first queen only needs to be tried in the left half of the first row, because mirror images are equivalent. That removes roughly a factor of two without changing correctness.

### Permutations: avoid copying the remaining list

```python
def permutations(items):
    out = []
    def rec(path, remaining):
        if not remaining:
            out.append(path[:])
            return
        for i in range(len(remaining)):
            rec(path + [remaining[i]], remaining[:i] + remaining[i + 1:])
    rec([], items)
    return out
```

`O(n! · n)` output and `O(n)` stack. The slicing is `O(n)` per node; a swap-in-place version keeps the same asymptotic bound but with smaller constants:

```python
def permutations(items):
    a = list(items)
    out = []
    def rec(k):
        if k == len(a):
            out.append(a[:])
            return
        for i in range(k, len(a)):
            a[k], a[i] = a[i], a[k]
            rec(k + 1)
            a[k], a[i] = a[i], a[k]
    rec(0)
    return out
```

### Subset sum with pruning

```python
def subset_sum(nums, target):
    nums.sort(reverse=True)                 # try large values first
    n = len(nums)
    suffix = [0] * (n + 1)
    for i in range(n - 1, -1, -1):
        suffix[i] = suffix[i + 1] + nums[i]

    def rec(i, current):
        if current == target:
            return True
        if current > target or current + suffix[i] < target:
            return False                    # prune: too big or cannot reach
        return rec(i + 1, current + nums[i]) or rec(i + 1, current)

    return rec(0, 0)
```

Worst case `O(2ⁿ)`, but the two prunes usually cut most of the tree. The `suffix` array is what makes the second prune `O(1)` instead of `O(n)`.

### Branch and bound: 0/1 knapsack

```python
def knapsack_bnb(weights, values, capacity):
    items = sorted(zip(weights, values), key=lambda x: x[1] / x[0], reverse=True)
    n = len(items)
    best = 0

    def bound(i, weight, value):
        # fractional relaxation: optimistic, so it is admissible
        total, cap = value, capacity - weight
        for w, v in items[i:]:
            if w <= cap:
                total += v; cap -= w
            else:
                total += v * cap / w
                break
        return total

    def rec(i, weight, value):
        nonlocal best
        if value > best:
            best = value
        if i == n or bound(i, weight, value) <= best:
            return
        w, v = items[i]
        if weight + w <= capacity:
            rec(i + 1, weight + w, value + v)
        rec(i + 1, weight, value)

    rec(0, 0, 0)
    return best
```

Worst case `O(2ⁿ)`, often far less. The relaxed bound is the key: without it this is plain enumeration.

### Alpha-beta pruning (game trees)

```python
def alphabeta(node, depth, alpha, beta, maximizing):
    if depth == 0 or node.is_terminal():
        return node.evaluate()
    if maximizing:
        value = float("-inf")
        for child in node.children():
            value = max(value, alphabeta(child, depth - 1, alpha, beta, False))
            alpha = max(alpha, value)
            if alpha >= beta:
                break                     # the opponent will avoid this branch
        return value
    value = float("inf")
    for child in node.children():
        value = min(value, alphabeta(child, depth - 1, alpha, beta, True))
        beta = min(beta, value)
        if alpha >= beta:
            break
    return value
```

Worst case is still the full tree; with good move ordering alpha-beta explores about `O(b^{d/2})` nodes. Move ordering is what makes it work.

## Anti-patterns

1. Checking constraints only at the leaves instead of pruning partial states early.
2. Recomputing feasibility with a linear scan on every node (use sets, bitsets, or incremental counters).
3. Copying the whole state at each node when apply/undo would do.
4. Exploring symmetric branches.
5. Using backtracking for overlapping subproblems that a DP could solve in polynomial time.
6. Reporting the pruned running time as a worst-case bound.

## Checklist

- Is the search space finite, and is every solution reachable exactly once?
- Are constraints checked as early as possible?
- Is the bound admissible?
- Is the state restored correctly after each recursive call?
- Did you measure the effective branching factor instead of assuming the worst case?
- Is the depth bounded, and is the stack large enough?

## Related

- `recursion.md` for the recursion mechanics and the recursion tree.
- `dynamicprog.md` when subproblems overlap and memoization is possible.
- `np-hardness.md` when the search space is inherently exponential.

