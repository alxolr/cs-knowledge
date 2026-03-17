# Heaps

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Intermediate
**Prerequisites:** Trees, Binary Search Trees

## Concept Overview

A heap is a specialized complete binary tree that satisfies the heap property: in a max-heap every parent is greater than or equal to its children, and in a min-heap every parent is less than or equal to its children. Because the tree is always complete, heaps are stored efficiently in a flat array — the children of the node at index `i` live at `2i + 1` and `2i + 2`, and its parent is at `floor((i - 1) / 2)`.

The two fundamental operations are insert (bubble up) and extract-min/max (bubble down), both running in O(log n) time. Building a heap from an unsorted array can be done in O(n) using the bottom-up heapify procedure, which is a key insight that separates heap construction from repeated insertion.

Heaps power priority queues, which appear everywhere: scheduling algorithms, graph shortest-path routines (Dijkstra), event-driven simulations, and streaming "top-K" problems. Understanding heaps also lays the groundwork for heap sort and for more advanced structures like Fibonacci heaps. The problems below explore insertion, extraction, streaming medians, K-way merging, and scheduling — each exercising a different facet of heap-based thinking.

### Core Operations — Pseudocode

A heap is stored as a flat array. For a node at index `i`: parent = `(i-1)/2`, left child = `2i+1`, right child = `2i+2`.

```
class MinHeap:
    data = []

    // Insert — O(log n): append to end, bubble up
    function insert(value):
        data.append(value)
        bubbleUp(length(data) - 1)

    function bubbleUp(index):
        while index > 0:
            parent = (index - 1) / 2
            if data[index] < data[parent]:
                swap(data[index], data[parent])
                index = parent
            else:
                break

    // Extract min — O(log n): swap root with last, remove last, bubble down
    function extractMin():
        if length(data) == 0: error "Heap is empty"
        min = data[0]
        data[0] = data[length(data) - 1]
        data.removeLast()
        bubbleDown(0)
        return min

    function bubbleDown(index):
        n = length(data)
        while true:
            smallest = index
            left = 2 * index + 1
            right = 2 * index + 2
            if left < n and data[left] < data[smallest]:
                smallest = left
            if right < n and data[right] < data[smallest]:
                smallest = right
            if smallest == index: break
            swap(data[index], data[smallest])
            index = smallest

    // Peek — O(1)
    function peekMin():
        return data[0]
```

Building a heap from an unsorted array can be done in O(n) using bottom-up heapify (start from the last non-leaf node and bubble down each one), which is faster than inserting elements one by one (O(n log n)).

---

## Problem 1 — Kth Largest Element in an Array

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given an unsorted integer array `nums` and an integer `k`, return the kth largest element in the array. The kth largest element is the element that would be in position `k` if the array were sorted in descending order. You must solve it without fully sorting the array.

### Example

**Input:** `nums = [3, 2, 1, 5, 6, 4]`, `k = 2`
**Output:** `5`
**Explanation:** Sorted in descending order the array is `[6, 5, 4, 3, 2, 1]`. The 2nd element is `5`.

**Input:** `nums = [3, 2, 3, 1, 2, 4, 5, 5, 6]`, `k = 4`
**Output:** `4`

<details>
<summary>Hints</summary>

1. A min-heap of size `k` always keeps the `k` largest elements seen so far. The root of that heap is the kth largest.

</details>

<details>
<summary>Solution</summary>

**Approach:** Maintain a min-heap of size k.

```
function kthLargest(nums, k):
    heap = new MinHeap()
    for num in nums:
        heap.insert(num)
        if heap.size() > k:
            heap.extractMin()
    return heap.peekMin()
```

Push every element onto a min-heap. Whenever the heap exceeds size `k`, remove the smallest. After processing all elements the root is the kth largest.

O(n log k) time, O(k) space.

</details>

---

## Problem 2 — Merge K Sorted Lists

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

You are given an array of `k` sorted linked lists. Merge all the lists into one sorted linked list and return it. Each input list is sorted in non-decreasing order.

### Example

**Input:** `lists = [[1, 4, 5], [1, 3, 4], [2, 6]]`
**Output:** `[1, 1, 2, 3, 4, 4, 5, 6]`
**Explanation:** Merging all three sorted lists into one sorted list produces the output above.

**Input:** `lists = [[], [1]]`
**Output:** `[1]`

<details>
<summary>Hints</summary>

1. Use a min-heap to always know which of the `k` list heads has the smallest value. Pop the smallest, append it to the result, and push that node's next element onto the heap.

</details>

<details>
<summary>Solution</summary>

**Approach:** K-way merge using a min-heap of size k.

```
function mergeKLists(lists):
    heap = new MinHeap()          // entries are (value, listIndex, nodePointer)
    for i from 0 to length(lists) - 1:
        if lists[i] is not empty:
            heap.insert((lists[i].val, i, lists[i]))
    dummy = new ListNode(0)
    tail = dummy
    while heap is not empty:
        (val, i, node) = heap.extractMin()
        tail.next = node
        tail = tail.next
        if node.next is not null:
            heap.insert((node.next.val, i, node.next))
    return dummy.next
```

Each of the N total nodes is inserted and extracted from the heap exactly once. Each heap operation is O(log k).

O(N log k) time, O(k) space.

</details>

---

## Problem 3 — Find Median from Data Stream

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a data structure that supports two operations on a stream of integers:

- `addNum(num)` — adds the integer `num` to the data structure.
- `findMedian()` — returns the median of all elements added so far. If the count is even, return the average of the two middle values.

### Example

**Input:**
```
addNum(1)
addNum(2)
findMedian()  -> 1.5
addNum(3)
findMedian()  -> 2
```
**Explanation:** After adding 1 and 2 the median is (1 + 2) / 2 = 1.5. After adding 3 the sorted sequence is [1, 2, 3] and the median is 2.

<details>
<summary>Hints</summary>

1. Split the stream into two halves: a max-heap for the lower half and a min-heap for the upper half. Keep their sizes balanced (differ by at most 1). The median is always at one or both of the heap roots.

</details>

<details>
<summary>Solution</summary>

**Approach:** Two-heap (max-heap + min-heap) partitioning.

```
class MedianFinder:
    maxHeapLow  = new MaxHeap()   // stores the smaller half
    minHeapHigh = new MinHeap()   // stores the larger half

    function addNum(num):
        maxHeapLow.insert(num)
        // ensure every element in low <= every element in high
        minHeapHigh.insert(maxHeapLow.extractMax())
        // rebalance so maxHeapLow has equal or one more element
        if minHeapHigh.size() > maxHeapLow.size():
            maxHeapLow.insert(minHeapHigh.extractMin())

    function findMedian():
        if maxHeapLow.size() > minHeapHigh.size():
            return maxHeapLow.peekMax()
        return (maxHeapLow.peekMax() + minHeapHigh.peekMin()) / 2.0
```

`addNum` is O(log n). `findMedian` is O(1). Space is O(n).

</details>

---

## Problem 4 — Task Scheduler

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

You are given an array of tasks represented by characters and a non-negative integer `n` representing the cooldown interval between two identical tasks. Each task takes one unit of time. In each unit of time you may either execute a task or idle. Return the minimum number of units of time the CPU will take to finish all the given tasks.

Tasks may be completed in any order, but identical tasks must be separated by at least `n` units of time.

### Example

**Input:** `tasks = ['A','A','A','B','B','B']`, `n = 2`
**Output:** `8`
**Explanation:** One valid schedule is `A B _ A B _ A B`. The CPU needs 8 units total.

**Input:** `tasks = ['A','A','A','B','B','B']`, `n = 0`
**Output:** `6`
**Explanation:** No cooldown required, so all 6 tasks run back-to-back.

<details>
<summary>Hints</summary>

1. Greedily schedule the most frequent remaining task first — this minimizes idle slots. A max-heap ordered by remaining count lets you always pick the highest-frequency task.
2. After executing a task, it enters a cooldown queue. Once the cooldown expires, push it back onto the heap.

</details>

<details>
<summary>Solution</summary>

**Approach:** Greedy scheduling with a max-heap and a cooldown queue.

```
function leastInterval(tasks, n):
    freq = countFrequencies(tasks)       // map char -> count
    maxHeap = new MaxHeap()
    for each count in freq.values():
        maxHeap.insert(count)

    time = 0
    cooldown = new Queue()               // entries: (remainingCount, availableAtTime)

    while maxHeap is not empty or cooldown is not empty:
        time += 1
        if maxHeap is not empty:
            cnt = maxHeap.extractMax() - 1
            if cnt > 0:
                cooldown.enqueue((cnt, time + n))
        if cooldown is not empty and cooldown.front().availableAtTime == time:
            cooldown.dequeue() into (cnt, _)
            maxHeap.insert(cnt)

    return time
```

Each task is inserted and extracted from the heap at most once per execution. Total operations are proportional to the answer length.

O(T · log U) time where T is the total number of task units (including idles) and U is the number of unique tasks. O(U) space.

</details>

---

## Problem 5 — Reorganize String

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Given a string `s`, rearrange the characters so that no two adjacent characters are the same. Return any valid rearrangement, or return an empty string if no valid rearrangement exists.

### Example

**Input:** `s = "aab"`
**Output:** `"aba"`
**Explanation:** Characters are rearranged so that no two adjacent characters are identical.

**Input:** `s = "aaab"`
**Output:** `""`
**Explanation:** No matter how you arrange the characters, two `'a'`s will always be adjacent.

<details>
<summary>Hints</summary>

1. If any character's frequency exceeds `ceil(length / 2)`, no valid arrangement exists.
2. Always place the most frequent remaining character next. A max-heap keyed by frequency lets you greedily pick the best candidate while holding back the character you just placed.

</details>

<details>
<summary>Solution</summary>

**Approach:** Greedy placement using a max-heap, holding back the previously placed character.

```
function reorganizeString(s):
    freq = countFrequencies(s)
    maxHeap = new MaxHeap()          // entries: (count, char)
    for (ch, count) in freq:
        if count > ceil(length(s) / 2):
            return ""
        maxHeap.insert((count, ch))

    result = []
    prev = null                      // (count, char) of the character just placed

    while maxHeap is not empty:
        (count, ch) = maxHeap.extractMax()
        result.append(ch)
        if prev is not null and prev.count > 0:
            maxHeap.insert(prev)
        prev = (count - 1, ch)

    return join(result)
```

After extracting the most frequent character, decrement its count and hold it aside as `prev`. On the next iteration, push `prev` back before extracting again. This guarantees no two adjacent characters are the same.

O(n log U) time where U is the number of unique characters. O(U) space for the heap.

</details>
