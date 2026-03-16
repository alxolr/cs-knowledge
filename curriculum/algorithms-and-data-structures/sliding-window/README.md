# Sliding Window

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Intermediate
**Prerequisites:** Arrays

## Concept Overview

The sliding window technique maintains a contiguous subarray (or substring) of a sequence and "slides" it across the data to compute running aggregates without redundant work. Instead of recalculating a property from scratch for every possible subarray — an O(n·k) or O(n²) approach — you add the new element entering the window and remove the element leaving it, keeping each update O(1).

There are two main variants. A fixed-size window slides one position at a time and is ideal for problems like "maximum sum of k consecutive elements." A variable-size (or dynamic) window expands by advancing the right boundary and contracts by advancing the left boundary, guided by a condition such as "the window must contain at most k distinct characters." The variable-size pattern is more general and appears in a wide range of substring and subarray optimization problems.

Sliding window builds directly on array traversal skills and pairs naturally with hash maps for frequency tracking. It is a prerequisite-free stepping stone toward more complex techniques like two-pointer partitioning on unsorted data and segment-tree range queries.

---

## Problem 1 — Maximum Sum Subarray of Size K

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given an array of integers `nums` and a positive integer `k`, find the maximum sum of any contiguous subarray of exactly size `k`.

### Example

**Input:** `nums = [2, 1, 5, 1, 3, 2]`, `k = 3`
**Output:** `9`
**Explanation:** The subarray `[5, 1, 3]` has the maximum sum of 9.

**Input:** `nums = [1, 4, 2, 10, 2, 3, 1, 0, 20]`, `k = 4`
**Output:** `24`
**Explanation:** The subarray `[2, 3, 1, 0]` sums to 6, but `[1, 0, 20]` is only size 3. The subarray `[10, 2, 3, 1]` sums to 16, and `[3, 1, 0, 20]` sums to 24 — the maximum.

<details>
<summary>Hints</summary>

1. Compute the sum of the first `k` elements. Then slide the window one position at a time: add the element entering on the right and subtract the element leaving on the left. Track the maximum sum seen.

</details>

<details>
<summary>Solution</summary>

**Approach:** Fixed-size sliding window — maintain a running sum and update it incrementally.

```
function maxSumSubarray(nums, k):
    windowSum = sum(nums[0..k-1])
    best = windowSum
    for i from k to length(nums) - 1:
        windowSum += nums[i] - nums[i - k]
        best = max(best, windowSum)
    return best
```

O(n) time, O(1) space.

</details>


---

## Problem 2 — Longest Substring Without Repeating Characters

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Given a string `s`, find the length of the longest substring that contains no repeating characters.

### Example

**Input:** `s = "abcabcbb"`
**Output:** `3`
**Explanation:** The longest substring without repeating characters is `"abc"`, which has length 3.

**Input:** `s = "bbbbb"`
**Output:** `1`
**Explanation:** Every character is the same, so the longest non-repeating substring is any single `"b"`.

<details>
<summary>Hints</summary>

1. Use a variable-size window with a set (or map) tracking characters currently in the window. Expand the right boundary one character at a time. When a duplicate is found, shrink from the left until the duplicate is removed.

</details>

<details>
<summary>Solution</summary>

**Approach:** Variable-size sliding window with a hash set for character membership.

```
function lengthOfLongestSubstring(s):
    charSet = empty set
    left = 0
    best = 0
    for right from 0 to length(s) - 1:
        while s[right] in charSet:
            charSet.remove(s[left])
            left += 1
        charSet.add(s[right])
        best = max(best, right - left + 1)
    return best
```

O(n) time (each character is added and removed at most once), O(min(n, alphabet)) space.

</details>


---

## Problem 3 — Minimum Size Subarray Sum

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Given an array of positive integers `nums` and a positive integer `target`, find the minimal length of a contiguous subarray whose sum is greater than or equal to `target`. If no such subarray exists, return `0`.

### Example

**Input:** `nums = [2, 3, 1, 2, 4, 3]`, `target = 7`
**Output:** `2`
**Explanation:** The subarray `[4, 3]` has sum 7 and length 2, which is the shortest.

**Input:** `nums = [1, 1, 1, 1, 1]`, `target = 11`
**Output:** `0`
**Explanation:** The total sum is 5, which is less than 11, so no valid subarray exists.

<details>
<summary>Hints</summary>

1. Expand the window to the right until the sum meets or exceeds `target`. Then shrink from the left to find the smallest valid window at that position. Continue expanding right and repeat.

</details>

<details>
<summary>Solution</summary>

**Approach:** Variable-size sliding window — expand right to satisfy the condition, shrink left to minimize length.

```
function minSubArrayLen(target, nums):
    left = 0
    windowSum = 0
    best = infinity
    for right from 0 to length(nums) - 1:
        windowSum += nums[right]
        while windowSum >= target:
            best = min(best, right - left + 1)
            windowSum -= nums[left]
            left += 1
    return best if best != infinity else 0
```

O(n) time, O(1) space.

</details>


---

## Problem 4 — Longest Substring with At Most K Distinct Characters

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Given a string `s` and an integer `k`, find the length of the longest substring that contains at most `k` distinct characters.

### Example

**Input:** `s = "eceba"`, `k = 2`
**Output:** `3`
**Explanation:** The longest substring with at most 2 distinct characters is `"ece"`, which has length 3.

**Input:** `s = "aabacbebebe"`, `k = 3`
**Output:** `7`
**Explanation:** The longest substring with at most 3 distinct characters is `"cbebebe"`, which has length 7.

<details>
<summary>Hints</summary>

1. Maintain a hash map of character → frequency for the current window. Expand right to add characters. When the map has more than `k` keys, shrink from the left, decrementing counts and removing entries that reach zero, until the window is valid again.

</details>

<details>
<summary>Solution</summary>

**Approach:** Variable-size sliding window with a frequency map tracking distinct character count.

```
function longestSubstringKDistinct(s, k):
    freq = empty hash map
    left = 0
    best = 0
    for right from 0 to length(s) - 1:
        freq[s[right]] += 1
        while size(freq) > k:
            freq[s[left]] -= 1
            if freq[s[left]] == 0:
                delete freq[s[left]]
            left += 1
        best = max(best, right - left + 1)
    return best
```

O(n) time, O(k) space.

</details>


---

## Problem 5 — Minimum Window Substring

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Given two strings `s` and `t`, find the minimum-length substring of `s` that contains every character of `t` (including duplicates). If no such substring exists, return the empty string `""`. When multiple answers of the same length exist, return the one that appears first.

### Example

**Input:** `s = "ADOBECODEBANC"`, `t = "ABC"`
**Output:** `"BANC"`
**Explanation:** `"BANC"` is the shortest substring of `s` that contains `'A'`, `'B'`, and `'C'`.

**Input:** `s = "a"`, `t = "aa"`
**Output:** `""`
**Explanation:** `t` requires two `'a'`s but `s` has only one, so no valid window exists.

<details>
<summary>Hints</summary>

1. Build a frequency map of `t`. Expand the window to the right, tracking how many required characters have been fully satisfied. Once all characters are satisfied, shrink from the left to minimize the window, updating the best result each time the window is still valid.
2. Keep a counter of how many distinct characters have met their required count. This lets you check validity in O(1) instead of scanning the entire frequency map.

</details>

<details>
<summary>Solution</summary>

**Approach:** Variable-size sliding window with two frequency maps and a "formed" counter.

```
function minWindow(s, t):
    if length(t) > length(s): return ""

    need = frequency map of t
    window = empty hash map
    required = size(need)       // distinct chars to satisfy
    formed = 0                  // distinct chars currently satisfied
    best = (infinity, -1, -1)   // (length, left, right)

    left = 0
    for right from 0 to length(s) - 1:
        ch = s[right]
        window[ch] += 1
        if ch in need and window[ch] == need[ch]:
            formed += 1

        while formed == required:
            // update best
            if right - left + 1 < best.length:
                best = (right - left + 1, left, right)
            // shrink from left
            out = s[left]
            window[out] -= 1
            if out in need and window[out] < need[out]:
                formed -= 1
            left += 1

    return "" if best.length == infinity
           else s[best.left .. best.right]
```

O(n + m) time where n = |s| and m = |t|, O(n + m) space.

</details>