# Dynamic Programming

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Advanced
**Prerequisites:** [Recursion](../recursion/README.md), [Backtracking](../backtracking/README.md), [Divide and Conquer](../divide-and-conquer/README.md)

## Concept Overview

Dynamic programming (DP) is an optimization technique that solves complex problems by breaking them into overlapping sub-problems, solving each sub-problem only once, and storing its result for future reuse. Where plain recursion may recompute the same state exponentially many times, DP eliminates that redundancy through either memoization (top-down) or tabulation (bottom-up), reducing time complexity dramatically — often from exponential to polynomial.

The technique builds on recursion and divide and conquer: you still decompose a problem into smaller instances, but the key insight is that those instances overlap. Recognizing this overlap — and defining a clear recurrence relation that expresses the answer to a larger problem in terms of smaller ones — is the core skill of DP. Backtracking experience helps because many DP problems start as brute-force explorations of a decision space; DP then optimizes that exploration by caching results.

DP problems generally fall into several families: linear sequences (climbing stairs, house robber), grid paths, string matching (edit distance, longest common subsequence), knapsack variants, interval scheduling, and tree/graph DP. Mastering the pattern of identifying states, writing the recurrence, choosing top-down vs. bottom-up, and optimizing space usage is essential for tackling hard algorithmic challenges.

### Core Concepts

DP problems follow a consistent methodology:

1. **Define the state** — what sub-problem does `dp[i]` (or `dp[i][j]`) represent?
2. **Write the recurrence** — how does the answer to a larger state depend on smaller states?
3. **Identify base cases** — what are the trivially solvable states?
4. **Choose top-down vs. bottom-up** — memoization (recursive + cache) or tabulation (iterative)?
5. **Optimize space** — can you reduce from 2D to 1D, or from O(n) to O(1)?

| Approach | Style | Pros | Cons |
|---|---|---|---|
| Top-down (memoization) | Recursive + cache | Only computes needed states | Recursion overhead, stack depth |
| Bottom-up (tabulation) | Iterative | No recursion overhead, easier space optimization | Must determine computation order |

Common DP families: linear sequences (1D state), grid paths (2D state), string matching (two-string 2D state), knapsack variants (item × capacity), and interval DP. The key skill is recognizing overlapping sub-problems and defining a clear recurrence relation.

---

## Problem 1 — Climbing Stairs

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

You are climbing a staircase that has `n` steps. Each time you can climb either 1 or 2 steps. Return the number of distinct ways you can reach the top.

### Example

**Input:** `n = 5`
**Output:** `8`
**Explanation:** The 8 ways are: 11111, 1112, 1121, 1211, 2111, 122, 212, 221 (where digits represent step sizes).

<details>
<summary>Hints</summary>

1. The number of ways to reach step `i` depends only on how many ways you can reach step `i-1` (then take 1 step) and step `i-2` (then take 2 steps). Write the recurrence: `dp[i] = dp[i-1] + dp[i-2]`.

</details>

<details>
<summary>Solution</summary>

**Approach:** Bottom-up tabulation with constant space, recognizing the Fibonacci-like recurrence.

```
function climbStairs(n):
    if n <= 2:
        return n
    prev2 = 1   // ways to reach step 0
    prev1 = 2   // ways to reach step 1
    for i from 3 to n:
        current = prev1 + prev2
        prev2 = prev1
        prev1 = current
    return prev1
```

O(n) time, O(1) space.

</details>

---

## Problem 2 — Longest Increasing Subsequence

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Given an integer array `nums`, return the length of the longest strictly increasing subsequence. A subsequence is derived by deleting zero or more elements from the array without changing the order of the remaining elements.

### Example

**Input:** `nums = [10, 9, 2, 5, 3, 7, 101, 18]`
**Output:** `4`
**Explanation:** One longest increasing subsequence is [2, 3, 7, 101].

<details>
<summary>Hints</summary>

1. Define `dp[i]` as the length of the longest increasing subsequence ending at index `i`. For each `i`, look at all `j < i` where `nums[j] < nums[i]` and take the maximum `dp[j] + 1`.
2. For an O(n log n) approach, maintain a list `tails` where `tails[k]` is the smallest tail element of all increasing subsequences of length `k+1`. Use binary search to update it.

</details>

<details>
<summary>Solution</summary>

**Approach — O(n²) DP:**

```
function lengthOfLIS(nums):
    n = length(nums)
    dp = array of n, all initialized to 1
    for i from 1 to n - 1:
        for j from 0 to i - 1:
            if nums[j] < nums[i]:
                dp[i] = max(dp[i], dp[j] + 1)
    return max(dp)
```

**Approach — O(n log n) patience sorting:**

```
function lengthOfLIS(nums):
    tails = []
    for num in nums:
        pos = binarySearchLeftmost(tails, num)
        if pos == length(tails):
            tails.append(num)
        else:
            tails[pos] = num
    return length(tails)
```

O(n²) or O(n log n) time; O(n) space.

</details>

---

## Problem 3 — 0/1 Knapsack

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

You are given `n` items, each with a weight `weights[i]` and a value `values[i]`, and a knapsack with capacity `W`. Each item can be taken at most once. Return the maximum total value that fits within the capacity.

### Example

**Input:** `weights = [2, 3, 4, 5]`, `values = [3, 4, 5, 6]`, `W = 8`
**Output:** `10`
**Explanation:** Take items with weights 3 and 5 (values 4 + 6 = 10). Total weight 8 ≤ capacity 8.

<details>
<summary>Hints</summary>

1. Define `dp[i][w]` as the maximum value achievable using items `0..i-1` with capacity `w`. For each item you either skip it or take it (if it fits): `dp[i][w] = max(dp[i-1][w], dp[i-1][w - weights[i-1]] + values[i-1])`.
2. You can reduce space to a single 1-D array by iterating capacity in reverse order for each item.

</details>

<details>
<summary>Solution</summary>

**Approach:** Bottom-up DP with 1-D space optimization.

```
function knapsack(weights, values, W):
    n = length(weights)
    dp = array of (W + 1) zeros

    for i from 0 to n - 1:
        for w from W down to weights[i]:
            dp[w] = max(dp[w], dp[w - weights[i]] + values[i])

    return dp[W]
```

O(n × W) time, O(W) space.

</details>

---

## Problem 4 — Edit Distance

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Given two strings `word1` and `word2`, return the minimum number of operations required to convert `word1` into `word2`. You have three operations available: insert a character, delete a character, or replace a character.

### Example

**Input:** `word1 = "horse"`, `word2 = "ros"`
**Output:** `3`
**Explanation:** horse → rorse (replace 'h' with 'r') → rose (remove 'r') → ros (remove 'e').

<details>
<summary>Hints</summary>

1. Define `dp[i][j]` as the edit distance between `word1[0..i-1]` and `word2[0..j-1]`. If the last characters match, `dp[i][j] = dp[i-1][j-1]`. Otherwise, take the minimum of insert (`dp[i][j-1] + 1`), delete (`dp[i-1][j] + 1`), and replace (`dp[i-1][j-1] + 1`).
2. Base cases: `dp[i][0] = i` (delete all of word1) and `dp[0][j] = j` (insert all of word2).

</details>

<details>
<summary>Solution</summary>

**Approach:** Classic 2-D DP with optional space optimization to two rows.

```
function minDistance(word1, word2):
    m = length(word1)
    n = length(word2)
    // Use two rows for space optimization
    prev = [0, 1, 2, ..., n]   // base case: dp[0][j] = j
    curr = array of (n + 1) zeros

    for i from 1 to m:
        curr[0] = i             // base case: dp[i][0] = i
        for j from 1 to n:
            if word1[i-1] == word2[j-1]:
                curr[j] = prev[j-1]
            else:
                curr[j] = 1 + min(prev[j],      // delete
                                  curr[j-1],     // insert
                                  prev[j-1])     // replace
        prev = copy(curr)

    return prev[n]
```

O(m × n) time, O(n) space with the two-row optimization.

</details>

---

## Problem 5 — Longest Common Subsequence

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Given two strings `text1` and `text2`, return the length of their longest common subsequence. A subsequence is a sequence that can be derived from a string by deleting zero or more characters without changing the relative order of the remaining characters. If there is no common subsequence, return `0`.

### Example

**Input:** `text1 = "abcde"`, `text2 = "ace"`
**Output:** `3`
**Explanation:** The longest common subsequence is "ace", which has length 3.

<details>
<summary>Hints</summary>

1. Define `dp[i][j]` as the length of the LCS of `text1[0..i-1]` and `text2[0..j-1]`. If the characters match, `dp[i][j] = dp[i-1][j-1] + 1`. Otherwise, `dp[i][j] = max(dp[i-1][j], dp[i][j-1])`.
2. Space can be reduced to a single row by processing left-to-right and keeping a variable for the "diagonal" value.

</details>

<details>
<summary>Solution</summary>

**Approach:** Bottom-up DP with single-row space optimization.

```
function longestCommonSubsequence(text1, text2):
    m = length(text1)
    n = length(text2)
    dp = array of (n + 1) zeros

    for i from 1 to m:
        prev_diag = 0
        for j from 1 to n:
            temp = dp[j]
            if text1[i-1] == text2[j-1]:
                dp[j] = prev_diag + 1
            else:
                dp[j] = max(dp[j], dp[j-1])
            prev_diag = temp

    return dp[n]
```

O(m × n) time, O(n) space.

</details>
