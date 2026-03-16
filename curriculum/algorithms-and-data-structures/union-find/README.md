# Union Find

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Advanced
**Prerequisites:** [Graphs](../graphs/README.md)

## Concept Overview

Union Find (also called Disjoint Set Union, or DSU) is a data structure that tracks a collection of non-overlapping sets and supports two primary operations: **find**, which determines which set a particular element belongs to, and **union**, which merges two sets into one. These operations make it exceptionally efficient for answering connectivity queries — "are these two elements in the same group?" — and for dynamically grouping elements together.

The naive implementation uses a parent array where each element points to a representative (root) of its set. Two critical optimizations bring the amortized cost of each operation down to nearly O(1): **path compression** (during find, make every visited node point directly to the root) and **union by rank** (or union by size), which attaches the smaller tree under the root of the larger tree to keep the structure shallow.

Union Find is the backbone of several classic algorithms and problem families: Kruskal's minimum spanning tree algorithm, detecting cycles in undirected graphs, computing connected components, and solving dynamic connectivity problems. It also appears in less obvious settings such as grouping accounts, determining redundant connections in networks, and region-merging problems on grids. A solid understanding of graph fundamentals is essential because most Union Find problems are framed in terms of nodes and edges.

---

## Problem 1 — Basic Disjoint Set Union

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Implement a Union Find data structure that supports the following operations on `n` elements (labeled `0` to `n - 1`):

- `union(x, y)` — merge the sets containing elements `x` and `y`.
- `find(x)` — return the representative (root) of the set containing `x`.
- `connected(x, y)` — return `true` if `x` and `y` are in the same set.

After processing a list of union operations, answer a list of connectivity queries.

### Example

**Input:**
```
n = 5
unions = [(0, 1), (1, 2), (3, 4)]
queries = [(0, 2), (0, 3), (3, 4)]
```
**Output:** `[true, false, true]`
**Explanation:** After unions, {0, 1, 2} and {3, 4} are the two connected components. 0 and 2 share a set, 0 and 3 do not, and 3 and 4 share a set.

<details>
<summary>Hints</summary>

1. Initialize a `parent` array where `parent[i] = i`. To find the root, follow parent pointers until you reach an element that is its own parent.
2. Apply path compression in `find` by setting each visited node's parent directly to the root. Use union by rank to keep the tree balanced.

</details>

<details>
<summary>Solution</summary>

**Approach:** Standard DSU with path compression and union by rank.

```
function initialize(n):
    parent = [0, 1, 2, ..., n - 1]
    rank = [0, 0, ..., 0]   // length n

function find(x):
    if parent[x] != x:
        parent[x] = find(parent[x])   // path compression
    return parent[x]

function union(x, y):
    rootX = find(x)
    rootY = find(y)
    if rootX == rootY:
        return
    if rank[rootX] < rank[rootY]:
        parent[rootX] = rootY
    else if rank[rootX] > rank[rootY]:
        parent[rootY] = rootX
    else:
        parent[rootY] = rootX
        rank[rootX] += 1

function connected(x, y):
    return find(x) == find(y)
```

Amortized O(α(n)) per operation, where α is the inverse Ackermann function — effectively constant. O(n) space.

</details>


---

## Problem 2 — Number of Connected Components

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

You are given `n` nodes labeled `0` to `n - 1` and a list of undirected edges. Return the number of connected components in the graph.

### Example

**Input:**
```
n = 5
edges = [(0, 1), (1, 2), (3, 4)]
```
**Output:** `2`
**Explanation:** The connected components are {0, 1, 2} and {3, 4}.

**Input:**
```
n = 4
edges = [(0, 1), (2, 3), (1, 2)]
```
**Output:** `1`
**Explanation:** All nodes end up in a single component after the three edges are processed.

<details>
<summary>Hints</summary>

1. Start with `n` components (each node is its own set). Each successful union — one that actually merges two different sets — reduces the component count by one.
2. After processing all edges, the remaining count is your answer. Use Union Find with path compression and union by rank for efficiency.

</details>

<details>
<summary>Solution</summary>

**Approach:** Initialize DSU with `n` components, decrement the count on each successful merge.

```
function countComponents(n, edges):
    initialize DSU with n elements
    components = n

    for (u, v) in edges:
        rootU = find(u)
        rootV = find(v)
        if rootU != rootV:
            union(rootU, rootV)
            components -= 1

    return components
```

O(E × α(n)) time where E is the number of edges. O(n) space.

</details>


---

## Problem 3 — Redundant Connection

**Difficulty:** Hard
**Estimated Time:** 45 minutes

### Problem Statement

You are given a graph that was originally a tree of `n` nodes (labeled `1` to `n`) with one additional edge added. The graph is provided as a list of `n` edges. Return the edge that, if removed, would restore the graph to a tree. If there are multiple answers, return the edge that appears last in the input.

### Example

**Input:** `edges = [[1,2], [1,3], [2,3]]`
**Output:** `[2, 3]`
**Explanation:** The tree is 1–2 and 1–3. The edge [2,3] creates a cycle, so removing it restores the tree.

**Input:** `edges = [[1,2], [2,3], [3,4], [1,4], [1,5]]`
**Output:** `[1, 4]`
**Explanation:** Removing [1,4] breaks the cycle 1–2–3–4–1 and leaves a valid tree.

<details>
<summary>Hints</summary>

1. Process edges one by one. The first edge that connects two nodes already in the same component is the one that creates a cycle — and since you want the last such edge in the input, simply keep processing all edges and record the latest cycle-forming edge.
2. In this specific problem, there is exactly one extra edge, so the first edge whose endpoints are already connected is the answer.

</details>

<details>
<summary>Solution</summary>

**Approach:** Iterate through edges, using Union Find to detect the first edge that forms a cycle.

```
function findRedundantConnection(edges):
    n = length(edges)
    initialize DSU with n + 1 elements   // 1-indexed

    for [u, v] in edges:
        if find(u) == find(v):
            return [u, v]
        union(u, v)

    return []   // should not reach here given problem constraints
```

O(n × α(n)) time. O(n) space.

</details>


---

## Problem 4 — Accounts Merge

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Given a list of accounts where each account is a list of strings — the first element is the owner's name and the remaining elements are email addresses belonging to that account — merge accounts that share at least one common email. Two accounts belong to the same person if they share any email, even if the names differ in casing or other accounts act as intermediaries. Return the merged accounts, where each account's emails are sorted and duplicates removed. The order of accounts in the output does not matter.

### Example

**Input:**
```
accounts = [
  ["Alice", "alice@mail.com", "alice@work.com"],
  ["Bob",   "bob@mail.com"],
  ["Alice", "alice@work.com", "alice@school.com"],
  ["Alice", "alice@school.com", "alicenew@mail.com"]
]
```
**Output:**
```
[
  ["Alice", "alice@mail.com", "alice@school.com", "alice@work.com", "alicenew@mail.com"],
  ["Bob",   "bob@mail.com"]
]
```
**Explanation:** The first, third, and fourth accounts all share emails transitively, so they merge into one. Bob's account stands alone.

<details>
<summary>Hints</summary>

1. Assign each unique email an ID. For every account, union the first email's ID with every other email's ID in that account — this links all emails of the same person.
2. After all unions, group emails by their root representative. Attach the owner name from any account that contained an email in that group.

</details>

<details>
<summary>Solution</summary>

**Approach:** Map emails to integer IDs, union emails within each account, then group by root.

```
function accountsMerge(accounts):
    emailToId = {}
    emailToName = {}
    nextId = 0

    // Assign IDs and record owner names
    for account in accounts:
        name = account[0]
        for email in account[1:]:
            if email not in emailToId:
                emailToId[email] = nextId
                nextId += 1
            emailToName[email] = name

    // Initialize DSU with nextId elements
    initialize DSU with nextId elements

    // Union emails within each account
    for account in accounts:
        firstId = emailToId[account[1]]
        for email in account[2:]:
            union(firstId, emailToId[email])

    // Group emails by root
    groups = defaultdict(list)
    for email, id in emailToId:
        root = find(id)
        groups[root].append(email)

    // Build result
    result = []
    for root, emails in groups:
        sort(emails)
        name = emailToName[emails[0]]
        result.append([name] + emails)

    return result
```

O(E × α(E) + E log E) time where E is the total number of emails. O(E) space.

</details>


---

## Problem 5 — Number of Islands II (Online)

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

You are given an `m × n` grid initially filled with water (`0`). You receive a sequence of "add land" operations, each specifying a cell `(r, c)` to turn into land (`1`). After each operation, return the current number of islands. An island is a maximal group of land cells connected horizontally or vertically.

### Example

**Input:**
```
m = 3, n = 3
positions = [(0,0), (0,1), (1,2), (2,1), (1,1)]
```
**Output:** `[1, 1, 2, 3, 1]`
**Explanation:**
- Add (0,0): one island → `1`
- Add (0,1): merges with (0,0) → `1`
- Add (1,2): new isolated island → `2`
- Add (2,1): new isolated island → `3`
- Add (1,1): connects all three islands → `1`

<details>
<summary>Hints</summary>

1. Flatten the 2D coordinate `(r, c)` into a 1D index `r * n + c` to use as the Union Find element ID.
2. When a new land cell is added, start a new component (increment count), then check all four neighbors. For each neighbor that is already land, union the new cell with it — each successful merge decrements the count by one.
3. Handle duplicate positions: if a cell is already land, skip it and return the current count unchanged.

</details>

<details>
<summary>Solution</summary>

**Approach:** Maintain a DSU over grid cells. For each new land cell, create a component and merge with adjacent land.

```
function numIslands2(m, n, positions):
    parent = array of size m * n, initialized to -1   // -1 means water
    rank = array of size m * n, initialized to 0
    count = 0
    results = []
    directions = [(-1,0), (1,0), (0,-1), (0,1)]

    function find(x):
        if parent[x] != x:
            parent[x] = find(parent[x])
        return parent[x]

    function union(x, y):
        rootX = find(x)
        rootY = find(y)
        if rootX == rootY:
            return false
        if rank[rootX] < rank[rootY]:
            parent[rootX] = rootY
        else if rank[rootX] > rank[rootY]:
            parent[rootY] = rootX
        else:
            parent[rootY] = rootX
            rank[rootX] += 1
        return true

    for (r, c) in positions:
        id = r * n + c
        if parent[id] != -1:       // already land
            results.append(count)
            continue
        parent[id] = id
        count += 1
        for (dr, dc) in directions:
            nr = r + dr
            nc = c + dc
            nid = nr * n + nc
            if 0 <= nr < m and 0 <= nc < n and parent[nid] != -1:
                if union(id, nid):
                    count -= 1
        results.append(count)

    return results
```

O(P × α(m × n)) time where P is the number of positions. O(m × n) space.

</details>
