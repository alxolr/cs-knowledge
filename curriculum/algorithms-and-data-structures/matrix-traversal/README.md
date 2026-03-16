# Matrix Traversal

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Intermediate
**Prerequisites:** [Arrays](../arrays/README.md)

## Concept Overview

Matrix traversal is the art of visiting elements in a two-dimensional grid in a controlled, purposeful order. While a flat array gives you a single dimension to walk, a matrix adds a second axis — and with it, a rich set of movement patterns: row-by-row, column-by-column, diagonal, spiral, or arbitrary neighbor-hopping via BFS and DFS.

Many classic interview and competition problems reduce to "explore a 2D grid under some constraint." Flood-fill, island counting, shortest path in a maze, and image rotation all rely on the same core skill: systematically visiting cells while tracking which ones you have already seen. The key decisions are traversal order (which cell next?), boundary handling (staying inside the grid), and state management (visited flags, distance counters, or auxiliary matrices).

Mastering matrix traversal builds directly on your array fundamentals and prepares you for graph algorithms, where a grid is simply a graph whose nodes are cells and whose edges connect neighbors.

---

## Problem 1 — Spiral Order

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given an `m × n` matrix of integers, return all elements of the matrix in spiral order — starting from the top-left corner, moving right across the top row, then down the right column, then left across the bottom row, then up the left column, and repeating inward until every element has been visited.

### Example

**Input:**
```
matrix = [
  [1, 2, 3],
  [4, 5, 6],
  [7, 8, 9]
]
```
**Output:** `[1, 2, 3, 6, 9, 8, 7, 4, 5]`
**Explanation:** Traverse right → down → left → up, then shrink the boundaries and repeat.

<details>
<summary>Hints</summary>

1. Maintain four boundary variables — `top`, `bottom`, `left`, `right` — and shrink them inward after completing each side of the spiral.

</details>

<details>
<summary>Solution</summary>

**Approach:** Layer-by-layer boundary shrinking.

```
function spiralOrder(matrix):
    result = []
    top = 0, bottom = rows - 1
    left = 0, right = cols - 1

    while top <= bottom and left <= right:
        // traverse right across top row
        for col from left to right:
            result.append(matrix[top][col])
        top += 1

        // traverse down right column
        for row from top to bottom:
            result.append(matrix[row][right])
        right -= 1

        // traverse left across bottom row (if still valid)
        if top <= bottom:
            for col from right down to left:
                result.append(matrix[bottom][col])
            bottom -= 1

        // traverse up left column (if still valid)
        if left <= right:
            for row from bottom down to top:
                result.append(matrix[row][left])
            left += 1

    return result
```

O(m·n) time, O(1) extra space (excluding output).

</details>


---

## Problem 2 — Rotate Image 90 Degrees

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Given an `n × n` matrix representing an image, rotate the image 90 degrees clockwise in-place. You must modify the input matrix directly — do not allocate a second matrix.

### Example

**Input:**
```
matrix = [
  [1, 2, 3],
  [4, 5, 6],
  [7, 8, 9]
]
```
**Output:**
```
[
  [7, 4, 1],
  [8, 5, 2],
  [9, 6, 3]
]
```
**Explanation:** Each element at position `(r, c)` moves to `(c, n-1-r)`.

<details>
<summary>Hints</summary>

1. A 90-degree clockwise rotation is equivalent to transposing the matrix (swap rows and columns) and then reversing each row.

</details>

<details>
<summary>Solution</summary>

**Approach:** Transpose then reverse each row — two simple in-place passes.

```
function rotate(matrix):
    n = length(matrix)

    // step 1: transpose
    for i from 0 to n - 1:
        for j from i + 1 to n - 1:
            swap(matrix[i][j], matrix[j][i])

    // step 2: reverse each row
    for i from 0 to n - 1:
        reverse(matrix[i])
```

O(n²) time, O(1) space.

</details>


---

## Problem 3 — Search a 2D Sorted Matrix

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

You are given an `m × n` matrix where each row is sorted in ascending order and the first element of each row is greater than the last element of the previous row. Given a target integer, determine whether it exists in the matrix. Your solution should run faster than O(m·n).

### Example

**Input:**
```
matrix = [
  [1,  3,  5,  7],
  [10, 11, 16, 20],
  [23, 30, 34, 60]
]
target = 3
```
**Output:** `true`
**Explanation:** 3 is found at row 0, column 1.

**Input:** same matrix, `target = 13`
**Output:** `false`

<details>
<summary>Hints</summary>

1. Because the rows are sorted and each row starts after the previous row ends, the entire matrix can be viewed as a single sorted array of length `m × n`. Apply binary search using index mapping: element at flat index `k` lives at `matrix[k / n][k % n]`.

</details>

<details>
<summary>Solution</summary>

**Approach:** Treat the matrix as a flat sorted array and binary search.

```
function searchMatrix(matrix, target):
    m = number of rows
    n = number of columns
    lo = 0, hi = m * n - 1

    while lo <= hi:
        mid = lo + (hi - lo) / 2
        val = matrix[mid / n][mid % n]
        if val == target:
            return true
        else if val < target:
            lo = mid + 1
        else:
            hi = mid - 1

    return false
```

O(log(m·n)) time, O(1) space.

</details>


---

## Problem 4 — Number of Islands

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Given an `m × n` grid of characters where `'1'` represents land and `'0'` represents water, count the number of islands. An island is a group of `'1'`s connected horizontally or vertically (not diagonally). You may assume all four edges of the grid are surrounded by water.

### Example

**Input:**
```
grid = [
  ['1','1','0','0','0'],
  ['1','1','0','0','0'],
  ['0','0','1','0','0'],
  ['0','0','0','1','1']
]
```
**Output:** `3`
**Explanation:** There are three connected components of land cells.

<details>
<summary>Hints</summary>

1. Scan the grid cell by cell. When you find an unvisited `'1'`, increment your island count and flood-fill (DFS or BFS) to mark every connected land cell as visited so you don't count it again.
2. You can mark cells visited by overwriting `'1'` with `'0'` (or using a separate visited matrix).

</details>

<details>
<summary>Solution</summary>

**Approach:** Iterate through every cell; on each unvisited land cell, run DFS to sink the entire island.

```
function numIslands(grid):
    count = 0
    for r from 0 to rows - 1:
        for c from 0 to cols - 1:
            if grid[r][c] == '1':
                count += 1
                dfs(grid, r, c)
    return count

function dfs(grid, r, c):
    if r < 0 or r >= rows or c < 0 or c >= cols:
        return
    if grid[r][c] != '1':
        return
    grid[r][c] = '0'          // mark visited
    dfs(grid, r - 1, c)       // up
    dfs(grid, r + 1, c)       // down
    dfs(grid, r, c - 1)       // left
    dfs(grid, r, c + 1)       // right
```

O(m·n) time, O(m·n) space in the worst case for the recursion stack.

</details>


---

## Problem 5 — Shortest Path in Binary Matrix

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Given an `n × n` binary grid where `0` represents an open cell and `1` represents a blocked cell, find the length of the shortest clear path from the top-left cell `(0, 0)` to the bottom-right cell `(n-1, n-1)`. A clear path consists of cells with value `0` connected by 8-directional moves (horizontal, vertical, and diagonal). The path length is the number of cells visited. If no such path exists, return `-1`.

### Example

**Input:**
```
grid = [
  [0, 1],
  [1, 0]
]
```
**Output:** `2`
**Explanation:** The path `(0,0) → (1,1)` uses a diagonal move and visits 2 cells.

**Input:**
```
grid = [
  [0, 0, 0],
  [1, 1, 0],
  [1, 1, 0]
]
```
**Output:** `4`
**Explanation:** The shortest path is `(0,0) → (0,1) → (1,2) → (2,2)`, using a diagonal from `(0,1)` to `(1,2)` and visiting 4 cells.

**Input:**
```
grid = [
  [1, 0],
  [0, 0]
]
```
**Output:** `-1`
**Explanation:** The start cell `(0,0)` is blocked, so no clear path exists.

<details>
<summary>Hints</summary>

1. BFS on a grid guarantees the shortest path when all moves have equal cost. Start BFS from `(0,0)` and explore all 8 neighbors at each step.
2. Track visited cells to avoid revisiting. The first time you reach `(n-1, n-1)` is the shortest path.

</details>

<details>
<summary>Solution</summary>

**Approach:** BFS from the top-left corner, exploring all 8 directions at each level.

```
function shortestPathBinaryMatrix(grid):
    n = length(grid)
    if grid[0][0] == 1 or grid[n-1][n-1] == 1:
        return -1

    directions = [(-1,-1),(-1,0),(-1,1),(0,-1),(0,1),(1,-1),(1,0),(1,1)]
    queue = [(0, 0, 1)]        // (row, col, path_length)
    grid[0][0] = 1             // mark visited

    while queue is not empty:
        (r, c, dist) = queue.dequeue()
        if r == n - 1 and c == n - 1:
            return dist
        for (dr, dc) in directions:
            nr, nc = r + dr, c + dc
            if 0 <= nr < n and 0 <= nc < n and grid[nr][nc] == 0:
                grid[nr][nc] = 1       // mark visited
                queue.enqueue((nr, nc, dist + 1))

    return -1
```

O(n²) time, O(n²) space for the queue.

</details>
