# 04 — Stacks & Queues

> LIFO and FIFO structures — the backbone of task scheduling, expression parsing, and BFS traversals.

---

## Table of Contents

1. [Stack Implementation](#1-stack-implementation)
2. [Monotonic Stack](#2-monotonic-stack)
3. [Queue Implementation](#3-queue-implementation)
4. [Priority Queue / Min-Heap](#4-priority-queue--min-heap)
5. [Classic Problems](#5-classic-problems)
6. [Node.js Real-World Connections](#6-nodejs-real-world-connections)
7. [Interview Questions & Answers](#7-interview-questions--answers)
8. [Practice Problems](#8-practice-problems)

---

## 1. Stack Implementation

```
Stack: Last In, First Out (LIFO)

  push(4) →  │ 4 │ ← top    pop() → 4
             │ 3 │
             │ 2 │
             │ 1 │
             └───┘
```

```typescript
// Stack using Array (good enough for most cases in JS)
class Stack<T> {
  private items: T[] = [];

  push(item: T): void { this.items.push(item); }         // O(1) amortized
  pop(): T | undefined { return this.items.pop(); }       // O(1)
  peek(): T | undefined { return this.items[this.items.length - 1]; } // O(1)
  isEmpty(): boolean { return this.items.length === 0; }  // O(1)
  get size(): number { return this.items.length; }
}

// Valid Parentheses — O(n) time, O(n) space
function isValid(s: string): boolean {
  const stack: string[] = [];
  const pairs: Record<string, string> = { ')': '(', ']': '[', '}': '{' };

  for (const char of s) {
    if ('([{'.includes(char)) {
      stack.push(char);
    } else {
      if (stack.pop() !== pairs[char]) return false;
    }
  }
  return stack.length === 0;
}

// Min Stack — O(1) for all operations including getMin
class MinStack {
  private stack: number[] = [];
  private minStack: number[] = [];

  push(val: number): void {
    this.stack.push(val);
    const currentMin = this.minStack.length === 0
      ? val
      : Math.min(val, this.minStack[this.minStack.length - 1]);
    this.minStack.push(currentMin);
  }

  pop(): void {
    this.stack.pop();
    this.minStack.pop();
  }

  top(): number { return this.stack[this.stack.length - 1]; }
  getMin(): number { return this.minStack[this.minStack.length - 1]; }
}
```

### Expression Parsing

```typescript
// Evaluate Reverse Polish Notation — O(n) time, O(n) space
function evalRPN(tokens: string[]): number {
  const stack: number[] = [];
  const ops: Record<string, (a: number, b: number) => number> = {
    '+': (a, b) => a + b,
    '-': (a, b) => a - b,
    '*': (a, b) => a * b,
    '/': (a, b) => Math.trunc(a / b),
  };

  for (const token of tokens) {
    if (token in ops) {
      const b = stack.pop()!;
      const a = stack.pop()!;
      stack.push(ops[token](a, b));
    } else {
      stack.push(parseInt(token, 10));
    }
  }
  return stack[0];
}

// Basic Calculator with +, -, (, ) — O(n) time
function calculate(s: string): number {
  const stack: number[] = [];
  let result = 0;
  let sign = 1;
  let i = 0;

  while (i < s.length) {
    if (s[i] >= '0' && s[i] <= '9') {
      let num = 0;
      while (i < s.length && s[i] >= '0' && s[i] <= '9') {
        num = num * 10 + parseInt(s[i], 10);
        i++;
      }
      result += sign * num;
      continue;
    }
    if (s[i] === '+') sign = 1;
    else if (s[i] === '-') sign = -1;
    else if (s[i] === '(') {
      stack.push(result);
      stack.push(sign);
      result = 0;
      sign = 1;
    } else if (s[i] === ')') {
      result = result * stack.pop()! + stack.pop()!;
    }
    i++;
  }
  return result;
}
```

---

## 2. Monotonic Stack

A stack that maintains elements in **increasing or decreasing order**.

```typescript
// Next Greater Element — O(n) time, O(n) space
function nextGreaterElement(nums: number[]): number[] {
  const result = new Array(nums.length).fill(-1);
  const stack: number[] = []; // Stack of indices

  for (let i = 0; i < nums.length; i++) {
    while (stack.length > 0 && nums[stack[stack.length - 1]] < nums[i]) {
      const idx = stack.pop()!;
      result[idx] = nums[i];
    }
    stack.push(i);
  }
  return result;
}
// [2, 1, 2, 4, 3] → [4, 2, 4, -1, -1]

// Daily Temperatures — O(n) time, O(n) space
function dailyTemperatures(temperatures: number[]): number[] {
  const result = new Array(temperatures.length).fill(0);
  const stack: number[] = [];

  for (let i = 0; i < temperatures.length; i++) {
    while (stack.length > 0 && temperatures[stack[stack.length - 1]] < temperatures[i]) {
      const idx = stack.pop()!;
      result[idx] = i - idx;
    }
    stack.push(i);
  }
  return result;
}

// Largest Rectangle in Histogram — O(n) time, O(n) space
function largestRectangleArea(heights: number[]): number {
  const stack: number[] = [];
  let maxArea = 0;
  const n = heights.length;

  for (let i = 0; i <= n; i++) {
    const currentHeight = i === n ? 0 : heights[i];
    while (stack.length > 0 && heights[stack[stack.length - 1]] > currentHeight) {
      const height = heights[stack.pop()!];
      const width = stack.length === 0 ? i : i - stack[stack.length - 1] - 1;
      maxArea = Math.max(maxArea, height * width);
    }
    stack.push(i);
  }
  return maxArea;
}
```

---

## 3. Queue Implementation

```
Queue: First In, First Out (FIFO)

  enqueue(4) → │ 1 │ 2 │ 3 │ 4 │ → dequeue() → 1
               front              back
```

```typescript
// Queue using Linked List (avoids O(n) shift on array)
class Queue<T> {
  private head: ListNode<T> | null = null;
  private tail: ListNode<T> | null = null;
  private _size = 0;

  enqueue(val: T): void {
    const node = new ListNode(val);
    if (this.tail) {
      this.tail.next = node;
    } else {
      this.head = node;
    }
    this.tail = node;
    this._size++;
  }

  dequeue(): T | undefined {
    if (!this.head) return undefined;
    const val = this.head.val;
    this.head = this.head.next;
    if (!this.head) this.tail = null;
    this._size--;
    return val;
  }

  peek(): T | undefined { return this.head?.val; }
  isEmpty(): boolean { return this._size === 0; }
  get size(): number { return this._size; }
}

// Implement Stack using Two Queues
class StackUsingQueues<T> {
  private q1 = new Queue<T>();
  private q2 = new Queue<T>();

  push(val: T): void {
    this.q2.enqueue(val);
    while (!this.q1.isEmpty()) {
      this.q2.enqueue(this.q1.dequeue()!);
    }
    [this.q1, this.q2] = [this.q2, this.q1];
  }

  pop(): T | undefined { return this.q1.dequeue(); }
  top(): T | undefined { return this.q1.peek(); }
  isEmpty(): boolean { return this.q1.isEmpty(); }
}

// Implement Queue using Two Stacks (amortized O(1) dequeue)
class QueueUsingStacks<T> {
  private inStack: T[] = [];
  private outStack: T[] = [];

  enqueue(val: T): void {
    this.inStack.push(val);
  }

  dequeue(): T | undefined {
    if (this.outStack.length === 0) {
      while (this.inStack.length > 0) {
        this.outStack.push(this.inStack.pop()!);
      }
    }
    return this.outStack.pop();
  }

  peek(): T | undefined {
    if (this.outStack.length === 0) {
      while (this.inStack.length > 0) {
        this.outStack.push(this.inStack.pop()!);
      }
    }
    return this.outStack[this.outStack.length - 1];
  }

  isEmpty(): boolean {
    return this.inStack.length === 0 && this.outStack.length === 0;
  }
}
```

---

## 4. Priority Queue / Min-Heap

```typescript
// Min-Heap implementation — O(log n) insert/extract, O(1) peek
class MinHeap {
  private heap: number[] = [];

  private parent(i: number): number { return Math.floor((i - 1) / 2); }
  private leftChild(i: number): number { return 2 * i + 1; }
  private rightChild(i: number): number { return 2 * i + 2; }

  private swap(i: number, j: number): void {
    [this.heap[i], this.heap[j]] = [this.heap[j], this.heap[i]];
  }

  insert(val: number): void {
    this.heap.push(val);
    this.bubbleUp(this.heap.length - 1);
  }

  extractMin(): number | undefined {
    if (this.heap.length === 0) return undefined;
    const min = this.heap[0];
    const last = this.heap.pop()!;
    if (this.heap.length > 0) {
      this.heap[0] = last;
      this.bubbleDown(0);
    }
    return min;
  }

  peek(): number | undefined { return this.heap[0]; }
  get size(): number { return this.heap.length; }

  private bubbleUp(i: number): void {
    while (i > 0 && this.heap[this.parent(i)] > this.heap[i]) {
      this.swap(i, this.parent(i));
      i = this.parent(i);
    }
  }

  private bubbleDown(i: number): void {
    let smallest = i;
    const left = this.leftChild(i);
    const right = this.rightChild(i);

    if (left < this.heap.length && this.heap[left] < this.heap[smallest]) {
      smallest = left;
    }
    if (right < this.heap.length && this.heap[right] < this.heap[smallest]) {
      smallest = right;
    }
    if (smallest !== i) {
      this.swap(i, smallest);
      this.bubbleDown(smallest);
    }
  }
}

// Kth Largest Element — O(n log k) using Min-Heap of size k
function findKthLargest(nums: number[], k: number): number {
  const heap = new MinHeap();
  for (const num of nums) {
    heap.insert(num);
    if (heap.size > k) heap.extractMin();
  }
  return heap.peek()!;
}

// Merge K Sorted Arrays — O(n log k) using Min-Heap
function mergeKSorted(arrays: number[][]): number[] {
  const heap = new MinHeap(); // Would need generic heap with source tracking
  const result: number[] = [];
  // (Simplified — full implementation would track array index + position)
  // Push first element from each array, extract min, push next from same array
  return result;
}
```

---

## 5. Classic Problems

### Sliding Window Maximum (Deque)

```typescript
// Maximum of each sliding window of size k — O(n) time
function maxSlidingWindow(nums: number[], k: number): number[] {
  const result: number[] = [];
  const deque: number[] = []; // Indices, decreasing order of values

  for (let i = 0; i < nums.length; i++) {
    // Remove elements outside window
    if (deque.length > 0 && deque[0] <= i - k) {
      deque.shift();
    }

    // Remove smaller elements from back
    while (deque.length > 0 && nums[deque[deque.length - 1]] < nums[i]) {
      deque.pop();
    }

    deque.push(i);

    if (i >= k - 1) {
      result.push(nums[deque[0]]);
    }
  }
  return result;
}
```

### Task Scheduler

```typescript
// Task Scheduler — O(n) time
function leastInterval(tasks: string[], n: number): number {
  const freq = new Array(26).fill(0);
  for (const task of tasks) freq[task.charCodeAt(0) - 65]++;

  freq.sort((a, b) => b - a);
  const maxFreq = freq[0];
  let maxCount = 0;

  for (const f of freq) {
    if (f === maxFreq) maxCount++;
    else break;
  }

  // Formula: (maxFreq - 1) * (n + 1) + maxCount
  const minSlots = (maxFreq - 1) * (n + 1) + maxCount;
  return Math.max(minSlots, tasks.length);
}
```

---

## 6. Node.js Real-World Connections

```
┌──────────────────────────┬─────────────────────────────────────────┐
│ DS Concept               │ Node.js Use Case                        │
├──────────────────────────┼─────────────────────────────────────────┤
│ Stack (LIFO)             │ Call stack, error stack traces           │
│ Queue (FIFO)             │ BullMQ, SQS, event loop microtask queue│
│ Priority Queue           │ Job scheduling by priority              │
│ Monotonic Stack          │ Price tracking, stock span problems     │
│ Deque                    │ Sliding window rate limiting             │
│ Stack + Recursion        │ AST parsing, template engines           │
└──────────────────────────┴─────────────────────────────────────────┘
```

```typescript
// Node.js Event Loop as Queue
// Simplified model:
// 1. Call Stack (LIFO) — currently executing code
// 2. Microtask Queue (FIFO) — Promise.then, queueMicrotask
// 3. Macrotask Queue (FIFO) — setTimeout, setInterval, I/O callbacks
// 4. After each macrotask, drain ALL microtasks before next macrotask

console.log('1');                    // Call stack
setTimeout(() => console.log('2'), 0); // Macrotask queue
Promise.resolve().then(() => console.log('3')); // Microtask queue
console.log('4');
// Output: 1, 4, 3, 2
```

---

## 7. Interview Questions & Answers

### Q1: When would you use a Stack vs a Queue?

**Answer:**
- **Stack (LIFO):** Undo operations, expression parsing, DFS, backtracking, function call management.
- **Queue (FIFO):** BFS, task scheduling, message processing, print job spooling.
- In Node.js: call stack is LIFO, event loop queues are FIFO.

### Q2: Why not use `Array.shift()` for a queue in JavaScript?

**Answer:**
- `shift()` is O(n) because all elements must be re-indexed.
- For a proper queue, use a linked list or a Map-based approach.
- For small queues, array performance is fine. For high-throughput (BullMQ-like), use linked list.

### Q3: What is a Monotonic Stack and when do you use it?

**Answer:**
- A stack where elements are maintained in strictly increasing or decreasing order.
- Used for: Next Greater Element, Daily Temperatures, Largest Rectangle in Histogram, Stock Span.
- Key insight: When a new element violates monotonic property, pop and process until order is restored.
- Time: O(n) — each element is pushed and popped at most once.

### Q4: How does a Priority Queue differ from a regular Queue?

**Answer:**
- Regular Queue: FIFO — first in, first out.
- Priority Queue: Elements dequeued by priority, not insertion order.
- Implemented as a **binary heap** — insert O(log n), extract-min/max O(log n).
- Use cases: Dijkstra's algorithm, task scheduling, merge K sorted lists.

---

## 8. Practice Problems

| # | Problem | Difficulty | Pattern |
|---|---------|------------|---------|
| 1 | Valid Parentheses | 🟢 Easy | Stack |
| 2 | Implement Queue using Stacks | 🟢 Easy | Two Stacks |
| 3 | Min Stack | 🟡 Medium | Dual Stack |
| 4 | Evaluate Reverse Polish Notation | 🟡 Medium | Stack |
| 5 | Daily Temperatures | 🟡 Medium | Monotonic Stack |
| 6 | Task Scheduler | 🟡 Medium | Greedy + Heap |
| 7 | Kth Largest Element | 🟡 Medium | Min-Heap |
| 8 | Sliding Window Maximum | 🔴 Hard | Deque |
| 9 | Largest Rectangle in Histogram | 🔴 Hard | Monotonic Stack |
| 10 | Basic Calculator | 🔴 Hard | Stack + Parsing |

---

**Previous:** [03 — Linked Lists ←](03-linked-lists.md) | **Next:** [05 — Trees & Graphs →](05-trees-graphs.md)
