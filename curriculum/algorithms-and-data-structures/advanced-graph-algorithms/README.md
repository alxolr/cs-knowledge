# Advanced Graph Algorithms

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Advanced
**Prerequisites:** [Graphs](../graphs/README.md), [Union Find](../union-find/README.md), [Topological Sort](../topological-sort/README.md)

## Concept Overview

Advanced graph algorithms build on foundational graph traversal (BFS, DFS) and introduce techniques for solving more complex structural and optimization problems on graphs. These algorithms address questions such as: What is the cheapest way to connect all nodes? What is the shortest path between two nodes when edges have weights? How do you find bottleneck connections whose removal would disconnect the graph? Can you visit every edge exactly once?

Key topics include minimum spanning trees (Kruskal's and Prim's algorithms), shortest-path algorithms (Dijkstra's, Bellman-Ford), detection of bridges and articulation points, strongly connected components (Tarjan's and Kosaraju's algorithms), and Eulerian path construction. Many of these algorithms rely on Union Find for efficient cycle detection and component merging, and on topological ordering for processing directed acyclic sub-structures.

Mastering these algorithms is essential for tackling real-world problems in network design, routing, dependency analysis, and infrastructure resilience. The problems in this module exercise distinct sub-skills — from building spanning trees to finding shortest paths to identifying critical graph structures — ensuring broad coverage of the advanced graph algorithm landscape.

---

## Problem 1 — Minimum Spanning Tree (Kruskal's)

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given `n` nodes labeled `0` to `n - 1` and a list of undirected weighted edges, find the minimum spanning tree (MST) — the subset of edges that connects all nodes with the smallest total weight. Return the total weight of the MST. If the graph is not connected, return `-1`.

### Example

**Input:** `n = 4`, `edges = [[0,1,1], [0,2,4], [1,2,2], [1,3,6], [2,3,3]]`
**Output:** `6`
**Explanation:** The MST uses edges (0,1,1), (1,2,2), (2,3,3) with total weight 1 + 2 + 3 = 6.

<details>
<summary>Hints</summary>

1. Sort all edges by weight. Use Union Find to greedily add the cheapest edge that connects two different components, skipping edges that would form a cycle.

</details>

<details>
<summary>Solution</summary>

**Approach:** Kruskal's algorithm — sort edges by weight, use Union Find to build the MST.

```
function kruskalMST(n, edges):
    sort edges by weight ascending
    uf = UnionFind(n)
    totalWeight = 0
    edgesUsed = 0

    for each (u, v, w) in edges:
        if uf.find(u) != uf.find(v):
            uf.union(u, v)
            totalWeight += w
            edgesUsed += 1
            if edgesUsed == n - 1:
                break

    if edgesUsed != n - 1:
        return -1
    return totalWeight
```

O(E log E) time for sorting, O(n) space for Union Find, where E = number of edges.

</details>


---

## Problem 2 — Shortest Path with Dijkstra's Algorithm

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Given `n` nodes labeled `0` to `n - 1` and a list of directed weighted edges (all weights non-negative), find the shortest path distance from a source node `src` to a destination node `dst`. Return the shortest distance, or `-1` if `dst` is unreachable from `src`.

### Example

**Input:** `n = 5`, `edges = [[0,1,2], [0,2,4], [1,2,1], [1,3,7], [2,4,3], [3,4,1]]`, `src = 0`, `dst = 4`
**Output:** `6`
**Explanation:** The shortest path is 0 → 1 → 2 → 4 with cost 2 + 1 + 3 = 6.

<details>
<summary>Hints</summary>

1. Use a min-heap (priority queue) seeded with `(0, src)`. Relax neighbors greedily — when you pop a node with the smallest tentative distance, update its neighbors if a shorter path is found through it.
2. Once you pop the destination node from the heap, you can return immediately since its distance is finalized.

</details>

<details>
<summary>Solution</summary>

**Approach:** Dijkstra's algorithm with a min-heap priority queue.

```
function dijkstra(n, edges, src, dst):
    graph = adjacency list from edges  // graph[u] = [(v, w), ...]
    dist = array of size n, filled with infinity
    dist[src] = 0
    heap = min-heap containing (0, src)

    while heap is not empty:
        (d, u) = heap.pop()
        if u == dst:
            return d
        if d > dist[u]:
            continue   // stale entry
        for each (v, w) in graph[u]:
            newDist = d + w
            if newDist < dist[v]:
                dist[v] = newDist
                heap.push((newDist, v))

    return -1
```

O((V + E) log V) time with a binary heap, O(V + E) space.

</details>


---

## Problem 3 — Find Bridges in a Graph

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Given `n` nodes labeled `0` to `n - 1` and a list of undirected edges, find all bridges in the graph. A bridge is an edge whose removal disconnects the graph (increases the number of connected components). Return the list of bridges as pairs `[u, v]`.

### Example

**Input:** `n = 5`, `edges = [[0,1], [1,2], [2,0], [1,3], [3,4]]`
**Output:** `[[1,3], [3,4]]`
**Explanation:** Removing edge (1,3) disconnects node 3 and 4 from the rest. Removing edge (3,4) disconnects node 4. Edges within the cycle 0-1-2 are not bridges.

<details>
<summary>Hints</summary>

1. Use Tarjan's bridge-finding algorithm. Perform a DFS and track two values for each node: the discovery time and the lowest discovery time reachable through back edges from the subtree rooted at that node.
2. An edge (u, v) is a bridge if the lowest reachable discovery time from v is strictly greater than the discovery time of u — meaning v's subtree has no back edge reaching u or any of u's ancestors.

</details>

<details>
<summary>Solution</summary>

**Approach:** Tarjan's algorithm — DFS with discovery and low-link tracking.

```
function findBridges(n, edges):
    graph = adjacency list from edges (undirected)
    disc = array of size n, filled with -1
    low = array of size n
    bridges = []
    timer = 0

    function dfs(u, parent):
        disc[u] = low[u] = timer
        timer += 1
        for each v in graph[u]:
            if v == parent:
                continue
            if disc[v] == -1:
                dfs(v, u)
                low[u] = min(low[u], low[v])
                if low[v] > disc[u]:
                    bridges.append([u, v])
            else:
                low[u] = min(low[u], disc[v])

    for i from 0 to n - 1:
        if disc[i] == -1:
            dfs(i, -1)

    return bridges
```

O(V + E) time and space.

</details>


---

## Problem 4 — Strongly Connected Components (Kosaraju's)

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Given `n` nodes labeled `0` to `n - 1` and a list of directed edges, find all strongly connected components (SCCs) of the graph. A strongly connected component is a maximal set of nodes such that every node is reachable from every other node in the set. Return the list of SCCs, where each SCC is a list of node labels.

### Example

**Input:** `n = 7`, `edges = [[0,1], [1,2], [2,0], [1,3], [3,4], [4,5], [5,3], [5,6]]`
**Output:** `[[0,1,2], [3,4,5], [6]]`
**Explanation:** Nodes 0, 1, 2 form a cycle. Nodes 3, 4, 5 form a cycle. Node 6 has no cycle back to any other node, so it is its own SCC.

<details>
<summary>Hints</summary>

1. Kosaraju's algorithm uses two passes of DFS. First, run DFS on the original graph and push nodes onto a stack in order of their finish times.
2. Build the transpose (reversed) graph. Pop nodes from the stack and run DFS on the transpose — each DFS from an unvisited node discovers one SCC.

</details>

<details>
<summary>Solution</summary>

**Approach:** Kosaraju's two-pass algorithm — finish-order DFS, then DFS on the transposed graph.

```
function findSCCs(n, edges):
    graph = adjacency list from edges
    transpose = reversed adjacency list from edges
    visited = array of booleans, all false
    stack = []

    // Pass 1: DFS on original graph, record finish order
    function dfs1(u):
        visited[u] = true
        for each v in graph[u]:
            if not visited[v]:
                dfs1(v)
        stack.push(u)

    for i from 0 to n - 1:
        if not visited[i]:
            dfs1(i)

    // Pass 2: DFS on transpose in reverse finish order
    visited = array of booleans, all false
    sccs = []

    function dfs2(u, component):
        visited[u] = true
        component.append(u)
        for each v in transpose[u]:
            if not visited[v]:
                dfs2(v, component)

    while stack is not empty:
        u = stack.pop()
        if not visited[u]:
            component = []
            dfs2(u, component)
            sccs.append(component)

    return sccs
```

O(V + E) time and space for both passes.

</details>


---

## Problem 5 — Shortest Path with Negative Weights (Bellman-Ford)

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Given `n` nodes labeled `0` to `n - 1` and a list of directed weighted edges (weights may be negative), find the shortest path distance from a source node `src` to every other node. If a negative-weight cycle is reachable from `src`, return an indicator that shortest paths are undefined. Otherwise, return an array `dist` where `dist[i]` is the shortest distance from `src` to node `i` (or infinity if unreachable).

### Example

**Input:** `n = 5`, `edges = [[0,1,4], [0,2,5], [1,2,-3], [2,3,3], [3,4,2]]`, `src = 0`
**Output:** `[0, 4, 1, 4, 6]`
**Explanation:** Shortest paths: 0→0 = 0, 0→1 = 4, 0→1→2 = 4 + (−3) = 1, 0→1→2→3 = 1 + 3 = 4, 0→1→2→3→4 = 4 + 2 = 6.

**Input:** `n = 3`, `edges = [[0,1,1], [1,2,-1], [2,0,-1]]`, `src = 0`
**Output:** `"Negative cycle detected"`
**Explanation:** The cycle 0→1→2→0 has total weight 1 + (−1) + (−1) = −1 < 0, so shortest paths are undefined.

<details>
<summary>Hints</summary>

1. Relax all edges `n - 1` times. In each pass, for every edge (u, v, w), update `dist[v] = min(dist[v], dist[u] + w)` if `dist[u]` is not infinity.
2. After `n - 1` passes, do one more pass over all edges. If any distance can still be reduced, a negative-weight cycle exists.

</details>

<details>
<summary>Solution</summary>

**Approach:** Bellman-Ford algorithm — iterative edge relaxation with negative cycle detection.

```
function bellmanFord(n, edges, src):
    dist = array of size n, filled with infinity
    dist[src] = 0

    // Relax all edges n - 1 times
    for i from 1 to n - 1:
        for each (u, v, w) in edges:
            if dist[u] != infinity and dist[u] + w < dist[v]:
                dist[v] = dist[u] + w

    // Check for negative-weight cycles
    for each (u, v, w) in edges:
        if dist[u] != infinity and dist[u] + w < dist[v]:
            return "Negative cycle detected"

    return dist
```

O(V × E) time, O(V) space.

</details>
