# Graphs

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Intermediate
**Prerequisites:** Queues, Trees

## Concept Overview

A graph is a collection of nodes (vertices) connected by edges. Unlike trees, graphs can contain cycles, disconnected components, and edges that point in only one direction. This flexibility makes graphs the go-to model for networks, maps, social connections, dependency chains, and countless other real-world structures.

Graphs come in two main flavors: directed (edges have a direction) and undirected (edges go both ways). They can also be weighted (edges carry a cost) or unweighted. The two foundational traversal algorithms — Breadth-First Search (BFS) and Depth-First Search (DFS) — form the backbone of nearly every graph problem. BFS explores level by level using a queue and naturally finds shortest paths in unweighted graphs, while DFS dives as deep as possible along each branch using a stack (or recursion) and is ideal for detecting cycles, exploring connected components, and backtracking through paths.

Representation matters too. An adjacency list maps each node to its list of neighbors and is space-efficient for sparse graphs. An adjacency matrix uses a 2D array for O(1) edge lookups but costs O(V²) space. Choosing the right representation and traversal strategy is the first decision in any graph problem, and mastering these fundamentals unlocks advanced topics like topological sort, shortest-path algorithms, and network flow.

### Core Concepts

Graphs are represented in two main ways:

| Representation | Space | Edge Lookup | Best For |
|---|---|---|---|
| Adjacency list | O(V + E) | O(degree) | Sparse graphs |
| Adjacency matrix | O(V²) | O(1) | Dense graphs |

The two foundational traversals:

| Traversal | Data Structure | Explores | Finds |
|---|---|---|---|
| BFS | Queue | Level by level | Shortest path (unweighted), connected components |
| DFS | Stack / Recursion | As deep as possible | Cycles, connected components, topological order |

Both BFS and DFS run in O(V + E) time. BFS naturally finds shortest paths in unweighted graphs because it visits nodes in order of increasing distance. DFS is ideal for detecting cycles (via back edges), exploring connected components, and backtracking through paths. Choosing the right representation and traversal strategy is the first decision in any graph problem.

---

## Problem 1 — Breadth-First Traversal of a Graph

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given an undirected graph represented as an adjacency list and a starting node `source`, return a list of all nodes reachable from `source` in BFS order (level by level). If multiple neighbors are available, visit them in ascending order. Each node should appear at most once in the output.

### Example

**Input:**
```
graph = {
  0: [1, 2],
  1: [0, 3],
  2: [0, 3, 4],
  3: [1, 2],
  4: [2]
}
source = 0
```
**Output:** `[0, 1, 2, 3, 4]`
**Explanation:** Starting from node 0, BFS visits neighbors 1 and 2 (level 1), then 3 and 4 (level 2).

<details>
<summary>Hints</summary>

1. Use a queue initialized with the source node and a set to track visited nodes. Dequeue a node, add it to the result, then enqueue all unvisited neighbors in sorted order.

</details>

<details>
<summary>Solution</summary>

**Approach:** Standard BFS using a queue and a visited set.

```
function bfsTraversal(graph, source):
    visited = set containing source
    queue = [source]
    result = []
    while queue is not empty:
        node = queue.dequeue()
        result.append(node)
        for neighbor in sorted(graph[node]):
            if neighbor not in visited:
                visited.add(neighbor)
                queue.enqueue(neighbor)
    return result
```

O(V + E) time, O(V) space.

</details>

---

## Problem 2 — Number of Connected Components

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Given `n` nodes labeled `0` to `n - 1` and a list of undirected edges, return the number of connected components in the graph. A connected component is a maximal set of nodes such that there is a path between every pair of nodes in the set.

### Example

**Input:** `n = 6`, `edges = [[0, 1], [1, 2], [3, 4]]`
**Output:** `3`
**Explanation:** Component 1: {0, 1, 2}. Component 2: {3, 4}. Component 3: {5} (isolated node). Total = 3.

<details>
<summary>Hints</summary>

1. Build an adjacency list from the edge list. Then iterate through all nodes 0 to n − 1. Each time you encounter an unvisited node, start a DFS or BFS from it and mark every reachable node as visited. Each such traversal discovers one connected component.

</details>

<details>
<summary>Solution</summary>

**Approach:** Build adjacency list, then count DFS/BFS launches from unvisited nodes.

```
function countComponents(n, edges):
    adj = empty adjacency list for n nodes
    for [u, v] in edges:
        adj[u].append(v)
        adj[v].append(u)

    visited = set()
    components = 0

    function dfs(node):
        visited.add(node)
        for neighbor in adj[node]:
            if neighbor not in visited:
                dfs(neighbor)

    for node from 0 to n - 1:
        if node not in visited:
            dfs(node)
            components += 1

    return components
```

O(V + E) time, O(V + E) space.

</details>

---

## Problem 3 — Detect a Cycle in an Undirected Graph

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Given `n` nodes labeled `0` to `n - 1` and a list of undirected edges, determine whether the graph contains a cycle. Return `true` if a cycle exists, `false` otherwise.

### Example

**Input:** `n = 4`, `edges = [[0, 1], [1, 2], [2, 3], [3, 1]]`
**Output:** `true`
**Explanation:** Nodes 1 → 2 → 3 → 1 form a cycle.

**Input:** `n = 3`, `edges = [[0, 1], [1, 2]]`
**Output:** `false`
**Explanation:** The graph is a simple path with no cycles.

<details>
<summary>Hints</summary>

1. Run DFS and pass the parent of each node along. If you encounter a visited neighbor that is not the parent of the current node, you have found a cycle.
2. Remember to handle disconnected components — start a DFS from every unvisited node.

</details>

<details>
<summary>Solution</summary>

**Approach:** DFS with parent tracking to detect back edges.

```
function hasCycle(n, edges):
    adj = empty adjacency list for n nodes
    for [u, v] in edges:
        adj[u].append(v)
        adj[v].append(u)

    visited = set()

    function dfs(node, parent):
        visited.add(node)
        for neighbor in adj[node]:
            if neighbor not in visited:
                if dfs(neighbor, node):
                    return true
            else if neighbor != parent:
                return true
        return false

    for node from 0 to n - 1:
        if node not in visited:
            if dfs(node, -1):
                return true
    return false
```

O(V + E) time, O(V + E) space.

</details>

---

## Problem 4 — Shortest Path in an Unweighted Graph

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Given a directed, unweighted graph represented as an adjacency list, a source node `src`, and a destination node `dst`, return the length of the shortest path from `src` to `dst`. If no path exists, return `-1`. The length of a path is the number of edges traversed.

### Example

**Input:**
```
graph = {
  0: [1, 2],
  1: [3],
  2: [3, 4],
  3: [5],
  4: [5],
  5: []
}
src = 0, dst = 5
```
**Output:** `3`
**Explanation:** Shortest path is 0 → 1 → 3 → 5 (3 edges). Another path 0 → 2 → 4 → 5 also has length 3.

**Input:**
```
graph = {0: [1], 1: [], 2: []}
src = 0, dst = 2
```
**Output:** `-1`
**Explanation:** Node 2 is not reachable from node 0.

<details>
<summary>Hints</summary>

1. BFS from the source naturally visits nodes in order of increasing distance. Track the distance of each node from the source. When you dequeue the destination, its recorded distance is the shortest path length.
2. Use a distance array initialized to -1 (unvisited). Set `distance[src] = 0` and for each neighbor update `distance[neighbor] = distance[current] + 1`.

</details>

<details>
<summary>Solution</summary>

**Approach:** BFS from source, tracking distance per node.

```
function shortestPath(graph, src, dst):
    if src == dst: return 0
    dist = array of size |V| initialized to -1
    dist[src] = 0
    queue = [src]

    while queue is not empty:
        node = queue.dequeue()
        for neighbor in graph[node]:
            if dist[neighbor] == -1:
                dist[neighbor] = dist[node] + 1
                if neighbor == dst:
                    return dist[neighbor]
                queue.enqueue(neighbor)

    return -1
```

O(V + E) time, O(V) space.

</details>

---

## Problem 5 — Course Schedule (Cycle Detection in Directed Graph)

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

There are `numCourses` courses labeled `0` to `numCourses - 1`. You are given a list of prerequisite pairs where `[a, b]` means course `b` must be completed before course `a`. Determine if it is possible to finish all courses. Return `true` if all courses can be completed (i.e., there is no cyclic dependency), `false` otherwise.

### Example

**Input:** `numCourses = 4`, `prerequisites = [[1, 0], [2, 1], [3, 2]]`
**Output:** `true`
**Explanation:** Take courses in order 0 → 1 → 2 → 3. No cycle exists.

**Input:** `numCourses = 3`, `prerequisites = [[0, 1], [1, 2], [2, 0]]`
**Output:** `false`
**Explanation:** 0 requires 1, 1 requires 2, 2 requires 0 — a circular dependency.

<details>
<summary>Hints</summary>

1. Model courses as nodes and prerequisites as directed edges. The problem reduces to detecting a cycle in a directed graph.
2. Use DFS with three states per node: unvisited, in-progress (on the current DFS path), and completed. If you visit a node that is in-progress, a cycle exists.
3. Alternatively, use Kahn's algorithm (BFS-based topological sort): compute in-degrees, enqueue all nodes with in-degree 0, and process. If the total processed count is less than `numCourses`, a cycle exists.

</details>

<details>
<summary>Solution</summary>

**Approach — DFS with three-color marking:**

```
function canFinish(numCourses, prerequisites):
    adj = empty adjacency list for numCourses nodes
    for [a, b] in prerequisites:
        adj[b].append(a)

    // 0 = unvisited, 1 = in-progress, 2 = completed
    state = array of size numCourses initialized to 0

    function dfs(node):
        state[node] = 1
        for neighbor in adj[node]:
            if state[neighbor] == 1:
                return false          // cycle detected
            if state[neighbor] == 0:
                if not dfs(neighbor):
                    return false
        state[node] = 2
        return true

    for course from 0 to numCourses - 1:
        if state[course] == 0:
            if not dfs(course):
                return false
    return true
```

O(V + E) time, O(V + E) space.

**Alternative — Kahn's algorithm (BFS topological sort):**

```
function canFinish(numCourses, prerequisites):
    adj = empty adjacency list for numCourses nodes
    inDegree = array of size numCourses initialized to 0
    for [a, b] in prerequisites:
        adj[b].append(a)
        inDegree[a] += 1

    queue = all nodes where inDegree == 0
    processed = 0

    while queue is not empty:
        node = queue.dequeue()
        processed += 1
        for neighbor in adj[node]:
            inDegree[neighbor] -= 1
            if inDegree[neighbor] == 0:
                queue.enqueue(neighbor)

    return processed == numCourses
```

O(V + E) time, O(V + E) space.

</details>
