# Trees

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Intermediate
**Prerequisites:** [Linked Lists](../linked-lists/README.md), [Recursion](../recursion/README.md)

## Concept Overview

A tree is a hierarchical data structure made up of nodes connected by edges, where each node has zero or more children and exactly one parent (except the root, which has none). Trees generalize linked lists — a linked list is simply a tree where every node has at most one child. This connection to linked lists makes pointer-based traversal feel familiar, while recursion provides the natural tool for processing a structure that is itself defined recursively.

The most common variant in interview and algorithm problems is the binary tree, where each node has at most two children (left and right). Binary trees support several traversal orders — pre-order, in-order, post-order (all depth-first), and level-order (breadth-first) — each revealing different structural information. Choosing the right traversal is often the key insight that unlocks a problem.

Beyond traversal, trees appear in problems involving depth calculation, path sums, subtree comparison, serialization, and reconstruction from traversal sequences. Mastering tree fundamentals here sets the stage for binary search trees, heaps, tries, and graph algorithms in later modules.

---

## Problem 1 — Maximum Depth of a Binary Tree

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given the root of a binary tree, return its maximum depth. The maximum depth is the number of nodes along the longest path from the root node down to the farthest leaf node.

### Example

**Input:**
```
        3
       / \
      9   20
         /  \
        15   7
```
**Output:** `3`
**Explanation:** The longest root-to-leaf path is 3 → 20 → 15 (or 3 → 20 → 7), which contains 3 nodes.

**Input:**
```
    1
     \
      2
```
**Output:** `2`

<details>
<summary>Hints</summary>

1. A leaf node has depth 1. The depth of any other node is 1 plus the maximum depth of its children. What does that suggest about the base case when the node is `null`?

</details>

<details>
<summary>Solution</summary>

**Approach:** Recursive depth-first traversal — compute depth bottom-up.

```
function maxDepth(node):
    if node is null:
        return 0
    leftDepth = maxDepth(node.left)
    rightDepth = maxDepth(node.right)
    return 1 + max(leftDepth, rightDepth)
```

O(n) time, O(h) space where h is the tree height (recursion stack).

</details>

---

## Problem 2 — Binary Tree Level-Order Traversal

**Difficulty:** Medium
**Estimated Time:** 35 minutes

### Problem Statement

Given the root of a binary tree, return the level-order traversal of its nodes' values — that is, the values grouped by level from left to right, top to bottom. Return the result as a list of lists, where each inner list contains the values at one level.

### Example

**Input:**
```
        3
       / \
      9   20
         /  \
        15   7
```
**Output:** `[[3], [9, 20], [15, 7]]`

**Input:**
```
    1
```
**Output:** `[[1]]`

<details>
<summary>Hints</summary>

1. Use a queue (BFS). At each step, record how many nodes are currently in the queue — that count is the size of the current level. Dequeue exactly that many nodes, collecting their values, and enqueue their children for the next level.

</details>

<details>
<summary>Solution</summary>

**Approach:** Breadth-first search with a queue, processing one level per iteration.

```
function levelOrder(root):
    if root is null: return []
    result = []
    queue = [root]
    while queue is not empty:
        levelSize = length(queue)
        currentLevel = []
        for i from 1 to levelSize:
            node = queue.dequeue()
            currentLevel.append(node.val)
            if node.left is not null: queue.enqueue(node.left)
            if node.right is not null: queue.enqueue(node.right)
        result.append(currentLevel)
    return result
```

O(n) time, O(n) space (the queue holds at most one full level).

</details>

---

## Problem 3 — Invert Binary Tree

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Given the root of a binary tree, invert the tree (mirror it) so that every node's left and right children are swapped, and return the root. The inversion must be applied recursively to every subtree.

### Example

**Input:**
```
        4
       / \
      2    7
     / \  / \
    1   3 6   9
```
**Output:**
```
        4
       / \
      7    2
     / \  / \
    9   6 3   1
```

**Input:**
```
    2
   / \
  1   3
```
**Output:**
```
    2
   / \
  3   1
```

<details>
<summary>Hints</summary>

1. Think recursively: if you invert the left subtree and invert the right subtree, then swap them, the whole tree is inverted. What is the base case?

</details>

<details>
<summary>Solution</summary>

**Approach:** Post-order recursive swap — invert children first, then swap them.

```
function invertTree(node):
    if node is null:
        return null
    invertTree(node.left)
    invertTree(node.right)
    swap(node.left, node.right)
    return node
```

O(n) time, O(h) space.

</details>

---

## Problem 4 — Lowest Common Ancestor of a Binary Tree

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Given the root of a binary tree and two nodes `p` and `q` that are guaranteed to exist in the tree, return their lowest common ancestor (LCA). The LCA of two nodes is the deepest node that is an ancestor of both `p` and `q` (a node is allowed to be an ancestor of itself).

### Example

**Input:**
```
            3
           / \
          5    1
         / \  / \
        6   2 0   8
           / \
          7   4
```
`p = 5`, `q = 1`
**Output:** `3`
**Explanation:** The LCA of nodes 5 and 1 is the root node 3.

**Input:** Same tree, `p = 5`, `q = 4`
**Output:** `5`
**Explanation:** Node 5 is an ancestor of node 4, and a node can be its own ancestor.

<details>
<summary>Hints</summary>

1. Recurse into both subtrees looking for `p` and `q`. If the left subtree returns a non-null result and the right subtree also returns a non-null result, the current node must be the LCA.
2. If only one side returns non-null, that side contains both targets — propagate that result upward.

</details>

<details>
<summary>Solution</summary>

**Approach:** Recursive post-order search — bubble up findings from left and right subtrees.

```
function lowestCommonAncestor(node, p, q):
    if node is null or node == p or node == q:
        return node
    left = lowestCommonAncestor(node.left, p, q)
    right = lowestCommonAncestor(node.right, p, q)
    if left is not null and right is not null:
        return node          // p and q are in different subtrees
    return left if left is not null else right
```

O(n) time, O(h) space.

</details>

---

## Problem 5 — Serialize and Deserialize a Binary Tree

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design an algorithm to serialize a binary tree into a string and deserialize that string back into the original tree. There is no restriction on how the serialization format works, as long as `deserialize(serialize(root))` produces a tree that is structurally identical to the original with the same node values.

### Example

**Input (serialize):**
```
        1
       / \
      2    3
          / \
         4   5
```
**Output (serialize):** `"1,2,#,#,3,4,#,#,5,#,#"` (one possible format using pre-order with `#` for null)

**Input (deserialize):** `"1,2,#,#,3,4,#,#,5,#,#"`
**Output (deserialize):** The tree shown above.

<details>
<summary>Hints</summary>

1. Pre-order traversal naturally records the root before its subtrees. If you also record null children as a sentinel (e.g., `#`), the sequence is unambiguous and can be deserialized with a simple recursive parser.
2. During deserialization, consume tokens one by one from the front. Each token either creates a node (and recurses for left and right children) or returns null when you hit the sentinel.

</details>

<details>
<summary>Solution</summary>

**Approach:** Pre-order traversal with null sentinels for serialization; recursive token consumption for deserialization.

```
function serialize(node):
    if node is null:
        return "#"
    return str(node.val) + "," + serialize(node.left) + "," + serialize(node.right)

function deserialize(data):
    tokens = data.split(",")
    index = 0

    function build():
        nonlocal index
        if tokens[index] == "#":
            index += 1
            return null
        node = new TreeNode(int(tokens[index]))
        index += 1
        node.left = build()
        node.right = build()
        return node

    return build()
```

Serialize: O(n) time, O(n) space (output string).
Deserialize: O(n) time, O(n) space (recursion + tree nodes).

</details>
