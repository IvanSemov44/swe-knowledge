# Big O Notation

## What it is
A mathematical notation describing how the runtime or space requirements of an algorithm grow relative to the input size (n). It describes the **upper bound** — worst case behavior.

## Why it matters
Choosing between O(n) and O(n²) is the difference between an API responding in 10ms vs timing out at 10 seconds for a list of 100,000 items.

---

## The Big O Classes (fastest to slowest)

| Notation | Name | Example |
|---|---|---|
| O(1) | Constant | Array access by index, HashMap lookup |
| O(log n) | Logarithmic | Binary search, balanced BST operations |
| O(n) | Linear | Linear scan, single loop |
| O(n log n) | Linearithmic | Merge sort, quick sort (avg) |
| O(n²) | Quadratic | Nested loops, bubble sort |
| O(n³) | Cubic | Triple nested loops |
| O(2ⁿ) | Exponential | Recursive Fibonacci without memo, subsets |
| O(n!) | Factorial | Permutations |

**Rule of thumb for n = 10,000:**
- O(1), O(log n), O(n) — fast
- O(n log n) — acceptable
- O(n²) — slow (100M operations)
- O(2ⁿ) — never use without optimization

---

## Rules for Calculating Big O

### 1. Drop constants
`O(2n)` → `O(n)` because constants don't change growth rate.

### 2. Drop lower-order terms
`O(n² + n)` → `O(n²)` because n² dominates for large n.

### 3. Different inputs = different variables
```csharp
// This is O(a + b), NOT O(n²)
foreach (var x in arrayA)    // O(a)
    Console.WriteLine(x);
foreach (var y in arrayB)    // O(b)
    Console.WriteLine(y);
```

```csharp
// This IS O(a * b)
foreach (var x in arrayA)
    foreach (var y in arrayB)
        Console.WriteLine(x + y);
```

### 4. Recursion
**Time:** O(branches^depth) in the general case.
Fibonacci naive: 2 branches, depth n → O(2ⁿ)
Binary search: 1 branch, depth log n → O(log n)

---

## Space Complexity

Space complexity = extra memory your algorithm uses (not counting input).

| Algorithm | Space |
|---|---|
| Iterative loop | O(1) |
| Recursive call stack (depth d) | O(d) |
| Creating copy of input | O(n) |
| HashMap of all elements | O(n) |
| 2D DP table n×n | O(n²) |

**Tradeoff:** Often you can trade space for time.
- Two sum O(n²) time O(1) space → O(n) time O(n) space with HashMap

---

## Amortized Analysis

Average cost per operation over a sequence of operations.

**Dynamic array push:**
- Most pushes are O(1)
- Occasional doubling is O(n)
- Amortized per operation: O(1)

---

## Best / Average / Worst Case

| Case | Meaning | Example (Quick Sort) |
|---|---|---|
| Best | Optimal input | Already sorted with random pivot → O(n log n) |
| Average | Random input | O(n log n) |
| Worst | Worst input | Sorted array + always pick first element as pivot → O(n²) |

Big O **technically** only says "upper bound." When people say "Quick Sort is O(n log n)" they usually mean average case. Worst case is O(n²).

Θ (Theta) = tight bound (both upper and lower)
Ω (Omega) = lower bound

In interviews, "Big O" usually means worst case unless specified.

---

## Common Pitfalls

**Is this O(n) or O(n²)?**
```csharp
// O(n) — string.Contains is O(m) but m is bounded by fixed alphabet
foreach (var s in strings)
    if (s.Contains("hello")) ...

// O(n * m) — if m varies with n, it's O(n²) in the worst case
foreach (var s in strings)     // O(n)
    foreach (var c in s)       // O(m) per string
        ...
```

**Hash map operations:**
- Average O(1) but don't forget worst-case O(n) with collisions
- In practice, .NET's `Dictionary<K,V>` has excellent hash distribution

**LINQ chains:**
```csharp
list.Where(...).Select(...).OrderBy(...).Take(10)
// Where: O(n), Select: O(n), OrderBy: O(n log n), Take: O(1)
// Total: O(n log n)
```

---

## How It Connects

- Every data structure and algorithm choice is a Big O decision
- EF Core LINQ queries translate to SQL — N+1 is O(n) queries instead of O(1)
- Caching turns repeated O(n) computations into O(1) lookups
- Index on a DB column turns O(n) table scan into O(log n) B-tree lookup
- Redis sorted sets use skip lists for O(log n) range queries

---

## Common Interview Questions

1. What is the time complexity of `Dictionary.ContainsKey`?
2. Is O(n/2) the same as O(n)? Why?
3. What is the space complexity of merge sort vs quick sort?
4. A loop runs n/2 times — what is the Big O?
5. You have two nested loops, but the inner runs from i to n — what is the Big O? (O(n²/2) → O(n²))

---

## My Confidence Level
- `[b]` Reading Big O of simple code
- `[b]` Knowing common data structure Big O
- `[~]` Analyzing recursive algorithms
- `[ ]` Amortized analysis

## My Notes
<!-- Add aha moments, things you keep forgetting -->
