# Arrays

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Beginner
**Prerequisites:** None

## Concept Overview

An array is a contiguous block of memory that stores a fixed-size sequence of elements of the same type. Arrays are the most fundamental data structure in computer science and form the basis for many other structures and algorithms. Because elements are stored contiguously, any element can be accessed in constant time given its index, making arrays ideal for random-access patterns.

Understanding arrays means understanding index arithmetic, in-place manipulation, traversal patterns, and the trade-offs between time and space when solving problems. Nearly every other data structure — from strings to heaps — builds on the mental model of sequential, indexed storage. Mastering array techniques early gives you a toolkit you will reuse throughout the entire curriculum.

### Core Operations — Pseudocode

```
// Access element by index — O(1)
function get(array, index):
    return array[index]

// Linear search — O(n)
function linearSearch(array, target):
    for i from 0 to length(array) - 1:
        if array[i] == target:
            return i
    return -1

// Insert at index (shifting elements right) — O(n)
function insertAt(array, index, value):
    for i from length(array) down to index + 1:
        array[i] = array[i - 1]
    array[index] = value

// Remove at index (shifting elements left) — O(n)
function removeAt(array, index):
    for i from index to length(array) - 2:
        array[i] = array[i + 1]
    shrink array by 1
```

---

## Problem 1 — Find the Maximum Element

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given an array of integers `nums`, return the maximum value in the array. The array is guaranteed to contain at least one element.

### Example

**Input:** `nums = [3, 1, 4, 1, 5, 9, 2, 6]`
**Output:** `9`
**Explanation:** 9 is the largest value in the array.

<details>
<summary>Hints</summary>

1. Walk through the array once, keeping track of the largest value seen so far.

</details>

<details>
<summary>Solution</summary>

**Approach:** Linear scan tracking the running maximum.

```
function findMax(nums):
    max_val = nums[0]
    for i from 1 to length(nums) - 1:
        if nums[i] > max_val:
            max_val = nums[i]
    return max_val
```

O(n) time, O(1) space.

</details>

---

## Problem 2 — Reverse an Array In-Place

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given an array of integers `nums`, reverse the array in-place (do not allocate a new array). Return the modified array.

### Example

**Input:** `nums = [1, 2, 3, 4, 5]`
**Output:** `[5, 4, 3, 2, 1]`
**Explanation:** The first and last elements swap, then the second and second-to-last, and so on.

<details>
<summary>Hints</summary>

1. Use two pointers — one at the start and one at the end — and swap elements while moving them toward each other.

</details>

<details>
<summary>Solution</summary>

**Approach:** Two-pointer swap from both ends toward the center.

```
function reverse(nums):
    left = 0
    right = length(nums) - 1
    while left < right:
        swap(nums[left], nums[right])
        left += 1
        right -= 1
    return nums
```

O(n) time, O(1) space.

</details>

---

## Problem 3 — Move Zeroes to End

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Given an array of integers `nums`, move all zeroes to the end while maintaining the relative order of the non-zero elements. Perform this in-place.

### Example

**Input:** `nums = [0, 1, 0, 3, 12]`
**Output:** `[1, 3, 12, 0, 0]`
**Explanation:** Non-zero elements keep their original order; zeroes are pushed to the back.

<details>
<summary>Hints</summary>

1. Maintain a write pointer that tracks where the next non-zero element should go.

</details>

<details>
<summary>Solution</summary>

**Approach:** Single pass with a write pointer, then zero-fill the tail.

```
function moveZeroes(nums):
    write = 0
    for i from 0 to length(nums) - 1:
        if nums[i] != 0:
            nums[write] = nums[i]
            write += 1
    for i from write to length(nums) - 1:
        nums[i] = 0
```

O(n) time, O(1) space.

</details>

---

## Problem 4 — Two Sum (Unsorted)

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Given an array of integers `nums` and an integer `target`, return the indices of the two elements that add up to `target`. You may assume exactly one solution exists and you may not use the same element twice.

### Example

**Input:** `nums = [2, 7, 11, 15]`, `target = 9`
**Output:** `[0, 1]`
**Explanation:** `nums[0] + nums[1] = 2 + 7 = 9`.

<details>
<summary>Hints</summary>

1. For each element, think about what complement value you need. A hash map can check for that complement in O(1).

</details>

<details>
<summary>Solution</summary>

**Approach:** Hash map storing seen values and their indices.

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

## Problem 5 — Rotate Array by K Steps

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Given an array `nums` and a non-negative integer `k`, rotate the array to the right by `k` steps in-place. For example, rotating `[1,2,3,4,5]` by 2 steps yields `[4,5,1,2,3]`.

### Example

**Input:** `nums = [1, 2, 3, 4, 5, 6, 7]`, `k = 3`
**Output:** `[5, 6, 7, 1, 2, 3, 4]`
**Explanation:** After 3 right rotations the last 3 elements wrap around to the front.

<details>
<summary>Hints</summary>

1. Rotating by `k` is the same as rotating by `k % len(nums)`.
2. Think about what happens if you reverse the entire array, then reverse the first `k` elements, then reverse the rest.

</details>

<details>
<summary>Solution</summary>

**Approach:** Three reversals — full array, first k, then remaining n-k.

```
function rotate(nums, k):
    n = length(nums)
    k = k % n
    if k == 0: return
    reverse(nums, 0, n - 1)
    reverse(nums, 0, k - 1)
    reverse(nums, k, n - 1)

function reverse(arr, left, right):
    while left < right:
        swap(arr[left], arr[right])
        left += 1
        right -= 1
```

O(n) time, O(1) space.

</details>
