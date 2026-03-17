# Tries

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Intermediate
**Prerequisites:** [Hash Maps](../hash-maps/README.md), [Trees](../trees/README.md)

## Concept Overview

A trie (pronounced "try"), also called a prefix tree, is a tree-shaped data structure used to store and retrieve strings efficiently. Each node represents a single character, and paths from the root to marked nodes spell out the strings in the collection. Unlike a hash map that stores whole keys, a trie shares common prefixes among its entries, making prefix-based queries — autocomplete, spell-check, prefix counting — natural and fast.

The standard trie uses an array or hash map of children at each node, one slot per possible character. Inserting or searching a word of length `m` takes O(m) time regardless of how many words are stored, because you simply walk down one level per character. This prefix-sharing property also makes tries space-efficient when the dataset contains many words with overlapping beginnings (e.g., "car", "card", "care", "careful").

Beyond basic insert/search, tries support powerful operations: finding all words with a given prefix, counting how many words share a prefix, and lexicographic ordering of stored strings. Variants like compressed tries (radix trees) reduce node count by collapsing single-child chains, and ternary search tries trade branching factor for lower memory overhead. Mastering the basic trie is essential before tackling these optimizations and related structures like suffix trees.

### Core Concepts

A trie node holds a map of children (character → child node) and a boolean flag marking end-of-word. Paths from root to marked nodes spell out stored strings.

| Operation | Time | Description |
|---|---|---|
| Insert | O(m) | Walk/create nodes for each character, mark end |
| Search | O(m) | Walk nodes for each character, check end flag |
| Prefix check | O(m) | Walk nodes for each character (no end flag check needed) |
| Collect all with prefix | O(m + k) | Walk to prefix node, then DFS to collect k matches |

Where m = length of the word/prefix. Space is O(total characters across all inserted words) in the worst case, but prefix sharing makes tries space-efficient when words overlap heavily. Variants include compressed tries (radix trees) that collapse single-child chains, and ternary search tries that trade branching factor for lower memory.

---

## Problem 1 — Implement a Basic Trie

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design and implement a trie that supports the following operations:

- `insert(word)` — Insert a string `word` into the trie.
- `search(word)` — Return `true` if the exact string `word` exists in the trie, `false` otherwise.
- `startsWith(prefix)` — Return `true` if any previously inserted string starts with the given `prefix`, `false` otherwise.

All inputs consist of lowercase English letters only.

### Example

**Input:**
```
insert("apple")
search("apple")    → true
search("app")      → false
startsWith("app")  → true
insert("app")
search("app")      → true
```

**Output:** `true, false, true, true` (for the four queries in order)

**Explanation:** After inserting "apple", the full word "apple" is found but "app" is not marked as a complete word. The prefix "app" does exist. After inserting "app" as its own word, `search("app")` returns true.

<details>
<summary>Hints</summary>

1. Each trie node needs a collection of children (one per character) and a boolean flag indicating whether this node marks the end of a complete word.
2. For `insert`, walk character by character from the root, creating child nodes as needed, and mark the final node as a word end.
3. `search` and `startsWith` both walk the trie the same way — the only difference is that `search` also checks the end-of-word flag at the last node.

</details>

<details>
<summary>Solution</summary>

**Approach:** Build a tree where each node holds a map of character → child node and an `isEnd` flag.

```
class TrieNode:
    children = {}      // map: character → TrieNode
    isEnd = false

class Trie:
    root = new TrieNode()

    function insert(word):
        node = root
        for ch in word:
            if ch not in node.children:
                node.children[ch] = new TrieNode()
            node = node.children[ch]
        node.isEnd = true

    function search(word):
        node = findNode(word)
        return node != null and node.isEnd

    function startsWith(prefix):
        return findNode(prefix) != null

    function findNode(s):
        node = root
        for ch in s:
            if ch not in node.children:
                return null
            node = node.children[ch]
        return node
```

O(m) time per operation where m is the length of the input string. Space is O(total characters across all inserted words) in the worst case.

</details>


---

## Problem 2 — Count Words with a Given Prefix

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Design a data structure that supports:

- `insert(word)` — Insert a string into the collection.
- `countWordsEqualTo(word)` — Return the number of times the exact string `word` has been inserted.
- `countWordsStartingWith(prefix)` — Return the number of inserted strings that have `prefix` as a prefix.

Duplicate insertions of the same word should each be counted. All inputs consist of lowercase English letters.

### Example

**Input:**
```
insert("apple")
insert("apple")
insert("app")
countWordsEqualTo("apple")       → 2
countWordsStartingWith("app")    → 3
countWordsStartingWith("apple")  → 2
```

**Output:** `2, 3, 2`

**Explanation:** "apple" was inserted twice and "app" once. Three words start with "app" (both "apple" entries and "app"). Two words start with "apple" (the two "apple" entries).

<details>
<summary>Hints</summary>

1. Augment each trie node with two counters: one that tracks how many words pass through this node (`prefixCount`) and one that tracks how many words end exactly at this node (`wordCount`).
2. On insert, increment `prefixCount` at every node along the path and `wordCount` at the final node.

</details>

<details>
<summary>Solution</summary>

**Approach:** Augmented trie with per-node prefix and word counters.

```
class TrieNode:
    children = {}
    prefixCount = 0
    wordCount = 0

class CountingTrie:
    root = new TrieNode()

    function insert(word):
        node = root
        for ch in word:
            if ch not in node.children:
                node.children[ch] = new TrieNode()
            node = node.children[ch]
            node.prefixCount += 1
        node.wordCount += 1

    function countWordsEqualTo(word):
        node = root
        for ch in word:
            if ch not in node.children:
                return 0
            node = node.children[ch]
        return node.wordCount

    function countWordsStartingWith(prefix):
        node = root
        for ch in prefix:
            if ch not in node.children:
                return 0
            node = node.children[ch]
        return node.prefixCount
```

O(m) time per operation. Space is O(total characters inserted).

</details>


---

## Problem 3 — Replace Words with Shortest Root

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Given a list of root words `roots` and a sentence string, replace every word in the sentence with the shortest root that is a prefix of that word. If a word has no matching root, leave it unchanged. Return the modified sentence.

Words in the sentence are separated by single spaces. All characters are lowercase English letters.

### Example

**Input:** `roots = ["cat", "bat", "rat"]`, `sentence = "the cattle was rattled by the battery"`
**Output:** `"the cat was rat by the bat"`
**Explanation:** "cattle" → "cat" (shortest prefix root), "rattled" → "rat", "battery" → "bat". "the", "was", "by" have no matching root and stay unchanged.

**Input:** `roots = ["a", "aa", "aaa"]`, `sentence = "aadsfasf aadaadadaf"`
**Output:** `"a a"`
**Explanation:** The shortest root "a" is a prefix of both words.

<details>
<summary>Hints</summary>

1. Insert all roots into a trie. For each word in the sentence, walk the trie character by character. The first node you reach that is marked as a word end gives you the shortest matching root.
2. If you exhaust the word or hit a missing child before finding a word end, the original word has no matching root.

</details>

<details>
<summary>Solution</summary>

**Approach:** Build a trie from the root list, then scan each sentence word against the trie to find the shortest prefix match.

```
function replaceWords(roots, sentence):
    trie = new Trie()
    for root in roots:
        trie.insert(root)          // mark root's last node as isEnd

    words = sentence.split(" ")
    result = []
    for word in words:
        node = trie.root
        replacement = ""
        found = false
        for i from 0 to length(word) - 1:
            ch = word[i]
            if ch not in node.children:
                break
            node = node.children[ch]
            replacement += ch
            if node.isEnd:
                found = true
                break
        if found:
            result.append(replacement)
        else:
            result.append(word)
    return join(result, " ")
```

O(n · m) time where n is the number of words in the sentence and m is the average word length. Trie construction is O(sum of root lengths).

</details>


---

## Problem 4 — Design Add-and-Search with Wildcards

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Design a data structure that supports:

- `addWord(word)` — Add a word to the data structure.
- `search(pattern)` — Return `true` if any previously added word matches the pattern. The pattern may contain the wildcard character `'.'` which matches any single letter.

All non-wildcard characters are lowercase English letters. A pattern matches a word if they have the same length and every non-wildcard character matches exactly.

### Example

**Input:**
```
addWord("bad")
addWord("dad")
addWord("mad")
search("pad")   → false
search("bad")   → true
search(".ad")   → true
search("b..")   → true
search("b.d")   → true
search("...")   → true
search("..")    → false
```

**Output:** `false, true, true, true, true, true, false`

**Explanation:** "pad" was never added. ".ad" matches "bad", "dad", and "mad". "b.." matches "bad". ".." has length 2 and no added word has length 2.

<details>
<summary>Hints</summary>

1. `addWord` works exactly like a standard trie insert. The wildcard logic only affects `search`.
2. When `search` encounters a `'.'`, it must try all children of the current node recursively. If any branch leads to a match, return true.
3. Use depth-first search (backtracking) for the wildcard case. Without wildcards, the search is a simple linear walk.

</details>

<details>
<summary>Solution</summary>

**Approach:** Standard trie insert; recursive DFS search that branches on wildcard characters.

```
class WildcardTrie:
    root = new TrieNode()

    function addWord(word):
        node = root
        for ch in word:
            if ch not in node.children:
                node.children[ch] = new TrieNode()
            node = node.children[ch]
        node.isEnd = true

    function search(pattern):
        return dfs(root, pattern, 0)

    function dfs(node, pattern, index):
        if index == length(pattern):
            return node.isEnd
        ch = pattern[index]
        if ch == '.':
            for child in node.children.values():
                if dfs(child, pattern, index + 1):
                    return true
            return false
        else:
            if ch not in node.children:
                return false
            return dfs(node.children[ch], pattern, index + 1)
```

`addWord`: O(m) time. `search` without wildcards: O(m) time. `search` with wildcards: O(26^w · m) worst case where w is the number of wildcards, but typically much faster due to sparse branching.

</details>


---

## Problem 5 — Maximum XOR of Two Numbers

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Given an array of non-negative integers `nums`, find the maximum value of `nums[i] XOR nums[j]` for any pair `0 <= i < j < length(nums)`. Return this maximum XOR value.

### Example

**Input:** `nums = [3, 10, 5, 25, 2, 8]`
**Output:** `28`
**Explanation:** The maximum XOR is `5 XOR 25 = 28`. In binary: `00101 XOR 11001 = 11100 = 28`.

**Input:** `nums = [0]`
**Output:** `0`
**Explanation:** Only one element, so the only pair is the element with itself (or no valid pair). The maximum XOR is 0.

<details>
<summary>Hints</summary>

1. Build a bitwise trie where each number is inserted as a sequence of bits from the most significant bit (MSB) to the least significant bit (LSB). Use a fixed bit width (e.g., 31 bits for values up to 10^9).
2. For each number, greedily traverse the trie trying to take the opposite bit at each level. Choosing the opposite bit at each step maximizes the XOR.
3. The answer is the maximum XOR found across all numbers.

</details>

<details>
<summary>Solution</summary>

**Approach:** Insert each number's binary representation into a bitwise trie (MSB first). For each number, greedily walk the trie choosing the opposite bit at each level to maximize XOR.

```
class BitTrieNode:
    children = [null, null]   // index 0 for bit 0, index 1 for bit 1

function maximumXOR(nums):
    MAX_BIT = 30              // enough for values up to ~10^9
    root = new BitTrieNode()

    function insert(num):
        node = root
        for i from MAX_BIT down to 0:
            bit = (num >> i) & 1
            if node.children[bit] == null:
                node.children[bit] = new BitTrieNode()
            node = node.children[bit]

    function queryMaxXor(num):
        node = root
        xorVal = 0
        for i from MAX_BIT down to 0:
            bit = (num >> i) & 1
            opposite = 1 - bit
            if node.children[opposite] != null:
                xorVal |= (1 << i)
                node = node.children[opposite]
            else:
                node = node.children[bit]
        return xorVal

    // Insert all numbers
    for num in nums:
        insert(num)

    // Find maximum XOR
    maxResult = 0
    for num in nums:
        maxResult = max(maxResult, queryMaxXor(num))
    return maxResult
```

O(n · B) time and space where n is the number of elements and B is the bit width (constant, e.g., 31). Effectively O(n).

</details>
