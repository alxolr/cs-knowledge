# Recursion

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Beginner
**Prerequisites:** None

## Concept Overview

Recursion is a problem-solving technique where a function calls itself to solve smaller instances of the same problem. Every recursive solution has two essential parts: a base case that stops the recursion, and a recursive case that breaks the problem into a smaller sub-problem and combines the results.

Recursion is the foundation for many advanced algorithmic paradigms — divide and conquer, backtracking, dynamic programming, and tree/graph traversals all rely on recursive thinking. Understanding how the call stack works, how to identify overlapping sub-problems, and how to convert between recursive and iterative solutions are skills you will use throughout the rest of this curriculum.

### General Template — Pseudocode

```
// Every recursive function has two parts:
//   1. Base case — stops the recursion
//   2. Recursive case — reduces the problem and calls itself

function solve(problem):
    if problem is base case:
        return base solution
    smaller = reduce(problem)
    subResult = solve(smaller)
    return combine(problem, subResult)
```

---

## Problem 1 — Factorial

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given a non-negative integer `n`, compute `n!` (n factorial) using recursion. By definition, `0! = 1`.

### Example

**Input:** `n = 5`
**Output:** `120`
**Explanation:** `5! = 5 × 4 × 3 × 2 × 1 = 120`.

<details>
<summary>Hints</summary>

1. The base case is `n == 0` (return 1). The recursive case is `n * factorial(n - 1)`.

</details>

<details>
<summary>Solution</summary>

**Approach:** Direct recursion with base case n == 0.

```
function factorial(n):
    if n == 0: return 1
    return n * factorial(n - 1)
```

O(n) time, O(n) call-stack space.

</details>

---

## Problem 2 — Fibonacci Number

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given a non-negative integer `n`, return the n-th Fibonacci number. The Fibonacci sequence is defined as `F(0) = 0`, `F(1) = 1`, and `F(n) = F(n-1) + F(n-2)` for `n > 1`.

### Example

**Input:** `n = 6`
**Output:** `8`
**Explanation:** The sequence is 0, 1, 1, 2, 3, 5, 8 — the 6th element (0-indexed) is 8.

<details>
<summary>Hints</summary>

1. A naive recursive solution has exponential time complexity. Think about how memoization or an iterative approach can bring it down to O(n).

</details>

<details>
<summary>Solution</summary>

**Approach:** Recursion with memoization to avoid recomputation.

```
memo = empty hash map

function fib(n):
    if n == 0: return 0
    if n == 1: return 1
    if n in memo: return memo[n]
    memo[n] = fib(n - 1) + fib(n - 2)
    return memo[n]
```

O(n) time and O(n) space with memoization. Naive recursion is O(2^n).

</details>

---

## Problem 3 — Power Function (Exponentiation)

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Implement a function `power(base, exp)` that computes `base` raised to the power `exp` using recursion, where `exp` is a non-negative integer.

### Example

**Input:** `base = 2`, `exp = 10`
**Output:** `1024`
**Explanation:** `2^10 = 1024`.

<details>
<summary>Hints</summary>

1. Use the property that `base^exp = base^(exp/2) × base^(exp/2)` for even exponents, and `base × base^(exp-1)` for odd exponents. This gives O(log n) recursive calls.

</details>

<details>
<summary>Solution</summary>

**Approach:** Fast exponentiation — halve the exponent at each step.

```
function power(base, exp):
    if exp == 0: return 1
    if exp is even:
        half = power(base, exp / 2)
        return half * half
    else:
        return base * power(base, exp - 1)
```

O(log n) time and call-stack space.

</details>

---

## Problem 4 — Flatten a Nested List

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Given a nested list of integers (where each element is either an integer or a list of integers, possibly nested further), return a flat list of all integers in the order they appear.

### Example

**Input:** `nested = [1, [2, [3, 4], 5], 6]`
**Output:** `[1, 2, 3, 4, 5, 6]`
**Explanation:** All nested levels are flattened into a single list.

<details>
<summary>Hints</summary>

1. For each element, check if it is an integer or a list. If it is a list, recursively flatten it and concatenate the results.

</details>

<details>
<summary>Solution</summary>

**Approach:** Recursive traversal — append integers, recurse into sub-lists.

```
function flatten(nested):
    result = []
    for element in nested:
        if element is an integer:
            result.append(element)
        else:
            result.extend(flatten(element))
    return result
```

O(n) time where n is the total number of integers, O(d) call-stack space where d is the maximum nesting depth.

</details>

---

## Problem 5 — Generate All Permutations of a String

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Given a string `s` with all unique characters, return all possible permutations of its characters. Return the permutations in any order.

### Example

**Input:** `s = "abc"`
**Output:** `["abc", "acb", "bac", "bca", "cab", "cba"]`
**Explanation:** There are 3! = 6 permutations of three distinct characters.

<details>
<summary>Hints</summary>

1. Fix one character at each position and recursively permute the remaining characters.
2. Use a "swap" approach: for each index, swap it with every index from that position onward, recurse, then swap back (backtrack).

</details>

<details>
<summary>Solution</summary>

**Approach:** Swap-based backtracking — fix each character at the current position, recurse on the rest.

```
function permutations(s):
    result = []
    arr = toCharArray(s)
    permute(arr, 0, result)
    return result

function permute(arr, start, result):
    if start == length(arr):
        result.append(toString(arr))
        return
    for i from start to length(arr) - 1:
        swap(arr[start], arr[i])
        permute(arr, start + 1, result)
        swap(arr[start], arr[i])    // backtrack
```

O(n × n!) time and space to store all permutations.

</details>
