# 03 — Linked Lists

> Linear data structures with dynamic memory allocation — essential for queues, caches, and event chains.

---

## Table of Contents

1. [Singly Linked List](#1-singly-linked-list)
2. [Doubly Linked List](#2-doubly-linked-list)
3. [Fast & Slow Pointer Technique](#3-fast--slow-pointer-technique)
4. [Common Operations](#4-common-operations)
5. [LRU Cache Implementation](#5-lru-cache-implementation)
6. [Node.js Real-World Connections](#6-nodejs-real-world-connections)
7. [Interview Questions & Answers](#7-interview-questions--answers)
8. [Practice Problems](#8-practice-problems)

---

## 1. Singly Linked List

```typescript
class ListNode<T> {
  val: T;
  next: ListNode<T> | null;

  constructor(val: T, next: ListNode<T> | null = null) {
    this.val = val;
    this.next = next;
  }
}

class SinglyLinkedList<T> {
  head: ListNode<T> | null = null;
  size: number = 0;

  // O(1) — Insert at head
  prepend(val: T): void {
    this.head = new ListNode(val, this.head);
    this.size++;
  }

  // O(n) — Insert at tail
  append(val: T): void {
    const node = new ListNode(val);
    if (!this.head) {
      this.head = node;
    } else {
      let current = this.head;
      while (current.next) current = current.next;
      current.next = node;
    }
    this.size++;
  }

  // O(1) — Remove head
  removeHead(): T | null {
    if (!this.head) return null;
    const val = this.head.val;
    this.head = this.head.next;
    this.size--;
    return val;
  }

  // O(n) — Remove by value
  remove(val: T): boolean {
    if (!this.head) return false;
    if (this.head.val === val) {
      this.head = this.head.next;
      this.size--;
      return true;
    }
    let current = this.head;
    while (current.next) {
      if (current.next.val === val) {
        current.next = current.next.next;
        this.size--;
        return true;
      }
      current = current.next;
    }
    return false;
  }

  // O(n) — Search
  find(val: T): ListNode<T> | null {
    let current = this.head;
    while (current) {
      if (current.val === val) return current;
      current = current.next;
    }
    return null;
  }

  // O(n) — Convert to array
  toArray(): T[] {
    const result: T[] = [];
    let current = this.head;
    while (current) {
      result.push(current.val);
      current = current.next;
    }
    return result;
  }
}
```

---

## 2. Doubly Linked List

```typescript
class DListNode<T> {
  val: T;
  prev: DListNode<T> | null;
  next: DListNode<T> | null;

  constructor(val: T) {
    this.val = val;
    this.prev = null;
    this.next = null;
  }
}

class DoublyLinkedList<T> {
  head: DListNode<T>;  // Sentinel head
  tail: DListNode<T>;  // Sentinel tail
  size: number = 0;

  constructor() {
    this.head = new DListNode<T>(null as T);
    this.tail = new DListNode<T>(null as T);
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  // O(1) — Insert after a node
  insertAfter(node: DListNode<T>, val: T): DListNode<T> {
    const newNode = new DListNode(val);
    newNode.prev = node;
    newNode.next = node.next;
    node.next!.prev = newNode;
    node.next = newNode;
    this.size++;
    return newNode;
  }

  // O(1) — Insert at front
  insertFront(val: T): DListNode<T> {
    return this.insertAfter(this.head, val);
  }

  // O(1) — Insert at back
  insertBack(val: T): DListNode<T> {
    return this.insertAfter(this.tail.prev!, val);
  }

  // O(1) — Remove a specific node
  removeNode(node: DListNode<T>): T {
    node.prev!.next = node.next;
    node.next!.prev = node.prev;
    this.size--;
    return node.val;
  }

  // O(1) — Remove and return last element
  removeLast(): T | null {
    if (this.size === 0) return null;
    return this.removeNode(this.tail.prev!);
  }

  // O(1) — Move existing node to front
  moveToFront(node: DListNode<T>): void {
    this.removeNode(node);
    this.size++; // removeNode decrements, so re-increment
    node.next = this.head.next;
    node.prev = this.head;
    this.head.next!.prev = node;
    this.head.next = node;
  }
}
```

---

## 3. Fast & Slow Pointer Technique

### Cycle Detection (Floyd's Algorithm)

```typescript
// Detect if linked list has a cycle — O(n) time, O(1) space
function hasCycle(head: ListNode<number> | null): boolean {
  let slow = head;
  let fast = head;

  while (fast && fast.next) {
    slow = slow!.next;
    fast = fast.next.next;
    if (slow === fast) return true;
  }
  return false;
}

// Find the start of the cycle
function detectCycleStart(head: ListNode<number> | null): ListNode<number> | null {
  let slow = head;
  let fast = head;

  // Phase 1: Detect cycle
  while (fast && fast.next) {
    slow = slow!.next;
    fast = fast.next.next;
    if (slow === fast) break;
  }

  if (!fast || !fast.next) return null; // No cycle

  // Phase 2: Find cycle start
  slow = head;
  while (slow !== fast) {
    slow = slow!.next;
    fast = fast!.next;
  }
  return slow;
}
```

### Find Middle Node

```typescript
// Find middle of linked list — O(n) time, O(1) space
function findMiddle(head: ListNode<number> | null): ListNode<number> | null {
  let slow = head;
  let fast = head;

  while (fast && fast.next) {
    slow = slow!.next;
    fast = fast.next.next;
  }
  return slow; // slow is at middle when fast reaches end
}
```

---

## 4. Common Operations

### Reverse a Linked List

```typescript
// Iterative — O(n) time, O(1) space
function reverseList(head: ListNode<number> | null): ListNode<number> | null {
  let prev: ListNode<number> | null = null;
  let current = head;

  while (current) {
    const next = current.next;
    current.next = prev;
    prev = current;
    current = next;
  }
  return prev;
}

// Recursive — O(n) time, O(n) space (call stack)
function reverseListRecursive(head: ListNode<number> | null): ListNode<number> | null {
  if (!head || !head.next) return head;
  const newHead = reverseListRecursive(head.next);
  head.next.next = head;
  head.next = null;
  return newHead;
}
```

### Merge Two Sorted Lists

```typescript
function mergeTwoLists(
  l1: ListNode<number> | null,
  l2: ListNode<number> | null
): ListNode<number> | null {
  const dummy = new ListNode(0);
  let current = dummy;

  while (l1 && l2) {
    if (l1.val <= l2.val) {
      current.next = l1;
      l1 = l1.next;
    } else {
      current.next = l2;
      l2 = l2.next;
    }
    current = current.next;
  }
  current.next = l1 || l2;
  return dummy.next;
}
```

### Remove Nth Node From End

```typescript
function removeNthFromEnd(head: ListNode<number> | null, n: number): ListNode<number> | null {
  const dummy = new ListNode(0, head);
  let fast: ListNode<number> | null = dummy;
  let slow: ListNode<number> | null = dummy;

  // Advance fast by n+1 steps
  for (let i = 0; i <= n; i++) {
    fast = fast!.next;
  }

  // Move both until fast reaches end
  while (fast) {
    slow = slow!.next;
    fast = fast.next;
  }

  // Remove the target node
  slow!.next = slow!.next!.next;
  return dummy.next;
}
```

---

## 5. LRU Cache Implementation

```typescript
// LRU Cache — O(1) get and put using HashMap + Doubly Linked List
class LRUCache {
  private capacity: number;
  private cache: Map<number, DListNode<{ key: number; val: number }>>;
  private list: DoublyLinkedList<{ key: number; val: number }>;

  constructor(capacity: number) {
    this.capacity = capacity;
    this.cache = new Map();
    this.list = new DoublyLinkedList();
  }

  get(key: number): number {
    if (!this.cache.has(key)) return -1;
    const node = this.cache.get(key)!;
    this.list.moveToFront(node);
    return node.val.val;
  }

  put(key: number, value: number): void {
    if (this.cache.has(key)) {
      const node = this.cache.get(key)!;
      node.val.val = value;
      this.list.moveToFront(node);
    } else {
      if (this.list.size >= this.capacity) {
        const removed = this.list.removeLast()!;
        this.cache.delete(removed.key);
      }
      const node = this.list.insertFront({ key, val: value });
      this.cache.set(key, node);
    }
  }
}

// Usage: API response cache with eviction
const cache = new LRUCache(100);
cache.put(1, 42);
cache.get(1); // 42 — moves to front (most recently used)
```

---

## 6. Node.js Real-World Connections

```
┌───────────────────────────┬──────────────────────────────────────┐
│ Linked List Concept       │ Node.js Use Case                     │
├───────────────────────────┼──────────────────────────────────────┤
│ Singly Linked List        │ Express middleware chain (next())     │
│ Doubly Linked List        │ Browser history (back/forward)        │
│ LRU Cache                 │ In-memory API response caching        │
│ Queue (LL-based)          │ BullMQ job queue processing           │
│ Cycle Detection           │ Detecting circular dependencies      │
│ Merge Sorted Lists        │ Merging sorted DB query results      │
└───────────────────────────┴──────────────────────────────────────┘
```

### Express Middleware as Linked List

```typescript
// Express middleware is essentially a linked list!
// Each middleware calls next() to pass to the next node
app.use((req, res, next) => {
  console.log('Middleware 1');
  next(); // → pointer to next middleware
});
app.use((req, res, next) => {
  console.log('Middleware 2');
  next(); // → pointer to next middleware
});
app.get('/', (req, res) => {
  res.send('End of chain');
});
```

---

## 7. Interview Questions & Answers

### Q1: When would you use a Linked List over an Array?

**Answer:**
- **Linked List** when you need frequent insertions/deletions at the beginning or middle (O(1) with reference).
- **Array** when you need random access by index (O(1)).
- In JS, arrays are optimized by V8 and usually faster than linked lists due to cache locality.
- Use linked lists for **queues**, **LRU caches**, and **middleware chains**.

### Q2: What is the dummy/sentinel node technique?

**Answer:**
- A **dummy node** is a placeholder node added before the real head.
- Simplifies edge cases: empty list, removing head, inserting at head.
- After operations, `dummy.next` is the new head.
- Avoids special-case `if (!head)` checks throughout the code.

### Q3: How do you detect if two linked lists intersect?

**Answer:**
- Calculate lengths of both lists. Advance the longer list by the difference.
- Then walk both lists simultaneously — they'll meet at the intersection.
- Alternative: Use two pointers — when one reaches the end, redirect it to the other list's head. They meet at the intersection or both reach null.
- Time: O(n + m), Space: O(1).

### Q4: Why is LRU Cache implemented with HashMap + Doubly Linked List?

**Answer:**
- **HashMap** gives O(1) key lookup.
- **Doubly Linked List** gives O(1) insertion, deletion, and move-to-front.
- Combined: O(1) for both `get` and `put`.
- Singly linked list won't work because removing a node requires access to the previous node.

---

## 8. Practice Problems

| # | Problem | Difficulty | Pattern |
|---|---------|------------|---------|
| 1 | Reverse Linked List | 🟢 Easy | Iteration / Recursion |
| 2 | Merge Two Sorted Lists | 🟢 Easy | Two Pointers |
| 3 | Linked List Cycle | 🟢 Easy | Fast-Slow Pointers |
| 4 | Remove Nth Node From End | 🟡 Medium | Two Pointers |
| 5 | Reorder List | 🟡 Medium | Find Middle + Reverse + Merge |
| 6 | Add Two Numbers | 🟡 Medium | Math + Linked List |
| 7 | Copy List with Random Pointer | 🟡 Medium | Hash Map |
| 8 | LRU Cache | 🟡 Medium | HashMap + Doubly LL |
| 9 | Reverse Nodes in K-Group | 🔴 Hard | Reverse + Recursion |
| 10 | Merge K Sorted Lists | 🔴 Hard | Divide & Conquer / Heap |

---

**Previous:** [02 — Arrays & Strings ←](02-arrays-strings.md) | **Next:** [04 — Stacks & Queues →](04-stacks-queues.md)
