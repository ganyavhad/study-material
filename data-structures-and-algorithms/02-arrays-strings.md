# 02 — Arrays & Strings

> The most fundamental and frequently tested data structures — mastered in TypeScript.

---

## Table of Contents

1. [Arrays in JavaScript/TypeScript](#1-arrays-in-javascripttypescript)
2. [Key Patterns](#2-key-patterns)
3. [Two Pointers](#3-two-pointers)
4. [Sliding Window](#4-sliding-window)
5. [Prefix Sum](#5-prefix-sum)
6. [Kadane's Algorithm](#6-kadanes-algorithm)
7. [String Techniques](#7-string-techniques)
8. [Interview Questions & Answers](#8-interview-questions--answers)
9. [Practice Problems](#9-practice-problems)

---

## 1. Arrays in JavaScript/TypeScript

### Internal Representation

```typescript
// JavaScript arrays are dynamic — they grow/shrink automatically
// V8 stores them as:
// - Packed SMI arrays: [1, 2, 3] — fastest
// - Packed Double arrays: [1.1, 2.2] — fast
// - Packed Object arrays: [{}, "a"] — slower
// - Holey arrays: [1, , 3] — slowest (has gaps)

// TypeScript typed arrays for performance-critical code:
const buffer = new ArrayBuffer(16);
const int32 = new Int32Array(buffer);  // Fixed-size, no boxing
const float64 = new Float64Array(4);   // 4 doubles, contiguous memory
```

### Essential Array Operations

```typescript
// Creation
const arr1: number[] = [1, 2, 3, 4, 5];
const arr2: number[] = Array.from({ length: 5 }, (_, i) => i + 1);
const arr3: number[] = new Array(5).fill(0);
const matrix: number[][] = Array.from({ length: 3 }, () => new Array(3).fill(0));

// Destructuring & Spread
const [first, second, ...rest] = arr1;
const combined = [...arr1, ...arr2];

// Immutable operations (functional style for Node.js APIs)
const doubled = arr1.map(x => x * 2);
const evens = arr1.filter(x => x % 2 === 0);
const sum = arr1.reduce((acc, val) => acc + val, 0);
```

---

## 2. Key Patterns

```
When to use which pattern:

┌─────────────────────────┬────────────────────────────────────────┐
│ Pattern                 │ Use When                               │
├─────────────────────────┼────────────────────────────────────────┤
│ Two Pointers            │ Sorted arrays, pairs, palindromes      │
│ Sliding Window          │ Subarrays, substrings of fixed/var len │
│ Prefix Sum              │ Range sum queries                      │
│ Kadane's Algorithm      │ Maximum subarray sum                   │
│ Hash Map Lookup         │ Finding pairs, frequency counting      │
│ Sort + Two Pointers     │ Three sum, closest pair                │
│ Binary Search           │ Sorted array, rotated array search     │
└─────────────────────────┴────────────────────────────────────────┘
```

---

## 3. Two Pointers

### Pattern: Opposite Direction

```typescript
// Two Sum (sorted array) — O(n) time, O(1) space
function twoSumSorted(nums: number[], target: number): [number, number] {
  let left = 0;
  let right = nums.length - 1;

  while (left < right) {
    const sum = nums[left] + nums[right];
    if (sum === target) return [left, right];
    if (sum < target) left++;
    else right--;
  }
  return [-1, -1];
}

// Valid Palindrome — O(n) time, O(1) space
function isPalindrome(s: string): boolean {
  const cleaned = s.toLowerCase().replace(/[^a-z0-9]/g, '');
  let left = 0;
  let right = cleaned.length - 1;

  while (left < right) {
    if (cleaned[left] !== cleaned[right]) return false;
    left++;
    right--;
  }
  return true;
}

// Container With Most Water — O(n) time, O(1) space
function maxArea(height: number[]): number {
  let left = 0, right = height.length - 1;
  let maxWater = 0;

  while (left < right) {
    const area = Math.min(height[left], height[right]) * (right - left);
    maxWater = Math.max(maxWater, area);
    if (height[left] < height[right]) left++;
    else right--;
  }
  return maxWater;
}
```

### Pattern: Same Direction (Fast & Slow)

```typescript
// Remove Duplicates from Sorted Array — O(n) time, O(1) space
function removeDuplicates(nums: number[]): number {
  if (nums.length === 0) return 0;
  let slow = 0;

  for (let fast = 1; fast < nums.length; fast++) {
    if (nums[fast] !== nums[slow]) {
      slow++;
      nums[slow] = nums[fast];
    }
  }
  return slow + 1;
}

// Move Zeroes — O(n) time, O(1) space
function moveZeroes(nums: number[]): void {
  let slow = 0;
  for (let fast = 0; fast < nums.length; fast++) {
    if (nums[fast] !== 0) {
      [nums[slow], nums[fast]] = [nums[fast], nums[slow]];
      slow++;
    }
  }
}
```

### Three Sum — O(n²) time, O(1) space

```typescript
function threeSum(nums: number[]): number[][] {
  nums.sort((a, b) => a - b);
  const result: number[][] = [];

  for (let i = 0; i < nums.length - 2; i++) {
    if (i > 0 && nums[i] === nums[i - 1]) continue; // skip duplicates

    let left = i + 1;
    let right = nums.length - 1;

    while (left < right) {
      const sum = nums[i] + nums[left] + nums[right];
      if (sum === 0) {
        result.push([nums[i], nums[left], nums[right]]);
        while (left < right && nums[left] === nums[left + 1]) left++;
        while (left < right && nums[right] === nums[right - 1]) right--;
        left++;
        right--;
      } else if (sum < 0) {
        left++;
      } else {
        right--;
      }
    }
  }
  return result;
}
```

---

## 4. Sliding Window

### Fixed-Size Window

```typescript
// Maximum Sum of Subarray of Size K — O(n) time, O(1) space
function maxSubarraySum(arr: number[], k: number): number {
  let windowSum = 0;
  let maxSum = -Infinity;

  for (let i = 0; i < arr.length; i++) {
    windowSum += arr[i];
    if (i >= k - 1) {
      maxSum = Math.max(maxSum, windowSum);
      windowSum -= arr[i - k + 1];
    }
  }
  return maxSum;
}

// Node.js use case: Calculate rolling average of API response times
function rollingAverage(responseTimes: number[], windowSize: number): number[] {
  const averages: number[] = [];
  let windowSum = 0;

  for (let i = 0; i < responseTimes.length; i++) {
    windowSum += responseTimes[i];
    if (i >= windowSize - 1) {
      averages.push(windowSum / windowSize);
      windowSum -= responseTimes[i - windowSize + 1];
    }
  }
  return averages;
}
```

### Variable-Size Window

```typescript
// Longest Substring Without Repeating Characters — O(n) time, O(min(n,m)) space
function lengthOfLongestSubstring(s: string): number {
  const charIndex = new Map<string, number>();
  let maxLength = 0;
  let left = 0;

  for (let right = 0; right < s.length; right++) {
    if (charIndex.has(s[right]) && charIndex.get(s[right])! >= left) {
      left = charIndex.get(s[right])! + 1;
    }
    charIndex.set(s[right], right);
    maxLength = Math.max(maxLength, right - left + 1);
  }
  return maxLength;
}

// Minimum Window Substring — O(n) time, O(m) space
function minWindow(s: string, t: string): string {
  const need = new Map<string, number>();
  for (const c of t) need.set(c, (need.get(c) || 0) + 1);

  let have = 0;
  const required = need.size;
  const window = new Map<string, number>();
  let result = '';
  let minLen = Infinity;
  let left = 0;

  for (let right = 0; right < s.length; right++) {
    const c = s[right];
    window.set(c, (window.get(c) || 0) + 1);

    if (need.has(c) && window.get(c) === need.get(c)) have++;

    while (have === required) {
      if (right - left + 1 < minLen) {
        minLen = right - left + 1;
        result = s.substring(left, right + 1);
      }
      const leftChar = s[left];
      window.set(leftChar, window.get(leftChar)! - 1);
      if (need.has(leftChar) && window.get(leftChar)! < need.get(leftChar)!) {
        have--;
      }
      left++;
    }
  }
  return result;
}
```

---

## 5. Prefix Sum

```typescript
// Build prefix sum array — O(n) time, O(n) space
function buildPrefixSum(arr: number[]): number[] {
  const prefix: number[] = [0];
  for (let i = 0; i < arr.length; i++) {
    prefix.push(prefix[i] + arr[i]);
  }
  return prefix;
}

// Range sum query in O(1) after O(n) preprocessing
function rangeSum(prefix: number[], left: number, right: number): number {
  return prefix[right + 1] - prefix[left];
}

// Usage:
const nums = [1, 2, 3, 4, 5];
const prefix = buildPrefixSum(nums);
console.log(rangeSum(prefix, 1, 3)); // sum of index 1..3 = 2+3+4 = 9

// Subarray Sum Equals K — O(n) time, O(n) space
function subarraySum(nums: number[], k: number): number {
  const prefixCount = new Map<number, number>([[0, 1]]);
  let sum = 0;
  let count = 0;

  for (const num of nums) {
    sum += num;
    if (prefixCount.has(sum - k)) {
      count += prefixCount.get(sum - k)!;
    }
    prefixCount.set(sum, (prefixCount.get(sum) || 0) + 1);
  }
  return count;
}
```

---

## 6. Kadane's Algorithm

```typescript
// Maximum Subarray Sum — O(n) time, O(1) space
function maxSubArray(nums: number[]): number {
  let currentMax = nums[0];
  let globalMax = nums[0];

  for (let i = 1; i < nums.length; i++) {
    currentMax = Math.max(nums[i], currentMax + nums[i]);
    globalMax = Math.max(globalMax, currentMax);
  }
  return globalMax;
}

// Variant: Return the subarray itself
function maxSubArrayWithIndices(nums: number[]): { sum: number; start: number; end: number } {
  let currentMax = nums[0];
  let globalMax = nums[0];
  let start = 0, end = 0, tempStart = 0;

  for (let i = 1; i < nums.length; i++) {
    if (nums[i] > currentMax + nums[i]) {
      currentMax = nums[i];
      tempStart = i;
    } else {
      currentMax += nums[i];
    }

    if (currentMax > globalMax) {
      globalMax = currentMax;
      start = tempStart;
      end = i;
    }
  }
  return { sum: globalMax, start, end };
}

// Node.js use case: Find the most profitable consecutive trading period
// Given daily profit/loss array, find max consecutive profit window
```

---

## 7. String Techniques

### Frequency Counting & Anagrams

```typescript
// Valid Anagram — O(n) time, O(1) space (26 letters)
function isAnagram(s: string, t: string): boolean {
  if (s.length !== t.length) return false;
  const count = new Map<string, number>();

  for (const c of s) count.set(c, (count.get(c) || 0) + 1);
  for (const c of t) {
    if (!count.has(c) || count.get(c) === 0) return false;
    count.set(c, count.get(c)! - 1);
  }
  return true;
}

// Group Anagrams — O(n * k log k) time
function groupAnagrams(strs: string[]): string[][] {
  const map = new Map<string, string[]>();

  for (const str of strs) {
    const key = str.split('').sort().join('');
    if (!map.has(key)) map.set(key, []);
    map.get(key)!.push(str);
  }
  return Array.from(map.values());
}

// Optimized Group Anagrams — O(n * k) time using character count as key
function groupAnagramsOptimized(strs: string[]): string[][] {
  const map = new Map<string, string[]>();

  for (const str of strs) {
    const count = new Array(26).fill(0);
    for (const c of str) count[c.charCodeAt(0) - 97]++;
    const key = count.join('#');
    if (!map.has(key)) map.set(key, []);
    map.get(key)!.push(str);
  }
  return Array.from(map.values());
}
```

### String Reversal & Manipulation

```typescript
// Reverse Words in a String — O(n) time
function reverseWords(s: string): string {
  return s.trim().split(/\s+/).reverse().join(' ');
}

// Longest Common Prefix — O(n * m) time
function longestCommonPrefix(strs: string[]): string {
  if (strs.length === 0) return '';
  let prefix = strs[0];

  for (let i = 1; i < strs.length; i++) {
    while (strs[i].indexOf(prefix) !== 0) {
      prefix = prefix.slice(0, -1);
      if (prefix === '') return '';
    }
  }
  return prefix;
}

// Longest Palindromic Substring — O(n²) time, O(1) space
function longestPalindrome(s: string): string {
  let result = '';

  function expandFromCenter(left: number, right: number): void {
    while (left >= 0 && right < s.length && s[left] === s[right]) {
      if (right - left + 1 > result.length) {
        result = s.substring(left, right + 1);
      }
      left--;
      right++;
    }
  }

  for (let i = 0; i < s.length; i++) {
    expandFromCenter(i, i);     // Odd length palindromes
    expandFromCenter(i, i + 1); // Even length palindromes
  }
  return result;
}
```

---

## 8. Interview Questions & Answers

### Q1: How does JavaScript's `Array.sort()` work internally?

**Answer:**
- V8 uses **TimSort** (hybrid of merge sort + insertion sort).
- Time: O(n log n) average and worst; O(n) best (nearly sorted data).
- Space: O(n) for the merge step.
- **Important:** Default sort is lexicographic! `[10, 2, 1].sort()` → `[1, 10, 2]`.
- Always pass a comparator: `.sort((a, b) => a - b)` for numeric sort.

### Q2: When would you use a TypedArray instead of a regular Array?

**Answer:**
- When working with **binary data** (WebSocket buffers, file I/O, crypto).
- When you need **fixed-size, contiguous memory** for performance.
- `Buffer` in Node.js is backed by `Uint8Array`.
- TypedArrays don't have `.push()`, `.pop()`, etc. — fixed length.

### Q3: Explain the two-pointer technique and when to use it.

**Answer:**
- Use **two pointers** when the array is sorted or you're searching for pairs.
- **Opposite direction:** Start one pointer at each end, move toward middle (e.g., two sum, palindrome).
- **Same direction:** One slow, one fast pointer (e.g., remove duplicates, detect cycles).
- Reduces O(n²) brute force to O(n) in many cases.

### Q4: What's the difference between `slice()` and `splice()`?

**Answer:**
- `slice(start, end)`: Returns a **new** array, doesn't modify original. O(k) where k = slice length.
- `splice(start, deleteCount, ...items)`: **Mutates** the original array. O(n) because elements shift.
- Use `slice` for immutable operations (functional style in Redux/NgRx).
- Use `splice` for in-place mutations.

### Q5: How would you efficiently find if a string contains all characters of another string?

**Answer:**
- Build a frequency map of the target string.
- Iterate through the source string, decrementing counts.
- When all counts reach 0, all characters are found.
- Time: O(n + m), Space: O(m).
- This is the basis of the **Minimum Window Substring** problem.

---

## 9. Practice Problems

| # | Problem | Difficulty | Pattern |
|---|---------|------------|---------|
| 1 | Two Sum | 🟢 Easy | Hash Map |
| 2 | Best Time to Buy and Sell Stock | 🟢 Easy | Kadane's / One Pass |
| 3 | Contains Duplicate | 🟢 Easy | Hash Set |
| 4 | Valid Anagram | 🟢 Easy | Frequency Count |
| 5 | Maximum Subarray | 🟡 Medium | Kadane's Algorithm |
| 6 | Product of Array Except Self | 🟡 Medium | Prefix/Suffix |
| 7 | 3Sum | 🟡 Medium | Sort + Two Pointers |
| 8 | Longest Substring Without Repeating Chars | 🟡 Medium | Sliding Window |
| 9 | Minimum Window Substring | 🔴 Hard | Sliding Window |
| 10 | Trapping Rain Water | 🔴 Hard | Two Pointers |

---

**Previous:** [01 — DSA Fundamentals & Big-O ←](01-dsa-fundamentals-bigO.md) | **Next:** [03 — Linked Lists →](03-linked-lists.md)
