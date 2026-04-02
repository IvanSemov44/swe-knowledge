# Data Structures

## What it is
The fundamental ways of organizing and storing data in memory, each with different performance characteristics for insertion, deletion, and lookup.

## Why it matters
Every algorithm operates on a data structure. Choosing the wrong one is the most common cause of poor performance. Interviewers test this constantly.

---

## Arrays

**What:** Contiguous block of memory. Fixed size (static) or resizable (dynamic/`List<T>` in C#).

**Operations:**
| Operation | Time |
|---|---|
| Access by index | O(1) |
| Search (unsorted) | O(n) |
| Search (sorted, binary) | O(log n) |
| Insert at end (amortized) | O(1) |
| Insert at middle | O(n) |
| Delete at middle | O(n) |

**In C#:** `int[]` (fixed), `List<T>` (dynamic)

**Common patterns:** Two pointers, sliding window, prefix sums — all start with arrays.

---

## Linked Lists

**What:** Nodes where each node holds a value and a pointer to the next node. No contiguous memory.

**Singly linked:** next pointer only.
**Doubly linked:** next + prev pointers.

**Operations:**
| Operation | Time |
|---|---|
| Access by index | O(n) |
| Insert/delete at head | O(1) |
| Insert/delete at tail (with tail ptr) | O(1) |
| Insert/delete in middle | O(n) to find + O(1) to splice |
| Search | O(n) |

**In C#:** `LinkedList<T>` (doubly linked)

**Traps:**
- Forgetting to handle `null` (empty list, single node)
- Off-by-one when traversing
- Losing a node's reference before rewiring pointers (always store `next` before reassigning)

---

## Stacks

**What:** Last-In-First-Out (LIFO). Push to top, pop from top.

**Operations:** Push O(1), Pop O(1), Peek O(1)

**In C#:** `Stack<T>`

**Use cases:** Undo/redo, expression parsing, DFS (iterative), balanced parentheses, call stack simulation.

---

## Queues

**What:** First-In-First-Out (FIFO). Enqueue to back, dequeue from front.

**Operations:** Enqueue O(1), Dequeue O(1)

**In C#:** `Queue<T>`

**Use cases:** BFS, task scheduling, rate limiting, print queues.

**Deque (double-ended queue):** Insert/remove from both ends. In C#: `LinkedList<T>` or implement manually.

---

## Hash Maps / Hash Tables

**What:** Key-value pairs. Hash function maps a key to a bucket index. Handles collisions via chaining or open addressing.

**Operations (average case):**
| Operation | Time |
|---|---|
| Get | O(1) |
| Put | O(1) |
| Delete | O(1) |
| Worst case (all collisions) | O(n) |

**In C#:** `Dictionary<TKey, TValue>`, `HashSet<T>`

**Use cases:** Frequency counting, two-sum, grouping, deduplication, caching.

**Traps:**
- Worst-case O(n) if hash function is terrible
- Key must be immutable or `GetHashCode` must be consistent
- `HashSet` when you only need membership check, not values

---

## Trees

**What:** Hierarchical structure with a root node. Each node has zero or more children.

### Binary Tree
Each node has at most 2 children (left, right). No ordering guarantee.

### Binary Search Tree (BST)
Left subtree < node < right subtree.

**BST Operations (balanced):**
| Operation | Time |
|---|---|
| Search | O(log n) |
| Insert | O(log n) |
| Delete | O(log n) |
| Worst case (skewed) | O(n) |

**Traversals:**
- **Inorder** (L → Node → R): gives sorted order for BST
- **Preorder** (Node → L → R): serialize/clone tree
- **Postorder** (L → R → Node): delete tree, evaluate expressions
- **Level-order** (BFS): shortest path, level-by-level processing

**In C#:** No built-in BST. `SortedDictionary<K,V>` uses a Red-Black tree internally.

---

## Heaps

**What:** Complete binary tree satisfying the heap property.
- **Min-heap:** parent ≤ children (root is minimum)
- **Max-heap:** parent ≥ children (root is maximum)

**Operations:**
| Operation | Time |
|---|---|
| Get min/max (peek) | O(1) |
| Insert | O(log n) |
| Extract min/max | O(log n) |
| Build heap from array | O(n) |

**In C#:** `PriorityQueue<TElement, TPriority>` (.NET 6+)

**Use cases:** K largest/smallest elements, Dijkstra's algorithm, merge K sorted lists, scheduling.

---

## Tries (Prefix Trees)

**What:** Tree where each node represents a character. Words are paths from root to leaf. Nodes share prefixes.

**Operations:** Insert O(m), Search O(m), StartsWith O(m) — where m = word length

**Use cases:** Autocomplete, spell check, IP routing, dictionary word searches.

---

## Graphs

**What:** Nodes (vertices) connected by edges. Can be directed or undirected, weighted or unweighted.

**Representations:**
- **Adjacency list:** `Dictionary<int, List<int>>` — space O(V + E), efficient for sparse graphs
- **Adjacency matrix:** `int[,]` — space O(V²), efficient for dense graphs, fast edge lookup

**Key traversals:** BFS (shortest path in unweighted), DFS (connectivity, cycle detection, topological sort)

---

## Big O Summary

| Structure | Access | Search | Insert | Delete |
|---|---|---|---|---|
| Array | O(1) | O(n) | O(n) | O(n) |
| LinkedList | O(n) | O(n) | O(1) head | O(1) head |
| Stack | O(n) | O(n) | O(1) | O(1) |
| Queue | O(n) | O(n) | O(1) | O(1) |
| HashMap | N/A | O(1) avg | O(1) avg | O(1) avg |
| BST (balanced) | N/A | O(log n) | O(log n) | O(log n) |
| Heap | O(1) top | O(n) | O(log n) | O(log n) |

---

## Common Interview Questions

1. Find duplicates in an array — HashMap frequency count O(n)
2. Two sum — HashMap complement lookup O(n)
3. Reverse a linked list — iterative with three pointers
4. Check balanced parentheses — Stack
5. Find Kth largest element — Min-heap of size K
6. Lowest common ancestor in BST
7. Serialize and deserialize a binary tree

---

## Common Mistakes / Traps

- Using a list when a set gives O(1) lookup instead of O(n)
- Forgetting null checks at list head/tail
- Modifying a collection while iterating it
- Thinking `Dictionary` lookup is always O(1) — worst case is O(n)
- Confusing pre/in/post order traversal direction

---

## How It Connects

- Algorithms need the right data structure to achieve their Big O target
- Hash maps underpin many O(n) solutions that would otherwise be O(n²)
- Trees appear in EF Core expression trees, JSON parsing, compiler ASTs
- Heaps appear in task schedulers, priority queues in message brokers
- Graphs model database relationships, dependency graphs (DI containers), microservice call graphs

---

## My Confidence Level
- `[~]` Arrays — know them, need to sharpen pattern recognition
- `[~]` Linked lists — know the concept, slow on implementation
- `[~]` Stacks/Queues — comfortable conceptually
- `[ ]` Trees — need practice
- `[ ]` Heaps — never implemented from scratch
- `[ ]` Tries — never studied deeply
- `[ ]` Graphs — need practice

## My Notes
<!-- Add personal notes, aha moments, things you always forget -->
