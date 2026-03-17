# String Manipulation

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Beginner
**Prerequisites:** Arrays

## Concept Overview

Strings are sequences of characters and, in most languages, are stored as arrays of characters under the hood. String manipulation problems test your ability to traverse, transform, compare, and search within character sequences. Common techniques include two-pointer scanning, sliding windows over characters, frequency counting, and in-place reversal.

Because strings are so pervasive — from parsing user input to processing log files — fluency with string operations is a must. Many string problems are really array problems in disguise, which is why Arrays is a prerequisite for this module. The key difference is that strings often involve character-specific logic such as case conversion, Unicode handling, and pattern matching.

### Core Operations — Pseudocode

```
// Character frequency count — O(n)
function charFrequency(s):
    freq = empty map
    for each char in s:
        freq[char] = freq.getOrDefault(char, 0) + 1
    return freq

// Simple substring search (brute force) — O(n × m)
function findSubstring(text, pattern):
    n = length(text)
    m = length(pattern)
    for i from 0 to n - m:
        match = true
        for j from 0 to m - 1:
            if text[i + j] != pattern[j]:
                match = false; break
        if match: return i
    return -1
```

---

## Problem 1 — Reverse a String In-Place

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given a character array `s`, reverse it in-place. Do not allocate a new array.

### Example

**Input:** `s = ['h', 'e', 'l', 'l', 'o']`
**Output:** `['o', 'l', 'l', 'e', 'h']`
**Explanation:** The first and last characters swap, then the second and second-to-last, and so on.

<details>
<summary>Hints</summary>

1. Use two pointers starting at opposite ends and swap characters while moving inward.

</details>

<details>
<summary>Solution</summary>

**Approach:** Two-pointer swap from both ends.

```
function reverseString(s):
    left = 0
    right = length(s) - 1
    while left < right:
        swap(s[left], s[right])
        left += 1
        right -= 1
```

O(n) time, O(1) space.

</details>

---

## Problem 2 — Check if a String Is a Palindrome

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given a string `s`, determine if it is a palindrome considering only alphanumeric characters and ignoring case.

### Example

**Input:** `s = "A man, a plan, a canal: Panama"`
**Output:** `true`
**Explanation:** After removing non-alphanumeric characters and lowering case, the string reads "amanaplanacanalpanama", which is the same forwards and backwards.

<details>
<summary>Hints</summary>

1. Use two pointers from each end, skipping non-alphanumeric characters and comparing in a case-insensitive manner.

</details>

<details>
<summary>Solution</summary>

**Approach:** Two pointers skipping non-alphanumeric, comparing lowercase.

```
function isPalindrome(s):
    left = 0
    right = length(s) - 1
    while left < right:
        while left < right and not isAlphanumeric(s[left]):
            left += 1
        while left < right and not isAlphanumeric(s[right]):
            right -= 1
        if toLower(s[left]) != toLower(s[right]):
            return false
        left += 1
        right -= 1
    return true
```

O(n) time, O(1) space.

</details>

---

## Problem 3 — First Unique Character in a String

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Given a string `s`, find the index of the first non-repeating character. If no such character exists, return `-1`.

### Example

**Input:** `s = "leetcode"`
**Output:** `0`
**Explanation:** The character 'l' at index 0 appears only once and is the first such character.

<details>
<summary>Hints</summary>

1. Build a frequency map of all characters, then scan the string again to find the first character with a count of 1.

</details>

<details>
<summary>Solution</summary>

**Approach:** Two-pass — build frequency map, then find first count-1 character.

```
function firstUniqChar(s):
    freq = empty hash map
    for c in s:
        freq[c] = freq.getOrDefault(c, 0) + 1
    for i from 0 to length(s) - 1:
        if freq[s[i]] == 1:
            return i
    return -1
```

O(n) time, O(k) space where k is the alphabet size.

</details>

---

## Problem 4 — Longest Common Prefix

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Given an array of strings `strs`, find the longest common prefix shared by all strings. If there is no common prefix, return an empty string.

### Example

**Input:** `strs = ["flower", "flow", "flight"]`
**Output:** `"fl"`
**Explanation:** All three strings start with "fl".

<details>
<summary>Hints</summary>

1. Compare characters column by column across all strings. Stop as soon as a mismatch is found or any string runs out of characters.

</details>

<details>
<summary>Solution</summary>

**Approach:** Vertical scanning — compare character at position i across all strings.

```
function longestCommonPrefix(strs):
    if strs is empty: return ""
    for i from 0 to length(strs[0]) - 1:
        c = strs[0][i]
        for j from 1 to length(strs) - 1:
            if i >= length(strs[j]) or strs[j][i] != c:
                return strs[0][0..i]    // substring up to i
    return strs[0]
```

O(S) time where S is the sum of all character lengths.

</details>

---

## Problem 5 — Longest Substring Without Repeating Characters

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Given a string `s`, find the length of the longest substring that contains no repeating characters.

### Example

**Input:** `s = "abcabcbb"`
**Output:** `3`
**Explanation:** The longest substring without repeating characters is "abc", with length 3.

<details>
<summary>Hints</summary>

1. Use a sliding window with two pointers. Maintain a set (or hash map) of characters in the current window.
2. When a duplicate is found, shrink the window from the left until the duplicate is removed.

</details>

<details>
<summary>Solution</summary>

**Approach:** Sliding window with a hash map tracking last-seen index of each character.

```
function lengthOfLongestSubstring(s):
    last_seen = empty hash map
    left = 0
    max_len = 0
    for right from 0 to length(s) - 1:
        if s[right] in last_seen and last_seen[s[right]] >= left:
            left = last_seen[s[right]] + 1
        last_seen[s[right]] = right
        max_len = max(max_len, right - left + 1)
    return max_len
```

O(n) time, O(min(n, alphabet_size)) space.

</details>
