# Advanced Algorithms

<!-- last-reviewed: 2026-04-09 | next-review: 2026-05-09 | confidence: ~ -->

---

## Two Pointers

Use two indices moving toward each other (or in the same direction) to avoid nested loops.

**Requires sorted array** — movement logic only works when you know:
- moving left pointer right → increases sum
- moving right pointer left → decreases sum

### Pattern: Two Sum (sorted array)

```typescript
function twoSum(nums: number[], target: number): number[] {
    let left = 0;
    let right = nums.length - 1;

    while (left < right) {
        const sum = nums[left] + nums[right];

        if (sum === target) return [nums[left], nums[right]];
        if (sum < target) left++;    // need bigger → move left right
        if (sum > target) right--;   // need smaller → move right left
    }

    return [];
}
```

**Time:** O(n) | **Space:** O(1)

### When to use two pointers

- Array is sorted (or you can sort it first)
- Problem asks for a pair/triplet that satisfies a condition
- Problem involves comparing elements from both ends

### Sorted vs unsorted — which approach

| Array state | Technique | Complexity |
|---|---|---|
| Sorted | Two pointers | O(n) |
| Unsorted | Hash set/map | O(n) |
| Unsorted + two pointers | Sort first, then two pointers | O(n log n) — worse than hash set |

**Rule:** If unsorted, use hash set. Sorting first to unlock two pointers is valid but suboptimal.

---

## Hash Set / Hash Map Pattern

For unsorted arrays. For each element, check if its complement has already been seen.

```typescript
function twoSum(nums: number[], target: number): number[] {
    const seen = new Set<number>();

    for (const num of nums) {
        const complement = target - num;
        if (seen.has(complement)) return [complement, num];
        seen.add(num);
    }

    return [];
}
```

**Time:** O(n) | **Space:** O(n)

When you need indices (not values):

```typescript
function twoSumIndices(nums: number[], target: number): number[] {
    const map = new Map<number, number>(); // value → index

    for (let i = 0; i < nums.length; i++) {
        const complement = target - nums[i];
        if (map.has(complement)) return [map.get(complement)!, i];
        map.set(nums[i], i);
    }

    return [];
}
```

---

## Sliding Window

For problems involving a contiguous subarray or substring. Maintain a window with two pointers, expand/shrink as needed.

### Fixed size window — max sum of k elements

```typescript
function maxSumSubarray(nums: number[], k: number): number {
    let windowSum = nums.slice(0, k).reduce((a, b) => a + b, 0);
    let maxSum = windowSum;

    for (let i = k; i < nums.length; i++) {
        windowSum += nums[i] - nums[i - k];  // slide: add new, remove old
        maxSum = Math.max(maxSum, windowSum);
    }

    return maxSum;
}
```

**Time:** O(n) | **Space:** O(1)

### Variable size window — longest subarray with sum ≤ target

```typescript
function longestSubarray(nums: number[], target: number): number {
    let left = 0;
    let sum = 0;
    let maxLen = 0;

    for (let right = 0; right < nums.length; right++) {
        sum += nums[right];

        while (sum > target) {
            sum -= nums[left];
            left++;
        }

        maxLen = Math.max(maxLen, right - left + 1);
    }

    return maxLen;
}
```

---

## Binary Search

Only works on sorted arrays. Eliminate half the search space each iteration.

```typescript
function binarySearch(nums: number[], target: number): number {
    let left = 0;
    let right = nums.length - 1;

    while (left <= right) {
        const mid = Math.floor((left + right) / 2);

        if (nums[mid] === target) return mid;
        if (nums[mid] < target) left = mid + 1;
        else right = mid - 1;
    }

    return -1;
}
```

**Time:** O(log n) | **Space:** O(1)

---

## Prefix Sums

Precompute cumulative sums to answer range queries in O(1).

```typescript
function buildPrefix(nums: number[]): number[] {
    const prefix = [0];
    for (const num of nums) {
        prefix.push(prefix[prefix.length - 1] + num);
    }
    return prefix;
}

// Sum from index i to j (inclusive)
function rangeSum(prefix: number[], i: number, j: number): number {
    return prefix[j + 1] - prefix[i];
}
```

**Build:** O(n) | **Query:** O(1)

---

## Dynamic Programming (Intro)

Break problem into subproblems. Store results to avoid recomputing.

### Fibonacci — top-down (memoization)

```typescript
function fib(n: number, memo = new Map<number, number>()): number {
    if (n <= 1) return n;
    if (memo.has(n)) return memo.get(n)!;

    const result = fib(n - 1, memo) + fib(n - 2, memo);
    memo.set(n, result);
    return result;
}
```

### Fibonacci — bottom-up (tabulation)

```typescript
function fib(n: number): number {
    if (n <= 1) return n;
    const dp = [0, 1];

    for (let i = 2; i <= n; i++) {
        dp[i] = dp[i - 1] + dp[i - 2];
    }

    return dp[n];
}
```

---

## Pattern Recognition Cheatsheet

| Problem type | Pattern |
|---|---|
| Pair with target sum, sorted array | Two pointers |
| Pair with target sum, unsorted array | Hash map |
| Max/min subarray of fixed size | Sliding window (fixed) |
| Longest subarray satisfying condition | Sliding window (variable) |
| Search in sorted array | Binary search |
| Range sum queries | Prefix sums |
| Overlapping subproblems | Dynamic programming |

---

## Gaps to Fill Next

- [ ] BFS / DFS (graph traversal)
- [ ] Dijkstra shortest path
- [ ] Topological sort
- [ ] Union-Find
- [ ] Greedy algorithms
- [ ] Backtracking
- [ ] DP patterns (0/1 knapsack, longest common subsequence)
