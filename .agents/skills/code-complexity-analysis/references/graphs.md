# Graphs

## Description

Which graph representation and algorithm to use, and what each one costs. Covers traversal, connectivity, topological order, shortest paths, minimum spanning trees, strongly connected components, and flows.

Throughout: `V` = vertices, `E` = edges, `W` = maximum edge weight (when relevant), `B` = block size for external memory.

## Use cases

- Reachability, connected components, cycle detection.
- Dependency ordering (build systems, task scheduling, package managers) — topological sort.
- Shortest route or cheapest cost — BFS, Dijkstra, Bellman-Ford, Floyd-Warshall.
- Network design and clustering — MST.
- Condensation and mutual reachability — SCC.
- Matching, assignment, scheduling with capacities — max flow / min cut.
- Social networks, web graphs, citations, road networks, state graphs.

## Representations

| Representation | Space | Edge lookup | Iterate neighbors | Use when |
|---|---|---|---|---|
| Adjacency list | `O(V + E)` | `O(deg)` | `O(deg)` | Sparse graphs (most real graphs) |
| Adjacency matrix | `O(V²)` | `O(1)` | `O(V)` | Dense graphs, tiny `V`, or `+`/`*` algebra |
| Edge list | `O(E)` | `O(E)` | `O(E)` | Sorting edges (Kruskal), input format |
| CSR (compressed sparse row) | `O(V + E)` | `O(log deg)` | `O(deg)` contiguous | High-performance / memory-bound traversal |

Use an adjacency list by default. An adjacency matrix on a sparse graph wastes memory and turns `O(V + E)` traversals into `O(V²)`.

## Algorithm selection

| Need | Algorithm | Time | Notes |
|---|---|---|---|
| Unweighted shortest path | BFS | `O(V + E)` | Also gives distance and a parent tree |
| Weighted, non-negative | Dijkstra + binary heap | `O((V + E) log V)` | No negative edges |
| Weighted, negative edges | Bellman-Ford | `O(V·E)` | Detects negative cycles |
| All-pairs, small/dense | Floyd-Warshall | `O(V³)` | Simple, cache-friendly for small `V` |
| All-pairs, sparse | Johnson | `O(V·E + V² log V)` | Reweighting + Dijkstra |
| DAG shortest path | Topological order + relax | `O(V + E)` | Faster than Dijkstra |
| Topological order | Kahn or DFS | `O(V + E)` | Only for DAGs; cycle detection |
| Cycle detection | DFS colors | `O(V + E)` | Directed: recursion stack; undirected: parent edge |
| Connected components | BFS/DFS or union-find | `O(V + E)` / `O(E α(V))` | DSU is good for incremental edges |
| MST | Kruskal / Prim | `O(E log E)` / `O(E log V)` | Kruskal with DSU; Prim with heap |
| SCC | Tarjan / Kosaraju | `O(V + E)` | Condensation is a DAG |
| Max flow | Dinic / push-relabel | `O(V²E)` / better in practice | Unit capacities often much faster |

## Code examples

### BFS: unweighted shortest paths

```python
from collections import deque

def bfs(adj, source):
    dist = {source: 0}
    parent = {source: None}
    q = deque([source])
    while q:
        u = q.popleft()
        for v in adj[u]:
            if v not in dist:
                dist[v] = dist[u] + 1
                parent[v] = u
                q.append(v)
    return dist, parent
```

`O(V + E)` time, `O(V)` space. Each vertex enters the queue once. The path can be reconstructed by following `parent` backwards.

### DFS: iterative (avoids stack overflow)

```python
def dfs(adj, source):
    seen = set()
    stack = [source]
    while stack:
        u = stack.pop()
        if u in seen:
            continue
        seen.add(u)
        for v in adj[u]:
            if v not in seen:
                stack.append(v)
    return seen
```

`O(V + E)` time, `O(V)` space. The recursive version has the same bound but depth `O(V)`; use the iterative form on deep graphs.

### Topological sort: Kahn's algorithm

```python
from collections import deque

def topological_sort(adj, n):
    indegree = [0] * n
    for u in range(n):
        for v in adj[u]:
            indegree[v] += 1
    q = deque(u for u in range(n) if indegree[u] == 0)
    order = []
    while q:
        u = q.popleft()
        order.append(u)
        for v in adj[u]:
            indegree[v] -= 1
            if indegree[v] == 0:
                q.append(v)
    return order if len(order) == n else None   # None means a cycle
```

`O(V + E)` time and `O(V)` space. The `len(order) == n` check detects a cycle.

### Dijkstra: non-negative weights

```python
import heapq

def dijkstra(adj, source):
    dist = {source: 0}
    pq = [(0, source)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist.get(u, float("inf")):
            continue                      # stale entry
        for v, w in adj[u]:
            nd = d + w
            if nd < dist.get(v, float("inf")):
                dist[v] = nd
                heapq.heappush(pq, (nd, v))
    return dist
```

`O((V + E) log V)` with a binary heap and a lazy-deletion implementation. With a Fibonacci heap it is `O(E + V log V)`, but the constant is usually worse. Dijkstra is **incorrect** with negative edges; use Bellman-Ford.

### Bellman-Ford: negative edges and cycle detection

```python
def bellman_ford(edges, n, source):
    INF = float("inf")
    dist = [INF] * n
    dist[source] = 0
    for _ in range(n - 1):
        for u, v, w in edges:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
    for u, v, w in edges:                 # one more pass detects a negative cycle
        if dist[u] + w < dist[v]:
            return None
    return dist
```

`O(V·E)` time, `O(V)` space.

### Union-find (DSU): connectivity and Kruskal

```python
class DSU:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n
        self.components = n

    def find(self, x):
        while self.parent[x] != x:
            self.parent[x] = self.parent[self.parent[x]]   # path halving
            x = self.parent[x]
        return x

    def union(self, a, b):
        a, b = self.find(a), self.find(b)
        if a == b:
            return False
        if self.rank[a] < self.rank[b]:
            a, b = b, a
        self.parent[b] = a
        if self.rank[a] == self.rank[b]:
            self.rank[a] += 1
        self.components -= 1
        return True
```

Amortized `O(α(V))` per operation, effectively constant. Path compression plus union by rank is what gives the inverse-Ackermann bound.

### Floyd-Warshall: all pairs

```python
def floyd_warshall(dist):
    n = len(dist)
    for k in range(n):
        for i in range(n):
            dik = dist[i][k]
            if dik == float("inf"):
                continue
            for j in range(n):
                nd = dik + dist[k][j]
                if nd < dist[i][j]:
                    dist[i][j] = nd
    return dist
```

`O(V³)` time, `O(V²)` space. The `k` loop must be outermost. Only reasonable for small or dense graphs (`V` up to a few hundred to a few thousand, depending on the constant).

## Anti-patterns

1. Adjacency matrix for a sparse graph (`V = 10⁵`, `E = 10⁵`).
2. Dijkstra with negative weights.
3. BFS with a weighted graph, expecting the cheapest path.
4. Recursive DFS on a deep graph without a stack check.
5. Rebuilding the graph per query.
6. Using Floyd-Warshall by default instead of running Dijkstra from each source when the graph is sparse.
7. Forgetting to skip stale heap entries in Dijkstra (turns the bound into something worse).
8. Ignoring the direction of edges or treating a DAG as undirected.

## Checklist

- Is the graph directed or undirected? Any self-loops or multi-edges?
- Are weights non-negative, negative, zero, integer, or floating point?
- Sparse or dense? Does the representation match?
- Is the graph a DAG? If yes, use topological order, not Dijkstra.
- Is the graph connected? What about isolated vertices?
- Are edge cases covered: `V = 0`, `V = 1`, parallel edges, disconnected components?
- Does the algorithm require `O(V)` recursion depth? Is the stack enough?
- Did you state `V` and `E` in the complexity, not just `n`?

## Related

- `dynamicprog.md` for shortest paths on DAGs and DP over subsets.
- `np-hardness.md` for Hamiltonian paths, TSP, and other hard graph problems.
- `memory.md` for CSR layout and cache behavior in large traversals.

