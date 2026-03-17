# Greedy Algorithms

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Advanced
**Prerequisites:** [Sorting Algorithms](../sorting-algorithms/README.md)

## Concept Overview

A greedy algorithm builds a solution piece by piece, always choosing the option that looks best at the current moment without reconsidering previous choices. At each step the algorithm makes a locally optimal decision and commits to it, hoping that the sequence of local optima leads to a globally optimal result. Unlike dynamic programming, a greedy approach never backtracks — once a choice is made, it is final.

Greedy algorithms work correctly only when the problem exhibits two properties: the greedy-choice property (a globally optimal solution can be assembled from locally optimal choices) and optimal substructure (an optimal solution to the problem contains optimal solutions to its sub-problems). Classic examples include Huffman coding, Kruskal's and Prim's minimum spanning tree algorithms, Dijkstra's shortest path, activity selection, and fractional knapsack. When these properties hold, greedy solutions are typically simpler and faster than their dynamic-programming counterparts.

The main challenge with greedy algorithms is proving correctness. A greedy strategy that seems intuitive can fail on subtle edge cases, so learning to construct exchange arguments or use matroids to justify a greedy choice is an important skill. Familiarity with sorting is essential because many greedy algorithms begin by sorting the input according to some criterion (deadline, weight, finish time, etc.) before making sequential choices.

### Core Concept

A greedy algorithm works correctly only when the problem exhibits two properties:

1. **Greedy-choice property** — a globally optimal solution can be assembled from locally optimal choices
2. **Optimal substructure** — an optimal solution contains optimal solutions to its sub-problems

The general pattern:
1. Sort the input by some criterion (deadline, weight, finish time, ratio, etc.)
2. Iterate through the sorted input, making the locally optimal choice at each step
3. Never reconsider previous choices

The main challenge is proving correctness — a greedy strategy that seems intuitive can fail on subtle edge cases. Learning to construct exchange arguments (showing that swapping any non-greedy choice for the greedy one doesn't worsen the solution) is an important skill. Classic greedy families include activity/interval scheduling, fractional optimization, and minimum spanning trees.

---

## Problem 1 — Assign Cookies

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

You are a parent with a set of children and a set of cookies. Each child `i` has a greed factor `g[i]`, which is the minimum cookie size that will make the child content. Each cookie `j` has a size `s[j]`. A cookie `j` can satisfy child `i` only if `s[j] >= g[i]`. Each child can receive at most one cookie, and each cookie can be given to at most one child. Return the maximum number of children you can make content.

### Example

**Input:** `g = [1, 2, 3]`, `s = [1, 1]`
**Output:** `1`
**Explanation:** You have 3 children with greed factors 1, 2, 3 and 2 cookies of size 1. Only the child with greed factor 1 can be satisfied. Output is 1.

**Input:** `g = [1, 2]`, `s = [1, 2, 3]`
**Output:** `2`
**Explanation:** Both children can be satisfied: give cookie of size 1 to child 1 and cookie of size 2 (or 3) to child 2.

<details>
<summary>Hints</summary>

1. Sort both the greed factors and the cookie sizes. Use two pointers: try to match the smallest available cookie to the least greedy unsatisfied child. If the current cookie is too small, move to the next larger cookie.

</details>

<details>
<summary>Solution</summary>

**Approach:** Sort both arrays and greedily assign the smallest sufficient cookie to the least greedy child.

```
function assignCookies(g, s):
    sort(g)
    sort(s)
    child = 0
    cookie = 0
    while child < length(g) and cookie < length(s):
        if s[cookie] >= g[child]:
            child++      // this child is satisfied
        cookie++         // move to next cookie either way
    return child
```

O(n log n + m log m) time for sorting, O(1) extra space (ignoring sort internals). The greedy choice — always use the smallest cookie that works — is optimal because wasting a large cookie on a less greedy child can never help.

</details>

---

## Problem 2 — Jump Game

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

You are given an integer array `nums` where each element represents the maximum jump length from that position. You start at index 0. Determine if you can reach the last index. Return `true` if you can reach the end, `false` otherwise.

### Example

**Input:** `nums = [2, 3, 1, 1, 4]`
**Output:** `true`
**Explanation:** Jump 1 step from index 0 to index 1, then 3 steps from index 1 to index 4 (the last index).

**Input:** `nums = [3, 2, 1, 0, 4]`
**Output:** `false`
**Explanation:** No matter what, you will always arrive at index 3 where the jump length is 0, so you cannot reach index 4.

<details>
<summary>Hints</summary>

1. Track the farthest index you can reach as you scan left to right. At each position, update the farthest reachable index. If you ever find that the current index exceeds the farthest reachable index, you are stuck.

</details>

<details>
<summary>Solution</summary>

**Approach:** Greedy scan maintaining the maximum reachable index.

```
function canJump(nums):
    farthest = 0
    for i from 0 to length(nums) - 1:
        if i > farthest:
            return false
        farthest = max(farthest, i + nums[i])
        if farthest >= length(nums) - 1:
            return true
    return true
```

O(n) time, O(1) space. The greedy insight is that you never need to track which specific path you take — only whether the farthest reachable position keeps advancing past each index.

</details>

---

## Problem 3 — Activity Selection (Weighted)

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

You are given `n` activities, each with a start time `start[i]`, an end time `end[i]`, and a weight (profit) `weight[i]`. Two activities are compatible if they do not overlap (one ends before or when the other starts). Find the maximum total weight you can achieve by selecting a set of non-overlapping activities.

Note: When all weights are equal this reduces to the classic activity selection problem. The weighted version requires combining greedy sorting with binary search.

### Example

**Input:**
```
start  = [1, 3, 0, 5, 8]
end    = [2, 4, 6, 7, 9]
weight = [5, 1, 8, 4, 2]
```
**Output:** `9`
**Explanation:** Select activities (1,2,w=5) and (5,7,w=4). They don't overlap and their total weight is 9. Selecting (0,6,w=8) alone gives only 8.

<details>
<summary>Hints</summary>

1. Sort activities by end time. For each activity, use binary search to find the latest non-overlapping previous activity. Build a DP table where `dp[i]` is the maximum weight considering the first `i` activities, choosing to include or exclude activity `i`.
2. The greedy component is the sorting by end time, which enables efficient binary search for compatible predecessors. The recurrence is `dp[i] = max(dp[i-1], weight[i] + dp[latestCompatible(i)])`.

</details>

<details>
<summary>Solution</summary>

**Approach:** Sort by end time, then use DP with binary search for the latest compatible activity.

```
function maxWeight(start, end, weight):
    n = length(start)
    activities = zip(start, end, weight)
    sort activities by end time

    // Binary search: find latest activity ending <= start of activity i
    function latestCompatible(i):
        lo = 0, hi = i - 1
        result = -1
        while lo <= hi:
            mid = (lo + hi) / 2
            if activities[mid].end <= activities[i].start:
                result = mid
                lo = mid + 1
            else:
                hi = mid - 1
        return result

    dp = array of size n
    dp[0] = activities[0].weight
    for i from 1 to n - 1:
        include = activities[i].weight
        p = latestCompatible(i)
        if p != -1:
            include += dp[p]
        dp[i] = max(dp[i - 1], include)
    return dp[n - 1]
```

O(n log n) time (sorting + binary search per activity), O(n) space for the DP table.

</details>

---

## Problem 4 — Minimum Number of Platforms

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Given arrival and departure times of `n` trains at a railway station, find the minimum number of platforms required so that no train has to wait. A platform is freed the moment a train departs (i.e., if one train departs at time `t` and another arrives at time `t`, they can share the same platform).

### Example

**Input:**
```
arrivals   = [900, 940, 950, 1100, 1500, 1800]
departures = [910, 1200, 1120, 1130, 1900, 2000]
```
**Output:** `3`
**Explanation:** Between times 950 and 1100, three trains are present simultaneously (trains arriving at 940, 950, and 1100 overlap with departures at 1200, 1120, and 1130). So 3 platforms are needed.

<details>
<summary>Hints</summary>

1. Sort the arrival times and departure times independently. Use two pointers to simulate a timeline: if the next event is an arrival, increment the platform count; if it is a departure, decrement it. Track the maximum count seen.
2. Think of it as a sweep-line algorithm. Each arrival is a +1 event and each departure is a −1 event. The answer is the peak of the running sum.

</details>

<details>
<summary>Solution</summary>

**Approach:** Sort arrivals and departures separately, then sweep through events with two pointers.

```
function minPlatforms(arrivals, departures):
    sort(arrivals)
    sort(departures)
    n = length(arrivals)
    platforms = 0
    maxPlatforms = 0
    i = 0   // pointer for arrivals
    j = 0   // pointer for departures

    while i < n:
        if arrivals[i] <= departures[j]:
            platforms++
            maxPlatforms = max(maxPlatforms, platforms)
            i++
        else:
            platforms--
            j++
    return maxPlatforms
```

O(n log n) time for sorting, O(1) extra space. The greedy insight is that processing events in chronological order and always handling arrivals before same-time departures (using `<=`) correctly counts the peak overlap.

</details>

---

## Problem 5 — Minimum Cost to Connect All Points (Prim's MST)

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

You are given `n` points in a 2D plane represented as `points[i] = (x_i, y_i)`. The cost of connecting two points `(x_i, y_i)` and `(x_j, y_j)` is the Manhattan distance: `|x_i - x_j| + |y_i - y_j|`. Return the minimum cost to connect all points such that there is exactly one path between every pair of points (i.e., find the minimum spanning tree cost).

### Example

**Input:** `points = [(0,0), (2,2), (3,10), (5,2), (7,0)]`
**Output:** `20`
**Explanation:** One optimal set of edges: (0,0)-(2,2) cost 4, (2,2)-(5,2) cost 3, (2,2)-(3,10) cost 9, (5,2)-(7,0) cost 4. Total = 20.

**Input:** `points = [(3,12), (-2,5), (-4,1)]`
**Output:** `18`
**Explanation:** Connect (-4,1) to (-2,5) with cost 6, then (-2,5) to (3,12) with cost 12. Total = 18.

<details>
<summary>Hints</summary>

1. Model the problem as a complete graph where each point is a node and edge weights are Manhattan distances. Then find the minimum spanning tree using Prim's or Kruskal's algorithm.
2. For Prim's approach: maintain a min-heap of (cost, node) pairs. Start from any node, and greedily add the cheapest edge that connects a new node to the growing MST. Track visited nodes to avoid cycles.

</details>

<details>
<summary>Solution</summary>

**Approach:** Prim's algorithm with a min-heap on the dense complete graph.

```
function minCostConnectPoints(points):
    n = length(points)
    visited = set()
    minHeap = [(0, 0)]   // (cost, pointIndex), start from point 0
    totalCost = 0

    while length(visited) < n:
        cost, u = heapPop(minHeap)
        if u in visited:
            continue
        visited.add(u)
        totalCost += cost

        for v from 0 to n - 1:
            if v not in visited:
                dist = |points[u].x - points[v].x| + |points[u].y - points[v].y|
                heapPush(minHeap, (dist, v))

    return totalCost
```

O(n² log n) time due to the complete graph having O(n²) edges and each heap operation costing O(log n). O(n²) space for the heap in the worst case. The greedy choice — always extending the MST with the cheapest available edge — is guaranteed optimal by the cut property of minimum spanning trees.

</details>
