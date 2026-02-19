# Chapter 19 — Divide and Conquer

## The Problem This Solves
Some problems look like they require O(n^2) brute force, but if the problem can be split into two independent halves, solved recursively, and merged — the complexity drops to O(n log n) or better. Divide and Conquer is the architectural pattern behind Merge Sort, Quick Sort, Binary Search, and Strassen's Matrix Multiplication.

---

## 1. The Pattern

Every D&C algorithm follows three strict phases:

```
DIVIDE AND CONQUER TEMPLATE:
════════════════════════════
  1. DIVIDE:   Split the problem into smaller subproblems (usually halves)
  2. CONQUER:  Solve each subproblem recursively (base case stops recursion)
  3. COMBINE:  Merge the subproblem solutions into the final answer

function solve(problem):
    if problem is small enough:
        return baseCaseSolution
    
    left = solve(leftHalf)
    right = solve(rightHalf)
    return combine(left, right)
```

**Key Insight:** The "combine" step is where the algorithm's intelligence lives. Merge Sort's combine is the merge operation. Quick Sort's combine is trivial (already partitioned). The cost of combining determines the final complexity.

---

## 2. Master Theorem — Analyzing D&C Complexity

For any recurrence of the form `T(n) = a * T(n/b) + O(n^c)`:

| Variable | Meaning |
|----------|---------|
| `a` | Number of subproblems created |
| `b` | Factor by which input shrinks |
| `c` | Cost of the divide + combine step |

| Case | Condition | Result | Intuition |
|------|-----------|--------|-----------|
| 1 | c < log_b(a) | O(n^(log_b(a))) | Recursion dominates (many subproblems) |
| 2 | c = log_b(a) | O(n^c * log n) | Balanced — equal work at every level |
| 3 | c > log_b(a) | O(n^c) | Combine step dominates |

### Manual Application

**Merge Sort:** `T(n) = 2T(n/2) + O(n)`
- a=2, b=2, c=1. log_2(2) = 1. c = log_b(a) → **Case 2: O(n log n)** ✓

**Binary Search:** `T(n) = T(n/2) + O(1)`
- a=1, b=2, c=0. log_2(1) = 0. c = log_b(a) → **Case 2: O(log n)** ✓

**Strassen's:** `T(n) = 7T(n/2) + O(n^2)`
- a=7, b=2, c=2. log_2(7) ≈ 2.81. c < log_b(a) → **Case 1: O(n^2.81)** ✓
  (Better than naive O(n^3) matrix multiplication!)

---

## 3. Quick Select — Find Kth Smallest in O(n) Average

The problem: Finding the 5th smallest element in an unsorted array normally requires sorting (O(n log n)). Quick Select does it in O(n) average.

```javascript
function quickSelect(arr, k) {
    // k is 0-indexed (k=0 → smallest element)
    function partition(lo, hi) {
        const pivot = arr[hi];
        let i = lo;
        for (let j = lo; j < hi; j++) {
            if (arr[j] <= pivot) {
                [arr[i], arr[j]] = [arr[j], arr[i]];
                i++;
            }
        }
        [arr[i], arr[hi]] = [arr[hi], arr[i]];
        return i; // Pivot's final position
    }
    
    function select(lo, hi) {
        const pivotIndex = partition(lo, hi);
        if (pivotIndex === k) return arr[pivotIndex];
        if (pivotIndex < k) return select(pivotIndex + 1, hi);
        return select(lo, pivotIndex - 1);
    }
    
    return select(0, arr.length - 1);
}
```

**Manual Walkthrough — Find 3rd smallest (k=2) in [7, 2, 1, 6, 8, 5]:**
```
Partition with pivot 5: [2, 1, 5, 6, 8, 7] → pivot at index 2
k=2, pivotIndex=2 → MATCH! Return arr[2] = 5

The 3rd smallest element is 5. Found in ONE partition (O(n)). ✓
```

**Complexity:** O(n) average, O(n^2) worst (bad pivot). Use median-of-3 to avoid worst case.

---

## 4. Count Inversions — Modified Merge Sort

An inversion is a pair (i, j) where i < j but arr[i] > arr[j]. Counting inversions measures "how unsorted" an array is. Brute force is O(n^2). Modified Merge Sort does it in O(n log n).

```javascript
function countInversions(arr) {
    let count = 0;
    
    function mergeSort(a) {
        if (a.length <= 1) return a;
        const mid = Math.floor(a.length / 2);
        const left = mergeSort(a.slice(0, mid));
        const right = mergeSort(a.slice(mid));
        return merge(left, right);
    }
    
    function merge(left, right) {
        const result = [];
        let i = 0, j = 0;
        while (i < left.length && j < right.length) {
            if (left[i] <= right[j]) {
                result.push(left[i++]);
            } else {
                result.push(right[j++]);
                count += left.length - i; // ALL remaining left elements are inversions
            }
        }
        return result.concat(left.slice(i)).concat(right.slice(j));
    }
    
    mergeSort(arr);
    return count;
}
```

**Manual Walkthrough — [2, 4, 1, 3, 5]:**
```
Split: [2,4] and [1,3,5]
  Merge [2] [4] → [2,4], inversions = 0
  Merge [1] [3,5]:
    Merge [3] [5] → [3,5], inversions = 0
    Merge [1] [3,5] → [1,3,5], inversions = 0
  Merge [2,4] [1,3,5]:
    1 < 2: take 1, inversions += 2 (both 2 and 4 > 1)
    2 < 3: take 2
    3 < 4: take 3, inversions += 1 (4 > 3)
    take 4, take 5
Total inversions: 3 ✓ (pairs: (2,1), (4,1), (4,3))
```

---

## 5. Closest Pair of Points — O(n log n)

Given N points in 2D, find the pair with the smallest Euclidean distance. Brute force is O(n^2).

```
D&C STRATEGY:
1. Sort points by X coordinate
2. DIVIDE: Split into left half and right half at the midpoint
3. CONQUER: Recursively find closest pair in each half (d_left, d_right)
4. Let d = min(d_left, d_right)
5. COMBINE: Check the "strip" of points within distance d of the midpoint
   - Only need to compare each point with at most 7 neighbors in the strip
   - This step is O(n), not O(n^2)!

Total: T(n) = 2T(n/2) + O(n) = O(n log n)
```

The insight: Points in the strip that could beat distance `d` must be within a d×2d rectangle, which can contain at most 8 points (due to the minimum distance constraint from each half).

---

## 6. D&C vs. Dynamic Programming

| Criteria | Divide & Conquer | Dynamic Programming |
|----------|-----------------|---------------------|
| Subproblem overlap | No overlap (independent) | Overlapping subproblems |
| Memoization needed | No | Yes (cache results) |
| Direction | Top-down | Top-down or bottom-up |
| Examples | Merge Sort, Quick Select | Fibonacci, Knapsack |

**Rule:** If subproblems are independent → D&C. If the same subproblem appears multiple times → DP.

---

## 7. Practice Problems

| Problem | Concept | Difficulty |
|---------|---------|------------|
| Merge Sort | Classic D&C | Medium |
| Quick Sort / Quick Select | Partition-based D&C | Medium |
| Count Inversions | Modified Merge Sort | Hard |
| Closest Pair of Points | 2D strip optimization | Hard |
| Maximum Subarray (D&C) | Alternative to Kadane's | Medium |
| Karatsuba Multiplication | Multiply large integers in O(n^1.59) | Hard |
