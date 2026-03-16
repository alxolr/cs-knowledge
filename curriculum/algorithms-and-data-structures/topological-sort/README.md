# Topological Sort

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Advanced
**Prerequisites:** [Graphs](../graphs/README.md)

## Concept Overview

Topological sorting produces a linear ordering of the vertices of a directed acyclic graph (DAG) such that for every directed edge (u, v), vertex u appears before vertex v in the ordering. If the graph contains a cycle, no valid topological order exists. This ordering is fundamental whenever you need to schedule tasks that have dependency relationships — compiling source files, resolving package dependencies, course prerequisite planning, and build-system execution order are all real-world instances of topological sort.

The two classic algorithms for topological sorting are Kahn's algorithm (BFS-based) and the DFS-based approach. Kahn's algorithm maintains an in-degree count for every vertex and repeatedly removes vertices with zero in-degree, appending them to the result. The DFS approach performs a post-order traversal and reverses the finishing order. Both run in O(V + E) time and O(V) space, where V is the number of vertices and E is the number of edges.

Beyond producing a single valid ordering, topological sort is a building block for more advanced graph algorithms: shortest and longest paths in DAGs, critical-path analysis, and detecting whether a directed graph is acyclic. Mastering this concept requires comfort with graph representations (adjacency lists), BFS/DFS traversals, and in-degree tracking — all of which build directly on the Graphs prerequisite module.

---

## Problem 1 — Course Schedule Feasibility

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

There are `n` courses labeled `0` to `n - 1`. You are given an array `prerequisites` where `prerequisites[i] = [a, b]` means you must complete course `b` before course `a`. Determine whether it is possible to finish all courses. In other words, return `true` if the dependency graph is a DAG, and `false` if it contains a cycle.

### Example

**Input:** `n = 4`, `prerequisites = [[1, 0], [2, 1], [3, 2]]`
**Output:** `true`
**Explanation:** A valid order is 0 → 1 → 2 → 3. There are no cycles.

**Input:** `n = 2`, `prerequisites = [[0, 1], [1, 0]]`
**Output:** `false`
**Explanation:** Courses 0 and 1 depend on each other, forming a cycle.

<details>
<summary>Hints</summary>

1. Build an adjacency list and compute the in-degree of every node. Use Kahn's algorithm: start with all zero-in-degree nodes, process them, and decrement the in-degree of their neighbors. If you process all n nodes, there is no cycle.

</details>

<details>
<summary>Solution</summary>

**Approach:** Kahn's algorithm — BFS with in-degree tracking.

```
function canFinish(n, prerequisites):
    adj = array of n empty lists
    inDegree = array of n zeros

    for each [a, b] in prerequisites:
        adj[b].append(a)
        inDegree[a] += 1

    queue = []
    for i from 0 to n - 1:
        if inDegree[i] == 0:
            queue.append(i)

    count = 0
    while queue is not empty:
        node = queue.removeFirst()
        count += 1
        for neighbor in adj[node]:
            inDegree[neighbor] -= 1
            if inDegree[neighbor] == 0:
                queue.append(neighbor)

    return count == n
```

O(V + E) time, O(V + E) space.

</details>

---

## Problem 2 — Course Schedule Ordering

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

There are `n` courses labeled `0` to `n - 1`. You are given an array `prerequisites` where `prerequisites[i] = [a, b]` means course `b` must be taken before course `a`. Return a valid ordering in which you can finish all courses. If there are multiple valid orderings, return any one. If it is impossible to finish all courses (a cycle exists), return an empty array.

### Example

**Input:** `n = 4`, `prerequisites = [[1, 0], [2, 0], [3, 1], [3, 2]]`
**Output:** `[0, 1, 2, 3]` (or `[0, 2, 1, 3]`)
**Explanation:** Course 0 has no prerequisites. Courses 1 and 2 both depend on 0. Course 3 depends on 1 and 2.

**Input:** `n = 2`, `prerequisites = [[0, 1], [1, 0]]`
**Output:** `[]`
**Explanation:** A cycle exists, so no valid ordering is possible.

<details>
<summary>Hints</summary>

1. This is a direct application of topological sort. Use Kahn's algorithm and collect the nodes in the order they are dequeued. If the result has fewer than n nodes, a cycle exists.
2. Alternatively, run DFS and record nodes in reverse post-order. Detect cycles using a "visiting" state during DFS.

</details>

<details>
<summary>Solution</summary>

**Approach:** Kahn's algorithm collecting the processing order.

```
function findOrder(n, prerequisites):
    adj = array of n empty lists
    inDegree = array of n zeros

    for each [a, b] in prerequisites:
        adj[b].append(a)
        inDegree[a] += 1

    queue = []
    for i from 0 to n - 1:
        if inDegree[i] == 0:
            queue.append(i)

    order = []
    while queue is not empty:
        node = queue.removeFirst()
        order.append(node)
        for neighbor in adj[node]:
            inDegree[neighbor] -= 1
            if inDegree[neighbor] == 0:
                queue.append(neighbor)

    if length(order) == n:
        return order
    return []
```

O(V + E) time, O(V + E) space.

</details>

---

## Problem 3 — Alien Dictionary

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

You are given a list of strings `words` sorted lexicographically according to the rules of an unknown alien language. Derive the ordering of the letters in this language. Return a string of the unique letters in the alien language sorted in the alien lexicographic order. If the order is invalid (contains a contradiction), return an empty string. If multiple valid orderings exist, return any one.

### Example

**Input:** `words = ["wrt", "wrf", "er", "ett", "rftt"]`
**Output:** `"wertf"`
**Explanation:** From comparing adjacent words: w < e (from "wrt" vs "er"), r < t (from "wrf" vs "wrt" — actually t < f from "wrt" vs "wrf"), e < r (from "er" vs "ett"), and r < f is inferred. One valid order is w → e → r → t → f.

**Input:** `words = ["z", "x", "z"]`
**Output:** `""`
**Explanation:** z < x and x < z is a contradiction (cycle).

<details>
<summary>Hints</summary>

1. Compare each pair of adjacent words to extract ordering constraints. The first position where the two words differ gives you a directed edge: `word1[i] → word2[i]`.
2. Watch for the edge case where a longer word appears before its prefix (e.g., `["abc", "ab"]`) — this is an invalid ordering.
3. After extracting all edges, run topological sort on the character graph. If a cycle is detected, return an empty string.

</details>

<details>
<summary>Solution</summary>

**Approach:** Build a character dependency graph from adjacent word comparisons, then topological sort.

```
function alienOrder(words):
    // Collect all unique characters
    chars = set of all characters in words
    adj = map from char to set of chars (empty)
    inDegree = map from char to 0

    // Extract edges from adjacent word pairs
    for i from 0 to length(words) - 2:
        w1 = words[i]
        w2 = words[i + 1]
        if w1 starts with w2 and length(w1) > length(w2):
            return ""          // invalid: prefix must come first
        for j from 0 to min(length(w1), length(w2)) - 1:
            if w1[j] != w2[j]:
                if w2[j] not in adj[w1[j]]:
                    adj[w1[j]].add(w2[j])
                    inDegree[w2[j]] += 1
                break

    // Kahn's algorithm
    queue = [c for c in chars if inDegree[c] == 0]
    result = []
    while queue is not empty:
        c = queue.removeFirst()
        result.append(c)
        for neighbor in adj[c]:
            inDegree[neighbor] -= 1
            if inDegree[neighbor] == 0:
                queue.append(neighbor)

    if length(result) != length(chars):
        return ""              // cycle detected
    return join(result)
```

O(C) time where C is the total number of characters across all words (to extract edges) plus O(V + E) for the topological sort on the character graph.

</details>

---

## Problem 4 — Parallel Job Scheduling

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

You are given `n` jobs labeled `0` to `n - 1`, an array `duration` where `duration[i]` is the time job `i` takes, and an array `dependencies` where `dependencies[i] = [a, b]` means job `b` must finish before job `a` can start. Jobs without dependency conflicts can run in parallel on unlimited processors. Return the minimum total time to complete all jobs.

### Example

**Input:** `n = 4`, `duration = [3, 2, 4, 1]`, `dependencies = [[2, 0], [3, 1], [3, 2]]`
**Output:** `8`
**Explanation:** Jobs 0 and 1 start at time 0 (no prerequisites). Job 0 finishes at t=3, job 1 at t=2. Job 2 starts at t=3 (after job 0) and finishes at t=7. Job 3 starts at t=7 (after jobs 1 and 2) and finishes at t=8. The critical path is 0 → 2 → 3 with total length 3 + 4 + 1 = 8.

<details>
<summary>Hints</summary>

1. This is a longest-path problem on a DAG. Process nodes in topological order and compute the earliest start time for each job as the maximum finish time among all its prerequisites.
2. The answer is the maximum finish time across all jobs — this corresponds to the critical path.

</details>

<details>
<summary>Solution</summary>

**Approach:** Topological sort followed by dynamic programming for the longest (critical) path.

```
function minCompletionTime(n, duration, dependencies):
    adj = array of n empty lists
    inDegree = array of n zeros

    for each [a, b] in dependencies:
        adj[b].append(a)
        inDegree[a] += 1

    // Kahn's algorithm to get topological order
    queue = []
    for i from 0 to n - 1:
        if inDegree[i] == 0:
            queue.append(i)

    topoOrder = []
    while queue is not empty:
        node = queue.removeFirst()
        topoOrder.append(node)
        for neighbor in adj[node]:
            inDegree[neighbor] -= 1
            if inDegree[neighbor] == 0:
                queue.append(neighbor)

    // Compute earliest start times
    earliest = array of n zeros
    for node in topoOrder:
        for neighbor in adj[node]:
            earliest[neighbor] = max(earliest[neighbor],
                                     earliest[node] + duration[node])

    // Answer is max finish time
    answer = 0
    for i from 0 to n - 1:
        answer = max(answer, earliest[i] + duration[i])
    return answer
```

O(V + E) time, O(V + E) space.

</details>

---

## Problem 5 — Longest Path in a DAG

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

You are given a directed acyclic graph with `n` nodes labeled `0` to `n - 1` and an array of edges `edges` where `edges[i] = [u, v]` represents a directed edge from `u` to `v`. Find the length of the longest path in the graph, measured by the number of edges. If the graph has no edges, return `0`.

### Example

**Input:** `n = 6`, `edges = [[0, 1], [0, 2], [1, 3], [2, 3], [3, 4], [2, 5]]`
**Output:** `3`
**Explanation:** The longest path is 0 → 1 → 3 → 4 (3 edges). The path 0 → 2 → 3 → 4 also has 3 edges. No path has more than 3 edges.

**Input:** `n = 4`, `edges = [[0, 1], [1, 2], [0, 3], [3, 2]]`
**Output:** `2`
**Explanation:** The longest paths are 0 → 1 → 2 and 0 → 3 → 2, each with 2 edges.

<details>
<summary>Hints</summary>

1. Process nodes in topological order. For each node, update the longest distance to each of its neighbors: `dist[neighbor] = max(dist[neighbor], dist[node] + 1)`.
2. The answer is the maximum value in the `dist` array after processing all nodes.

</details>

<details>
<summary>Solution</summary>

**Approach:** Topological sort then relax edges in topological order to find the longest path.

```
function longestPath(n, edges):
    adj = array of n empty lists
    inDegree = array of n zeros

    for each [u, v] in edges:
        adj[u].append(v)
        inDegree[v] += 1

    // Kahn's algorithm
    queue = []
    for i from 0 to n - 1:
        if inDegree[i] == 0:
            queue.append(i)

    topoOrder = []
    while queue is not empty:
        node = queue.removeFirst()
        topoOrder.append(node)
        for neighbor in adj[node]:
            inDegree[neighbor] -= 1
            if inDegree[neighbor] == 0:
                queue.append(neighbor)

    // Relax edges in topological order
    dist = array of n zeros
    for node in topoOrder:
        for neighbor in adj[node]:
            dist[neighbor] = max(dist[neighbor], dist[node] + 1)

    return max(dist)
```

O(V + E) time, O(V + E) space.

</details>