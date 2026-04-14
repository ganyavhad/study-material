# 01 — DSA Fundamentals & Big-O Notation

> Understanding complexity analysis and how JavaScript/TypeScript handles data under the hood.

---

## Table of Contents

1. [Why DSA Matters for Node.js Developers](#1-why-dsa-matters-for-nodejs-developers)
2. [Big-O Notation](#2-big-o-notation)
3. [Time Complexity](#3-time-complexity)
4. [Space Complexity](#4-space-complexity)
5. [JavaScript Engine Internals](#5-javascript-engine-internals)
6. [Amortized Analysis](#6-amortized-analysis)
7. [Common Patterns & Complexity Cheat Sheet](#7-common-patterns--complexity-cheat-sheet)
8. [Interview Questions & Answers](#8-interview-questions--answers)

---

## 1. Why DSA Matters for Node.js Developers

```
Real-World Mapping:
┌─────────────────────────┬──────────────────────────────────┐
│ DSA Concept             │ Node.js Backend Use Case         │
├─────────────────────────┼──────────────────────────────────┤
│ Hash Maps               │ In-memory caching (Redis logic)  │
│ Queues                  │ BullMQ, SQS message processing   │
│ Trees (Trie)            │ Route matching in Express        │
│ Graphs                  │ Dependency resolution (npm)      │
│ Sorting                 │ API response ordering            │
│ Binary Search           │ Searching sorted DB results      │
│ Linked Lists            │ Event listener chains            │
│ Dynamic Programming     │ Rate limiting / throttle logic   │
└─────────────────────────┴──────────────────────────────────┘
```

---

## 2. Big-O Notation

Big-O describes the **upper bound** of an algorithm's growth rate as input size increases.

### The Key Complexities (Best → Worst)

```
O(1) → O(log n) → O(n) → O(n log n) → O(n²) → O(2ⁿ) → O(n!)
```

### Visual Growth Comparison

```
Input Size (n) │  O(1)  │ O(log n) │  O(n)  │ O(n log n) │  O(n²)    │  O(2ⁿ)
───────────────┼────────┼──────────┼────────┼────────────┼───────────┼─────────
        10     │   1    │    3     │   10   │     33     │   100     │  1,024
       100     │   1    │    7     │  100   │    664     │ 10,000    │  1.27e30
     1,000     │   1    │   10     │ 1,000  │   9,966    │ 1,000,000 │  ∞
    10,000     │   1    │   13     │ 10,000 │  132,877   │ 100M      │  ∞
```

### Rules for Calculating Big-O

```typescript
// Rule 1: Drop constants
// O(2n) → O(n)
function printTwice(arr: number[]): void {
  for (const item of arr) console.log(item);  // O(n)
  for (const item of arr) console.log(item);  // O(n)
  // Total: O(2n) → O(n)
}

// Rule 2: Drop non-dominant terms
// O(n + n²) → O(n²)
function mixedComplexity(arr: number[]): void {
  for (const item of arr) console.log(item);         // O(n)
  for (const a of arr)
    for (const b of arr) console.log(a, b);           // O(n²)
  // Total: O(n + n²) → O(n²)
}

// Rule 3: Different inputs → different variables
function compareTwoArrays(a: number[], b: number[]): void {
  for (const x of a) console.log(x);  // O(a)
  for (const y of b) console.log(y);  // O(b)
  // Total: O(a + b), NOT O(n)
}
```

---

## 3. Time Complexity

### O(1) — Constant Time

```typescript
// Array index access, object property lookup, Map.get()
function getFirst(arr: number[]): number {
  return arr[0]; // Always 1 operation regardless of array size
}

const map = new Map<string, number>();
map.set('key', 42);
map.get('key'); // O(1) average
```

### O(log n) — Logarithmic Time

```typescript
// Binary Search — halves search space each step
function binarySearch(arr: number[], target: number): number {
  let left = 0;
  let right = arr.length - 1;

  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    if (arr[mid] === target) return mid;
    if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  return -1;
}
// For n=1000, only ~10 comparisons needed (log₂1000 ≈ 10)
```

### O(n) — Linear Time

```typescript
// Single pass through data
function findMax(arr: number[]): number {
  let max = -Infinity;
  for (const num of arr) {
    if (num > max) max = num;
  }
  return max;
}

// Node.js: Reading a stream line by line
// Each line processed once → O(n) where n = number of lines
```

### O(n log n) — Linearithmic Time

```typescript
// Built-in sort uses TimSort → O(n log n)
const sorted = [5, 2, 8, 1, 9].sort((a, b) => a - b);

// Merge Sort implementation
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
  return [...result, ...left.slice(i), ...right.slice(j)];
}
```

### O(n²) — Quadratic Time

```typescript
// Nested loops over same collection
function hasDuplicate(arr: number[]): boolean {
  for (let i = 0; i < arr.length; i++) {
    for (let j = i + 1; j < arr.length; j++) {
      if (arr[i] === arr[j]) return true;
    }
  }
  return false;
}
// Better approach: Use Set → O(n)
function hasDuplicateOptimized(arr: number[]): boolean {
  return new Set(arr).size !== arr.length;
}
```

### O(2ⁿ) — Exponential Time

```typescript
// Naive recursive Fibonacci
function fib(n: number): number {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2);
}
// fib(40) → over 1 billion function calls!

// Optimized with memoization → O(n)
function fibMemo(n: number, memo: Map<number, number> = new Map()): number {
  if (memo.has(n)) return memo.get(n)!;
  if (n <= 1) return n;
  const result = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
  memo.set(n, result);
  return result;
}
```

---

## 4. Space Complexity

Space complexity measures **additional memory** used by an algorithm (excluding input).

```typescript
// O(1) Space — In-place operations
function reverseInPlace(arr: number[]): void {
  let left = 0, right = arr.length - 1;
  while (left < right) {
    [arr[left], arr[right]] = [arr[right], arr[left]];
    left++;
    right--;
  }
}

// O(n) Space — Creating new data structures
function removeDuplicates(arr: number[]): number[] {
  const seen = new Set<number>();       // O(n) space
  const result: number[] = [];          // O(n) space
  for (const num of arr) {
    if (!seen.has(num)) {
      seen.add(num);
      result.push(num);
    }
  }
  return result;
}

// O(n) Space — Recursive call stack
function factorial(n: number): number {
  if (n <= 1) return 1;
  return n * factorial(n - 1);
  // Call stack depth: n → O(n) space
}
```

### Space Complexity in Node.js Context

```typescript
// ❌ Bad: Loading entire file into memory → O(n) space
import { readFileSync } from 'fs';
const data = readFileSync('large-file.csv', 'utf-8');
const lines = data.split('\n'); // Entire file in memory

// ✅ Good: Stream processing → O(1) space
import { createReadStream } from 'fs';
import { createInterface } from 'readline';

const rl = createInterface({
  input: createReadStream('large-file.csv'),
});
rl.on('line', (line) => {
  // Process one line at a time → constant memory
});
```

---

## 5. JavaScript Engine Internals

### V8 Array Storage

```
V8 uses different internal representations based on content:

┌─────────────────────┬──────────────────────────────────┐
│ Array Type           │ V8 Internal Representation      │
├─────────────────────┼──────────────────────────────────┤
│ [1, 2, 3]           │ SMI (Small Integer) — fastest    │
│ [1.1, 2.2, 3.3]    │ Double — packed doubles           │
│ [1, "a", {}]        │ Tagged/Boxed — slowest           │
│ [1, , 3]            │ Holey — has gaps, slower lookups  │
└─────────────────────┴──────────────────────────────────┘

Tip: Keep arrays homogeneous for best V8 performance.
```

### Built-in Method Complexities

```typescript
// Array methods and their complexities:
const arr = [1, 2, 3, 4, 5];

arr.push(6);          // O(1) amortized — append to end
arr.pop();            // O(1) — remove from end
arr.shift();          // O(n) — remove from start (shifts all elements)
arr.unshift(0);       // O(n) — insert at start (shifts all elements)
arr.splice(2, 1);     // O(n) — remove/insert at index
arr.includes(3);      // O(n) — linear search
arr.indexOf(3);       // O(n) — linear search
arr.slice(1, 3);      // O(k) — k = slice length
arr.concat([6, 7]);   // O(n + m)
arr.sort();           // O(n log n) — TimSort
arr.map(x => x * 2);  // O(n)
arr.filter(x => x > 2); // O(n)
arr.reduce((a, b) => a + b, 0); // O(n)

// Object/Map methods:
const obj: Record<string, number> = { a: 1 };
obj['b'] = 2;           // O(1) average
delete obj['a'];         // O(1) average
'a' in obj;              // O(1) average
Object.keys(obj);        // O(n)

// Set methods:
const set = new Set([1, 2, 3]);
set.add(4);              // O(1) average
set.has(3);              // O(1) average
set.delete(2);           // O(1) average

// Map methods:
const map = new Map();
map.set('key', 'val');   // O(1) average
map.get('key');           // O(1) average
map.has('key');           // O(1) average
map.delete('key');        // O(1) average
```

---

## 6. Amortized Analysis

Some operations are **usually fast** but **occasionally slow**. Amortized analysis averages the cost.

```typescript
// Example: Array.push() in JavaScript
// V8 pre-allocates array capacity. When full, it doubles the capacity.

// Push operations:
// [_, _, _, _]  ← capacity 4
// push(1): O(1) → [1, _, _, _]
// push(2): O(1) → [1, 2, _, _]
// push(3): O(1) → [1, 2, 3, _]
// push(4): O(1) → [1, 2, 3, 4]
// push(5): O(n) → copy all to new array of capacity 8
//          [1, 2, 3, 4, 5, _, _, _]

// Most pushes: O(1)
// Occasional resize: O(n)
// Amortized: O(1) per push
```

---

## 7. Common Patterns & Complexity Cheat Sheet

```
┌──────────────────────┬──────────┬──────────┬───────────┐
│ Data Structure       │ Access   │ Search   │ Insert    │
├──────────────────────┼──────────┼──────────┼───────────┤
│ Array                │ O(1)     │ O(n)     │ O(n)      │
│ Sorted Array         │ O(1)     │ O(log n) │ O(n)      │
│ Linked List          │ O(n)     │ O(n)     │ O(1)*     │
│ Stack                │ O(n)     │ O(n)     │ O(1)      │
│ Queue                │ O(n)     │ O(n)     │ O(1)      │
│ Hash Map / Set       │ N/A      │ O(1)     │ O(1)      │
│ BST (balanced)       │ O(log n) │ O(log n) │ O(log n)  │
│ Heap (Priority Queue)│ N/A      │ O(n)     │ O(log n)  │
│ Trie                 │ N/A      │ O(k)     │ O(k)      │
└──────────────────────┴──────────┴──────────┴───────────┘
* Insert at head; insert at arbitrary position is O(n)

┌──────────────────────┬──────────────┬──────────┐
│ Sorting Algorithm    │ Average      │ Space    │
├──────────────────────┼──────────────┼──────────┤
│ Bubble Sort          │ O(n²)        │ O(1)     │
│ Selection Sort       │ O(n²)        │ O(1)     │
│ Insertion Sort       │ O(n²)        │ O(1)     │
│ Merge Sort           │ O(n log n)   │ O(n)     │
│ Quick Sort           │ O(n log n)   │ O(log n) │
│ Heap Sort            │ O(n log n)   │ O(1)     │
│ Tim Sort (JS built-in)│ O(n log n)  │ O(n)     │
│ Counting/Radix Sort  │ O(n + k)     │ O(n + k) │
└──────────────────────┴──────────────┴──────────┘
```

---

## 8. Interview Questions & Answers

### Q1: What is the difference between Big-O, Big-Ω, and Big-Θ?

**Answer:**
- **Big-O (O):** Upper bound — worst case. "At most this slow."
- **Big-Ω (Omega):** Lower bound — best case. "At least this fast."
- **Big-Θ (Theta):** Tight bound — average case. "Exactly this growth rate."
- In interviews, we mostly use **Big-O** because we care about worst-case guarantees.

### Q2: Why is `Array.shift()` O(n) but `Array.pop()` O(1) in JavaScript?

**Answer:**
- `pop()` removes the last element — no other elements need to move.
- `shift()` removes the first element — **all remaining elements must be re-indexed** (shifted left by one position).
- For queue behavior, use a **linked list** or a **Map-based queue** instead of `shift()`.

### Q3: When would you choose a Map over a plain Object in Node.js?

**Answer:**
- **Map** when keys are dynamic, non-string, or you need insertion order and `.size`.
- **Object** for static structures, JSON serialization, and when keys are always strings.
- **Map** has better performance for frequent additions/deletions.
- **Object** has prototype chain overhead; Map does not.

### Q4: How does `Set` achieve O(1) lookup in JavaScript?

**Answer:**
- Internally uses a **hash table**. Each value is hashed to a bucket index.
- Collision resolved via chaining or open addressing.
- Average O(1), worst case O(n) if all values hash to the same bucket (rare with good hash functions).

### Q5: Explain amortized O(1) for `Array.push()`.

**Answer:**
- V8 pre-allocates extra capacity. Most pushes just write to the next slot → O(1).
- When capacity is exceeded, V8 allocates a new array (typically 1.5x or 2x size) and copies elements → O(n).
- Over n pushes, total work ≈ 3n → amortized cost = O(1) per push.

### Q6: How would you measure the time complexity of a Node.js API endpoint?

**Answer:**
- Identify the **dominant operation**: DB query, loop, sorting, etc.
- DB query with index → O(log n); full table scan → O(n).
- If endpoint loops through results and sorts → O(n log n).
- Use `console.time()` / `performance.now()` to benchmark.
- Use **Big-O** to reason about behavior as data grows.

---

## Practice Problems

| # | Problem | Difficulty | Concept |
|---|---------|------------|---------|
| 1 | Determine Big-O of given code snippets | 🟢 Easy | Complexity Analysis |
| 2 | Compare two solutions and identify the more efficient one | 🟢 Easy | Big-O Comparison |
| 3 | Optimize a brute-force O(n²) solution to O(n) | 🟡 Medium | Optimization |
| 4 | Analyze space complexity of recursive vs iterative solutions | 🟡 Medium | Space Complexity |
| 5 | Design an API endpoint and analyze its complexity | 🟡 Medium | Real-World Application |

---

**Next:** [02 — Arrays & Strings →](02-arrays-strings.md)
