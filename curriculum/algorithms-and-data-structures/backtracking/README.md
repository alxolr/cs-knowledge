# Backtracking

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Advanced
**Prerequisites:** [Stacks](../stacks/README.md), [Recursion](../recursion/README.md)

## Concept Overview

Backtracking is a systematic method for exploring all possible configurations of a solution by building candidates incrementally and abandoning ("backtracking" from) a candidate as soon as it is clear that it cannot lead to a valid or optimal result. It is essentially a depth-first search over the space of potential solutions, enhanced with pruning to cut off branches that violate constraints.

The general pattern is: make a choice, recurse to extend the partial solution, then undo the choice and try the next option. This "choose → explore → un-choose" loop naturally maps onto a recursive call stack, which is why a solid understanding of recursion is a prerequisite. The stack-like nature of the exploration — pushing a decision, going deeper, then popping it — also connects directly to the stack data structure.

Backtracking appears in a wide range of classic problems: generating permutations and combinations, solving constraint-satisfaction puzzles (Sudoku, N-Queens), partitioning sets, and finding paths in grids or graphs. The key to writing efficient backtracking solutions is designing strong pruning conditions that eliminate invalid branches early, dramatically reducing the effective search space compared to brute-force enumeration.

### Core Template — Pseudocode

```
// General backtracking template
function backtrack(state, choices, result):
    if state is a complete solution:
        result.append(copy of state)
        return
    for each choice in choices:
        if choice is valid (pruning check):
            make choice (modify state)
            backtrack(state, remaining choices, result)
            undo choice (restore state)
```

The "choose → explore → un-choose" loop is the heart of backtracking. The key to efficiency is designing strong pruning conditions that eliminate invalid branches early, dramatically reducing the effective search space compared to brute-force enumeration. Common pruning strategies include: skipping duplicate values, checking constraints before recursing, and sorting input to enable early termination.

---

## Problem 1 — Generate All Subsets

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given an array `nums` of unique integers, return all possible subsets (the power set). The solution set must not contain duplicate subsets, and you may return the subsets in any order.

### Example

**Input:** `nums = [1, 2, 3]`
**Output:** `[[], [1], [2], [3], [1, 2], [1, 3], [2, 3], [1, 2, 3]]`
**Explanation:** There are 2³ = 8 subsets of a 3-element set, including the empty set and the full set.

<details>
<summary>Hints</summary>

1. At each index, you have two choices: include the current element in the subset or skip it. Recurse on the remaining elements after each choice.

</details>

<details>
<summary>Solution</summary>

**Approach:** Backtrack through the array, at each position choosing to include or exclude the element.

```
function subsets(nums):
    result = []
    current = []

    function backtrack(index):
        if index == length(nums):
            result.append(copy(current))
            return
        // Include nums[index]
        current.append(nums[index])
        backtrack(index + 1)
        current.pop()
        // Exclude nums[index]
        backtrack(index + 1)

    backtrack(0)
    return result
```

O(2ⁿ) time and space to store all subsets, where n = length(nums).

</details>


---

## Problem 2 — Permutations

**Difficulty:** Medium
**Estimated Time:** 35 minutes

### Problem Statement

Given an array `nums` of distinct integers, return all possible permutations. You may return the answer in any order.

### Example

**Input:** `nums = [1, 2, 3]`
**Output:** `[[1,2,3], [1,3,2], [2,1,3], [2,3,1], [3,1,2], [3,2,1]]`
**Explanation:** There are 3! = 6 permutations of three distinct elements.

<details>
<summary>Hints</summary>

1. At each level of recursion, pick one of the remaining unused elements to place at the current position. Use a "used" set or swap elements in-place to track which elements are still available.

</details>

<details>
<summary>Solution</summary>

**Approach:** Backtrack by swapping each remaining element into the current position.

```
function permutations(nums):
    result = []

    function backtrack(start):
        if start == length(nums):
            result.append(copy(nums))
            return
        for i from start to length(nums) - 1:
            swap(nums[start], nums[i])
            backtrack(start + 1)
            swap(nums[start], nums[i])   // undo swap

    backtrack(0)
    return result
```

O(n! × n) time, O(n) recursion depth.

</details>


---

## Problem 3 — N-Queens

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Place `n` queens on an `n × n` chessboard so that no two queens threaten each other. Two queens threaten each other if they share the same row, column, or diagonal. Return all distinct solutions, where each solution is represented as a list of strings: each string is a row of the board with `'Q'` for a queen and `'.'` for an empty square.

### Example

**Input:** `n = 4`
**Output:**
```
[[".Q..",
  "...Q",
  "Q...",
  "..Q."],
 ["..Q.",
  "Q...",
  "...Q",
  ".Q.."]]
```
**Explanation:** There are exactly two ways to place 4 non-attacking queens on a 4×4 board.

<details>
<summary>Hints</summary>

1. Place queens one row at a time. For each row, try every column and check whether the placement conflicts with queens already placed.
2. Track attacked columns, main diagonals (row − col), and anti-diagonals (row + col) in sets for O(1) conflict checks.

</details>

<details>
<summary>Solution</summary>

**Approach:** Row-by-row backtracking with three sets for conflict detection.

```
function solveNQueens(n):
    result = []
    queens = []          // queens[i] = column of queen in row i
    cols = set()
    diag1 = set()        // row - col
    diag2 = set()        // row + col

    function backtrack(row):
        if row == n:
            board = []
            for r from 0 to n - 1:
                line = "." repeated n times
                line[queens[r]] = 'Q'
                board.append(line)
            result.append(board)
            return
        for col from 0 to n - 1:
            if col in cols or (row - col) in diag1 or (row + col) in diag2:
                continue
            queens.append(col)
            cols.add(col)
            diag1.add(row - col)
            diag2.add(row + col)
            backtrack(row + 1)
            queens.pop()
            cols.remove(col)
            diag1.remove(row - col)
            diag2.remove(row + col)

    backtrack(0)
    return result
```

O(n!) time in the worst case (heavily pruned), O(n) space for recursion and tracking sets.

</details>


---

## Problem 4 — Sudoku Solver

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Write a program to solve a 9×9 Sudoku puzzle by filling the empty cells. Each empty cell is indicated by the character `'.'`. The solution must satisfy the standard Sudoku rules: each digit `1`–`9` must appear exactly once in each row, each column, and each of the nine 3×3 sub-boxes.

### Example

**Input:**
```
board = [
  ["5","3",".",".","7",".",".",".","."],
  ["6",".",".","1","9","5",".",".","."],
  [".","9","8",".",".",".",".","6","."],
  ["8",".",".",".","6",".",".",".","3"],
  ["4",".",".","8",".","3",".",".","1"],
  ["7",".",".",".","2",".",".",".","6"],
  [".","6",".",".",".",".","2","8","."],
  [".",".",".","4","1","9",".",".","5"],
  [".",".",".",".","8",".",".","7","9"]
]
```
**Output:** The completed board with all `'.'` cells filled so every row, column, and 3×3 box contains digits 1–9 exactly once.

<details>
<summary>Hints</summary>

1. Scan for the next empty cell, try digits 1–9, and recurse. If no digit works, backtrack to the previous cell.
2. Maintain sets for each row, column, and 3×3 box to check validity in O(1) before placing a digit.

</details>

<details>
<summary>Solution</summary>

**Approach:** Cell-by-cell backtracking with constraint sets for rows, columns, and boxes.

```
function solveSudoku(board):
    rows = array of 9 empty sets
    cols = array of 9 empty sets
    boxes = array of 9 empty sets

    // Initialize constraint sets from pre-filled cells
    for r from 0 to 8:
        for c from 0 to 8:
            if board[r][c] != '.':
                d = board[r][c]
                rows[r].add(d)
                cols[c].add(d)
                boxes[(r / 3) * 3 + c / 3].add(d)

    function backtrack(pos):
        if pos == 81:
            return true
        r = pos / 9
        c = pos % 9
        if board[r][c] != '.':
            return backtrack(pos + 1)
        box_id = (r / 3) * 3 + c / 3
        for d from '1' to '9':
            if d in rows[r] or d in cols[c] or d in boxes[box_id]:
                continue
            board[r][c] = d
            rows[r].add(d)
            cols[c].add(d)
            boxes[box_id].add(d)
            if backtrack(pos + 1):
                return true
            board[r][c] = '.'
            rows[r].remove(d)
            cols[c].remove(d)
            boxes[box_id].remove(d)
        return false

    backtrack(0)
```

O(9^m) worst case where m is the number of empty cells, but pruning makes it practical. O(m) recursion depth.

</details>


---

## Problem 5 — Partition to K Equal Sum Subsets

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Given an integer array `nums` and an integer `k`, return `true` if it is possible to divide the array into `k` non-empty subsets whose sums are all equal, and `false` otherwise.

### Example

**Input:** `nums = [4, 3, 2, 3, 5, 2, 1]`, `k = 4`
**Output:** `true`
**Explanation:** The total sum is 20, so each subset must sum to 5. One valid partition: {5}, {4, 1}, {3, 2}, {3, 2}.

**Input:** `nums = [1, 2, 3, 4]`, `k = 3`
**Output:** `false`
**Explanation:** The total sum is 10, which is not divisible by 3.

<details>
<summary>Hints</summary>

1. If the total sum is not divisible by `k`, return false immediately. The target sum per subset is `total / k`.
2. Sort `nums` in descending order before backtracking — larger elements are harder to place, so trying them first prunes failures earlier.
3. Track the current sum of each of the `k` buckets. For each number, try placing it in a bucket that would not exceed the target. Skip buckets that have the same current sum as a previously tried bucket to avoid redundant work.

</details>

<details>
<summary>Solution</summary>

**Approach:** Backtracking over bucket assignments with sorting and symmetry-breaking pruning.

```
function canPartitionKSubsets(nums, k):
    total = sum(nums)
    if total % k != 0:
        return false
    target = total / k
    sort(nums, descending)
    if nums[0] > target:
        return false
    buckets = array of k zeros

    function backtrack(index):
        if index == length(nums):
            return true
        seen = set()
        for i from 0 to k - 1:
            if buckets[i] + nums[index] > target:
                continue
            if buckets[i] in seen:
                continue          // skip duplicate bucket states
            seen.add(buckets[i])
            buckets[i] += nums[index]
            if backtrack(index + 1):
                return true
            buckets[i] -= nums[index]
        return false

    return backtrack(0)
```

Worst case exponential, but pruning (sorting, duplicate-state skipping) makes it efficient for typical inputs. O(n) space for buckets and recursion.

</details>
