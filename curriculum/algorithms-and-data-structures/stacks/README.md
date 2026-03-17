# Stacks

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Beginner
**Prerequisites:** None

## Concept Overview

A stack is a linear data structure that follows the Last-In, First-Out (LIFO) principle: the most recently added element is the first one removed. The two core operations are `push` (add to the top) and `pop` (remove from the top), both running in O(1) time.

Stacks appear everywhere in computing — from the call stack that manages function invocations, to undo mechanisms in text editors, to expression parsing in compilers. Many algorithmic problems that involve matching, nesting, or backtracking can be elegantly solved with a stack. Building fluency with stack-based thinking is essential before tackling recursion and backtracking later in the curriculum.

### Core Operations — Pseudocode

```
// Stack backed by a dynamic array
class Stack:
    items = []

    // Push — O(1) amortized
    function push(value):
        items.append(value)

    // Pop — O(1)
    function pop():
        if isEmpty(): error "Stack underflow"
        return items.removeLast()

    // Peek — O(1)
    function peek():
        if isEmpty(): error "Stack is empty"
        return items[length(items) - 1]

    // isEmpty — O(1)
    function isEmpty():
        return length(items) == 0
```

---

## Problem 1 — Valid Parentheses

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given a string `s` containing only the characters `(`, `)`, `{`, `}`, `[`, and `]`, determine if the input string is valid. A string is valid if every open bracket is closed by the same type of bracket in the correct order.

### Example

**Input:** `s = "({[]})"`
**Output:** `true`
**Explanation:** Each opening bracket has a matching closing bracket in the correct nesting order.

<details>
<summary>Hints</summary>

1. Push each opening bracket onto a stack. When you encounter a closing bracket, check that the top of the stack is the matching opener.

</details>

<details>
<summary>Solution</summary>

**Approach:** Stack-based bracket matching with a closing→opening map.

```
function isValid(s):
    match = {')': '(', '}': '{', ']': '['}
    stack = []
    for char in s:
        if char in match.values():
            stack.push(char)
        else if char in match:
            if stack is empty or stack.pop() != match[char]:
                return false
    return stack is empty
```

O(n) time, O(n) space.

</details>

---

## Problem 2 — Implement a Stack Using an Array

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Implement a stack class that supports `push(val)`, `pop()`, `peek()`, and `isEmpty()` using a dynamic array (list) as the underlying storage. `pop()` and `peek()` should raise an error if the stack is empty.

### Example

**Input:** `push(1)`, `push(2)`, `peek()` → `2`, `pop()` → `2`, `isEmpty()` → `false`
**Output:** Operations return the values shown above.
**Explanation:** Elements are added and removed from the end of the internal array.

<details>
<summary>Hints</summary>

1. Use the end of the array as the "top" of the stack so that push and pop are both O(1) amortized.

</details>

<details>
<summary>Solution</summary>

**Approach:** Wrap a dynamic array; push/pop operate on the last element.

```
class Stack:
    data = []

    function push(val):
        data.append(val)

    function pop():
        if isEmpty(): raise error
        return data.removeLast()

    function peek():
        if isEmpty(): raise error
        return data[length(data) - 1]

    function isEmpty():
        return length(data) == 0
```

All operations O(1) amortized.

</details>

---

## Problem 3 — Evaluate Reverse Polish Notation

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Evaluate an arithmetic expression given in Reverse Polish Notation (postfix). The expression is provided as an array of strings where each element is either an integer or one of the operators `+`, `-`, `*`, `/`. Division truncates toward zero.

### Example

**Input:** `tokens = ["2", "1", "+", "3", "*"]`
**Output:** `9`
**Explanation:** `((2 + 1) * 3) = 9`.

<details>
<summary>Hints</summary>

1. Push numbers onto a stack. When you encounter an operator, pop two operands, apply the operator, and push the result back.

</details>

<details>
<summary>Solution</summary>

**Approach:** Stack-based evaluation — push operands, pop-and-apply on operators.

```
function evalRPN(tokens):
    stack = []
    for token in tokens:
        if token is a number:
            stack.push(toInteger(token))
        else:
            b = stack.pop()
            a = stack.pop()
            stack.push(apply(a, token, b))
    return stack.pop()
```

O(n) time, O(n) space.

</details>

---

## Problem 4 — Daily Temperatures

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Given an array of integers `temperatures` representing daily temperatures, return an array `answer` where `answer[i]` is the number of days you have to wait after day `i` to get a warmer temperature. If there is no future day with a warmer temperature, set `answer[i] = 0`.

### Example

**Input:** `temperatures = [73, 74, 75, 71, 69, 72, 76, 73]`
**Output:** `[1, 1, 4, 2, 1, 1, 0, 0]`
**Explanation:** On day 0 (73°), the next warmer day is day 1 (74°), so `answer[0] = 1`.

<details>
<summary>Hints</summary>

1. Use a monotonic decreasing stack that stores indices. When the current temperature is warmer than the temperature at the stack's top index, you have found the answer for that index.

</details>

<details>
<summary>Solution</summary>

**Approach:** Monotonic stack storing indices in decreasing temperature order.

```
function dailyTemperatures(temps):
    n = length(temps)
    answer = array of n zeroes
    stack = []                      // stores indices
    for i from 0 to n - 1:
        while stack is not empty and temps[i] > temps[stack.top()]:
            j = stack.pop()
            answer[j] = i - j
        stack.push(i)
    return answer
```

O(n) time, O(n) space.

</details>

---

## Problem 5 — Largest Rectangle in Histogram

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Given an array of non-negative integers `heights` representing the heights of bars in a histogram where each bar has width 1, find the area of the largest rectangle that can be formed within the histogram.

### Example

**Input:** `heights = [2, 1, 5, 6, 2, 3]`
**Output:** `10`
**Explanation:** The largest rectangle spans bars at indices 2 and 3 (heights 5 and 6), giving area = 5 × 2 = 10.

<details>
<summary>Hints</summary>

1. Use a stack to track bar indices in increasing order of height. When a bar shorter than the stack top is encountered, calculate the area that the popped bar could form.
2. Think about what the "width" of the rectangle is when you pop a bar — it extends from the current index back to the new stack top.

</details>

<details>
<summary>Solution</summary>

**Approach:** Monotonic increasing stack; on each pop compute the rectangle width using the new stack top as the left boundary.

```
function largestRectangle(heights):
    n = length(heights)
    stack = []
    max_area = 0
    for i from 0 to n - 1:
        while stack is not empty and heights[i] < heights[stack.top()]:
            h = heights[stack.pop()]
            width = i if stack is empty else i - stack.top() - 1
            max_area = max(max_area, h * width)
        stack.push(i)
    while stack is not empty:
        h = heights[stack.pop()]
        width = n if stack is empty else n - stack.top() - 1
        max_area = max(max_area, h * width)
    return max_area
```

O(n) time, O(n) space.

</details>
