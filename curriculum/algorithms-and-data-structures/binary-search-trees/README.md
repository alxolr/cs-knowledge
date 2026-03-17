# Binary Search Trees

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Intermediate
**Prerequisites:** [Binary Search](../binary-search/README.md), [Trees](../trees/README.md)

## Concept Overview

A binary search tree (BST) is a binary tree where every node's left subtree contains only values less than the node, and every right subtree contains only values greater. This ordering invariant turns the tree into a dynamic data structure that supports search, insertion, and deletion in O(h) time, where h is the tree's height.

When a BST is balanced, h = O(log n) and operations rival the speed of binary search on a sorted array — with the added benefit that insertions and deletions don't require shifting elements. When the tree degenerates into a chain (e.g., inserting sorted data without rebalancing), h = O(n) and performance drops to that of a linked list. Understanding this trade-off is the first step toward self-balancing variants like AVL trees and red-black trees.

BSTs are the backbone of ordered collections in standard libraries (e.g., `TreeMap` in Java, `std::set` in C++). Mastering them means you can reason about in-order traversals, predecessor/successor queries, range searches, and the structural properties that keep trees efficient. The problems below explore these facets from basic validation to advanced structural transformations.

### Core Operations

A BST node stores a value and pointers to left/right children. The BST invariant ensures: all left subtree values < node value < all right subtree values.

| Operation | Time (balanced) | Time (degenerate) | Description |
|---|---|---|---|
| Search | O(log n) | O(n) | Follow left/right based on comparison |
| Insert | O(log n) | O(n) | Search for position, create new leaf |
| Delete | O(log n) | O(n) | Three cases: no children, one child, two children (replace with in-order successor) |
| In-order traversal | O(n) | O(n) | Visits nodes in sorted order |

Key concepts: the in-order traversal of a BST produces a sorted sequence. When deleting a node with two children, replace it with its in-order successor (smallest node in right subtree) or predecessor (largest in left subtree). A degenerate BST (all nodes in a chain) has O(n) operations, motivating self-balancing variants like AVL and red-black trees.

---

## Problem 1 — Validate Binary Search Tree

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given the root of a binary tree, determine whether it is a valid binary search tree. A valid BST is defined as follows: for every node, all values in its left subtree are strictly less than the node's value, and all values in its right subtree are strictly greater. Each node's value is a unique integer.

### Example

**Input:**
```
    5
   / \
  3   7
 / \   \
1   4    9
```
**Output:** `true`
**Explanation:** Every node satisfies the BST property — left children are smaller, right children are larger.

**Input:**
```
    5
   / \
  3   7
 / \
1   6
```
**Output:** `false`
**Explanation:** Node 6 is in the left subtree of 5 but is greater than 5, violating the BST invariant.

<details>
<summary>Hints</summary>

1. A common mistake is only checking a node against its immediate parent. You need to enforce an allowable range (min, max) that tightens as you recurse deeper.

</details>

<details>
<summary>Solution</summary>

**Approach:** Recursive range validation — pass down the valid (min, max) interval for each node.

```
function isValidBST(node, min, max):
    if node is null:
        return true
    if node.val <= min or node.val >= max:
        return false
    return isValidBST(node.left, min, node.val)
       and isValidBST(node.right, node.val, max)

// Initial call: isValidBST(root, -infinity, +infinity)
```

O(n) time, O(h) space for the recursion stack.

</details>

---

## Problem 2 — Kth Smallest Element in a BST

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Given the root of a BST and an integer `k`, return the kth smallest value (1-indexed) among all nodes in the tree. You may assume that `k` is always valid (1 ≤ k ≤ number of nodes).

### Example

**Input:**
```
      5
     / \
    3    7
   / \
  2    4
 /
1
```
`k = 3`
**Output:** `3`
**Explanation:** The in-order traversal is [1, 2, 3, 4, 5, 7]. The 3rd smallest element is 3.

<details>
<summary>Hints</summary>

1. An in-order traversal of a BST visits nodes in ascending order. You don't need to collect all values — just count as you go and stop at the kth visit.

</details>

<details>
<summary>Solution</summary>

**Approach:** Iterative in-order traversal with an early exit after k nodes.

```
function kthSmallest(root, k):
    stack = []
    current = root
    count = 0
    while current is not null or stack is not empty:
        while current is not null:
            stack.push(current)
            current = current.left
        current = stack.pop()
        count += 1
        if count == k:
            return current.val
        current = current.right
```

O(h + k) time, O(h) space.

</details>

---

## Problem 3 — Lowest Common Ancestor in a BST

**Difficulty:** Medium
**Estimated Time:** 35 minutes

### Problem Statement

Given a BST and two nodes `p` and `q` that are guaranteed to exist in the tree, find their lowest common ancestor (LCA). The LCA is the deepest node that has both `p` and `q` as descendants (a node is allowed to be a descendant of itself).

### Example

**Input:**
```
        6
       / \
      2    8
     / \  / \
    0   4 7   9
       / \
      3   5
```
`p = 2`, `q = 8`
**Output:** `6`
**Explanation:** Node 6 is the root and is the first node that has both 2 and 8 in its subtrees.

**Input:** `p = 2`, `q = 4`
**Output:** `2`
**Explanation:** Node 2 is an ancestor of 4 and is itself one of the target nodes.

<details>
<summary>Hints</summary>

1. Exploit the BST ordering: if both values are less than the current node, the LCA must be in the left subtree; if both are greater, it must be in the right subtree. Otherwise the current node is the split point.

</details>

<details>
<summary>Solution</summary>

**Approach:** Iterative traversal using the BST property to choose a direction.

```
function lowestCommonAncestor(root, p, q):
    node = root
    while node is not null:
        if p.val < node.val and q.val < node.val:
            node = node.left
        else if p.val > node.val and q.val > node.val:
            node = node.right
        else:
            return node
```

O(h) time, O(1) space.

</details>

---

## Problem 4 — Delete Node in a BST

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Given the root of a BST and a key value, delete the node with that key and return the root of the modified tree. If the key does not exist, return the tree unchanged. The resulting tree must remain a valid BST. When a node with two children is deleted, replace it with its in-order successor (the smallest node in its right subtree).

### Example

**Input:**
```
      5
     / \
    3    7
   / \  / \
  2   4 6   8
```
`key = 5`
**Output:**
```
      6
     / \
    3    7
   / \    \
  2   4    8
```
**Explanation:** Node 5 has two children. Its in-order successor is 6. Replace 5 with 6 and remove the original 6 from the right subtree.

<details>
<summary>Hints</summary>

1. Handle three cases separately: the target node has no children (just remove it), one child (replace it with that child), or two children (find the in-order successor, copy its value up, then recursively delete the successor from the right subtree).
2. The in-order successor of a node is the leftmost node in its right subtree.

</details>

<details>
<summary>Solution</summary>

**Approach:** Recursive search and case-based deletion.

```
function deleteNode(root, key):
    if root is null:
        return null
    if key < root.val:
        root.left = deleteNode(root.left, key)
    else if key > root.val:
        root.right = deleteNode(root.right, key)
    else:
        // Found the node to delete
        if root.left is null:
            return root.right
        if root.right is null:
            return root.left
        // Two children: find in-order successor
        successor = root.right
        while successor.left is not null:
            successor = successor.left
        root.val = successor.val
        root.right = deleteNode(root.right, successor.val)
    return root
```

O(h) time, O(h) space for the recursion stack.

</details>

---

## Problem 5 — Convert Sorted Array to Balanced BST

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Given a sorted (ascending) integer array `nums` with unique elements, construct a height-balanced BST. A height-balanced BST is one in which the depth of the left and right subtrees of every node differs by at most 1.

### Example

**Input:** `nums = [-10, -3, 0, 5, 9]`
**Output (one valid tree):**
```
       0
      / \
    -3    9
    /    /
  -10   5
```
**Explanation:** Choosing the middle element as the root at each step produces a balanced tree. Other valid balanced BSTs exist.

<details>
<summary>Hints</summary>

1. Think recursively: the middle element of the current range becomes the root, the left half becomes the left subtree, and the right half becomes the right subtree. This mirrors how binary search divides an array.
2. Any consistent rule for picking the middle (lower-mid or upper-mid when the range has even length) will produce a valid balanced BST.

</details>

<details>
<summary>Solution</summary>

**Approach:** Divide-and-conquer — recursively pick the middle element as root.

```
function sortedArrayToBST(nums):
    return build(nums, 0, length(nums) - 1)

function build(nums, left, right):
    if left > right:
        return null
    mid = left + (right - left) / 2
    node = new TreeNode(nums[mid])
    node.left = build(nums, left, mid - 1)
    node.right = build(nums, mid + 1, right)
    return node
```

O(n) time, O(log n) space for the recursion stack (balanced tree height).

</details>