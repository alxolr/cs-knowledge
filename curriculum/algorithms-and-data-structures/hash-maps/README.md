# Hash Maps

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Beginner
**Prerequisites:** None

## Concept Overview

A hash map (also called a hash table or dictionary) is a data structure that maps keys to values using a hash function. The hash function converts a key into an array index, allowing average-case O(1) lookups, insertions, and deletions. When two keys hash to the same index (a collision), strategies like chaining or open addressing resolve the conflict.

Hash maps are one of the most frequently used data structures in practice. They power database indexing, caching, symbol tables in compilers, and countless algorithmic tricks where you need fast membership testing or frequency counting. Understanding how hash maps work — and their worst-case O(n) behavior under heavy collisions — is foundational knowledge for every programmer.

### Core Operations — Pseudocode

```
// Hash map with separate chaining
class HashMap:
    buckets = array of empty lists, size = capacity

    function hash(key):
        return hashCode(key) mod capacity

    // Put — O(1) average
    function put(key, value):
        index = hash(key)
        for each (k, v) in buckets[index]:
            if k == key:
                update v to value
                return
        buckets[index].append((key, value))

    // Get — O(1) average
    function get(key):
        index = hash(key)
        for each (k, v) in buckets[index]:
            if k == key:
                return v
        return null

    // Delete — O(1) average
    function delete(key):
        index = hash(key)
        remove entry with matching key from buckets[index]
```

---

## Problem 1 — Two Sum with Hash Map

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given an array of integers `nums` and an integer `target`, return the indices of the two elements that add up to `target`. Use a hash map for an efficient solution. Exactly one solution exists and you may not use the same element twice.

### Example

**Input:** `nums = [3, 2, 4]`, `target = 6`
**Output:** `[1, 2]`
**Explanation:** `nums[1] + nums[2] = 2 + 4 = 6`.

<details>
<summary>Hints</summary>

1. For each element, compute the complement (`target - element`) and check if it already exists in your hash map.

</details>

<details>
<summary>Solution</summary>

**Approach:** Single-pass hash map storing value → index.

```
function twoSum(nums, target):
    seen = empty hash map
    for i from 0 to length(nums) - 1:
        complement = target - nums[i]
        if complement in seen:
            return [seen[complement], i]
        seen[nums[i]] = i
```

O(n) time, O(n) space.

</details>

---

## Problem 2 — Count Character Frequencies

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given a string `s`, return a hash map where each key is a character in `s` and each value is the number of times that character appears.

### Example

**Input:** `s = "hello"`
**Output:** `{'h': 1, 'e': 1, 'l': 2, 'o': 1}`
**Explanation:** The character 'l' appears twice; all others appear once.

<details>
<summary>Hints</summary>

1. Iterate through the string and increment the count for each character in the map.

</details>

<details>
<summary>Solution</summary>

**Approach:** Single pass building a frequency map.

```
function charFrequency(s):
    freq = empty hash map
    for c in s:
        freq[c] = freq.getOrDefault(c, 0) + 1
    return freq
```

O(n) time, O(k) space where k is the number of distinct characters.

</details>

---

## Problem 3 — Check if Two Strings Are Anagrams

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given two strings `s` and `t`, determine if `t` is an anagram of `s`. An anagram uses exactly the same characters with the same frequencies.

### Example

**Input:** `s = "anagram"`, `t = "nagaram"`
**Output:** `true`
**Explanation:** Both strings contain the same characters with identical counts.

<details>
<summary>Hints</summary>

1. Build a frequency map for one string, then decrement counts using the other string. If all counts reach zero, the strings are anagrams.

</details>

<details>
<summary>Solution</summary>

**Approach:** Frequency map from `s`, then decrement with `t`.

```
function isAnagram(s, t):
    if length(s) != length(t): return false
    freq = empty hash map
    for c in s:
        freq[c] = freq.getOrDefault(c, 0) + 1
    for c in t:
        freq[c] = freq.getOrDefault(c, 0) - 1
        if freq[c] < 0: return false
    return true
```

O(n) time, O(k) space.

</details>

---

## Problem 4 — Group Anagrams

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Given an array of strings `strs`, group the anagrams together. You may return the groups in any order.

### Example

**Input:** `strs = ["eat", "tea", "tan", "ate", "nat", "bat"]`
**Output:** `[["eat", "tea", "ate"], ["tan", "nat"], ["bat"]]`
**Explanation:** Strings that are anagrams of each other are placed in the same group.

<details>
<summary>Hints</summary>

1. Two strings are anagrams if they produce the same key when their characters are sorted. Use that sorted string as a hash map key.

</details>

<details>
<summary>Solution</summary>

**Approach:** Group by sorted-character key in a hash map.

```
function groupAnagrams(strs):
    groups = empty hash map   // key: sorted string, value: list of strings
    for s in strs:
        key = sort(characters of s)
        groups[key].append(s)
    return all values of groups
```

O(n · k log k) time where k is the maximum string length.

</details>

---

## Problem 5 — Design a Hash Map from Scratch

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a hash map without using any built-in hash table libraries. Implement `put(key, value)`, `get(key)`, and `remove(key)`. Keys and values are integers. `get` should return `-1` if the key is not found.

### Example

**Input:** `put(1, 1)`, `put(2, 2)`, `get(1)` → `1`, `get(3)` → `-1`, `put(2, 1)`, `get(2)` → `1`, `remove(2)`, `get(2)` → `-1`
**Output:** Operations return the values shown above.
**Explanation:** The map stores, updates, retrieves, and deletes key-value pairs.

<details>
<summary>Hints</summary>

1. Choose a prime number for the bucket count to reduce collisions.
2. Use chaining (a linked list or dynamic array at each bucket) to handle collisions.

</details>

<details>
<summary>Solution</summary>

**Approach:** Array of N buckets with chaining; hash function is `key % N`.

```
class MyHashMap:
    N = 1009
    buckets = array of N empty lists

    function hash(key):
        return key % N

    function put(key, value):
        bucket = buckets[hash(key)]
        for pair in bucket:
            if pair.key == key:
                pair.value = value
                return
        bucket.append((key, value))

    function get(key):
        bucket = buckets[hash(key)]
        for pair in bucket:
            if pair.key == key:
                return pair.value
        return -1

    function remove(key):
        bucket = buckets[hash(key)]
        remove pair with pair.key == key from bucket
```

Average O(1) per operation with a good load factor.

</details>
