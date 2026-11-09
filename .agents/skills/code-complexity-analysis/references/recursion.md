# Recursion

## Description

How to design recursive algorithms and turn them into cost bounds. Covers reductions, the "simplify and delegate" pattern, recursion trees, and the recurrences that appear most often in practice.

A recursive algorithm is correct when:

1. Every call reduces the problem to a **strictly smaller** instance of the same problem.
2. A **base case** solves the smallest instance directly.
3. The combination step is correct assuming the recursive calls are correct (the "Recursion Fairy", i.e. the induction hypothesis).

If the reduction can cycle, the algorithm does not terminate. If subproblems repeat, recursion is doing redundant work and should become dynamic programming or memoization.

## Use cases

- Divide and conquer: merge sort, quicksort, binary search, fast exponentiation, FFT.
- Tree and graph traversal where the structure itself is recursive.
- Problems defined by a recurrence (Fibonacci, binomial coefficients, edit distance).
- Backtracking and enumeration (see `backtracking.md`).
- Designing an interface before knowing its implementation: reduce problem X to black-box problem Y.

## Recurrences

Write `T(n)` for the cost of the recursive call and solve it rather than guessing.

| Recurrence | Solution | Example |
|---|---|---|
| `T(n) = T(n/2) + O(1)` | `O(log n)` | Binary search |
| `T(n) = T(n-1) + O(1)` | `O(n)` | Linear recursion, factorial |
| `T(n) = 2T(n/2) + O(1)` | `O(n)` | Tree traversal, fast power on lists |
| `T(n) = 2T(n/2) + O(n)` | `O(n log n)` | Merge sort, naive FFT split |
| `T(n) = T(n-1) + O(n)` | `O(n²)` | Selection sort recurrence |
| `T(n) = 2T(n-1) + O(1)` | `O(2ⁿ)` | Hanoi, subset enumeration |
| `T(n) = T(n-1) + T(n-2) + O(1)` | `O(φⁿ) = O(2ⁿ)` | Naive Fibonacci |
| `T(n) = aT(n/b) + f(n)` | Master theorem | Divide and conquer |

Use the **recursion tree** or **substitution** when the Master theorem does not apply (`a < 1`, uneven splits, additive lower-order work). For space, count the maximum depth of the call stack, not the number of calls.

## Code examples

### Naive Fibonacci: exponential time, linear stack

```python
def fib(n):
    if n < 2:
        return n
    return fib(n - 1) + fib(n - 2)
```

`T(n) = T(n-1) + T(n-2) + O(1) = O(2ⁿ)`; space `O(n)` stack. The same `fib(k)` is recomputed exponentially many times.

**Fix:** memoize or use two rolling values.

```python
def fib(n):
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a
```

Time `O(n)`, space `O(1)`.

### Binary search: halve the space

```python
def binary_search(a, target):
    lo, hi = 0, len(a) - 1
    while lo <= hi:
        mid = lo + (hi - lo) // 2   # avoids overflow in fixed-width languages
        if a[mid] == target:
            return mid
        if a[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return -1
```

`T(n) = T(n/2) + O(1) = O(log n)` time, `O(1)` space. Precondition: `a` is sorted. If it is not sorted, sorting costs `O(n log n)` once; sorting inside a loop is a different, much worse problem.

### Merge sort: split, solve, combine

```python
def merge_sort(a):
    if len(a) <= 1:
        return a
    mid = len(a) // 2
    left = merge_sort(a[:mid])
    right = merge_sort(a[mid:])
    out, i, j = [], 0, 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            out.append(left[i]); i += 1
        else:
            out.append(right[j]); j += 1
    out.extend(left[i:])
    out.extend(right[j:])
    return out
```

`T(n) = 2T(n/2) + O(n) = O(n log n)` time, `O(n)` auxiliary space. The `a[:mid]` slices are copies; in an in-place version they disappear but the bound does not change.

### Tower of Hanoi: two subproblems, no overlap

```python
def hanoi(n, src, dst, aux):
    if n == 0:
        return
    hanoi(n - 1, src, aux, dst)
    print(src, "->", dst)
    hanoi(n - 1, aux, dst, src)
```

`T(n) = 2T(n-1) + O(1) = O(2ⁿ)`. This is inherent: the optimal solution moves `2ⁿ - 1` disks. Stack depth `O(n)`.

### Peasant multiplication: recursion with a logarithmic depth

```python
def peasant_mul(x, y):
    if x == 0:
        return 0
    half = peasant_mul(x // 2, y + y)
    return half + y if x % 2 else half
```

`T(x) = T(x/2) + O(1) = O(log x)` time and stack. The recursive call is not in tail position, so it cannot be eliminated automatically; write the loop if depth matters.

### Tail recursion is not free

```python
def count_down(n):
    if n == 0:
        return
    count_down(n - 1)   # tail call
```

Python does **not** guarantee tail-call elimination, and the default recursion limit is about 1000. Convert deep tail recursion to a loop, or raise the limit deliberately. C/C++/Rust also do not guarantee it (LLVM may do it at `-O2`, but it is not a language promise).

## Checklist

- Is there a base case for every branch?
- Does every recursive call receive a strictly smaller input?
- Is the recursion depth bounded? (`O(n)` depth on `n = 10⁶` will overflow most stacks.)
- Do subproblems overlap? If yes, memoize or switch to bottom-up DP.
- Did you solve the recurrence, or only guess it?
- Is the recursive version clearer than the loop? If not, prefer iteration.

## Related

- `dynamicprog.md` for overlapping subproblems and memoization.
- `backtracking.md` for search trees and pruning.
- `asymptotic.md` for the meaning of the bounds.

