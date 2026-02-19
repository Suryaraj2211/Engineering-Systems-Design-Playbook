# Chapter 03 — Recursion

## Simple Explanation (ELI5)

Imagine you're in a line, and you want to know your position. Instead of counting from the front, you ask the person ahead: "What's your position?" They ask the person ahead of them, and so on, until the first person says "I'm #1." Then answers flow back: "#2", "#3", "#4"... That's recursion — **a function asking a smaller version of itself**.

---

## Technical Definition

Recursion is when a function **calls itself** with a smaller input until it reaches a **base case** (stopping condition). Each call creates a new frame on the **call stack**.

---

## Why It's Needed

- Trees and graphs are **naturally recursive** (a subtree is itself a tree)
- Divide & Conquer algorithms (Merge Sort, Quick Sort) require recursion
- Backtracking problems (N-Queens, Sudoku) are recursive
- Dynamic Programming starts with recursive solutions
- Many problems have **elegant recursive solutions** that are hard to write iteratively

---

## Real-world Analogy

**Russian nesting dolls (Matryoshka):** Open the biggest doll → there's a smaller doll inside → open that → smaller doll → ... → smallest doll (base case). Then you put them back together (unwinding).

---

## 1. Anatomy of Recursion

Every recursive function has **exactly two parts**:

```javascript
function recursive(input) {
  // 1. BASE CASE — When to stop
  if (input is small enough) {
    return simple answer;
  }

  // 2. RECURSIVE CASE — Break down + call self
  return combine(recursive(smaller input));
}
```

### Example: Factorial

```javascript
function factorial(n) {
  if (n <= 1) return 1;           // Base case
  return n * factorial(n - 1);   // Recursive case
}
```

```python
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)
```

---

## 2. The Call Stack — Internal Working

When a function is called, it's **pushed** onto the call stack. When it returns, it's **popped** off.

### Dry Run: factorial(5)

```
CALL PHASE (pushing frames):
┌─────────────────────┐
│ factorial(1) → 1    │  ← Base case reached!
├─────────────────────┤
│ factorial(2)        │  waiting for factorial(1)
├─────────────────────┤
│ factorial(3)        │  waiting for factorial(2)
├─────────────────────┤
│ factorial(4)        │  waiting for factorial(3)
├─────────────────────┤
│ factorial(5)        │  waiting for factorial(4)
└─────────────────────┘

RETURN PHASE (popping frames):
factorial(1) returns 1
factorial(2) returns 2 × 1 = 2
factorial(3) returns 3 × 2 = 6
factorial(4) returns 4 × 6 = 24
factorial(5) returns 5 × 24 = 120   ← FINAL ANSWER
```

**Time: O(n)** — n function calls
**Space: O(n)** — n frames on the call stack

---

## 3. Types of Recursion

### Linear Recursion (One call)
```javascript
function sum(n) {
  if (n === 0) return 0;
  return n + sum(n - 1);  // Single recursive call
}
// T(n) = T(n-1) + O(1) → O(n)
```

### Binary/Tree Recursion (Multiple calls)
```javascript
function fib(n) {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2);  // TWO recursive calls
}
// T(n) = T(n-1) + T(n-2) + O(1) → O(2ⁿ)  ← Exponential!
```

Recursion tree for fib(5):
```
                    fib(5)
                   /      \
              fib(4)       fib(3)
             /    \        /    \
          fib(3)  fib(2) fib(2) fib(1)
          / \      / \    / \
      fib(2) fib(1) fib(1) fib(0) fib(1) fib(0)
       / \
   fib(1) fib(0)

Notice: fib(3) is calculated TWICE, fib(2) THREE times!
This is why memoization is crucial (Chapter 22 - DP).
```

### Tail Recursion
The recursive call is the **last thing** the function does. Some compilers optimize this to use constant stack space.

```javascript
// NOT tail recursive — multiplication happens AFTER recursive call
function factorial(n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1);  // Must wait for result, then multiply
}

// Tail recursive — accumulator carries the result
function factorialTail(n, acc = 1) {
  if (n <= 1) return acc;
  return factorialTail(n - 1, n * acc);  // Nothing to do after call
}
```

```python
# Tail recursive (Python doesn't optimize this, but concept matters)
def factorial_tail(n, acc=1):
    if n <= 1:
        return acc
    return factorial_tail(n - 1, n * acc)
```

### Mutual/Indirect Recursion
Function A calls B, B calls A.

```javascript
function isEven(n) {
  if (n === 0) return true;
  return isOdd(n - 1);
}
function isOdd(n) {
  if (n === 0) return false;
  return isEven(n - 1);
}
```

---

## 4. Recursion vs Iteration

| Aspect | Recursion | Iteration |
|--------|-----------|-----------|
| Uses | Call stack | Loop variable |
| Memory | O(n) stack | O(1) usually |
| Risk | Stack overflow | None |
| Readability | Often cleaner | Can be verbose |
| Best for | Trees, graphs, D&C | Simple linear tasks |
| Speed | Slightly slower (function call overhead) | Slightly faster |

### Converting Recursion to Iteration

```javascript
// Recursive
function sumRecursive(n) {
  if (n === 0) return 0;
  return n + sumRecursive(n - 1);
}

// Iterative equivalent
function sumIterative(n) {
  let total = 0;
  for (let i = 1; i <= n; i++) total += i;
  return total;
}
```

> **Rule:** Any recursion can be converted to iteration using an explicit stack. But for trees/graphs, recursion is usually cleaner.

---

## 5. Classic Recursive Problems

### Power Function

```javascript
// O(n) — Linear
function power(base, exp) {
  if (exp === 0) return 1;
  return base * power(base, exp - 1);
}

// O(log n) — Fast Power (divide exponent)
function fastPower(base, exp) {
  if (exp === 0) return 1;
  let half = fastPower(base, Math.floor(exp / 2));
  if (exp % 2 === 0) return half * half;
  return half * half * base;
}
```

```python
def fast_power(base, exp):
    if exp == 0:
        return 1
    half = fast_power(base, exp // 2)
    if exp % 2 == 0:
        return half * half
    return half * half * base
```

### Reverse a String

```javascript
function reverseStr(s) {
  if (s.length <= 1) return s;
  return reverseStr(s.slice(1)) + s[0];
}
// "hello" → reverse("ello") + "h"
//         → reverse("llo") + "e" + "h"
//         → ... → "olleh"
```

### Check Palindrome

```javascript
function isPalindrome(s, left = 0, right = s.length - 1) {
  if (left >= right) return true;
  if (s[left] !== s[right]) return false;
  return isPalindrome(s, left + 1, right - 1);
}
```

### Sum of Digits

```javascript
function digitSum(n) {
  if (n < 10) return n;             // Single digit
  return (n % 10) + digitSum(Math.floor(n / 10));
}
console.log(digitSum(1234)); // 10 (1+2+3+4)
```

### Tower of Hanoi

```javascript
function hanoi(n, from, to, aux) {
  if (n === 0) return;
  hanoi(n - 1, from, aux, to);  // Move n-1 disks to auxiliary
  console.log(`Move disk ${n}: ${from} → ${to}`);
  hanoi(n - 1, aux, to, from);  // Move n-1 disks from aux to target
}

hanoi(3, 'A', 'C', 'B');
// Move disk 1: A → C
// Move disk 2: A → B
// Move disk 1: C → B
// Move disk 3: A → C
// Move disk 1: B → A
// Move disk 2: B → C
// Move disk 1: A → C
// Total moves = 2ⁿ - 1 = 7
```

---

## 6. Recursion Complexity Analysis

### How to Find Time Complexity

| Pattern | Recurrence | Complexity |
|---------|-----------|-----------|
| Linear (1 call, reduce by 1) | T(n) = T(n-1) + O(1) | O(n) |
| Linear (1 call, work at each level) | T(n) = T(n-1) + O(n) | O(n²) |
| Divide by 2 (1 call) | T(n) = T(n/2) + O(1) | O(log n) |
| Divide by 2 (2 calls, no extra work) | T(n) = 2T(n/2) + O(1) | O(n) |
| Divide by 2 (2 calls, O(n) work) | T(n) = 2T(n/2) + O(n) | O(n log n) |
| Two branches (subtract by 1) | T(n) = 2T(n-1) + O(1) | O(2ⁿ) |

### Space = Max Depth of Recursion Tree

```
Linear recursion → depth = n → O(n)
Binary Search recursion → depth = log n → O(log n)
Merge Sort → depth = log n → O(log n) (but O(n) for merge arrays)
```

---

## 7. Common Mistakes

| Mistake | Fix |
|---------|-----|
| No base case | Always define when to stop |
| Base case doesn't return | Must `return` a value |
| Not reducing toward base case | Each call must make input SMALLER |
| Modifying shared mutable state | Pass copies or use indices |
| Not cloning arrays in backtracking | Use `[...arr]` or `path.slice()` |
| Ignoring stack space | Recursion depth = space complexity |

### Stack Overflow Prevention
```javascript
// BAD — infinite recursion
function bad(n) {
  return bad(n);  // Never reaches base case!
}

// BAD — doesn't shrink input
function bad2(n) {
  if (n === 0) return 0;
  return bad2(n);  // n never decreases!
}

// GOOD — always approaches base case
function good(n) {
  if (n <= 0) return 0;
  return n + good(n - 1);  // n shrinks by 1
}
```

---

## 8. When NOT to Use Recursion

- **Simple loops** — `for` loop is faster and uses less memory
- **When depth > ~10,000** — Stack overflow risk (convert to iterative)
- **When the same subproblem is solved many times** without memoization
- **Performance-critical code** — function call overhead adds up

---

## 9. Recursion → Memoization Preview

Tree recursion often recomputes the same values. Adding a **cache** makes it efficient:

```javascript
// Without memo: O(2ⁿ) — terrible
function fib(n) {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2);
}

// With memo: O(n) — excellent
function fibMemo(n, memo = {}) {
  if (n in memo) return memo[n];
  if (n <= 1) return n;
  memo[n] = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
  return memo[n];
}
```

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)
```

> This is the bridge to **Dynamic Programming** (Chapter 22).

---

## 10. Practice Problems

| Problem | Type | Difficulty |
|---------|------|-----------|
| Factorial | Linear recursion | Easy |
| Sum of digits | Linear recursion | Easy |
| Fibonacci | Tree recursion | Easy |
| Power of x^n | Divide & conquer | Easy |
| Reverse string | Linear recursion | Easy |
| Check palindrome | Two-pointer recursion | Easy |
| Tower of Hanoi | Classic recursion | Medium |
| Print all subsequences | Tree recursion | Medium |
| Flatten nested array | Recursion | Medium |
| Count paths in grid | Grid recursion | Medium |
| Generate parentheses | Backtracking preview | Medium |
| Recursion to iteration conversion | Understanding | Medium |
