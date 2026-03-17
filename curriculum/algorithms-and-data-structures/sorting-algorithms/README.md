# Sorting Algorithms

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Beginner
**Prerequisites:** Arrays

## Concept Overview

Sorting is the process of arranging elements in a defined order — typically ascending or descending. Sorting algorithms are among the most studied topics in computer science because sorted data unlocks efficient searching, merging, and deduplication. Understanding different sorting strategies also builds intuition about time-space trade-offs and algorithmic paradigms like divide-and-conquer.

Common sorting algorithms include comparison-based methods (bubble sort, insertion sort, merge sort, quicksort) and non-comparison-based methods (counting sort, radix sort). Each has different best-case, average-case, and worst-case complexities, as well as stability and in-place characteristics. This module focuses on understanding these trade-offs and applying sorting as a building block for more complex problems.

### Key Concepts

| Algorithm | Time (Worst / Avg / Best) | Space | Stable? | In-place? |
|---|---|---|---|---|
| Insertion Sort | O(n²) / O(n²) / O(n) | O(1) | Yes | Yes |
| Merge Sort | O(n log n) / O(n log n) / O(n log n) | O(n) | Yes | No |
| Quick Sort | O(n²) / O(n log n) / O(n log n) | O(log n) | No | Yes |
| Counting Sort | O(n + k) | O(k) | Yes | No |

Key trade-offs to consider: stability (preserving relative order of equal elements), in-place vs. extra memory, and how the algorithm performs on nearly-sorted or adversarial inputs. Divide-and-conquer sorts (merge sort, quicksort) achieve O(n log n) by splitting the problem, while comparison-based lower bound proves no comparison sort can do better.

---

## Problem 1 — Implement Insertion Sort

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given an array of integers `nums`, sort it in ascending order using the insertion sort algorithm. Return the sorted array.

### Example

**Input:** `nums = [5, 2, 4, 6, 1, 3]`
**Output:** `[1, 2, 3, 4, 5, 6]`
**Explanation:** Each element is inserted into its correct position within the already-sorted prefix.

<details>
<summary>Hints</summary>

1. Start from the second element. For each element, shift larger elements in the sorted prefix one position to the right, then insert the current element.

</details>

<details>
<summary>Solution</summary>

**Approach:** Iterate from index 1; shift sorted prefix right to make room for the key.

```
function insertionSort(nums):
    for i from 1 to length(nums) - 1:
        key = nums[i]
        j = i - 1
        while j >= 0 and nums[j] > key:
            nums[j + 1] = nums[j]
            j -= 1
        nums[j + 1] = key
    return nums
```

O(n²) worst case, O(n) best case (already sorted), O(1) extra space, stable.

</details>

---

## Problem 2 — Implement Merge Sort

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Given an array of integers `nums`, sort it in ascending order using the merge sort algorithm. Return the sorted array.

### Example

**Input:** `nums = [38, 27, 43, 3, 9, 82, 10]`
**Output:** `[3, 9, 10, 27, 38, 43, 82]`
**Explanation:** The array is recursively split in half, sorted, and merged back together.

<details>
<summary>Hints</summary>

1. Divide the array into two halves, recursively sort each half, then merge the two sorted halves.

</details>

<details>
<summary>Solution</summary>

**Approach:** Recursive split at midpoint, then merge two sorted halves.

```
function mergeSort(nums):
    if length(nums) <= 1: return nums
    mid = length(nums) / 2
    left = mergeSort(nums[0..mid])
    right = mergeSort(nums[mid..end])
    return merge(left, right)

function merge(a, b):
    result = []
    i = 0, j = 0
    while i < length(a) and j < length(b):
        if a[i] <= b[j]:
            result.append(a[i]); i += 1
        else:
            result.append(b[j]); j += 1
    append remaining elements of a and b to result
    return result
```

O(n log n) time in all cases, O(n) extra space, stable.

</details>

---

## Problem 3 — Sort an Array of 0s, 1s, and 2s (Dutch National Flag)

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Given an array `nums` containing only the values 0, 1, and 2, sort it in-place so that all 0s come first, then all 1s, then all 2s. Do not use a library sort function.

### Example

**Input:** `nums = [2, 0, 2, 1, 1, 0]`
**Output:** `[0, 0, 1, 1, 2, 2]`
**Explanation:** Elements are partitioned into three groups in a single pass.

<details>
<summary>Hints</summary>

1. Use three pointers: `low`, `mid`, and `high`. Process elements at `mid` and swap them to the correct region.

</details>

<details>
<summary>Solution</summary>

**Approach:** Dutch National Flag — three pointers partitioning in one pass.

```
function sortColors(nums):
    low = 0
    mid = 0
    high = length(nums) - 1
    while mid <= high:
        if nums[mid] == 0:
            swap(nums[low], nums[mid])
            low += 1; mid += 1
        else if nums[mid] == 1:
            mid += 1
        else:
            swap(nums[mid], nums[high])
            high -= 1
```

O(n) time, O(1) space.

</details>

---

## Problem 4 — Find the K-th Largest Element

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Given an unsorted array of integers `nums` and an integer `k`, return the k-th largest element. For example, if `k = 1`, return the maximum; if `k = n`, return the minimum.

### Example

**Input:** `nums = [3, 2, 1, 5, 6, 4]`, `k = 2`
**Output:** `5`
**Explanation:** The sorted array is `[1, 2, 3, 4, 5, 6]`; the 2nd largest is 5.

<details>
<summary>Hints</summary>

1. Sorting the array and picking the element at index `n - k` works in O(n log n). Can you do better with a partition-based approach (Quickselect)?

</details>

<details>
<summary>Solution</summary>

**Approach:** Quickselect — partition around a pivot and recurse into the relevant side.

```
function findKthLargest(nums, k):
    target = length(nums) - k
    return quickselect(nums, 0, length(nums) - 1, target)

function quickselect(nums, lo, hi, target):
    pivot = nums[hi]
    i = lo
    for j from lo to hi - 1:
        if nums[j] <= pivot:
            swap(nums[i], nums[j])
            i += 1
    swap(nums[i], nums[hi])
    if i == target: return nums[i]
    else if i < target: return quickselect(nums, i + 1, hi, target)
    else: return quickselect(nums, lo, i - 1, target)
```

Average O(n), worst O(n²). Sorting approach: O(n log n).

</details>

---

## Problem 5 — Merge Intervals After Sorting

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Given an array of intervals where `intervals[i] = [start_i, end_i]`, merge all overlapping intervals and return an array of the non-overlapping intervals that cover all the input intervals.

### Example

**Input:** `intervals = [[1,3], [2,6], [8,10], [15,18]]`
**Output:** `[[1,6], [8,10], [15,18]]`
**Explanation:** Intervals [1,3] and [2,6] overlap, so they merge into [1,6].

<details>
<summary>Hints</summary>

1. Sort the intervals by their start value first. Then iterate and merge overlapping intervals greedily.
2. Two intervals overlap if the current interval's start is less than or equal to the previous interval's end.

</details>

<details>
<summary>Solution</summary>

**Approach:** Sort by start, then greedily extend or append intervals.

```
function mergeIntervals(intervals):
    sort intervals by start value
    merged = [intervals[0]]
    for i from 1 to length(intervals) - 1:
        last = merged[length(merged) - 1]
        current = intervals[i]
        if current.start <= last.end:
            last.end = max(last.end, current.end)
        else:
            merged.append(current)
    return merged
```

O(n log n) time for sorting, O(n) space for output.

</details>
