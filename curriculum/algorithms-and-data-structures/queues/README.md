# Queues

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Beginner
**Prerequisites:** None

## Concept Overview

A queue is a linear data structure that follows the First-In, First-Out (FIFO) principle: the element added earliest is the first one removed. The two core operations are `enqueue` (add to the back) and `dequeue` (remove from the front), both typically running in O(1) time.

Queues model real-world scenarios like task scheduling, print job management, and breadth-first traversal of graphs and trees. Variations such as circular queues, double-ended queues (deques), and priority queues extend the basic idea to cover a wide range of algorithmic patterns. Building comfort with queue-based reasoning is essential preparation for graph algorithms later in the curriculum.

### Core Operations — Pseudocode

```
// Queue backed by a linked list (O(1) enqueue and dequeue)
class Queue:
    head = null
    tail = null

    // Enqueue — O(1)
    function enqueue(value):
        newNode = Node(value)
        if tail != null:
            tail.next = newNode
        tail = newNode
        if head == null:
            head = newNode

    // Dequeue — O(1)
    function dequeue():
        if head == null: error "Queue underflow"
        value = head.value
        head = head.next
        if head == null:
            tail = null
        return value

    // Peek — O(1)
    function peek():
        if head == null: error "Queue is empty"
        return head.value

    // isEmpty — O(1)
    function isEmpty():
        return head == null
```

---

## Problem 1 — Implement a Queue Using an Array

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Implement a queue class that supports `enqueue(val)`, `dequeue()`, `front()`, and `isEmpty()` using a list as the underlying storage. `dequeue()` and `front()` should raise an error if the queue is empty.

### Example

**Input:** `enqueue(1)`, `enqueue(2)`, `front()` → `1`, `dequeue()` → `1`, `isEmpty()` → `false`
**Output:** Operations return the values shown above.
**Explanation:** Elements leave in the same order they entered.

<details>
<summary>Hints</summary>

1. Track a front index instead of shifting the entire array on each dequeue, or use a linked-list-backed approach for true O(1) operations.

</details>

<details>
<summary>Solution</summary>

**Approach:** Array with a front-index pointer for amortized O(1) dequeue.

```
class Queue:
    data = []
    front_index = 0

    function enqueue(val):
        data.append(val)

    function dequeue():
        if isEmpty(): raise error
        val = data[front_index]
        front_index += 1
        return val

    function front():
        if isEmpty(): raise error
        return data[front_index]

    function isEmpty():
        return front_index >= length(data)
```

Amortized O(1) per operation.

</details>

---

## Problem 2 — Number of Recent Calls

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Implement a `RecentCounter` class that counts the number of requests received in the last 3000 milliseconds. It supports a single method `ping(t)` where `t` is the timestamp of a new request (in milliseconds). Return the number of requests that occurred in the inclusive range `[t - 3000, t]`. Calls to `ping` are guaranteed to have strictly increasing `t` values.

### Example

**Input:** `ping(1)` → `1`, `ping(100)` → `2`, `ping(3001)` → `3`, `ping(3002)` → `3`
**Output:** Counts shown above.
**Explanation:** At `t = 3002`, the valid window is `[2, 3002]`, which includes timestamps 100, 3001, and 3002.

<details>
<summary>Hints</summary>

1. Use a queue to store timestamps. On each `ping`, remove timestamps from the front that fall outside the window.

</details>

<details>
<summary>Solution</summary>

**Approach:** Queue that evicts expired timestamps on each ping.

```
class RecentCounter:
    queue = []

    function ping(t):
        queue.enqueue(t)
        while queue.front() < t - 3000:
            queue.dequeue()
        return length(queue)
```

Amortized O(1) per call.

</details>

---

## Problem 3 — Implement a Queue Using Two Stacks

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Implement a FIFO queue using only two stacks. The queue should support `enqueue(val)`, `dequeue()`, `front()`, and `isEmpty()`.

### Example

**Input:** `enqueue(1)`, `enqueue(2)`, `dequeue()` → `1`, `front()` → `2`
**Output:** Operations return the values shown above.
**Explanation:** Despite using LIFO stacks internally, the external behavior is FIFO.

<details>
<summary>Hints</summary>

1. Use one stack for incoming elements and another for outgoing. Transfer elements from the incoming stack to the outgoing stack (reversing order) only when the outgoing stack is empty.

</details>

<details>
<summary>Solution</summary>

**Approach:** Lazy transfer between two stacks — push to `stack_in`, pop from `stack_out`.

```
class QueueFromStacks:
    stack_in = []
    stack_out = []

    function enqueue(val):
        stack_in.push(val)

    function transfer():
        if stack_out is empty:
            while stack_in is not empty:
                stack_out.push(stack_in.pop())

    function dequeue():
        transfer()
        return stack_out.pop()

    function front():
        transfer()
        return stack_out.top()

    function isEmpty():
        return stack_in is empty and stack_out is empty
```

Amortized O(1) per operation.

</details>

---

## Problem 4 — Design a Circular Queue

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a circular queue with a fixed capacity `k`. Implement `enqueue(val)`, `dequeue()`, `front()`, `rear()`, `isEmpty()`, and `isFull()`. All operations should run in O(1) time.

### Example

**Input:** `CircularQueue(3)`, `enqueue(1)` → `true`, `enqueue(2)` → `true`, `enqueue(3)` → `true`, `enqueue(4)` → `false`, `rear()` → `3`, `isFull()` → `true`, `dequeue()` → `true`, `front()` → `2`
**Output:** Operations return the values shown above.
**Explanation:** The queue wraps around a fixed-size array, reusing slots freed by dequeue.

<details>
<summary>Hints</summary>

1. Use a fixed-size array and two pointers (`head`, `tail`) with modular arithmetic to wrap around.
2. Track the current count to distinguish between empty and full states.

</details>

<details>
<summary>Solution</summary>

**Approach:** Fixed array with head/tail pointers and modular arithmetic.

```
class CircularQueue:
    arr = array of size k
    head = 0
    tail = -1
    count = 0

    function enqueue(val):
        if isFull(): return false
        tail = (tail + 1) % k
        arr[tail] = val
        count += 1
        return true

    function dequeue():
        if isEmpty(): return false
        head = (head + 1) % k
        count -= 1
        return true

    function front():  return arr[head]
    function rear():   return arr[tail]
    function isEmpty(): return count == 0
    function isFull():  return count == k
```

All operations O(1).

</details>

---

## Problem 5 — Sliding Window Maximum

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Given an array of integers `nums` and an integer `k`, return an array of the maximum value in each sliding window of size `k` as it moves from left to right across the array.

### Example

**Input:** `nums = [1, 3, -1, -3, 5, 3, 6, 7]`, `k = 3`
**Output:** `[3, 3, 5, 5, 6, 7]`
**Explanation:** Window `[1,3,-1]` → max 3; window `[3,-1,-3]` → max 3; and so on.

<details>
<summary>Hints</summary>

1. A brute-force approach checks all `k` elements per window (O(nk)). You can do better with a deque that maintains indices of useful elements in decreasing order of value.
2. Before adding a new index, remove all indices from the back of the deque whose corresponding values are less than or equal to the new value.

</details>

<details>
<summary>Solution</summary>

**Approach:** Monotonic decreasing deque storing indices; front always holds the current window max.

```
function maxSlidingWindow(nums, k):
    n = length(nums)
    dq = empty deque       // stores indices
    result = []
    for i from 0 to n - 1:
        // remove indices outside the window
        while dq is not empty and dq.front() <= i - k:
            dq.popFront()
        // remove smaller elements from back
        while dq is not empty and nums[dq.back()] <= nums[i]:
            dq.popBack()
        dq.pushBack(i)
        if i >= k - 1:
            result.append(nums[dq.front()])
    return result
```

O(n) time, O(k) space.

</details>
