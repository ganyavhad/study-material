# 05 — Trees & Graphs

> Hierarchical and network structures — critical for routing, dependency resolution, and search algorithms.

---

## Table of Contents

1. [Binary Tree Basics](#1-binary-tree-basics)
2. [Binary Search Tree (BST)](#2-binary-search-tree-bst)
3. [Tree Traversals](#3-tree-traversals)
4. [Trie (Prefix Tree)](#4-trie-prefix-tree)
5. [Graph Fundamentals](#5-graph-fundamentals)
6. [Graph Traversals (BFS & DFS)](#6-graph-traversals-bfs--dfs)
7. [Shortest Path Algorithms](#7-shortest-path-algorithms)
8. [Topological Sort](#8-topological-sort)
9. [Interview Questions & Answers](#9-interview-questions--answers)
10. [Practice Problems](#10-practice-problems)

---

## 1. Binary Tree Basics

```typescript
class TreeNode {
  val: number;
  left: TreeNode | null;
  right: TreeNode | null;

  constructor(val: number, left: TreeNode | null = null, right: TreeNode | null = null) {
    this.val = val;
    this.left = left;
    this.right = right;
  }
}

// Max Depth — O(n) time, O(h) space
function maxDepth(root: TreeNode | null): number {
  if (!root) return 0;
  return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}

// Invert Binary Tree — O(n) time
function invertTree(root: TreeNode | null): TreeNode | null {
  if (!root) return null;
  [root.left, root.right] = [invertTree(root.right), invertTree(root.left)];
  return root;
}

// Check if Symmetric — O(n) time
function isSymmetric(root: TreeNode | null): boolean {
  function isMirror(a: TreeNode | null, b: TreeNode | null): boolean {
    if (!a && !b) return true;
    if (!a || !b) return false;
    return a.val === b.val && isMirror(a.left, b.right) && isMirror(a.right, b.left);
  }
  return isMirror(root?.left ?? null, root?.right ?? null);
}

// Lowest Common Ancestor — O(n) time
function lowestCommonAncestor(
  root: TreeNode | null, p: TreeNode, q: TreeNode
): TreeNode | null {
  if (!root || root === p || root === q) return root;
  const left = lowestCommonAncestor(root.left, p, q);
  const right = lowestCommonAncestor(root.right, p, q);
  if (left && right) return root;
  return left || right;
}
```

---

## 2. Binary Search Tree (BST)

```
BST Property: left.val < node.val < right.val

        8
       / \
      3   10
     / \    \
    1   6    14
       / \   /
      4   7 13
```

```typescript
class BST {
  root: TreeNode | null = null;

  // O(log n) average, O(n) worst (skewed)
  insert(val: number): void {
    this.root = this._insert(this.root, val);
  }

  private _insert(node: TreeNode | null, val: number): TreeNode {
    if (!node) return new TreeNode(val);
    if (val < node.val) node.left = this._insert(node.left, val);
    else if (val > node.val) node.right = this._insert(node.right, val);
    return node;
  }

  // O(log n) average
  search(val: number): TreeNode | null {
    let current = this.root;
    while (current) {
      if (val === current.val) return current;
      current = val < current.val ? current.left : current.right;
    }
    return null;
  }

  // O(log n) average
  delete(val: number): void {
    this.root = this._delete(this.root, val);
  }

  private _delete(node: TreeNode | null, val: number): TreeNode | null {
    if (!node) return null;
    if (val < node.val) {
      node.left = this._delete(node.left, val);
    } else if (val > node.val) {
      node.right = this._delete(node.right, val);
    } else {
      // Node found
      if (!node.left) return node.right;
      if (!node.right) return node.left;
      // Two children: replace with inorder successor
      let successor = node.right;
      while (successor.left) successor = successor.left;
      node.val = successor.val;
      node.right = this._delete(node.right, successor.val);
    }
    return node;
  }

  // Validate BST — O(n)
  isValid(): boolean {
    function validate(node: TreeNode | null, min: number, max: number): boolean {
      if (!node) return true;
      if (node.val <= min || node.val >= max) return false;
      return validate(node.left, min, node.val) && validate(node.right, node.val, max);
    }
    return validate(this.root, -Infinity, Infinity);
  }
}
```

---

## 3. Tree Traversals

```typescript
// DFS Traversals — O(n) time, O(h) space

// Inorder: Left → Root → Right (gives sorted order for BST)
function inorder(root: TreeNode | null): number[] {
  if (!root) return [];
  return [...inorder(root.left), root.val, ...inorder(root.right)];
}

// Preorder: Root → Left → Right (useful for serialization)
function preorder(root: TreeNode | null): number[] {
  if (!root) return [];
  return [root.val, ...preorder(root.left), ...preorder(root.right)];
}

// Postorder: Left → Right → Root (useful for deletion, calc)
function postorder(root: TreeNode | null): number[] {
  if (!root) return [];
  return [...postorder(root.left), ...postorder(root.right), root.val];
}

// BFS / Level-Order Traversal — O(n) time, O(w) space (w = max width)
function levelOrder(root: TreeNode | null): number[][] {
  if (!root) return [];
  const result: number[][] = [];
  const queue: TreeNode[] = [root];

  while (queue.length > 0) {
    const levelSize = queue.length;
    const level: number[] = [];

    for (let i = 0; i < levelSize; i++) {
      const node = queue.shift()!;
      level.push(node.val);
      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }
    result.push(level);
  }
  return result;
}

// Iterative Inorder using Stack
function inorderIterative(root: TreeNode | null): number[] {
  const result: number[] = [];
  const stack: TreeNode[] = [];
  let current = root;

  while (current || stack.length > 0) {
    while (current) {
      stack.push(current);
      current = current.left;
    }
    current = stack.pop()!;
    result.push(current.val);
    current = current.right;
  }
  return result;
}
```

---

## 4. Trie (Prefix Tree)

```typescript
class TrieNode {
  children: Map<string, TrieNode> = new Map();
  isEnd: boolean = false;
}

class Trie {
  private root = new TrieNode();

  // O(m) where m = word length
  insert(word: string): void {
    let node = this.root;
    for (const char of word) {
      if (!node.children.has(char)) {
        node.children.set(char, new TrieNode());
      }
      node = node.children.get(char)!;
    }
    node.isEnd = true;
  }

  // O(m)
  search(word: string): boolean {
    const node = this.findNode(word);
    return node !== null && node.isEnd;
  }

  // O(m)
  startsWith(prefix: string): boolean {
    return this.findNode(prefix) !== null;
  }

  private findNode(prefix: string): TrieNode | null {
    let node = this.root;
    for (const char of prefix) {
      if (!node.children.has(char)) return null;
      node = node.children.get(char)!;
    }
    return node;
  }

  // Autocomplete — return all words with given prefix
  autocomplete(prefix: string): string[] {
    const node = this.findNode(prefix);
    if (!node) return [];
    const results: string[] = [];
    this.dfs(node, prefix, results);
    return results;
  }

  private dfs(node: TrieNode, prefix: string, results: string[]): void {
    if (node.isEnd) results.push(prefix);
    for (const [char, child] of node.children) {
      this.dfs(child, prefix + char, results);
    }
  }
}

// Node.js use case: Express-like route matching
const routeTrie = new Trie();
routeTrie.insert('/api/users');
routeTrie.insert('/api/users/profile');
routeTrie.insert('/api/products');
routeTrie.autocomplete('/api/u'); // ['/api/users', '/api/users/profile']
```

---

## 5. Graph Fundamentals

```typescript
// Adjacency List representation (most common for sparse graphs)
class Graph {
  private adjacencyList: Map<string, string[]> = new Map();

  addVertex(vertex: string): void {
    if (!this.adjacencyList.has(vertex)) {
      this.adjacencyList.set(vertex, []);
    }
  }

  addEdge(v1: string, v2: string, directed = false): void {
    this.adjacencyList.get(v1)?.push(v2);
    if (!directed) {
      this.adjacencyList.get(v2)?.push(v1);
    }
  }

  getNeighbors(vertex: string): string[] {
    return this.adjacencyList.get(vertex) || [];
  }

  get vertices(): string[] {
    return Array.from(this.adjacencyList.keys());
  }
}

// Weighted Graph with Adjacency List
class WeightedGraph {
  private adj: Map<string, Array<{ node: string; weight: number }>> = new Map();

  addVertex(vertex: string): void {
    if (!this.adj.has(vertex)) this.adj.set(vertex, []);
  }

  addEdge(v1: string, v2: string, weight: number, directed = false): void {
    this.adj.get(v1)?.push({ node: v2, weight });
    if (!directed) this.adj.get(v2)?.push({ node: v1, weight });
  }

  getNeighbors(vertex: string) {
    return this.adj.get(vertex) || [];
  }
}
```

---

## 6. Graph Traversals (BFS & DFS)

```typescript
// BFS — O(V + E) time, O(V) space
function bfs(graph: Graph, start: string): string[] {
  const visited = new Set<string>();
  const queue: string[] = [start];
  const result: string[] = [];
  visited.add(start);

  while (queue.length > 0) {
    const vertex = queue.shift()!;
    result.push(vertex);

    for (const neighbor of graph.getNeighbors(vertex)) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push(neighbor);
      }
    }
  }
  return result;
}

// DFS — O(V + E) time, O(V) space
function dfs(graph: Graph, start: string): string[] {
  const visited = new Set<string>();
  const result: string[] = [];

  function explore(vertex: string): void {
    visited.add(vertex);
    result.push(vertex);
    for (const neighbor of graph.getNeighbors(vertex)) {
      if (!visited.has(neighbor)) explore(neighbor);
    }
  }

  explore(start);
  return result;
}

// Number of Islands (Grid BFS) — O(m * n)
function numIslands(grid: string[][]): number {
  const rows = grid.length;
  const cols = grid[0].length;
  let count = 0;

  function bfsFlood(r: number, c: number): void {
    const queue: [number, number][] = [[r, c]];
    grid[r][c] = '0';

    while (queue.length > 0) {
      const [row, col] = queue.shift()!;
      const dirs = [[1, 0], [-1, 0], [0, 1], [0, -1]];
      for (const [dr, dc] of dirs) {
        const nr = row + dr, nc = col + dc;
        if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] === '1') {
          grid[nr][nc] = '0';
          queue.push([nr, nc]);
        }
      }
    }
  }

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (grid[r][c] === '1') {
        count++;
        bfsFlood(r, c);
      }
    }
  }
  return count;
}

// Detect Cycle in Directed Graph (DFS with coloring) — O(V + E)
function hasCycleDirected(graph: Map<number, number[]>, numNodes: number): boolean {
  const WHITE = 0, GRAY = 1, BLACK = 2;
  const color = new Array(numNodes).fill(WHITE);

  function dfs(node: number): boolean {
    color[node] = GRAY;
    for (const neighbor of (graph.get(node) || [])) {
      if (color[neighbor] === GRAY) return true;  // Back edge = cycle
      if (color[neighbor] === WHITE && dfs(neighbor)) return true;
    }
    color[node] = BLACK;
    return false;
  }

  for (let i = 0; i < numNodes; i++) {
    if (color[i] === WHITE && dfs(i)) return true;
  }
  return false;
}
```

---

## 7. Shortest Path Algorithms

### Dijkstra's Algorithm — O((V + E) log V)

```typescript
function dijkstra(
  graph: WeightedGraph,
  start: string,
  vertices: string[]
): Map<string, number> {
  const distances = new Map<string, number>();
  const visited = new Set<string>();

  for (const v of vertices) distances.set(v, Infinity);
  distances.set(start, 0);

  // Simple priority queue using sorted array (use MinHeap for production)
  const pq: Array<{ node: string; dist: number }> = [{ node: start, dist: 0 }];

  while (pq.length > 0) {
    pq.sort((a, b) => a.dist - b.dist);
    const { node } = pq.shift()!;

    if (visited.has(node)) continue;
    visited.add(node);

    for (const neighbor of graph.getNeighbors(node)) {
      if (visited.has(neighbor.node)) continue;
      const newDist = distances.get(node)! + neighbor.weight;
      if (newDist < distances.get(neighbor.node)!) {
        distances.set(neighbor.node, newDist);
        pq.push({ node: neighbor.node, dist: newDist });
      }
    }
  }
  return distances;
}
```

### BFS for Unweighted Shortest Path

```typescript
// Shortest path in unweighted graph — O(V + E)
function shortestPath(graph: Graph, start: string, end: string): string[] | null {
  const visited = new Set<string>([start]);
  const queue: Array<{ node: string; path: string[] }> = [
    { node: start, path: [start] },
  ];

  while (queue.length > 0) {
    const { node, path } = queue.shift()!;
    if (node === end) return path;

    for (const neighbor of graph.getNeighbors(node)) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push({ node: neighbor, path: [...path, neighbor] });
      }
    }
  }
  return null;
}
```

---

## 8. Topological Sort

```typescript
// Topological Sort (Kahn's Algorithm — BFS) — O(V + E)
// Use case: Build order, task dependency, npm install order
function topologicalSort(numCourses: number, prerequisites: number[][]): number[] {
  const inDegree = new Array(numCourses).fill(0);
  const adj = new Map<number, number[]>();

  for (let i = 0; i < numCourses; i++) adj.set(i, []);

  for (const [course, prereq] of prerequisites) {
    adj.get(prereq)!.push(course);
    inDegree[course]++;
  }

  const queue: number[] = [];
  for (let i = 0; i < numCourses; i++) {
    if (inDegree[i] === 0) queue.push(i);
  }

  const order: number[] = [];
  while (queue.length > 0) {
    const node = queue.shift()!;
    order.push(node);
    for (const neighbor of adj.get(node)!) {
      inDegree[neighbor]--;
      if (inDegree[neighbor] === 0) queue.push(neighbor);
    }
  }

  return order.length === numCourses ? order : []; // Empty = cycle exists
}

// Node.js use case: NestJS module dependency resolution
// Modules declare imports → forms a DAG → topological sort determines init order
```

---

## 9. Interview Questions & Answers

### Q1: BFS vs DFS — when to use which?

**Answer:**
- **BFS:** Shortest path in unweighted graphs, level-order traversal, nearest neighbor.
- **DFS:** Cycle detection, topological sort, connected components, path existence.
- BFS uses **queue** (O(w) space, w = max width); DFS uses **stack/recursion** (O(h) space, h = height).
- For trees: BFS = level order; DFS = inorder/preorder/postorder.

### Q2: How would you detect a cycle in a graph?

**Answer:**
- **Undirected graph:** DFS — if we visit an already-visited node that isn't the parent, there's a cycle. Or use Union-Find.
- **Directed graph:** DFS with 3-color marking (white/gray/black). A back edge to a gray node = cycle.
- **Topological sort:** If sort result has fewer nodes than total, cycle exists.

### Q3: What is a Trie and where is it used in Node.js?

**Answer:**
- A tree where each node represents a character; paths from root to leaf form words.
- **Express/Fastify** use trie-based routers for URL matching.
- Autocomplete, spell checking, IP routing tables, DNS lookup.
- O(m) insert/search where m = key length (independent of total keys stored).

### Q4: Explain Dijkstra's algorithm limitations.

**Answer:**
- Cannot handle **negative edge weights** — use Bellman-Ford instead.
- Greedy approach: once a node is finalized, its distance is optimal.
- Time: O((V + E) log V) with a min-heap.
- For unweighted graphs, BFS is simpler and faster: O(V + E).

---

## 10. Practice Problems

| # | Problem | Difficulty | Pattern |
|---|---------|------------|---------|
| 1 | Maximum Depth of Binary Tree | 🟢 Easy | DFS/Recursion |
| 2 | Invert Binary Tree | 🟢 Easy | DFS |
| 3 | Validate BST | 🟡 Medium | DFS + Bounds |
| 4 | Level Order Traversal | 🟡 Medium | BFS |
| 5 | Number of Islands | 🟡 Medium | BFS/DFS Grid |
| 6 | Course Schedule | 🟡 Medium | Topological Sort |
| 7 | Implement Trie | 🟡 Medium | Trie |
| 8 | Lowest Common Ancestor | 🟡 Medium | DFS |
| 9 | Word Search II | 🔴 Hard | Trie + DFS |
| 10 | Serialize/Deserialize Binary Tree | 🔴 Hard | BFS/DFS |

---

**Previous:** [04 — Stacks & Queues ←](04-stacks-queues.md) | **Next:** [06 — Hash Maps & Sets →](06-hashmaps-sets.md)
