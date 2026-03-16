# Bit Manipulation

**Track:** Algorithms & Data Structures
**Difficulty Tier:** Intermediate
**Prerequisites:** None

## Concept Overview

Bit manipulation is the practice of using bitwise operators — AND, OR, XOR, NOT, and shifts — to operate directly on the binary representations of integers. Because these operations map to single CPU instructions, they execute in constant time and are among the fastest primitives available. Many problems that look like they require hash sets, sorting, or arithmetic can be solved more elegantly (and more efficiently) with a handful of bit tricks.

The key insight behind most bit-manipulation techniques is that each bit in an integer is independent. XOR, for example, cancels identical bits and preserves differences, which is why it can isolate a unique element in a list of duplicates without any extra memory. Shifts let you inspect or construct numbers one bit at a time, and masks let you read or write specific bit positions. Combining these primitives unlocks solutions to counting, parity, subset enumeration, and power-of-two problems that would otherwise require more complex data structures.

Understanding bit manipulation also deepens your grasp of how computers represent data. Signed vs. unsigned integers, two's complement, and overflow behavior all become clearer once you think in binary. These fundamentals surface in systems programming, cryptography, graphics, and networking — anywhere performance or compact representation matters.

---

## Problem 1 — Single Number

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given a non-empty array of integers `nums` where every element appears exactly twice except for one element that appears exactly once, find that single element. Your solution must run in linear time and use constant extra space.

### Example

**Input:** `nums = [4, 1, 2, 1, 2]`
**Output:** `4`
**Explanation:** 1 and 2 each appear twice. 4 appears once, so it is the answer.

**Input:** `nums = [2, 2, 1]`
**Output:** `1`

<details>
<summary>Hints</summary>

1. XOR of a number with itself is 0, and XOR of a number with 0 is the number itself. What happens if you XOR all elements together?

</details>

<details>
<summary>Solution</summary>

**Approach:** XOR all elements. Pairs cancel to 0, leaving the unique element.

```
function singleNumber(nums):
    result = 0
    for num in nums:
        result = result XOR num
    return result
```

O(n) time, O(1) space.

</details>

---

## Problem 2 — Number of 1 Bits (Hamming Weight)

**Difficulty:** Medium
**Estimated Time:** 35 minutes

### Problem Statement

Given a positive integer `n`, return the number of `1` bits in its binary representation (also known as the Hamming weight).

### Example

**Input:** `n = 11` (binary `1011`)
**Output:** `3`
**Explanation:** The binary representation has three 1-bits.

**Input:** `n = 128` (binary `10000000`)
**Output:** `1`

<details>
<summary>Hints</summary>

1. The expression `n & (n - 1)` clears the lowest set bit of `n`. How many times can you apply this before `n` becomes 0?

</details>

<details>
<summary>Solution</summary>

**Approach:** Repeatedly clear the lowest set bit using `n & (n - 1)` and count iterations.

```
function hammingWeight(n):
    count = 0
    while n != 0:
        n = n AND (n - 1)
        count += 1
    return count
```

O(k) time where k is the number of set bits, O(1) space.

</details>

---

## Problem 3 — Counting Bits

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Given a non-negative integer `n`, return an array `ans` of length `n + 1` where `ans[i]` is the number of `1` bits in the binary representation of `i`, for every `0 <= i <= n`.

### Example

**Input:** `n = 5`
**Output:** `[0, 1, 1, 2, 1, 2]`
**Explanation:** Binary representations: 0→0, 1→1, 2→10, 3→11, 4→100, 5→101.

**Input:** `n = 2`
**Output:** `[0, 1, 1]`

<details>
<summary>Hints</summary>

1. You already know that `i & (i - 1)` drops the lowest set bit. That means `ans[i] = ans[i & (i - 1)] + 1`. Can you build the answer array bottom-up using this recurrence?

</details>

<details>
<summary>Solution</summary>

**Approach:** Dynamic programming using the relation `ans[i] = ans[i & (i - 1)] + 1`.

```
function countBits(n):
    ans = array of size (n + 1), initialized to 0
    for i from 1 to n:
        ans[i] = ans[i AND (i - 1)] + 1
    return ans
```

O(n) time, O(n) space (for the output array).

</details>

---

## Problem 4 — Reverse Bits

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Given a 32-bit unsigned integer `n`, reverse the order of its bits and return the resulting integer. For example, the input's least-significant bit becomes the output's most-significant bit.

### Example

**Input:** `n = 43261596` (binary `00000010100101000001111010011100`)
**Output:** `964176192` (binary `00111001011110000010100101000000`)
**Explanation:** Reading the input bits from right to left gives the output bits from left to right.

**Input:** `n = 4294967293` (binary `11111111111111111111111111111101`)
**Output:** `3221225471` (binary `10111111111111111111111111111111`)

<details>
<summary>Hints</summary>

1. Build the result one bit at a time: extract the lowest bit of `n`, shift the result left to make room, OR the extracted bit in, then shift `n` right. Repeat 32 times.
2. For a follow-up, consider a divide-and-conquer approach: swap adjacent single bits, then adjacent 2-bit groups, then 4-bit groups, and so on up to 16-bit halves.

</details>

<details>
<summary>Solution</summary>

**Approach:** Iterate through all 32 bits, building the reversed number from the bottom up.

```
function reverseBits(n):
    result = 0
    for i from 0 to 31:
        bit = n AND 1
        result = (result LEFT_SHIFT 1) OR bit
        n = n RIGHT_SHIFT 1
    return result
```

O(1) time (fixed 32 iterations), O(1) space.

</details>

---

## Problem 5 — Find Two Non-Repeating Elements

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Given an array of integers `nums` where every element appears exactly twice except for two elements that each appear exactly once, find and return those two unique elements. Your solution must run in linear time and use constant extra space. Return the two elements in any order.

### Example

**Input:** `nums = [1, 2, 1, 3, 2, 5]`
**Output:** `[3, 5]`
**Explanation:** 1 and 2 each appear twice. 3 and 5 each appear once.

**Input:** `nums = [0, 1, 0, 6]`
**Output:** `[1, 6]`

<details>
<summary>Hints</summary>

1. XOR of all elements gives you `a XOR b` where `a` and `b` are the two unique numbers. Since `a != b`, at least one bit in this XOR result is set.
2. Use any set bit from the XOR result as a mask to partition the array into two groups — one where that bit is set and one where it is not. Each group contains exactly one of the unique numbers, and you can XOR within each group to isolate it.

</details>

<details>
<summary>Solution</summary>

**Approach:** XOR all elements to get `a XOR b`, find a distinguishing bit, then partition and XOR each group.

```
function findTwoUnique(nums):
    xor_all = 0
    for num in nums:
        xor_all = xor_all XOR num

    // Isolate the lowest set bit (any set bit works)
    diff_bit = xor_all AND (-xor_all)

    a = 0
    b = 0
    for num in nums:
        if (num AND diff_bit) != 0:
            a = a XOR num
        else:
            b = b XOR num

    return [a, b]
```

O(n) time, O(1) space.

</details>
