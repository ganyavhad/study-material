# 08 — Recursion & Dynamic Programming

> Transform exponential brute-force into polynomial solutions — the ultimate interview skill.

---

## Table of Contents

1. [Recursion Fundamentals](#1-recursion-fundamentals)
2. [Backtracking](#2-backtracking)
3. [DP Fundamentals: Memoization vs Tabulation](#3-dp-fundamentals-memoization-vs-tabulation)
4. [1D Dynamic Programming](#4-1d-dynamic-programming)
5. [2D Dynamic Programming](#5-2d-dynamic-programming)
6. [Classic DP Patterns](#6-classic-dp-patterns)
7. [Node.js Real-World Applications](#7-nodejs-real-world-applications)
8. [Interview Questions & Answers](#8-interview-questions--answers)
9. [Practice Problems](#9-practice-problems)

---

## 1. Recursion Fundamentals

```
Every recursive solution needs:
1. Base case — when to stop
2. Recursive case — break problem into smaller subproblems
3. Progress toward base case — avoid infinite recursion

Call Stack Visualization:
  factorial(4)
    → 4 * factorial(3)
      → 3 * factorial(2)
        → 2 * factorial(1)
          → returns 1        ← base case
        → returns 2 * 1 = 2
      → returns 3 * 2 = 6
    → returns 4 * 6 = 24
```

```typescript
// Classic recursion examples
function factorial(n: number): number {
  if (n <= 1) return 1;           // Base case
  return n * factorial(n - 1);     // Recursive case
}

// Power function — O(log n) time using divide-and-conquer
function power(base: number, exp: number): number {
  if (exp === 0) return 1;
  if (exp < 0) return 1 / power(base, -exp);
  if (exp % 2 === 0) {
    const half = power(base, exp / 2);
    return half * half;           // Avoid redundant computation
  }
  return base * power(base, exp - 1);
}

// Flatten nested array — recursion on unknown depth
function flattenDeep(arr: any[]): any[] {
  const result: any[] = [];
  for (const item of arr) {
    if (Array.isArray(item)) {
      result.push(...flattenDeep(item));
    } else {
      result.push(item);
    }
  }
  return result;
}
// flattenDeep([1, [2, [3, [4]]]]) → [1, 2, 3, 4]

// Deep clone object — recursion on nested structure
function deepClone<T>(obj: T): T {
  if (obj === null || typeof obj !== 'object') return obj;
  if (Array.isArray(obj)) return obj.map(item => deepClone(item)) as T;
  const clone = {} as T;
  for (const key in obj) {
    if (Object.prototype.hasOwnProperty.call(obj, key)) {
      (clone as any)[key] = deepClone((obj as any)[key]);
    }
  }
  return clone;
}
```

---

## 2. Backtracking

```
Backtracking Template:
  function backtrack(candidates, path, result):
    if (goal reached): add path to result; return
    for each candidate:
      if (valid choice):
        make choice (add to path)
        backtrack(remaining candidates, path, result)
        undo choice (remove from path)  ← BACKTRACK
```

```typescript
// Permutations — O(n! * n) time
function permute(nums: number[]): number[][] {
  const result: number[][] = [];

  function backtrack(path: number[], remaining: number[]): void {
    if (remaining.length === 0) {
      result.push([...path]);
      return;
    }
    for (let i = 0; i < remaining.length; i++) {
      path.push(remaining[i]);
      backtrack(path, [...remaining.slice(0, i), ...remaining.slice(i + 1)]);
      path.pop(); // Backtrack
    }
  }

  backtrack([], nums);
  return result;
}

// Subsets — O(2ⁿ * n) time
function subsets(nums: number[]): number[][] {
  const result: number[][] = [];

  function backtrack(start: number, path: number[]): void {
    result.push([...path]);
    for (let i = start; i < nums.length; i++) {
      path.push(nums[i]);
      backtrack(i + 1, path);
      path.pop();
    }
  }

  backtrack(0, []);
  return result;
}

// Combination Sum — O(2^target) time
function combinationSum(candidates: number[], target: number): number[][] {
  const result: number[][] = [];

  function backtrack(start: number, path: number[], remaining: number): void {
    if (remaining === 0) { result.push([...path]); return; }
    if (remaining < 0) return;

    for (let i = start; i < candidates.length; i++) {
      path.push(candidates[i]);
      backtrack(i, path, remaining - candidates[i]); // Can reuse same element
      path.pop();
    }
  }

  backtrack(0, [], target);
  return result;
}

// N-Queens — O(n!) time
function solveNQueens(n: number): string[][] {
  const result: string[][] = [];
  const board: string[][] = Array.from({ length: n }, () => new Array(n).fill('.'));

  function isValid(row: number, col: number): boolean {
    for (let i = 0; i < row; i++) {
      if (board[i][col] === 'Q') return false;
      if (col - (row - i) >= 0 && board[i][col - (row - i)] === 'Q') return false;
      if (col + (row - i) < n && board[i][col + (row - i)] === 'Q') return false;
    }
    return true;
  }

  function backtrack(row: number): void {
    if (row === n) {
      result.push(board.map(r => r.join('')));
      return;
    }
    for (let col = 0; col < n; col++) {
      if (isValid(row, col)) {
        board[row][col] = 'Q';
        backtrack(row + 1);
        board[row][col] = '.';
      }
    }
  }

  backtrack(0);
  return result;
}
```

---

## 3. DP Fundamentals: Memoization vs Tabulation

```
Dynamic Programming = Recursion + Memoization (or Tabulation)

Key Properties:
1. Optimal Substructure — optimal solution contains optimal sub-solutions
2. Overlapping Subproblems — same subproblems are solved repeatedly

Two Approaches:
┌─────────────────┬──────────────────────────────────────┐
│ Top-Down        │ Recursion + Memo (cache results)     │
│ (Memoization)   │ Start from problem, break down       │
│                 │ Natural thinking, easier to code      │
├─────────────────┼──────────────────────────────────────┤
│ Bottom-Up       │ Iterative + Table (build up)          │
│ (Tabulation)    │ Start from base case, build up        │
│                 │ No recursion stack, often faster       │
└─────────────────┴──────────────────────────────────────┘
```

```typescript
// Fibonacci — Classic DP Example

// ❌ Naive Recursion — O(2ⁿ) time, O(n) space
function fibNaive(n: number): number {
  if (n <= 1) return n;
  return fibNaive(n - 1) + fibNaive(n - 2);
}

// ✅ Top-Down (Memoization) — O(n) time, O(n) space
function fibMemo(n: number, memo: Map<number, number> = new Map()): number {
  if (memo.has(n)) return memo.get(n)!;
  if (n <= 1) return n;
  const result = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
  memo.set(n, result);
  return result;
}

// ✅ Bottom-Up (Tabulation) — O(n) time, O(n) space
function fibTab(n: number): number {
  if (n <= 1) return n;
  const dp = new Array(n + 1);
  dp[0] = 0;
  dp[1] = 1;
  for (let i = 2; i <= n; i++) {
    dp[i] = dp[i - 1] + dp[i - 2];
  }
  return dp[n];
}

// ✅ Space-Optimized — O(n) time, O(1) space
function fibOptimal(n: number): number {
  if (n <= 1) return n;
  let prev2 = 0, prev1 = 1;
  for (let i = 2; i <= n; i++) {
    const curr = prev1 + prev2;
    prev2 = prev1;
    prev1 = curr;
  }
  return prev1;
}
```

---

## 4. 1D Dynamic Programming

```typescript
// Climbing Stairs — O(n) time, O(1) space
function climbStairs(n: number): number {
  if (n <= 2) return n;
  let prev2 = 1, prev1 = 2;
  for (let i = 3; i <= n; i++) {
    const curr = prev1 + prev2;
    prev2 = prev1;
    prev1 = curr;
  }
  return prev1;
}

// House Robber — O(n) time, O(1) space
// Can't rob two adjacent houses
function rob(nums: number[]): number {
  let prev2 = 0, prev1 = 0;
  for (const num of nums) {
    const curr = Math.max(prev1, prev2 + num);
    prev2 = prev1;
    prev1 = curr;
  }
  return prev1;
}

// Coin Change — O(amount * coins) time, O(amount) space
function coinChange(coins: number[], amount: number): number {
  const dp = new Array(amount + 1).fill(Infinity);
  dp[0] = 0;

  for (let i = 1; i <= amount; i++) {
    for (const coin of coins) {
      if (coin <= i && dp[i - coin] + 1 < dp[i]) {
        dp[i] = dp[i - coin] + 1;
      }
    }
  }
  return dp[amount] === Infinity ? -1 : dp[amount];
}

// Longest Increasing Subsequence — O(n log n) with binary search
function lengthOfLIS(nums: number[]): number {
  const tails: number[] = []; // Smallest tail of increasing subsequence of each length

  for (const num of nums) {
    let left = 0, right = tails.length;
    while (left < right) {
      const mid = Math.floor((left + right) / 2);
      if (tails[mid] < num) left = mid + 1;
      else right = mid;
    }
    tails[left] = num;
  }
  return tails.length;
}

// Decode Ways — O(n) time, O(1) space
function numDecodings(s: string): number {
  if (s[0] === '0') return 0;
  let prev2 = 1, prev1 = 1;

  for (let i = 1; i < s.length; i++) {
    let curr = 0;
    if (s[i] !== '0') curr += prev1;
    const twoDigit = parseInt(s.substring(i - 1, i + 1), 10);
    if (twoDigit >= 10 && twoDigit <= 26) curr += prev2;
    prev2 = prev1;
    prev1 = curr;
  }
  return prev1;
}

// Word Break — O(n² * m) time
function wordBreak(s: string, wordDict: string[]): boolean {
  const wordSet = new Set(wordDict);
  const dp = new Array(s.length + 1).fill(false);
  dp[0] = true;

  for (let i = 1; i <= s.length; i++) {
    for (let j = 0; j < i; j++) {
      if (dp[j] && wordSet.has(s.substring(j, i))) {
        dp[i] = true;
        break;
      }
    }
  }
  return dp[s.length];
}
```

---

## 5. 2D Dynamic Programming

```typescript
// Unique Paths — O(m * n) time, O(n) space
function uniquePaths(m: number, n: number): number {
  const dp = new Array(n).fill(1);
  for (let i = 1; i < m; i++) {
    for (let j = 1; j < n; j++) {
      dp[j] += dp[j - 1];
    }
  }
  return dp[n - 1];
}

// Longest Common Subsequence — O(m * n) time, O(m * n) space
function longestCommonSubsequence(text1: string, text2: string): number {
  const m = text1.length, n = text2.length;
  const dp: number[][] = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));

  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (text1[i - 1] === text2[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1] + 1;
      } else {
        dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
      }
    }
  }
  return dp[m][n];
}

// Edit Distance — O(m * n) time
function minDistance(word1: string, word2: string): number {
  const m = word1.length, n = word2.length;
  const dp: number[][] = Array.from({ length: m + 1 }, (_, i) =>
    new Array(n + 1).fill(0).map((_, j) => (i === 0 ? j : j === 0 ? i : 0))
  );

  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (word1[i - 1] === word2[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1];
      } else {
        dp[i][j] = 1 + Math.min(
          dp[i - 1][j],     // Delete
          dp[i][j - 1],     // Insert
          dp[i - 1][j - 1]  // Replace
        );
      }
    }
  }
  return dp[m][n];
}

// 0/1 Knapsack — O(n * W) time
function knapsack(weights: number[], values: number[], capacity: number): number {
  const n = weights.length;
  const dp = new Array(capacity + 1).fill(0);

  for (let i = 0; i < n; i++) {
    for (let w = capacity; w >= weights[i]; w--) { // Reverse to avoid reusing items
      dp[w] = Math.max(dp[w], dp[w - weights[i]] + values[i]);
    }
  }
  return dp[capacity];
}
```

---

## 6. Classic DP Patterns

```
Pattern Recognition Guide:

┌──────────────────────┬─────────────────────────────────────────┐
│ Pattern              │ Problems                                │
├──────────────────────┼─────────────────────────────────────────┤
│ Fibonacci-like       │ Climbing Stairs, House Robber, Decode   │
│ Knapsack             │ 0/1 Knapsack, Coin Change, Subset Sum  │
│ LCS/LIS              │ Longest Common Subseq, Edit Distance   │
│ Grid DP              │ Unique Paths, Min Path Sum, Dungeon     │
│ Interval DP          │ Burst Balloons, Matrix Chain            │
│ String DP            │ Palindrome Partitioning, Word Break     │
│ State Machine        │ Buy/Sell Stock (with cooldown/fee)      │
└──────────────────────┴─────────────────────────────────────────┘

Steps to Solve Any DP Problem:
1. Define state: What does dp[i] (or dp[i][j]) represent?
2. Find transition: How does dp[i] relate to previous states?
3. Set base cases: What are the starting values?
4. Determine iteration order: Left-to-right? Bottom-up?
5. Optimize space if possible (rolling array, two variables)
```

```typescript
// Buy and Sell Stock with Cooldown — State Machine DP
function maxProfit(prices: number[]): number {
  const n = prices.length;
  if (n <= 1) return 0;

  // States: hold, sold, rest
  let hold = -prices[0];  // Bought/holding stock
  let sold = 0;           // Just sold (must cooldown next)
  let rest = 0;           // Not holding, can buy

  for (let i = 1; i < n; i++) {
    const newHold = Math.max(hold, rest - prices[i]); // Hold or buy
    const newSold = hold + prices[i];                   // Sell
    const newRest = Math.max(rest, sold);               // Rest or post-cooldown
    hold = newHold;
    sold = newSold;
    rest = newRest;
  }
  return Math.max(sold, rest);
}

// Palindrome Partitioning — Backtracking + DP precomputation
function partition(s: string): string[][] {
  const n = s.length;
  // Precompute palindrome lookup — O(n²)
  const isPalin: boolean[][] = Array.from({ length: n }, () => new Array(n).fill(false));
  for (let i = n - 1; i >= 0; i--) {
    for (let j = i; j < n; j++) {
      isPalin[i][j] = s[i] === s[j] && (j - i <= 2 || isPalin[i + 1][j - 1]);
    }
  }

  const result: string[][] = [];
  function backtrack(start: number, path: string[]): void {
    if (start === n) { result.push([...path]); return; }
    for (let end = start; end < n; end++) {
      if (isPalin[start][end]) {
        path.push(s.substring(start, end + 1));
        backtrack(end + 1, path);
        path.pop();
      }
    }
  }

  backtrack(0, []);
  return result;
}
```

---

## 7. Node.js Real-World Applications

```
┌───────────────────────────┬──────────────────────────────────────┐
│ DP/Recursion Concept      │ Node.js Use Case                     │
├───────────────────────────┼──────────────────────────────────────┤
│ Memoization               │ API response caching (lru-cache)     │
│ Recursion                 │ JSON deep merge, directory traversal │
│ Backtracking              │ Regex matching, route resolution      │
│ Knapsack                  │ Resource allocation, bundle splitting│
│ Edit Distance             │ Fuzzy search, spell correction        │
│ LCS                       │ Git diff algorithm                    │
│ Interval DP               │ Meeting room scheduling               │
│ State Machine             │ Workflow engines, payment states      │
└───────────────────────────┴──────────────────────────────────────┘
```

```typescript
// Memoize decorator for Node.js functions
function memoize<T extends (...args: any[]) => any>(fn: T): T {
  const cache = new Map<string, ReturnType<T>>();

  return ((...args: Parameters<T>): ReturnType<T> => {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key)!;
    const result = fn(...args);
    cache.set(key, result);
    return result;
  }) as T;
}

// Usage: memoize expensive DB aggregation results
const getReport = memoize(async (userId: string, month: string) => {
  // Expensive aggregation query...
  return { revenue: 10000, users: 500 };
});

// Directory tree traversal — recursion
import { readdirSync, statSync } from 'fs';
import { join } from 'path';

function getDirectoryTree(dir: string): object {
  const items = readdirSync(dir);
  const tree: Record<string, any> = {};

  for (const item of items) {
    const fullPath = join(dir, item);
    if (statSync(fullPath).isDirectory()) {
      tree[item] = getDirectoryTree(fullPath); // Recursive
    } else {
      tree[item] = 'file';
    }
  }
  return tree;
}
```

---

## 8. Interview Questions & Answers

### Q1: When should you use DP vs Greedy?

**Answer:**
- **DP:** When the problem has overlapping subproblems and optimal substructure. You explore all combinations. Example: Coin Change (need minimum coins).
- **Greedy:** When locally optimal choices lead to global optimum. No need to explore all options. Example: Activity Selection, Interval Scheduling.
- **Test:** If greedy gives wrong answers on edge cases, you likely need DP.

### Q2: How do you identify a DP problem?

**Answer:**
- Keywords: "minimum/maximum", "count ways", "is it possible", "longest/shortest".
- Ask: Can I break this into smaller subproblems? Are subproblems repeated?
- If brute force is exponential and subproblems overlap → DP.
- Draw the recursion tree — if you see repeated nodes, memoize.

### Q3: Top-down vs Bottom-up — which to choose?

**Answer:**
- **Top-down (memoization):** Easier to write, only computes needed states, natural for tree-shaped dependencies.
- **Bottom-up (tabulation):** No recursion stack overflow risk, sometimes faster (no function call overhead), easier to optimize space.
- In interviews: start with top-down (easier to think), optimize to bottom-up if asked.

### Q4: What's the maximum recursion depth in Node.js?

**Answer:**
- Default call stack: ~10,000–15,000 frames (V8 dependent).
- `RangeError: Maximum call stack size exceeded` if exceeded.
- Solutions: convert to iterative, use tail call optimization (limited support), increase stack with `--stack-size` flag.
- For deep recursion: always prefer iterative DP (bottom-up).

### Q5: Explain the Knapsack problem and its variants.

**Answer:**
- **0/1 Knapsack:** Each item used at most once. DP on items × capacity.
- **Unbounded Knapsack:** Items can be reused. Like Coin Change.
- **Fractional Knapsack:** Can take fractions → Greedy (sort by value/weight ratio).
- State: `dp[i][w]` = max value using first i items with capacity w.
- Space optimization: single 1D array, iterate capacity in reverse (0/1) or forward (unbounded).

---

## 9. Practice Problems

| # | Problem | Difficulty | Pattern |
|---|---------|------------|---------|
| 1 | Climbing Stairs | 🟢 Easy | Fibonacci DP |
| 2 | House Robber | 🟡 Medium | 1D DP |
| 3 | Coin Change | 🟡 Medium | Unbounded Knapsack |
| 4 | Longest Increasing Subsequence | 🟡 Medium | LIS + Binary Search |
| 5 | Word Break | 🟡 Medium | String DP |
| 6 | Unique Paths | 🟡 Medium | Grid DP |
| 7 | Longest Common Subsequence | 🟡 Medium | 2D DP (LCS) |
| 8 | Edit Distance | 🟡 Medium | 2D DP |
| 9 | Partition Equal Subset Sum | 🟡 Medium | 0/1 Knapsack |
| 10 | Burst Balloons | 🔴 Hard | Interval DP |
| 11 | Regular Expression Matching | 🔴 Hard | 2D DP |
| 12 | N-Queens | 🔴 Hard | Backtracking |

---

**Previous:** [07 — Sorting & Searching ←](07-sorting-searching.md)
