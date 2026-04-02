# Algorithms

## What it is
Step-by-step procedures for solving computational problems. Algorithms are analyzed by time and space complexity and chosen based on the data structure and constraints.

## Why it matters
Every coding interview tests algorithms. More importantly, production systems hit performance issues when the wrong algorithm is applied at scale.

---

## Sorting Algorithms

### Bubble Sort
- Repeatedly swap adjacent elements if out of order
- **Time:** O(n²) avg/worst, O(n) best (already sorted with early exit)
- **Space:** O(1) in-place
- Use: never in production; useful for teaching

### Selection Sort
- Find minimum, put it at front. Repeat.
- **Time:** O(n²) always
- **Space:** O(1)
- Use: never in production

### Insertion Sort
- Build sorted array one element at a time
- **Time:** O(n²) avg/worst, O(n) best
- **Space:** O(1)
- Use: small arrays or nearly sorted data; used inside Timsort for small runs

### Merge Sort
- Divide array in half, sort each half, merge
- **Time:** O(n log n) always
- **Space:** O(n) — needs auxiliary array
- **Stable:** yes
- Use: when stability required; linked lists (no random access needed)

```csharp
void MergeSort(int[] arr, int left, int right) {
    if (left >= right) return;
    int mid = (left + right) / 2;
    MergeSort(arr, left, mid);
    MergeSort(arr, mid + 1, right);
    Merge(arr, left, mid, right);
}
```

### Quick Sort
- Pick pivot, partition around it, recurse
- **Time:** O(n log n) avg, O(n²) worst (sorted input + bad pivot)
- **Space:** O(log n) stack space
- **Stable:** no (standard implementation)
- Use: default choice; fast in practice; Array.Sort uses this (Introsort variant)

```csharp
// Partition: elements < pivot go left, > pivot go right
// Return pivot's final index
```

### Heap Sort
- Build max-heap, repeatedly extract max
- **Time:** O(n log n) always
- **Space:** O(1)
- **Stable:** no
- Use: when guaranteed O(n log n) needed with O(1) space

### Counting / Radix Sort
- Non-comparison sorts
- **Time:** O(n + k) where k is range of values
- Use: when values have a bounded range (e.g., ages, scores)

---

## Searching Algorithms

### Linear Search
- Scan every element
- **Time:** O(n)
- Use: unsorted data

### Binary Search
- Requires sorted array. Halve the search space each step.
- **Time:** O(log n)
- **Space:** O(1) iterative, O(log n) recursive

```csharp
int BinarySearch(int[] arr, int target) {
    int left = 0, right = arr.Length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2; // avoid overflow vs (l+r)/2
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```

**Trap:** `(left + right) / 2` overflows for large indices — use `left + (right - left) / 2`

**Variants:** Find first occurrence, last occurrence, leftmost position where condition is true (binary search on answer).

---

## Graph Algorithms

### BFS (Breadth-First Search)
- Explore all neighbors at current depth before going deeper
- Uses a **queue**
- Finds **shortest path** in unweighted graphs

```csharp
void BFS(Dictionary<int, List<int>> graph, int start) {
    var visited = new HashSet<int>();
    var queue = new Queue<int>();
    queue.Enqueue(start);
    visited.Add(start);
    while (queue.Count > 0) {
        int node = queue.Dequeue();
        foreach (int neighbor in graph[node]) {
            if (!visited.Contains(neighbor)) {
                visited.Add(neighbor);
                queue.Enqueue(neighbor);
            }
        }
    }
}
```

**Use:** Shortest path (unweighted), level-order tree traversal, connected components, web crawlers.

### DFS (Depth-First Search)
- Go as deep as possible before backtracking
- Uses a **stack** (recursion or explicit)

```csharp
void DFS(Dictionary<int, List<int>> graph, int node, HashSet<int> visited) {
    visited.Add(node);
    foreach (int neighbor in graph[node]) {
        if (!visited.Contains(neighbor))
            DFS(graph, neighbor, visited);
    }
}
```

**Use:** Cycle detection, topological sort, connected components, maze solving, tree traversals.

### Dijkstra's Algorithm
- Shortest path in a **weighted** graph (non-negative weights)
- Uses a **min-heap** (priority queue)
- **Time:** O((V + E) log V)

**Use:** GPS routing, network packet routing, game pathfinding.

### Topological Sort
- Linear ordering of vertices where for every edge u→v, u comes before v
- Only works on **DAGs** (directed acyclic graphs)
- Methods: DFS-based (reverse postorder) or Kahn's algorithm (BFS with in-degree)

**Use:** Build systems, task scheduling, course prerequisites.

### Union-Find (Disjoint Set Union)
- Track which elements belong to the same set
- Operations: `Find` (which set?), `Union` (merge two sets)
- With path compression + union by rank: nearly O(1) amortized

**Use:** Kruskal's MST, detecting cycles in undirected graphs, network connectivity.

---

## Two Pointers

Use two pointers that move toward each other or in the same direction.

**Pattern 1 — Opposite ends (sorted array):**
```csharp
// Two sum on sorted array
int left = 0, right = arr.Length - 1;
while (left < right) {
    int sum = arr[left] + arr[right];
    if (sum == target) return (left, right);
    else if (sum < target) left++;
    else right--;
}
```

**Pattern 2 — Fast/slow pointers:**
- Detect cycle in linked list (Floyd's algorithm)
- Find middle of linked list

---

## Sliding Window

Maintain a window (subarray/substring) and slide it across the input.

**Fixed size window:**
```csharp
// Max sum of subarray of size k
int windowSum = arr[0..k].Sum();
int maxSum = windowSum;
for (int i = k; i < arr.Length; i++) {
    windowSum += arr[i] - arr[i - k];
    maxSum = Math.Max(maxSum, windowSum);
}
```

**Variable size window (expand/shrink):**
```csharp
// Longest substring without repeating characters
int left = 0, maxLen = 0;
var seen = new HashSet<char>();
for (int right = 0; right < s.Length; right++) {
    while (seen.Contains(s[right])) seen.Remove(s[left++]);
    seen.Add(s[right]);
    maxLen = Math.Max(maxLen, right - left + 1);
}
```

---

## Prefix Sums

Precompute cumulative sums so any subarray sum is O(1).

```csharp
// prefix[i] = sum of arr[0..i-1]
int[] prefix = new int[arr.Length + 1];
for (int i = 0; i < arr.Length; i++)
    prefix[i + 1] = prefix[i] + arr[i];

// Sum of arr[l..r] = prefix[r+1] - prefix[l]
```

---

## Recursion & Backtracking

**Recursion:** Function calls itself with a smaller subproblem. Always needs a base case.

**Backtracking:** Try a choice, recurse, if it fails — undo the choice (backtrack).

```csharp
// Generate all subsets
void Backtrack(int[] nums, int start, List<int> current, List<List<int>> result) {
    result.Add(new List<int>(current));
    for (int i = start; i < nums.Length; i++) {
        current.Add(nums[i]);
        Backtrack(nums, i + 1, current, result);
        current.RemoveAt(current.Count - 1); // backtrack
    }
}
```

**Use:** Permutations, combinations, N-Queens, Sudoku solver, word search.

---

## Dynamic Programming

Break problem into overlapping subproblems. Store results to avoid recomputing.

**Top-down (memoization):** Recursive + cache
**Bottom-up (tabulation):** Iterative, fill table from base cases up

**Fibonacci example:**
```csharp
// Bottom-up O(n) time, O(1) space
int Fib(int n) {
    if (n <= 1) return n;
    int a = 0, b = 1;
    for (int i = 2; i <= n; i++) (a, b) = (b, a + b);
    return b;
}
```

**Common DP patterns:**
- 1D: Fibonacci, climbing stairs, house robber
- 2D: Longest common subsequence, edit distance, coin change
- Knapsack: 0/1 knapsack, unbounded knapsack

---

## Common Interview Questions

1. Binary search on sorted rotated array
2. Merge two sorted arrays in-place
3. Find all permutations of a string (backtracking)
4. Longest increasing subsequence (DP)
5. Coin change — minimum coins (DP)
6. Number of islands (BFS/DFS on grid)
7. Course schedule — detect cycle (topological sort)

---

## Common Mistakes

- Binary search: `<=` vs `<` in the while condition (off-by-one)
- BFS: forgetting to mark nodes visited **when enqueuing** (not when dequeuing) — causes duplicates
- DP: not identifying overlapping subproblems → re-solving instead of memoizing
- Recursion: no base case → stack overflow
- Two pointers on unsorted array when the pattern requires sorted input

---

## How It Connects

- Algorithms + Data Structures are inseparable — algorithm choice depends on data structure
- BFS/DFS are used in EF Core's query plan traversal, compiler AST walks
- Binary search appears in `List.BinarySearch()`, `SortedSet`
- DP thinking transfers to caching strategies in backend (memoize expensive computations)

---

## My Confidence Level
- `[~]` Sorting — know the concepts, slow on implementation
- `[~]` Binary search — know it, make off-by-one errors
- `[ ]` BFS/DFS — need practice
- `[ ]` Two pointers — need practice
- `[ ]` Sliding window — need practice
- `[ ]` DP — weak area

## My Notes
<!-- Add personal notes -->
