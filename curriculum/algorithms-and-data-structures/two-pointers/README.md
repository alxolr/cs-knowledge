# Two Pointers

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Intermediate
**Prerequisites:** Arrays

## Concept Overview

The two-pointer technique uses two index variables that move through a data structure — usually an array or string — to solve problems in linear or near-linear time. Instead of brute-forcing every pair of elements with nested loops (O(n²)), you position one pointer at each end, or advance both from the start at different speeds, and converge toward a solution in a single pass.

Two pointers come in several flavors. The most common are the "opposite-direction" pattern (one pointer at the start, one at the end, moving inward) and the "same-direction" or "fast-and-slow" pattern (both start at the beginning, one advancing faster). Choosing the right variant depends on the problem's structure: sorted arrays often call for opposite-direction pointers, while linked-list cycle detection and partitioning problems favor fast-and-slow.

Mastering two pointers is a gateway to more advanced sliding-window and interval techniques. The core insight — that maintaining two positions lets you eliminate large swaths of the search space — recurs throughout competitive programming and real-world algorithm design.

### Core Patterns

Two pointers come in several flavors:

| Pattern | Pointer Setup | Typical Use Case |
|---|---|---|
| Opposite-direction | One at start, one at end, moving inward | Pair-sum on sorted data, container problems |
| Same-direction (fast/slow) | Both start at beginning, one advances faster | Deduplication, partitioning, linked-list cycle detection |
| Sliding window hybrid | Left and right define a window | Substring/subarray optimization (see Sliding Window module) |

The key insight is that maintaining two positions lets you eliminate large swaths of the search space in a single pass, turning O(n²) brute-force pair enumeration into O(n) linear scans. Choosing the right variant depends on the problem's structure: sorted arrays often call for opposite-direction pointers, while partitioning and cycle problems favor fast-and-slow.

---

## Problem 1 — Pair with Target Sum (Sorted Array)

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given a sorted array of integers `nums` and an integer `target`, return the indices of two numbers that add up to `target`. The array is sorted in non-decreasing order, exactly one solution exists, and you may not use the same element twice.

### Example

**Input:** `nums = [1, 3, 4, 6, 8, 11]`, `target = 10`
**Output:** `[2, 3]`
**Explanation:** `nums[2] + nums[3] = 4 + 6 = 10`.

**Input:** `nums = [2, 5, 9, 11]`, `target = 11`
**Output:** `[0, 2]`
**Explanation:** `nums[0] + nums[2] = 2 + 9 = 11`.

<details>
<summary>Hints</summary>

1. Place one pointer at the start and one at the end. Because the array is sorted, the sum of the two pointed-to elements tells you which pointer to move.

</details>

<details>
<summary>Solution</summary>

**Approach:** Opposite-direction two pointers on a sorted array.

```
function pairWithTarget(nums, target):
    left = 0
    right = length(nums) - 1
    while left < right:
        sum = nums[left] + nums[right]
        if sum == target:
            return [left, right]
        else if sum < target:
            left += 1
        else:
            right -= 1
```

O(n) time, O(1) space.

</details>


---

## Problem 2 — Remove Duplicates from Sorted Array

**Difficulty:** Medium
**Estimated Time:** 35 minutes

### Problem Statement

Given a sorted array `nums`, remove the duplicates in-place so that each unique element appears only once. Return the number of unique elements `k`. The first `k` elements of `nums` must hold the unique values in their original order. Elements beyond position `k` do not matter.

### Example

**Input:** `nums = [1, 1, 2, 3, 3, 3, 4]`
**Output:** `k = 4`, `nums = [1, 2, 3, 4, ...]`
**Explanation:** Four unique values remain at the front of the array.

<details>
<summary>Hints</summary>

1. Use a slow pointer to track the write position and a fast pointer to scan ahead. Whenever the fast pointer finds a value different from the one at the slow pointer, copy it forward.

</details>

<details>
<summary>Solution</summary>

**Approach:** Same-direction two pointers — slow writes, fast reads.

```
function removeDuplicates(nums):
    if length(nums) == 0: return 0
    slow = 0
    for fast from 1 to length(nums) - 1:
        if nums[fast] != nums[slow]:
            slow += 1
            nums[slow] = nums[fast]
    return slow + 1
```

O(n) time, O(1) space.

</details>


---

## Problem 3 — Container with Most Water

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Given an array `height` of `n` non-negative integers where each element represents the height of a vertical line drawn at that index, find two lines that, together with the x-axis, form a container that holds the most water. Return the maximum amount of water the container can store. The container cannot be tilted.

### Example

**Input:** `height = [1, 8, 6, 2, 5, 4, 8, 3, 7]`
**Output:** `49`
**Explanation:** Lines at indices 1 (height 8) and 8 (height 7) form a container of width 7 and height min(8, 7) = 7, giving area 49.

<details>
<summary>Hints</summary>

1. Start with the widest container (pointers at both ends). The only way to potentially find a larger area is to move the pointer at the shorter line inward, because moving the taller one can never increase the limiting height.

</details>

<details>
<summary>Solution</summary>

**Approach:** Opposite-direction greedy narrowing — always move the shorter side.

```
function maxArea(height):
    left = 0
    right = length(height) - 1
    best = 0
    while left < right:
        w = right - left
        h = min(height[left], height[right])
        best = max(best, w * h)
        if height[left] < height[right]:
            left += 1
        else:
            right -= 1
    return best
```

O(n) time, O(1) space.

</details>


---

## Problem 4 — Three Sum

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Given an array of integers `nums`, return all unique triplets `[nums[i], nums[j], nums[k]]` such that `i != j`, `i != k`, `j != k`, and `nums[i] + nums[j] + nums[k] == 0`. The solution set must not contain duplicate triplets.

### Example

**Input:** `nums = [-1, 0, 1, 2, -1, -4]`
**Output:** `[[-1, -1, 2], [-1, 0, 1]]`
**Explanation:** The two unique triplets that sum to zero are shown above.

<details>
<summary>Hints</summary>

1. Sort the array first. Then fix one element and use the two-pointer pair-sum technique on the remaining subarray to find pairs that complete the target.
2. Skip duplicate values for the fixed element and for both pointers to avoid duplicate triplets.

</details>

<details>
<summary>Solution</summary>

**Approach:** Sort + fix one element + opposite-direction two pointers for the remaining pair.

```
function threeSum(nums):
    sort(nums)
    result = []
    for i from 0 to length(nums) - 3:
        if i > 0 and nums[i] == nums[i - 1]: continue   // skip duplicate fixed element
        left = i + 1
        right = length(nums) - 1
        while left < right:
            total = nums[i] + nums[left] + nums[right]
            if total == 0:
                result.append([nums[i], nums[left], nums[right]])
                while left < right and nums[left] == nums[left + 1]: left += 1
                while left < right and nums[right] == nums[right - 1]: right -= 1
                left += 1
                right -= 1
            else if total < 0:
                left += 1
            else:
                right -= 1
    return result
```

O(n²) time, O(1) extra space (ignoring output).

</details>


---

## Problem 5 — Trapping Rain Water

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Given an array `height` of `n` non-negative integers representing an elevation map where the width of each bar is 1, compute how much water can be trapped after raining.

### Example

**Input:** `height = [0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1]`
**Output:** `6`
**Explanation:** Water fills the gaps between bars. The total trapped water across all positions is 6 units.

<details>
<summary>Hints</summary>

1. The water above any position `i` is `min(max_left, max_right) - height[i]`, where `max_left` and `max_right` are the tallest bars to the left and right of `i`.
2. You can compute this in one pass with two pointers: maintain running maximums from each side and always process the side with the smaller maximum, because that side is the bottleneck.

</details>

<details>
<summary>Solution</summary>

**Approach:** Opposite-direction two pointers with running left-max and right-max.

```
function trap(height):
    left = 0
    right = length(height) - 1
    left_max = 0
    right_max = 0
    water = 0
    while left < right:
        if height[left] < height[right]:
            if height[left] >= left_max:
                left_max = height[left]
            else:
                water += left_max - height[left]
            left += 1
        else:
            if height[right] >= right_max:
                right_max = height[right]
            else:
                water += right_max - height[right]
            right -= 1
    return water
```

O(n) time, O(1) space.

</details>