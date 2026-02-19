# Chapter 02 — Time & Space Complexity

## Simple Explanation (ELI5)

Imagine two friends racing to count all the stars. Friend A counts one by one. Friend B uses a telescope that halves the sky each time. Even though both get the answer, Friend B finishes **way faster** when there are millions of stars.

**Time Complexity** = How the number of steps grows as input grows.
**Space Complexity** = How much extra memory is needed as input grows.

---

## Technical Definition

**Time Complexity** is a mathematical function T(n) that describes the number of primitive operations an algorithm performs relative to input size n, expressed using asymptotic notation.

**Space Complexity** is the total memory consumed by an algorithm, including input storage plus any auxiliary space.

---

## Why It's Needed

Without complexity analysis, you can't:
- Compare two solutions objectively
- Predict if a solution will pass within time limits
- Optimize code that's too slow
- Choose the right data structure

---

## 1. Asymptotic Notations

### Big O — O(n) — Upper Bound (Worst Case)

```
"This algorithm will NEVER be slower than this."
f(n) = O(g(n)) means f(n) ≤ c × g(n) for large n
```

```
     Performance
       ↑
       |         O(n²)
       |       /
       |     /   ← Your algorithm is somewhere below this line
       |   /
       | /
       └──────────→ Input Size (n)
```

### Big Omega — Ω(n) — Lower Bound (Best Case)

```
"This algorithm will NEVER be faster than this."
f(n) = Ω(g(n)) means f(n) ≥ c × g(n) for large n
```

### Big Theta — Θ(n) — Tight Bound (Average Case)

```
"This algorithm grows EXACTLY at this rate."
f(n) = Θ(g(n)) means c₁ × g(n) ≤ f(n) ≤ c₂ × g(n)
```

### Example: Linear Search

```javascript
function linearSearch(arr, target) {
  for (let i = 0; i < arr.length; i++) {
    if (arr[i] === target) return i;
  }
  return -1;
}
```

| Case | Scenario | Complexity |
|------|----------|-----------|
| Best | Target is first element | Ω(1) |
| Worst | Target is last or missing | O(n) |
| Average | Target is somewhere in middle | Θ(n/2) = Θ(n) |

> **Convention:** When someone says "complexity is O(n)", they mean **worst case** unless stated otherwise.

---

## 2. All Complexity Classes

### O(1) — Constant

Operations don't change with input size.

```javascript
function getFirst(arr) {
  return arr[0];  // Always 1 operation
}
// Array access, hash map lookup, stack push/pop
```

```python
def get_first(arr):
    return arr[0]
```

### O(log n) — Logarithmic

Input is halved each step.

```javascript
function binarySearch(arr, target) {
  let lo = 0, hi = arr.length - 1;
  while (lo <= hi) {
    let mid = lo + Math.floor((hi - lo) / 2);
    if (arr[mid] === target) return mid;
    else if (arr[mid] < target) lo = mid + 1;
    else hi = mid - 1;
  }
  return -1;
}
// 1 billion items → only ~30 steps!
```

### O(n) — Linear

Visit every element once.

```javascript
function findMax(arr) {
  let max = arr[0];
  for (let i = 1; i < arr.length; i++) {
    if (arr[i] > max) max = arr[i];
  }
  return max;
}
```

### O(n log n) — Linearithmic

Divide-and-conquer algorithms. Split (log n) × process (n).

```javascript
// Merge Sort, Quick Sort (average), Heap Sort
arr.sort((a, b) => a - b); // Built-in uses O(n log n)
```

### O(n²) — Quadratic

Nested loops over input.

```javascript
function allPairs(arr) {
  for (let i = 0; i < arr.length; i++) {        // n
    for (let j = i + 1; j < arr.length; j++) {  // n
      console.log(arr[i], arr[j]);
    }
  }
}
```

### O(2ⁿ) — Exponential

Every element has two choices (take/skip).

```javascript
// Recursive Fibonacci (without memo), subset generation
function fib(n) {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2);
}
// fib(40) → over 1 billion operations!
```

### O(n!) — Factorial

All permutations.

```javascript
// Generating all permutations: n! arrangements
// n=10 → 3,628,800
// n=13 → 6 billion
// n=20 → 2.4 × 10¹⁸ (impossible)
```

### Complete Comparison Table

| Big O | Name | n=10 | n=100 | n=1000 | n=10⁶ | Verdict |
|-------|------|------|-------|--------|--------|---------|
| O(1) | Constant | 1 | 1 | 1 | 1 | ✅ Perfect |
| O(log n) | Logarithmic | 3 | 7 | 10 | 20 | ✅ Excellent |
| O(n) | Linear | 10 | 100 | 1000 | 10⁶ | ✅ Good |
| O(n log n) | Linearithmic | 33 | 664 | 10K | 2×10⁷ | ✅ Acceptable |
| O(n²) | Quadratic | 100 | 10K | 10⁶ | 10¹² ❌ | ⚠️ Small n only |
| O(2ⁿ) | Exponential | 1024 | 10³⁰ | 🔥 | 🔥 | ❌ n ≤ 20 |
| O(n!) | Factorial | 3.6M | 🔥 | 🔥 | 🔥 | ❌ n ≤ 12 |

```
    Time
     ↑
     |                    n!
     |                  /
     |               2ⁿ
     |             /
     |          n²
     |        /
     |     n log n
     |    /
     |  n
     | / log n
     |________1____________→ Input Size
```

---

## 3. How to Calculate Complexity — Rules

### Rule 1: Drop Constants
```
O(2n) → O(n)
O(100n) → O(n)
O(n/2) → O(n)
O(500) → O(1)
```

### Rule 2: Drop Smaller Terms
```
O(n² + n) → O(n²)
O(n + log n) → O(n)
O(n³ + n² + n) → O(n³)
```

### Rule 3: Different Inputs = Different Variables
```javascript
function twoArrays(arr1, arr2) {
  for (let a of arr1) { /* ... */ }  // O(a)
  for (let b of arr2) { /* ... */ }  // O(b)
}
// Total: O(a + b), NOT O(2n)
```

### Rule 4: Sequential = Add
```javascript
function sequential(arr) {
  for (let i = 0; i < arr.length; i++) { }  // O(n)
  for (let i = 0; i < arr.length; i++) { }  // O(n)
}
// Total: O(n + n) = O(n)
```

### Rule 5: Nested = Multiply
```javascript
function nested(arr) {
  for (let i = 0; i < arr.length; i++) {      // O(n)
    for (let j = 0; j < arr.length; j++) {    //   × O(n)
      console.log(arr[i], arr[j]);
    }
  }
}
// Total: O(n × n) = O(n²)
```

### Dry Run: Calculating Complexity

```javascript
function mystery(n) {
  let a = 0;                          // O(1)
  for (let i = 0; i < n; i++) {       // O(n) ─┐
    a += i;                           //        │  O(n)
  }                                   // ───────┘
  for (let i = 0; i < n; i++) {       // O(n) ─┐
    for (let j = 0; j < n; j++) {     //   O(n) │  O(n²)
      a += i + j;                     //        │
    }                                 //        │
  }                                   // ───────┘
  return a;                           // O(1)
}
// Total: O(1) + O(n) + O(n²) + O(1) = O(n²)
```

---

## 4. Space Complexity

### O(1) — Constant Space
```javascript
function sum(arr) {
  let total = 0;  // Only 1 extra variable
  for (let num of arr) total += num;
  return total;
}
```

### O(n) — Linear Space
```javascript
function duplicate(arr) {
  let copy = [...arr];  // New array of size n
  return copy;
}
```

### O(n) — Recursion Call Stack
```javascript
function factorial(n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1);
}
// Call stack grows to depth n → O(n) space

// Visualization for factorial(5):
// Stack: [factorial(5)]
// Stack: [factorial(5), factorial(4)]
// Stack: [factorial(5), factorial(4), factorial(3)]
// Stack: [factorial(5), factorial(4), factorial(3), factorial(2)]
// Stack: [factorial(5), factorial(4), factorial(3), factorial(2), factorial(1)]
// Max depth = 5 = n → O(n)
```

### O(log n) — Balanced Recursion
```javascript
function binarySearchRecursive(arr, target, lo, hi) {
  if (lo > hi) return -1;
  let mid = Math.floor((lo + hi) / 2);
  if (arr[mid] === target) return mid;
  if (arr[mid] < target) return binarySearchRecursive(arr, target, mid + 1, hi);
  return binarySearchRecursive(arr, target, lo, mid - 1);
}
// Max recursion depth = log n → O(log n) space
```

---

## 5. Amortized Complexity

### What Is It?
Some operations are **usually fast** but **occasionally slow**. Amortized analysis averages the cost over many operations.

### Example: Dynamic Array (push)
```
Array capacity: 4
Push 1: [1, _, _, _]         → O(1)
Push 2: [1, 2, _, _]         → O(1)
Push 3: [1, 2, 3, _]         → O(1)
Push 4: [1, 2, 3, 4]         → O(1)
Push 5: Capacity full!
  → Create new array of size 8
  → Copy all 4 elements        → O(n) just this once!
  → [1, 2, 3, 4, 5, _, _, _]

Most pushes: O(1). Occasional resize: O(n).
Over n pushes, total work ≈ 2n → Amortized O(1) per push.
```

---

## 6. Common Mistakes

| Mistake | Correct Understanding |
|---------|----------------------|
| "O(2n) is worse than O(n)" | They're the same: O(2n) = O(n) |
| "O(n+m) = O(n)" | Wrong if m could be large! Keep O(n+m) |
| "Best case matters" | Almost never — focus on worst case |
| "Recursive = always O(n)" | Depends on branching: tree recursion can be O(2ⁿ) |
| "Space = only new arrays" | Call stack counts too! |
| "O(log n) base doesn't matter" | True! log₂(n) and log₁₀(n) differ by constant |

---

## 7. Space-Time Tradeoff

Often you **trade space for time** or vice versa:

| Approach | Time | Space | Example |
|----------|------|-------|---------|
| Brute force Two Sum | O(n²) | O(1) | Nested loop |
| HashMap Two Sum | O(n) | O(n) | Store seen values |
| Memoized Fibonacci | O(n) | O(n) | Cache results |
| Tabulated Fibonacci | O(n) | O(1) | Only store prev two |

---

## 8. Interview Questions

1. What's the difference between O(n) and Θ(n)?
2. What's the time complexity of accessing an element in a linked list vs array?
3. If a function has O(n²) time but O(1) space, and another has O(n log n) time but O(n) space, which would you prefer for n = 10⁶?
4. What's the amortized time complexity of `push` in a dynamic array?
5. Can an algorithm have O(1) time but O(n) space? (Yes — lookup in a precomputed table)

---

## 9. Practice Problems

| Problem | What to Analyze | Difficulty |
|---------|----------------|-----------|
| What is O(n + n²)? | Drop smaller terms → O(n²) | Easy |
| Nested loop: outer 1..n, inner 1..m | O(n × m) | Easy |
| Binary search recursive space | O(log n) call stack | Easy |
| HashMap lookup vs sorted array search | O(1) avg vs O(log n) | Medium |
| Merge sort: time AND space | O(n log n) time, O(n) space | Medium |
| Recursive fib without memo | O(2ⁿ) — draw recursion tree | Medium |
