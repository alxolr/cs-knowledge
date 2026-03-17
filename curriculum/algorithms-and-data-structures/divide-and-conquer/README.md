# Divide and Conquer

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Advanced
**Prerequisites:** [Recursion](../recursion/README.md)

## Concept Overview

Divide and conquer is an algorithm design paradigm that solves a problem by breaking it into smaller sub-problems of the same type, solving each sub-problem recursively, and then combining the results to produce the final answer. The three canonical steps are: divide the input into smaller pieces, conquer each piece via recursion (with a base case for trivially small inputs), and combine the sub-solutions.

This strategy underpins many of the most important algorithms in computer science. Merge sort and quicksort split an array, sort the halves, and merge or partition the results. Binary search repeatedly halves the search space. Strassen's algorithm multiplies matrices faster than the naive cubic approach by decomposing the multiplication into fewer recursive sub-multiplications. The efficiency of divide and conquer often comes from reducing the number of sub-problems or the cost of the combine step, which is formally analyzed using the Master Theorem.

A solid grasp of recursion is essential because every divide-and-conquer algorithm is expressed as a recursive function. The key skill beyond basic recursion is learning how to identify the right way to split the problem and how to merge partial results efficiently — the "combine" step is frequently where the real algorithmic insight lives.

### Core Template — Pseudocode

```
// General divide-and-conquer template
function divideAndConquer(problem):
    if problem is small enough:          // base case
        return directSolve(problem)
    subproblems = divide(problem)        // divide
    subresults = []
    for each sub in subproblems:
        subresults.append(divideAndConquer(sub))  // conquer
    return combine(subresults)           // combine
```

The efficiency of divide and conquer often comes from reducing the number of sub-problems or the cost of the combine step. The Master Theorem provides a framework for analyzing the time complexity: for recurrences of the form T(n) = aT(n/b) + O(n^d), the solution depends on the relationship between log_b(a) and d. The "combine" step is frequently where the real algorithmic insight lives.

---

## Problem 1 — Merge Sort

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given an array of integers `nums`, sort the array in non-decreasing order using the merge sort algorithm and return the sorted array. You must implement the divide-and-conquer approach: split the array into two halves, recursively sort each half, and merge the two sorted halves.

### Example

**Input:** `nums = [5, 2, 8, 1, 9, 3]`
**Output:** `[1, 2, 3, 5, 8, 9]`
**Explanation:** The array is recursively split into halves until single elements remain, then merged back in sorted order.

<details>
<summary>Hints</summary>

1. The base case is an array of length 0 or 1 — it is already sorted. Split the array at the midpoint, sort each half, and merge by comparing elements from the front of each half.

</details>

<details>
<summary>Solution</summary>

**Approach:** Classic two-way merge sort.

```
function mergeSort(nums):
    if length(nums) <= 1:
        return nums
    mid = length(nums) / 2
    left = mergeSort(nums[0..mid-1])
    right = mergeSort(nums[mid..end])
    return merge(left, right)

function merge(a, b):
    result = []
    i = 0, j = 0
    while i < length(a) and j < length(b):
        if a[i] <= b[j]:
            result.append(a[i]); i++
        else:
            result.append(b[j]); j++
    append remaining elements of a or b to result
    return result
```

O(n log n) time, O(n) auxiliary space for the merge buffer.

</details>


---

## Problem 2 — Count Inversions

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

An inversion in an array is a pair of indices `(i, j)` where `i < j` but `nums[i] > nums[j]`. Given an array of integers `nums`, return the total number of inversions. Use a divide-and-conquer approach to achieve better than O(n²) time.

### Example

**Input:** `nums = [2, 4, 1, 3, 5]`
**Output:** `3`
**Explanation:** The inversions are (2,1), (4,1), and (4,3).

<details>
<summary>Hints</summary>

1. Modify merge sort so that during the merge step you count how many elements from the right half are placed before elements from the left half — each such placement represents one or more inversions.

</details>

<details>
<summary>Solution</summary>

**Approach:** Merge sort with an inversion counter accumulated during the merge step.

```
function countInversions(nums):
    if length(nums) <= 1:
        return nums, 0
    mid = length(nums) / 2
    left, leftInv = countInversions(nums[0..mid-1])
    right, rightInv = countInversions(nums[mid..end])
    merged, splitInv = mergeCount(left, right)
    return merged, leftInv + rightInv + splitInv

function mergeCount(a, b):
    result = []
    count = 0
    i = 0, j = 0
    while i < length(a) and j < length(b):
        if a[i] <= b[j]:
            result.append(a[i]); i++
        else:
            result.append(b[j]); j++
            count += length(a) - i   // all remaining in a are inversions with b[j]
    append remaining elements of a or b to result
    return result, count
```

O(n log n) time, O(n) space — same as merge sort with a constant-time addition per merge comparison.

</details>


---

## Problem 3 — Kth Largest Element (Quickselect)

**Difficulty:** Hard
**Estimated Time:** 45 minutes

### Problem Statement

Given an unsorted array of integers `nums` and an integer `k`, return the kth largest element in the array. You must solve it using a divide-and-conquer selection algorithm (quickselect) with expected O(n) time, not by fully sorting the array.

### Example

**Input:** `nums = [3, 2, 1, 5, 6, 4]`, `k = 2`
**Output:** `5`
**Explanation:** The sorted array is [1, 2, 3, 4, 5, 6]. The 2nd largest element is 5.

**Input:** `nums = [3, 2, 3, 1, 2, 4, 5, 5, 6]`, `k = 4`
**Output:** `4`

<details>
<summary>Hints</summary>

1. The kth largest element is the element at index `n - k` in the sorted array. Use a partition function (like Lomuto or Hoare) to place a pivot in its correct position, then recurse only into the half that contains the target index.
2. Randomize the pivot choice to avoid worst-case O(n²) behavior on already-sorted or adversarial inputs.

</details>

<details>
<summary>Solution</summary>

**Approach:** Quickselect — partition around a random pivot and recurse into one side.

```
function findKthLargest(nums, k):
    target = length(nums) - k
    return quickselect(nums, 0, length(nums) - 1, target)

function quickselect(nums, lo, hi, target):
    if lo == hi:
        return nums[lo]
    pivotIndex = randomInt(lo, hi)
    pivotIndex = partition(nums, lo, hi, pivotIndex)
    if target == pivotIndex:
        return nums[pivotIndex]
    else if target < pivotIndex:
        return quickselect(nums, lo, pivotIndex - 1, target)
    else:
        return quickselect(nums, pivotIndex + 1, hi, target)

function partition(nums, lo, hi, pivotIndex):
    pivotValue = nums[pivotIndex]
    swap(nums[pivotIndex], nums[hi])
    storeIndex = lo
    for i from lo to hi - 1:
        if nums[i] < pivotValue:
            swap(nums[storeIndex], nums[i])
            storeIndex++
    swap(nums[storeIndex], nums[hi])
    return storeIndex
```

O(n) expected time, O(n²) worst case (mitigated by random pivot). O(log n) expected recursion depth.

</details>


---

## Problem 4 — Closest Pair of Points

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Given `n` points in a 2D plane, each represented as `(x, y)`, find the pair of points with the smallest Euclidean distance between them. Return that minimum distance. Implement a divide-and-conquer solution that runs in O(n log n) time rather than the brute-force O(n²).

### Example

**Input:** `points = [(2, 3), (12, 30), (40, 50), (5, 1), (12, 10), (3, 4)]`
**Output:** `1.414` (approximately √2, the distance between (2,3) and (3,4))
**Explanation:** Among all pairs, (2,3) and (3,4) are the closest with distance √((3−2)² + (4−3)²) = √2.

<details>
<summary>Hints</summary>

1. Sort points by x-coordinate. Split the set into left and right halves at the median x-value. Recursively find the closest pair in each half. The tricky part is handling pairs that straddle the dividing line.
2. After finding the minimum distance `d` from both halves, build a "strip" of points within distance `d` of the dividing line. Sort this strip by y-coordinate and compare each point only to the next few points (at most 7) in the strip — this keeps the combine step O(n log n) overall.

</details>

<details>
<summary>Solution</summary>

**Approach:** Divide the point set by x-coordinate, recurse on each half, then check the strip around the dividing line.

```
function closestPair(points):
    sort points by x-coordinate
    return closestRec(points)

function closestRec(P):
    n = length(P)
    if n <= 3:
        return bruteForce(P)
    mid = n / 2
    midX = P[mid].x
    dLeft = closestRec(P[0..mid-1])
    dRight = closestRec(P[mid..n-1])
    d = min(dLeft, dRight)

    // Build strip of points within distance d of the dividing line
    strip = [p for p in P if |p.x - midX| < d]
    sort strip by y-coordinate

    // Check strip pairs
    for i from 0 to length(strip) - 1:
        for j from i + 1 to length(strip) - 1:
            if strip[j].y - strip[i].y >= d:
                break
            d = min(d, distance(strip[i], strip[j]))
    return d
```

O(n log² n) time with the simple version (sorting the strip each time). Can be optimized to O(n log n) by pre-sorting by y and merging. O(n) auxiliary space.

</details>


---

## Problem 5 — Median of Two Sorted Arrays

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Given two sorted arrays `nums1` and `nums2` of sizes `m` and `n` respectively, return the median of the two sorted arrays. The overall run-time complexity must be O(log(m + n)). The median is the middle value if the combined length is odd, or the average of the two middle values if even.

### Example

**Input:** `nums1 = [1, 3]`, `nums2 = [2]`
**Output:** `2.0`
**Explanation:** The merged array is [1, 2, 3]. The median is 2.

**Input:** `nums1 = [1, 2]`, `nums2 = [3, 4]`
**Output:** `2.5`
**Explanation:** The merged array is [1, 2, 3, 4]. The median is (2 + 3) / 2 = 2.5.

<details>
<summary>Hints</summary>

1. Think of the problem as finding a partition of both arrays such that all elements on the left side are ≤ all elements on the right side, and the left side has exactly half the total elements. Binary search on the partition position of the shorter array.
2. If you partition `nums1` at index `i` and `nums2` at index `j = (m+n+1)/2 - i`, check whether `nums1[i-1] <= nums2[j]` and `nums2[j-1] <= nums1[i]`. Adjust the binary search bounds based on which condition fails.

</details>

<details>
<summary>Solution</summary>

**Approach:** Binary search on the partition index of the shorter array.

```
function findMedianSortedArrays(nums1, nums2):
    // Ensure nums1 is the shorter array
    if length(nums1) > length(nums2):
        swap(nums1, nums2)
    m = length(nums1)
    n = length(nums2)
    lo = 0, hi = m
    halfLen = (m + n + 1) / 2

    while lo <= hi:
        i = (lo + hi) / 2          // partition index in nums1
        j = halfLen - i             // partition index in nums2

        leftMax1  = nums1[i - 1] if i > 0 else -∞
        rightMin1 = nums1[i]     if i < m else +∞
        leftMax2  = nums2[j - 1] if j > 0 else -∞
        rightMin2 = nums2[j]     if j < n else +∞

        if leftMax1 <= rightMin2 and leftMax2 <= rightMin1:
            // Correct partition found
            if (m + n) is odd:
                return max(leftMax1, leftMax2)
            else:
                return (max(leftMax1, leftMax2) + min(rightMin1, rightMin2)) / 2.0
        else if leftMax1 > rightMin2:
            hi = i - 1
        else:
            lo = i + 1
```

O(log(min(m, n))) time, O(1) space.

</details>
