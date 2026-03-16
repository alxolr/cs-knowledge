# Binary Search

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Beginner
**Prerequisites:** Arrays, Sorting Algorithms

## Concept Overview

Binary search is an efficient algorithm for finding a target value within a sorted sequence. By repeatedly dividing the search space in half, it achieves O(log n) time complexity — a dramatic improvement over the O(n) linear scan. The core idea is simple: compare the target with the middle element; if they match you are done; if the target is smaller, search the left half; if larger, search the right half.

Beyond the classic "find a value in a sorted array" use case, binary search generalizes to any situation where you can define a monotonic predicate over a search space. This makes it applicable to problems like finding boundaries, searching in rotated arrays, and optimizing over continuous or discrete domains. Mastering binary search and its edge cases (off-by-one errors, choosing between `left <= right` vs. `left < right`) is a rite of passage for every algorithm practitioner.

---

## Problem 1 — Classic Binary Search

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given a sorted array of integers `nums` and a target integer `target`, return the index of `target` if it exists, or `-1` if it does not.

### Example

**Input:** `nums = [-1, 0, 3, 5, 9, 12]`, `target = 9`
**Output:** `4`
**Explanation:** The value 9 is at index 4.

<details>
<summary>Hints</summary>

1. Maintain `left` and `right` pointers. Compute `mid = left + (right - left) // 2` to avoid overflow.

</details>

<details>
<summary>Solution</summary>

**Approach:** Standard binary search with left/right pointers.

```
function binarySearch(nums, target):
    left = 0
    right = length(nums) - 1
    while left <= right:
        mid = left + (right - left) / 2
        if nums[mid] == target: return mid
        if nums[mid] < target: left = mid + 1
        else: right = mid - 1
    return -1
```

O(log n) time, O(1) space.

</details>

---

## Problem 2 — Find the Insertion Position

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given a sorted array of distinct integers `nums` and a target value, return the index where the target would be inserted to keep the array sorted. If the target already exists, return its index.

### Example

**Input:** `nums = [1, 3, 5, 6]`, `target = 5`
**Output:** `2`

**Input:** `nums = [1, 3, 5, 6]`, `target = 2`
**Output:** `1`
**Explanation:** The value 2 would be inserted at index 1 to maintain sorted order.

<details>
<summary>Hints</summary>

1. This is a standard binary search where, if the target is not found, the `left` pointer ends up at the correct insertion index.

</details>

<details>
<summary>Solution</summary>

**Approach:** Binary search; when target not found, `left` is the insertion point.

```
function searchInsert(nums, target):
    left = 0
    right = length(nums) - 1
    while left <= right:
        mid = left + (right - left) / 2
        if nums[mid] == target: return mid
        if nums[mid] < target: left = mid + 1
        else: right = mid - 1
    return left
```

O(log n) time, O(1) space.

</details>

---

## Problem 3 — First and Last Position of Target

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Given a sorted array of integers `nums` (which may contain duplicates) and a target value, find the starting and ending position of the target. If the target is not found, return `[-1, -1]`.

### Example

**Input:** `nums = [5, 7, 7, 8, 8, 10]`, `target = 8`
**Output:** `[3, 4]`
**Explanation:** The value 8 first appears at index 3 and last appears at index 4.

<details>
<summary>Hints</summary>

1. Run two binary searches: one to find the leftmost occurrence and one to find the rightmost occurrence.

</details>

<details>
<summary>Solution</summary>

**Approach:** Two binary searches — one biased left, one biased right.

```
function searchRange(nums, target):
    return [findLeft(nums, target), findRight(nums, target)]

function findLeft(nums, target):
    left = 0; right = length(nums) - 1; result = -1
    while left <= right:
        mid = left + (right - left) / 2
        if nums[mid] == target:
            result = mid; right = mid - 1
        else if nums[mid] < target: left = mid + 1
        else: right = mid - 1
    return result

function findRight(nums, target):
    left = 0; right = length(nums) - 1; result = -1
    while left <= right:
        mid = left + (right - left) / 2
        if nums[mid] == target:
            result = mid; left = mid + 1
        else if nums[mid] < target: left = mid + 1
        else: right = mid - 1
    return result
```

O(log n) time, O(1) space.

</details>

---

## Problem 4 — Search in a Rotated Sorted Array

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

A sorted array has been rotated at some unknown pivot (e.g., `[4,5,6,7,0,1,2]` was originally `[0,1,2,4,5,6,7]`). Given the rotated array `nums` and a target value, return the index of the target or `-1` if not found. All values are distinct.

### Example

**Input:** `nums = [4, 5, 6, 7, 0, 1, 2]`, `target = 0`
**Output:** `4`
**Explanation:** The value 0 is at index 4 in the rotated array.

<details>
<summary>Hints</summary>

1. At each step of binary search, one half of the array is guaranteed to be sorted. Determine which half is sorted and whether the target lies within that sorted half.

</details>

<details>
<summary>Solution</summary>

**Approach:** Modified binary search — identify the sorted half and check if target falls within it.

```
function searchRotated(nums, target):
    left = 0; right = length(nums) - 1
    while left <= right:
        mid = left + (right - left) / 2
        if nums[mid] == target: return mid
        if nums[left] <= nums[mid]:       // left half is sorted
            if nums[left] <= target < nums[mid]:
                right = mid - 1
            else:
                left = mid + 1
        else:                              // right half is sorted
            if nums[mid] < target <= nums[right]:
                left = mid + 1
            else:
                right = mid - 1
    return -1
```

O(log n) time, O(1) space.

</details>

---

## Problem 5 — Find Peak Element

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

A peak element is an element that is strictly greater than its neighbors. Given an integer array `nums` where `nums[-1]` and `nums[n]` are conceptually negative infinity, find any peak element and return its index. The array may contain multiple peaks; returning any one is acceptable. Your solution must run in O(log n) time.

### Example

**Input:** `nums = [1, 2, 1, 3, 5, 6, 4]`
**Output:** `5` (or `1`)
**Explanation:** Index 5 has value 6, which is greater than neighbors 5 and 4. Index 1 (value 2) is also a valid peak.

<details>
<summary>Hints</summary>

1. If `nums[mid] < nums[mid + 1]`, a peak must exist to the right (the slope is going up). Otherwise, a peak exists to the left (including `mid` itself).
2. This is a binary search on the "direction of the slope."

</details>

<details>
<summary>Solution</summary>

**Approach:** Binary search on slope direction — move toward the ascending side.

```
function findPeakElement(nums):
    left = 0
    right = length(nums) - 1
    while left < right:
        mid = left + (right - left) / 2
        if nums[mid] < nums[mid + 1]:
            left = mid + 1
        else:
            right = mid
    return left
```

O(log n) time, O(1) space.

</details>
