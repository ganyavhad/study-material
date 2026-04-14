# 07 — Sorting & Searching

> Mastering the core algorithms that power database queries, API responses, and system design.

---

## Table of Contents

1. [Sorting Algorithms Overview](#1-sorting-algorithms-overview)
2. [QuickSort](#2-quicksort)
3. [MergeSort](#3-mergesort)
4. [Counting & Radix Sort](#4-counting--radix-sort)
5. [Binary Search & Variants](#5-binary-search--variants)
6. [Search in Rotated Array](#6-search-in-rotated-array)
7. [Kth Element Problems](#7-kth-element-problems)
8. [Interview Questions & Answers](#8-interview-questions--answers)
9. [Practice Problems](#9-practice-problems)

---

## 1. Sorting Algorithms Overview

```
┌──────────────────┬──────────┬──────────┬──────────┬────────┬──────────┐
│ Algorithm        │ Best     │ Average  │ Worst    │ Space  │ Stable?  │
├──────────────────┼──────────┼──────────┼──────────┼────────┼──────────┤
│ Bubble Sort      │ O(n)     │ O(n²)    │ O(n²)    │ O(1)   │ Yes      │
│ Selection Sort   │ O(n²)    │ O(n²)    │ O(n²)    │ O(1)   │ No       │
│ Insertion Sort   │ O(n)     │ O(n²)    │ O(n²)    │ O(1)   │ Yes      │
│ Merge Sort       │ O(n lg n)│ O(n lg n)│ O(n lg n)│ O(n)   │ Yes      │
│ Quick Sort       │ O(n lg n)│ O(n lg n)│ O(n²)    │ O(lg n)│ No       │
│ Heap Sort        │ O(n lg n)│ O(n lg n)│ O(n lg n)│ O(1)   │ No       │
│ Tim Sort (JS)    │ O(n)     │ O(n lg n)│ O(n lg n)│ O(n)   │ Yes      │
│ Counting Sort    │ O(n+k)   │ O(n+k)   │ O(n+k)   │ O(k)   │ Yes      │
│ Radix Sort       │ O(nk)    │ O(nk)    │ O(nk)    │ O(n+k) │ Yes      │
└──────────────────┴──────────┴──────────┴──────────┴────────┴──────────┘

Stable = equal elements maintain their relative order
```

### Basic Sorts (for understanding — rarely used in production)

```typescript
// Bubble Sort — O(n²) time, O(1) space
function bubbleSort(arr: number[]): number[] {
  const n = arr.length;
  for (let i = 0; i < n - 1; i++) {
    let swapped = false;
    for (let j = 0; j < n - 1 - i; j++) {
      if (arr[j] > arr[j + 1]) {
        [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
        swapped = true;
      }
    }
    if (!swapped) break; // Already sorted — O(n) best case
  }
  return arr;
}

// Insertion Sort — O(n²) avg, O(n) best (nearly sorted)
function insertionSort(arr: number[]): number[] {
  for (let i = 1; i < arr.length; i++) {
    const key = arr[i];
    let j = i - 1;
    while (j >= 0 && arr[j] > key) {
      arr[j + 1] = arr[j];
      j--;
    }
    arr[j + 1] = key;
  }
  return arr;
}
// Tim Sort uses insertion sort for small subarrays (< 64 elements)
```

---

## 2. QuickSort

```typescript
// QuickSort — O(n log n) avg, O(n²) worst, O(log n) space
function quickSort(arr: number[], low = 0, high = arr.length - 1): number[] {
  if (low < high) {
    const pivotIndex = partition(arr, low, high);
    quickSort(arr, low, pivotIndex - 1);
    quickSort(arr, pivotIndex + 1, high);
  }
  return arr;
}

function partition(arr: number[], low: number, high: number): number {
  // Median-of-three pivot selection (avoids O(n²) on sorted input)
  const mid = Math.floor((low + high) / 2);
  if (arr[low] > arr[mid]) [arr[low], arr[mid]] = [arr[mid], arr[low]];
  if (arr[low] > arr[high]) [arr[low], arr[high]] = [arr[high], arr[low]];
  if (arr[mid] > arr[high]) [arr[mid], arr[high]] = [arr[high], arr[mid]];
  [arr[mid], arr[high]] = [arr[high], arr[mid]]; // pivot at end

  const pivot = arr[high];
  let i = low - 1;

  for (let j = low; j < high; j++) {
    if (arr[j] <= pivot) {
      i++;
      [arr[i], arr[j]] = [arr[j], arr[i]];
    }
  }
  [arr[i + 1], arr[high]] = [arr[high], arr[i + 1]];
  return i + 1;
}

// 3-Way QuickSort (Dutch National Flag) — handles many duplicates
function quickSort3Way(arr: number[], low = 0, high = arr.length - 1): void {
  if (low >= high) return;
  const pivot = arr[low];
  let lt = low, gt = high, i = low;

  while (i <= gt) {
    if (arr[i] < pivot) {
      [arr[lt], arr[i]] = [arr[i], arr[lt]];
      lt++; i++;
    } else if (arr[i] > pivot) {
      [arr[i], arr[gt]] = [arr[gt], arr[i]];
      gt--;
    } else {
      i++;
    }
  }
  quickSort3Way(arr, low, lt - 1);
  quickSort3Way(arr, gt + 1, high);
}
```

---

## 3. MergeSort

```typescript
// MergeSort — O(n log n) guaranteed, O(n) space, stable
function mergeSort(arr: number[]): number[] {
  if (arr.length <= 1) return arr;
  const mid = Math.floor(arr.length / 2);
  const left = mergeSort(arr.slice(0, mid));
  const right = mergeSort(arr.slice(mid));
  return merge(left, right);
}

function merge(left: number[], right: number[]): number[] {
  const result: number[] = [];
  let i = 0, j = 0;

  while (i < left.length && j < right.length) {
    if (left[i] <= right[j]) result.push(left[i++]);
    else result.push(right[j++]);
  }
  while (i < left.length) result.push(left[i++]);
  while (j < right.length) result.push(right[j++]);
  return result;
}

// In-place MergeSort (for interviews — reduces space to O(1) auxiliary)
// Uses bottom-up approach with in-place merging
function mergeSortInPlace(arr: number[]): void {
  const n = arr.length;
  for (let size = 1; size < n; size *= 2) {
    for (let start = 0; start < n - size; start += 2 * size) {
      const mid = start + size - 1;
      const end = Math.min(start + 2 * size - 1, n - 1);
      mergeInPlace(arr, start, mid, end);
    }
  }
}

function mergeInPlace(arr: number[], start: number, mid: number, end: number): void {
  const leftArr = arr.slice(start, mid + 1);
  const rightArr = arr.slice(mid + 1, end + 1);
  let i = 0, j = 0, k = start;

  while (i < leftArr.length && j < rightArr.length) {
    if (leftArr[i] <= rightArr[j]) arr[k++] = leftArr[i++];
    else arr[k++] = rightArr[j++];
  }
  while (i < leftArr.length) arr[k++] = leftArr[i++];
  while (j < rightArr.length) arr[k++] = rightArr[j++];
}

// Sort Linked List — O(n log n) time, O(log n) space (recursion stack)
// MergeSort is ideal for linked lists — no random access needed
```

---

## 4. Counting & Radix Sort

```typescript
// Counting Sort — O(n + k) time, O(k) space
// Best when k (range) is small relative to n
function countingSort(arr: number[], maxVal: number): number[] {
  const count = new Array(maxVal + 1).fill(0);
  for (const num of arr) count[num]++;

  const result: number[] = [];
  for (let i = 0; i <= maxVal; i++) {
    while (count[i] > 0) {
      result.push(i);
      count[i]--;
    }
  }
  return result;
}

// Radix Sort — O(d * (n + k)) where d = digits, k = base
function radixSort(arr: number[]): number[] {
  const max = Math.max(...arr);
  let exp = 1;

  while (Math.floor(max / exp) > 0) {
    arr = countingSortByDigit(arr, exp);
    exp *= 10;
  }
  return arr;
}

function countingSortByDigit(arr: number[], exp: number): number[] {
  const output = new Array(arr.length);
  const count = new Array(10).fill(0);

  for (const num of arr) count[Math.floor(num / exp) % 10]++;
  for (let i = 1; i < 10; i++) count[i] += count[i - 1];

  for (let i = arr.length - 1; i >= 0; i--) {
    const digit = Math.floor(arr[i] / exp) % 10;
    output[count[digit] - 1] = arr[i];
    count[digit]--;
  }
  return output;
}

// Node.js use case: Sort user IDs, sort by age (bounded range)
```

---

## 5. Binary Search & Variants

```typescript
// Classic Binary Search — O(log n) time, O(1) space
function binarySearch(arr: number[], target: number): number {
  let left = 0, right = arr.length - 1;
  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2); // Avoids overflow
    if (arr[mid] === target) return mid;
    if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  return -1;
}

// Find First Occurrence (leftmost) — O(log n)
function findFirst(arr: number[], target: number): number {
  let left = 0, right = arr.length - 1, result = -1;
  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2);
    if (arr[mid] === target) {
      result = mid;
      right = mid - 1; // Keep searching left
    } else if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  return result;
}

// Find Last Occurrence (rightmost) — O(log n)
function findLast(arr: number[], target: number): number {
  let left = 0, right = arr.length - 1, result = -1;
  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2);
    if (arr[mid] === target) {
      result = mid;
      left = mid + 1; // Keep searching right
    } else if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  return result;
}

// Binary Search on Answer — O(n * log(maxVal))
// "Can we achieve X?" → binary search on X
function minEatingSpeed(piles: number[], h: number): number {
  let left = 1;
  let right = Math.max(...piles);

  while (left < right) {
    const mid = Math.floor((left + right) / 2);
    const hours = piles.reduce((sum, pile) => sum + Math.ceil(pile / mid), 0);
    if (hours <= h) right = mid;
    else left = mid + 1;
  }
  return left;
}

// Search Insert Position — O(log n)
function searchInsert(nums: number[], target: number): number {
  let left = 0, right = nums.length - 1;
  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    if (nums[mid] === target) return mid;
    if (nums[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  return left;
}

// Find Peak Element — O(log n)
function findPeakElement(nums: number[]): number {
  let left = 0, right = nums.length - 1;
  while (left < right) {
    const mid = Math.floor((left + right) / 2);
    if (nums[mid] < nums[mid + 1]) left = mid + 1;
    else right = mid;
  }
  return left;
}
```

---

## 6. Search in Rotated Array

```typescript
// Search Rotated Sorted Array — O(log n)
function searchRotated(nums: number[], target: number): number {
  let left = 0, right = nums.length - 1;

  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    if (nums[mid] === target) return mid;

    // Left half is sorted
    if (nums[left] <= nums[mid]) {
      if (target >= nums[left] && target < nums[mid]) {
        right = mid - 1;
      } else {
        left = mid + 1;
      }
    }
    // Right half is sorted
    else {
      if (target > nums[mid] && target <= nums[right]) {
        left = mid + 1;
      } else {
        right = mid - 1;
      }
    }
  }
  return -1;
}

// Find Minimum in Rotated Sorted Array — O(log n)
function findMin(nums: number[]): number {
  let left = 0, right = nums.length - 1;

  while (left < right) {
    const mid = Math.floor((left + right) / 2);
    if (nums[mid] > nums[right]) left = mid + 1;
    else right = mid;
  }
  return nums[left];
}
```

---

## 7. Kth Element Problems

```typescript
// Kth Largest — QuickSelect O(n) average, O(n²) worst
function findKthLargest(nums: number[], k: number): number {
  const targetIndex = nums.length - k;
  return quickSelect(nums, 0, nums.length - 1, targetIndex);
}

function quickSelect(
  arr: number[], low: number, high: number, target: number
): number {
  const pivotIndex = partitionRandom(arr, low, high);
  if (pivotIndex === target) return arr[pivotIndex];
  if (pivotIndex < target) return quickSelect(arr, pivotIndex + 1, high, target);
  return quickSelect(arr, low, pivotIndex - 1, target);
}

function partitionRandom(arr: number[], low: number, high: number): number {
  const randomIdx = low + Math.floor(Math.random() * (high - low + 1));
  [arr[randomIdx], arr[high]] = [arr[high], arr[randomIdx]];

  const pivot = arr[high];
  let i = low - 1;
  for (let j = low; j < high; j++) {
    if (arr[j] <= pivot) {
      i++;
      [arr[i], arr[j]] = [arr[j], arr[i]];
    }
  }
  [arr[i + 1], arr[high]] = [arr[high], arr[i + 1]];
  return i + 1;
}

// Merge K Sorted Arrays using Min-Heap — O(n log k)
// See Stacks & Queues module for MinHeap implementation
```

---

## 8. Interview Questions & Answers

### Q1: Why does JavaScript use TimSort instead of QuickSort?

**Answer:**
- TimSort is **stable** (equal elements keep their original order) — crucial for sorting objects by multiple keys.
- TimSort performs well on **partially sorted data** (O(n) best case) which is common in real applications.
- QuickSort is unstable and O(n²) worst case on sorted data.
- V8 switched from QuickSort to TimSort in 2019 for `Array.sort()`.

### Q2: When would you use binary search in a Node.js backend?

**Answer:**
- Searching sorted **in-memory datasets** (cached DB results, config lookups).
- **Rate limiting:** binary search on sorted timestamp arrays.
- **Pagination:** finding the starting point in sorted results.
- **Database internals:** B-tree indexes use binary search at each node.

### Q3: QuickSort vs MergeSort — which is better?

**Answer:**
- **QuickSort:** Better cache locality (in-place), O(log n) space, but O(n²) worst case.
- **MergeSort:** Guaranteed O(n log n), stable, but O(n) extra space.
- For arrays: QuickSort usually faster due to cache. For linked lists: MergeSort better (no random access needed).
- In practice: use the built-in `.sort()` which uses TimSort.

### Q4: What is "binary search on the answer"?

**Answer:**
- Instead of searching in an array, you search over **possible answer values**.
- If the problem asks "what is the minimum X such that condition holds?" — binary search on X.
- Examples: Koko eating bananas, ship capacity, split array largest sum.
- The key insight: if `f(x)` is monotonic (condition holds for all values ≥ threshold), use binary search.

---

## 9. Practice Problems

| # | Problem | Difficulty | Pattern |
|---|---------|------------|---------|
| 1 | Binary Search | 🟢 Easy | Classic Binary Search |
| 2 | Search Insert Position | 🟢 Easy | Binary Search |
| 3 | Sort Colors (Dutch Flag) | 🟡 Medium | 3-Way Partition |
| 4 | Kth Largest Element | 🟡 Medium | QuickSelect / Heap |
| 5 | Search in Rotated Array | 🟡 Medium | Modified Binary Search |
| 6 | Find Peak Element | 🟡 Medium | Binary Search |
| 7 | Find Minimum in Rotated Array | 🟡 Medium | Binary Search |
| 8 | Merge Intervals | 🟡 Medium | Sort + Sweep |
| 9 | Koko Eating Bananas | 🟡 Medium | Binary Search on Answer |
| 10 | Median of Two Sorted Arrays | 🔴 Hard | Binary Search |

---

**Previous:** [06 — Hash Maps & Sets ←](06-hashmaps-sets.md) | **Next:** [08 — Recursion & DP →](08-recursion-dp.md)
