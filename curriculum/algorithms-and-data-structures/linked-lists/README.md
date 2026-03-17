# Linked Lists

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Beginner
**Prerequisites:** None

## Concept Overview

A linked list is a linear data structure where each element (node) contains a value and a pointer to the next node. Unlike arrays, linked lists do not store elements contiguously in memory, which means insertions and deletions at arbitrary positions can be done in O(1) time if you already have a reference to the relevant node — but random access by index requires O(n) traversal.

Linked lists come in several flavors: singly linked (each node points to the next), doubly linked (each node points to both next and previous), and circular (the tail connects back to the head). Mastering linked list manipulation builds strong pointer-reasoning skills that transfer directly to trees, graphs, and many interview-style problems.

### Core Operations — Pseudocode

```
// Node structure
class Node:
    value
    next = null

// Traverse — O(n)
function traverse(head):
    current = head
    while current != null:
        visit(current.value)
        current = current.next

// Insert at head — O(1)
function insertAtHead(head, value):
    newNode = Node(value)
    newNode.next = head
    return newNode

// Insert at tail — O(n)
function insertAtTail(head, value):
    newNode = Node(value)
    if head == null: return newNode
    current = head
    while current.next != null:
        current = current.next
    current.next = newNode
    return head

// Delete first occurrence of value — O(n)
function delete(head, value):
    if head == null: return null
    if head.value == value: return head.next
    current = head
    while current.next != null:
        if current.next.value == value:
            current.next = current.next.next
            return head
        current = current.next
    return head
```

---

## Problem 1 — Traverse and Collect Values

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given the head of a singly linked list, return an array containing all node values in order from head to tail.

### Example

**Input:** `head = 1 -> 2 -> 3 -> 4 -> null`
**Output:** `[1, 2, 3, 4]`
**Explanation:** Walk the list from head to tail, collecting each value.

<details>
<summary>Hints</summary>

1. Start at the head and follow `.next` pointers until you reach `null`.

</details>

<details>
<summary>Solution</summary>

**Approach:** Linear traversal collecting values into a result array.

```
function collectValues(head):
    result = []
    current = head
    while current != null:
        result.append(current.val)
        current = current.next
    return result
```

O(n) time, O(n) space for the output.

</details>

---

## Problem 2 — Reverse a Singly Linked List

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given the head of a singly linked list, reverse the list in-place and return the new head.

### Example

**Input:** `head = 1 -> 2 -> 3 -> 4 -> 5 -> null`
**Output:** `5 -> 4 -> 3 -> 2 -> 1 -> null`
**Explanation:** Every pointer is flipped so the former tail becomes the new head.

<details>
<summary>Hints</summary>

1. Keep three pointers: `prev`, `current`, and `next`. At each step, redirect `current.next` to `prev`.

</details>

<details>
<summary>Solution</summary>

**Approach:** Iterative pointer reversal with prev/current/next.

```
function reverseList(head):
    prev = null
    current = head
    while current != null:
        next = current.next
        current.next = prev
        prev = current
        current = next
    return prev
```

O(n) time, O(1) space.

</details>

---

## Problem 3 — Detect a Cycle

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Given the head of a singly linked list, determine whether the list contains a cycle. A cycle exists if some node's `next` pointer points back to a previously visited node.

### Example

**Input:** `head = 3 -> 2 -> 0 -> -4 -> (back to node 2)`
**Output:** `true`
**Explanation:** Node -4 points back to node 2, forming a cycle.

<details>
<summary>Hints</summary>

1. Use two pointers moving at different speeds (Floyd's tortoise and hare). If they ever meet, there is a cycle.

</details>

<details>
<summary>Solution</summary>

**Approach:** Floyd's cycle detection — slow moves 1 step, fast moves 2 steps.

```
function hasCycle(head):
    slow = head
    fast = head
    while fast != null and fast.next != null:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            return true
    return false
```

O(n) time, O(1) space.

</details>

---

## Problem 4 — Remove the N-th Node from End

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Given the head of a singly linked list and an integer `n`, remove the n-th node from the end of the list and return the head. Assume `n` is always valid.

### Example

**Input:** `head = 1 -> 2 -> 3 -> 4 -> 5`, `n = 2`
**Output:** `1 -> 2 -> 3 -> 5`
**Explanation:** The 2nd node from the end is node 4, which is removed.

<details>
<summary>Hints</summary>

1. Use two pointers separated by a gap of `n` nodes. When the leading pointer reaches the end, the trailing pointer is just before the target node.

</details>

<details>
<summary>Solution</summary>

**Approach:** Two-pointer gap technique with a dummy head.

```
function removeNthFromEnd(head, n):
    dummy = new Node(0)
    dummy.next = head
    fast = dummy
    slow = dummy
    for i from 0 to n:
        fast = fast.next
    while fast != null:
        fast = fast.next
        slow = slow.next
    slow.next = slow.next.next
    return dummy.next
```

O(n) time, O(1) space.

</details>

---

## Problem 5 — Merge Two Sorted Linked Lists

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Given the heads of two sorted singly linked lists, merge them into a single sorted linked list and return its head. The merged list should be made by splicing together the nodes of the two input lists.

### Example

**Input:** `l1 = 1 -> 2 -> 4`, `l2 = 1 -> 3 -> 4`
**Output:** `1 -> 1 -> 2 -> 3 -> 4 -> 4`
**Explanation:** Nodes are interleaved in sorted order.

<details>
<summary>Hints</summary>

1. Use a dummy head node and a tail pointer. At each step, compare the current nodes of both lists and attach the smaller one to the tail.

</details>

<details>
<summary>Solution</summary>

**Approach:** Iterative merge with dummy node and tail pointer.

```
function mergeTwoLists(l1, l2):
    dummy = new Node(0)
    tail = dummy
    while l1 != null and l2 != null:
        if l1.val <= l2.val:
            tail.next = l1
            l1 = l1.next
        else:
            tail.next = l2
            l2 = l2.next
        tail = tail.next
    if l1 != null: tail.next = l1
    if l2 != null: tail.next = l2
    return dummy.next
```

O(n + m) time, O(1) space.

</details>
