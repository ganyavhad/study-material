# 06 — Hash Maps & Sets

> O(1) average-case lookup, insertion, and deletion — the most versatile problem-solving tool.

---

## Table of Contents

1. [JavaScript Map & Set Internals](#1-javascript-map--set-internals)
2. [Hash Function Concepts](#2-hash-function-concepts)
3. [Frequency Counting Pattern](#3-frequency-counting-pattern)
4. [Two Sum & Variants](#4-two-sum--variants)
5. [Grouping & Bucketing](#5-grouping--bucketing)
6. [Sliding Window + Hash Map](#6-sliding-window--hash-map)
7. [Design Problems](#7-design-problems)
8. [Interview Questions & Answers](#8-interview-questions--answers)
9. [Practice Problems](#9-practice-problems)

---

## 1. JavaScript Map & Set Internals

```typescript
// Map vs Object
// ┌─────────────────┬────────────────────────────────────────┐
// │ Feature         │ Map              │ Object              │
// ├─────────────────┼──────────────────┼─────────────────────┤
// │ Key types       │ Any (obj, func)  │ String/Symbol only  │
// │ Order           │ Insertion order  │ Mixed (spec varies) │
// │ Size            │ .size property   │ Object.keys().length│
// │ Iteration       │ for..of built-in │ Object.entries()    │
// │ Performance     │ Frequent add/del │ Static structures   │
// │ Serialization   │ No JSON.stringify│ JSON.stringify works│
// └─────────────────┴──────────────────┴─────────────────────┘

// Map — O(1) average for get, set, has, delete
const map = new Map<string, number>();
map.set('users', 100);
map.get('users');      // 100
map.has('users');      // true
map.delete('users');   // true
map.size;              // 0

// Set — O(1) average for add, has, delete
const set = new Set<number>();
set.add(1);
set.add(2);
set.add(1);            // Ignored — duplicates not allowed
set.has(1);            // true
set.size;              // 2

// WeakMap/WeakSet — keys are weakly held (GC can collect them)
// Use for: metadata on objects without preventing garbage collection
const weakMap = new WeakMap<object, string>();
let obj = { id: 1 };
weakMap.set(obj, 'metadata');
obj = null!; // Object can now be garbage collected
```

---

## 2. Hash Function Concepts

```
How Hash Maps Work:

  key → hash(key) → index → bucket
  
  "alice" → 97+108+105+99+101 = 510 → 510 % 16 = 14 → bucket[14]

Collision Resolution:
1. Chaining: Each bucket holds a linked list of entries
2. Open Addressing: Find next empty slot (linear/quadratic probing)

Load Factor = entries / buckets
- When load factor > 0.75, resize (double buckets, rehash all)
- This is why amortized O(1) — occasional O(n) rehash
```

```typescript
// Simple HashMap implementation (for understanding)
class SimpleHashMap<K, V> {
  private buckets: Array<Array<[K, V]>>;
  private _size: number = 0;
  private capacity: number;

  constructor(capacity: number = 16) {
    this.capacity = capacity;
    this.buckets = new Array(capacity).fill(null).map(() => []);
  }

  private hash(key: K): number {
    const str = String(key);
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = (hash * 31 + str.charCodeAt(i)) % this.capacity;
    }
    return hash;
  }

  set(key: K, value: V): void {
    const index = this.hash(key);
    const bucket = this.buckets[index];
    const existing = bucket.find(([k]) => k === key);
    if (existing) {
      existing[1] = value;
    } else {
      bucket.push([key, value]);
      this._size++;
    }
  }

  get(key: K): V | undefined {
    const index = this.hash(key);
    const entry = this.buckets[index].find(([k]) => k === key);
    return entry?.[1];
  }

  get size(): number { return this._size; }
}
```

---

## 3. Frequency Counting Pattern

```typescript
// Count character frequency — O(n) time, O(k) space
function charFrequency(s: string): Map<string, number> {
  const freq = new Map<string, number>();
  for (const char of s) {
    freq.set(char, (freq.get(char) || 0) + 1);
  }
  return freq;
}

// First Unique Character — O(n) time
function firstUniqChar(s: string): number {
  const freq = new Map<string, number>();
  for (const c of s) freq.set(c, (freq.get(c) || 0) + 1);
  for (let i = 0; i < s.length; i++) {
    if (freq.get(s[i]) === 1) return i;
  }
  return -1;
}

// Top K Frequent Elements — O(n) time using bucket sort
function topKFrequent(nums: number[], k: number): number[] {
  const freq = new Map<number, number>();
  for (const num of nums) freq.set(num, (freq.get(num) || 0) + 1);

  // Bucket sort: index = frequency, value = list of elements
  const buckets: number[][] = new Array(nums.length + 1).fill(null).map(() => []);
  for (const [num, count] of freq) {
    buckets[count].push(num);
  }

  const result: number[] = [];
  for (let i = buckets.length - 1; i >= 0 && result.length < k; i--) {
    result.push(...buckets[i]);
  }
  return result.slice(0, k);
}

// Node.js use case: API rate limiting
class RateLimiter {
  private requests: Map<string, number[]> = new Map();

  isAllowed(clientId: string, windowMs: number, maxRequests: number): boolean {
    const now = Date.now();
    const timestamps = this.requests.get(clientId) || [];

    // Remove expired timestamps
    const valid = timestamps.filter(t => now - t < windowMs);

    if (valid.length >= maxRequests) return false;

    valid.push(now);
    this.requests.set(clientId, valid);
    return true;
  }
}
```

---

## 4. Two Sum & Variants

```typescript
// Two Sum — O(n) time, O(n) space
function twoSum(nums: number[], target: number): [number, number] {
  const seen = new Map<number, number>(); // value → index

  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i];
    if (seen.has(complement)) {
      return [seen.get(complement)!, i];
    }
    seen.set(nums[i], i);
  }
  return [-1, -1];
}

// Four Sum Count — O(n²) time, O(n²) space
function fourSumCount(
  nums1: number[], nums2: number[], nums3: number[], nums4: number[]
): number {
  const sumAB = new Map<number, number>();

  for (const a of nums1) {
    for (const b of nums2) {
      const sum = a + b;
      sumAB.set(sum, (sumAB.get(sum) || 0) + 1);
    }
  }

  let count = 0;
  for (const c of nums3) {
    for (const d of nums4) {
      const target = -(c + d);
      if (sumAB.has(target)) count += sumAB.get(target)!;
    }
  }
  return count;
}

// Subarray Sum Equals K (using prefix sum + hash map) — O(n)
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

## 5. Grouping & Bucketing

```typescript
// Group Anagrams — O(n * k) time
function groupAnagrams(strs: string[]): string[][] {
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

// Longest Consecutive Sequence — O(n) time
function longestConsecutive(nums: number[]): number {
  const set = new Set(nums);
  let maxLength = 0;

  for (const num of set) {
    // Only start counting from sequence start
    if (!set.has(num - 1)) {
      let length = 1;
      let current = num;
      while (set.has(current + 1)) {
        current++;
        length++;
      }
      maxLength = Math.max(maxLength, length);
    }
  }
  return maxLength;
}

// Isomorphic Strings — O(n) time
function isIsomorphic(s: string, t: string): boolean {
  if (s.length !== t.length) return false;
  const sToT = new Map<string, string>();
  const tToS = new Map<string, string>();

  for (let i = 0; i < s.length; i++) {
    const sc = s[i], tc = t[i];
    if (sToT.has(sc) && sToT.get(sc) !== tc) return false;
    if (tToS.has(tc) && tToS.get(tc) !== sc) return false;
    sToT.set(sc, tc);
    tToS.set(tc, sc);
  }
  return true;
}
```

---

## 6. Sliding Window + Hash Map

```typescript
// Minimum Window Substring — O(n) time
function minWindow(s: string, t: string): string {
  const need = new Map<string, number>();
  for (const c of t) need.set(c, (need.get(c) || 0) + 1);

  const window = new Map<string, number>();
  let have = 0;
  const required = need.size;
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
      const lc = s[left];
      window.set(lc, window.get(lc)! - 1);
      if (need.has(lc) && window.get(lc)! < need.get(lc)!) have--;
      left++;
    }
  }
  return result;
}

// Find All Anagrams in String — O(n) time
function findAnagrams(s: string, p: string): number[] {
  if (s.length < p.length) return [];
  const result: number[] = [];
  const pCount = new Array(26).fill(0);
  const sCount = new Array(26).fill(0);

  for (const c of p) pCount[c.charCodeAt(0) - 97]++;

  for (let i = 0; i < s.length; i++) {
    sCount[s.charCodeAt(i) - 97]++;
    if (i >= p.length) sCount[s.charCodeAt(i - p.length) - 97]--;

    if (pCount.every((v, idx) => v === sCount[idx])) {
      result.push(i - p.length + 1);
    }
  }
  return result;
}
```

---

## 7. Design Problems

```typescript
// Design a Logger Rate Limiter — O(1) per call
class Logger {
  private timestamps: Map<string, number> = new Map();

  shouldPrintMessage(timestamp: number, message: string): boolean {
    if (!this.timestamps.has(message) || timestamp - this.timestamps.get(message)! >= 10) {
      this.timestamps.set(message, timestamp);
      return true;
    }
    return false;
  }
}

// Design a Hit Counter — O(1) amortized
class HitCounter {
  private hits: Map<number, number> = new Map();
  private total = 0;

  hit(timestamp: number): void {
    this.hits.set(timestamp, (this.hits.get(timestamp) || 0) + 1);
    this.total++;
  }

  getHits(timestamp: number): number {
    let count = 0;
    for (const [time, freq] of this.hits) {
      if (timestamp - time < 300) { // 5-minute window
        count += freq;
      } else {
        this.hits.delete(time);
      }
    }
    return count;
  }
}
```

---

## 8. Interview Questions & Answers

### Q1: Map vs Object — when to use which in Node.js?

**Answer:**
- **Map:** Dynamic keys, frequent add/delete, need `.size`, non-string keys, iteration order matters.
- **Object:** JSON serialization, static config, prototype methods needed, destructuring.
- **Performance:** Map is faster for frequent insertions/deletions. Object slightly faster for small static lookups.

### Q2: How does a Set guarantee uniqueness?

**Answer:**
- Uses **SameValueZero** comparison (like `===` but `NaN === NaN` is true).
- Backed by a hash table — hashes each value to a bucket.
- For objects, uniqueness is by **reference**, not by value. `new Set([{a:1}, {a:1}])` has size 2.

### Q3: What's the time complexity of `Object.keys()` vs `Map.forEach()`?

**Answer:**
- Both are O(n) — must iterate all entries.
- `Object.keys()` creates a new array → O(n) extra space.
- `Map.forEach()` iterates in-place → O(1) extra space.

### Q4: How would you implement a cache with expiration in Node.js?

**Answer:**
- Use a `Map<key, { value, expiry }>`.
- On `get()`, check if `Date.now() > expiry` — if so, delete and return miss.
- Periodically sweep expired entries (or lazily on access).
- For production: use `node-cache`, `lru-cache`, or Redis.

---

## 9. Practice Problems

| # | Problem | Difficulty | Pattern |
|---|---------|------------|---------|
| 1 | Two Sum | 🟢 Easy | Hash Map Lookup |
| 2 | Contains Duplicate | 🟢 Easy | Hash Set |
| 3 | Valid Anagram | 🟢 Easy | Frequency Count |
| 4 | Group Anagrams | 🟡 Medium | Hash Map Grouping |
| 5 | Top K Frequent Elements | 🟡 Medium | Frequency + Bucket Sort |
| 6 | Longest Consecutive Sequence | 🟡 Medium | Hash Set |
| 7 | Subarray Sum Equals K | 🟡 Medium | Prefix Sum + Map |
| 8 | Find All Anagrams in String | 🟡 Medium | Sliding Window + Map |
| 9 | Minimum Window Substring | 🔴 Hard | Sliding Window + Map |
| 10 | LRU Cache | 🟡 Medium | Map + Doubly LL |

---

**Previous:** [05 — Trees & Graphs ←](05-trees-graphs.md) | **Next:** [07 — Sorting & Searching →](07-sorting-searching.md)
