# Interval Problems

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Advanced
**Prerequisites:** [Sorting Algorithms](../sorting-algorithms/README.md), [Greedy Algorithms](../greedy-algorithms/README.md)

## Concept Overview

Interval problems deal with ranges defined by a start and end point — time slots, meeting schedules, coordinate spans, and similar structures. The core challenge is reasoning about how intervals relate to one another: do they overlap, are they adjacent, does one contain the other, or are they completely disjoint?

Nearly every interval problem begins the same way: sort the intervals by their start (or end) point so that you can process them in a single left-to-right sweep. Once sorted, overlapping or adjacent intervals become neighbours in the array, which lets you merge, count, or partition them with greedy logic. This is why a solid grasp of sorting algorithms and greedy strategies is essential before tackling this topic.

Common patterns include merging overlapping intervals, inserting a new interval into a sorted list, finding the minimum number of resources to cover all intervals (the "meeting rooms" family), and computing intersections between two interval lists. Mastering these patterns builds strong intuition for sweep-line techniques and scheduling problems that appear frequently in interviews and real-world systems.

---

## Problem 1 — Merge Overlapping Intervals

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given an array of intervals where `intervals[i] = [start_i, end_i]`, merge all overlapping intervals and return an array of the non-overlapping intervals that cover all the ranges in the input.

Two intervals overlap if one starts before or when the other ends (i.e., `a.start <= b.end` and `b.start <= a.end` after sorting).

### Example

**Input:** `intervals = [[1,3], [2,6], [8,10], [15,18]]`
**Output:** `[[1,6], [8,10], [15,18]]`
**Explanation:** Intervals `[1,3]` and `[2,6]` overlap, so they merge into `[1,6]`. The remaining intervals are already disjoint.

<details>
<summary>Hints</summary>

1. Sort the intervals by their start value. After sorting, overlapping intervals will be adjacent in the array.
2. Walk through the sorted list and compare each interval's start with the end of the last merged interval. If they overlap, extend the end; otherwise, start a new merged interval.

</details>

<details>
<summary>Solution</summary>

**Approach:** Sort by start, then greedily merge consecutive overlapping intervals.

```
function merge(intervals):
    sort intervals by start value
    merged = [intervals[0]]

    for i from 1 to length(intervals) - 1:
        last = merged[length(merged) - 1]
        if intervals[i].start <= last.end:
            last.end = max(last.end, intervals[i].end)
        else:
            merged.append(intervals[i])

    return merged
```

O(n log n) time for sorting, O(n) space for the output.

</details>

---

## Problem 2 — Insert Interval

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

You are given a sorted (by start) array of non-overlapping intervals and a new interval. Insert the new interval into the array, merging with any existing intervals that overlap, and return the resulting array of non-overlapping intervals. The output must remain sorted by start.

### Example

**Input:** `intervals = [[1,3], [6,9]]`, `newInterval = [2,5]`
**Output:** `[[1,5], [6,9]]`
**Explanation:** The new interval `[2,5]` overlaps with `[1,3]`, producing `[1,5]`. Interval `[6,9]` is untouched.

**Input:** `intervals = [[1,2], [3,5], [6,7], [8,10], [12,16]]`, `newInterval = [4,8]`
**Output:** `[[1,2], [3,10], [12,16]]`
**Explanation:** `[4,8]` overlaps with `[3,5]`, `[6,7]`, and `[8,10]`, merging them into `[3,10]`.

<details>
<summary>Hints</summary>

1. Split the work into three phases: (a) add all intervals that end before the new interval starts, (b) merge all intervals that overlap with the new interval, (c) add all intervals that start after the new interval ends.
2. During the merge phase, keep expanding the new interval's start and end to absorb each overlapping interval.

</details>

<details>
<summary>Solution</summary>

**Approach:** Three-pass linear scan — before, overlap, after.

```
function insert(intervals, newInterval):
    result = []
    i = 0
    n = length(intervals)

    // Phase 1: intervals entirely before newInterval
    while i < n and intervals[i].end < newInterval.start:
        result.append(intervals[i])
        i += 1

    // Phase 2: merge overlapping intervals
    while i < n and intervals[i].start <= newInterval.end:
        newInterval.start = min(newInterval.start, intervals[i].start)
        newInterval.end   = max(newInterval.end,   intervals[i].end)
        i += 1
    result.append(newInterval)

    // Phase 3: intervals entirely after newInterval
    while i < n:
        result.append(intervals[i])
        i += 1

    return result
```

O(n) time and space.

</details>

---

## Problem 3 — Minimum Number of Meeting Rooms

**Difficulty:** Hard
**Estimated Time:** 45 minutes

### Problem Statement

Given an array of meeting time intervals `intervals[i] = [start_i, end_i]`, determine the minimum number of conference rooms required so that no two overlapping meetings share a room.

### Example

**Input:** `intervals = [[0,30], [5,10], [15,20]]`
**Output:** `2`
**Explanation:** The meeting `[0,30]` overlaps with both `[5,10]` and `[15,20]`, but `[5,10]` and `[15,20]` do not overlap with each other. Two rooms are enough: one for `[0,30]` and one shared sequentially by `[5,10]` then `[15,20]`.

<details>
<summary>Hints</summary>

1. Think of each meeting as two events: a "start" event and an "end" event. The number of rooms needed at any moment equals the number of meetings that have started but not yet ended.
2. Separate all start times and end times into two sorted arrays. Sweep through them with two pointers: increment a counter on a start, decrement on an end. Track the maximum counter value.

</details>

<details>
<summary>Solution</summary>

**Approach:** Event-based sweep line with two sorted arrays.

```
function minMeetingRooms(intervals):
    starts = sort([iv.start for iv in intervals])
    ends   = sort([iv.end   for iv in intervals])

    rooms = 0
    maxRooms = 0
    s = 0
    e = 0

    while s < length(starts):
        if starts[s] < ends[e]:
            rooms += 1
            maxRooms = max(maxRooms, rooms)
            s += 1
        else:
            rooms -= 1
            e += 1

    return maxRooms
```

O(n log n) time for sorting, O(n) space for the event arrays.

</details>

---

## Problem 4 — Non-Overlapping Intervals (Minimum Removals)

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Given an array of intervals `intervals[i] = [start_i, end_i]`, find the minimum number of intervals you need to remove so that the remaining intervals are pairwise non-overlapping. Two intervals overlap if one starts strictly before the other ends.

### Example

**Input:** `intervals = [[1,2], [2,3], [3,4], [1,3]]`
**Output:** `1`
**Explanation:** Removing `[1,3]` leaves `[1,2], [2,3], [3,4]`, which are non-overlapping (touching endpoints are allowed).

**Input:** `intervals = [[1,2], [1,2], [1,2]]`
**Output:** `2`
**Explanation:** Two of the three identical intervals must be removed.

<details>
<summary>Hints</summary>

1. This is equivalent to finding the maximum number of non-overlapping intervals you can keep, then subtracting from the total count.
2. Sort by end time. Greedily pick the interval that ends earliest and skip any interval whose start is before the current end. This is the classic activity-selection / interval-scheduling greedy strategy.

</details>

<details>
<summary>Solution</summary>

**Approach:** Greedy interval scheduling — sort by end, keep as many non-overlapping intervals as possible.

```
function eraseOverlapIntervals(intervals):
    sort intervals by end value
    kept = 0
    currentEnd = -infinity

    for each interval in intervals:
        if interval.start >= currentEnd:
            kept += 1
            currentEnd = interval.end

    return length(intervals) - kept
```

O(n log n) time for sorting, O(1) extra space.

</details>

---

## Problem 5 — Interval List Intersections

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

You are given two lists of closed intervals, `firstList` and `secondList`, where each list is pairwise disjoint and sorted by start. Return the intersection of these two interval lists. A closed interval `[a, b]` (with `a <= b`) denotes the set of real numbers `x` with `a <= x <= b`. The intersection of two closed intervals is either empty or a closed interval.

### Example

**Input:**
`firstList = [[0,2], [5,10], [13,23], [24,25]]`
`secondList = [[1,5], [8,12], [15,24], [25,26]]`
**Output:** `[[1,2], [5,5], [8,10], [15,23], [24,24], [25,25]]`
**Explanation:** Each output interval is the overlap between one interval from each list. For instance, `[0,2]` and `[1,5]` intersect at `[1,2]`.

<details>
<summary>Hints</summary>

1. Use two pointers, one for each list. At each step, check whether the current pair of intervals overlaps. If they do, the intersection is `[max(start1, start2), min(end1, end2)]`.
2. After processing a pair, advance the pointer whose current interval ends first, because that interval cannot overlap with anything further in the other list.

</details>

<details>
<summary>Solution</summary>

**Approach:** Two-pointer merge with overlap detection.

```
function intervalIntersection(firstList, secondList):
    result = []
    i = 0
    j = 0

    while i < length(firstList) and j < length(secondList):
        lo = max(firstList[i].start, secondList[j].start)
        hi = min(firstList[i].end,   secondList[j].end)

        if lo <= hi:
            result.append([lo, hi])

        // Advance the pointer with the smaller end
        if firstList[i].end < secondList[j].end:
            i += 1
        else:
            j += 1

    return result
```

O(n + m) time and space, where n and m are the lengths of the two lists.

</details>
