# Chapter 01 — Mathematical Foundations for DSA

## Why Math for DSA?

> **ELI5:** Before building a house, you need to understand measurements. Before solving DSA problems, you need basic math tools.

**Technical:** Many algorithms rely on mathematical concepts — logarithms (binary search), modular arithmetic (hashing), combinatorics (permutations), and powers of 2 (bit manipulation). Without these, you'll memorize algorithms instead of understanding them.

---

## 1. Logarithms

### Simple Explanation
A logarithm answers: **"How many times do I divide by X to reach 1?"**

### Real-world Analogy
You have a 1000-page phone book. If you tear it in half repeatedly: 1000 → 500 → 250 → 125 → 63 → 32 → 16 → 8 → 4 → 2 → 1. That's ~10 steps. So log₂(1000) ≈ 10.

### Key Properties

```
log₂(1) = 0          → 2⁰ = 1
log₂(2) = 1          → 2¹ = 2
log₂(4) = 2          → 2² = 4
log₂(8) = 3          → 2³ = 8
log₂(1024) = 10      → 2¹⁰ = 1024
log₂(1,000,000) ≈ 20 → Why binary search is so fast!
```

```javascript
// Important: In DSA, "log" almost always means log base 2
console.log(Math.log2(1024)); // 10
console.log(Math.log2(1000000)); // ~19.93
```

```python
import math
print(math.log2(1024))     # 10.0
print(math.log2(1000000))  # ~19.93
```

### Log Rules You Must Know

```
log(a × b) = log(a) + log(b)
log(a / b) = log(a) - log(b)
log(aⁿ) = n × log(a)
log₂(n) = log₁₀(n) / log₁₀(2)    ← Change of base
```

### Where Logs Appear in DSA
| Algorithm/DS | Why Log? |
|-------------|----------|
| Binary Search | Halves the search space each step |
| Balanced BST | Height = log n |
| Heap operations | Bubble up/down = log n levels |
| Merge Sort | Splits into log n levels |
| Quick Sort (avg) | Partitions log n times |

---

## 2. Powers of 2

### Why It Matters
Computers work in binary. Powers of 2 appear everywhere in DSA.

```
2⁰  = 1
2¹  = 2
2²  = 4
2³  = 8
2⁴  = 16
2⁵  = 32
2⁶  = 64
2⁷  = 128
2⁸  = 256
2⁹  = 512
2¹⁰ = 1024      ← ~1 Thousand (1 KB)
2²⁰ = 1,048,576  ← ~1 Million (1 MB)
2³⁰ = ~1 Billion  ← ~1 GB
2³² = ~4.3 Billion ← Max 32-bit integer
```

### Important for Interview Estimation

```
"Given n ≤ 10⁶, will O(n²) work?"
10⁶ × 10⁶ = 10¹² → Too slow (computers do ~10⁸ operations/sec)

"Given n ≤ 10⁶, will O(n log n) work?"
10⁶ × 20 = 2 × 10⁷ → Fast enough ✅
```

| n | O(n) | O(n log n) | O(n²) | O(2ⁿ) |
|---|------|-----------|-------|--------|
| 10 | 10 | 33 | 100 | 1024 |
| 100 | 100 | 664 | 10⁴ | 10³⁰ |
| 10³ | 10³ | 10⁴ | 10⁶ | 🔥 |
| 10⁶ | 10⁶ | 2×10⁷ | 10¹² ❌ | 🔥 |
| 10⁸ | 10⁸ | 2.6×10⁹ ❌ | 🔥 | 🔥 |

> **Rule of thumb:** If operations > 10⁸, it's too slow for a 1-second time limit.

---

## 3. Modular Arithmetic

### Simple Explanation
Modulo (%) gives the **remainder** after division.

### Why It Matters
- Many problems say "return answer modulo 10⁹ + 7" to prevent integer overflow
- Hashing uses modulo to fit keys into array indices

```javascript
console.log(10 % 3);  // 1 (10 ÷ 3 = 3 remainder 1)
console.log(7 % 2);   // 1 (odd number check!)
console.log(8 % 2);   // 0 (even number check!)

const MOD = 1e9 + 7;  // 1000000007 (common in competitive programming)

// Properties:
// (a + b) % m = ((a % m) + (b % m)) % m
// (a × b) % m = ((a % m) × (b % m)) % m
// (a - b) % m = ((a % m) - (b % m) + m) % m  ← Add m to avoid negatives
```

```python
MOD = 10**9 + 7

# Modular exponentiation — compute (base^exp) % mod efficiently
def mod_pow(base, exp, mod):
    result = 1
    base %= mod
    while exp > 0:
        if exp % 2 == 1:
            result = (result * base) % mod
        exp //= 2
        base = (base * base) % mod
    return result

print(mod_pow(2, 100, MOD))  # Fast even for huge exponents
```

---

## 4. Floor, Ceil, and Integer Division

```javascript
// Floor: Round DOWN to nearest integer
Math.floor(7 / 2);   // 3 (not 3.5)
Math.floor(-2.3);     // -3

// Ceil: Round UP to nearest integer
Math.ceil(7 / 2);     // 4
Math.ceil(2.1);        // 3

// Integer division (floor for positives)
Math.trunc(7 / 2);    // 3

// Common DSA usage: finding middle index
let mid = Math.floor((left + right) / 2);
// Better (avoids overflow in other languages):
let mid2 = left + Math.floor((right - left) / 2);
```

```python
# Floor division
7 // 2   # 3
-7 // 2  # -4 (Python floors toward negative infinity!)

import math
math.ceil(7 / 2)   # 4
math.floor(7 / 2)  # 3
```

---

## 5. Combinatorics Basics

### Factorial

```
n! = n × (n-1) × (n-2) × ... × 1
5! = 5 × 4 × 3 × 2 × 1 = 120
0! = 1 (by definition)
```

### Permutations (Order matters)
Ways to arrange r items from n items: **P(n,r) = n! / (n-r)!**

```
"How many ways to arrange 3 letters from A,B,C,D?"
P(4,3) = 4! / 1! = 24
```

### Combinations (Order doesn't matter)
Ways to choose r items from n items: **C(n,r) = n! / (r! × (n-r)!)**

```
"How many ways to choose 2 items from {A,B,C,D}?"
C(4,2) = 4! / (2! × 2!) = 6 → {AB, AC, AD, BC, BD, CD}
```

```javascript
function factorial(n) {
  if (n <= 1) return 1;
  let result = 1;
  for (let i = 2; i <= n; i++) result *= i;
  return result;
}

function combination(n, r) {
  return factorial(n) / (factorial(r) * factorial(n - r));
}

console.log(combination(4, 2)); // 6
```

### Where Combinatorics Appear in DSA
- **Subsets:** A set of n elements has 2ⁿ subsets
- **Permutations:** n elements have n! arrangements
- **Paths in grid:** m×n grid has C(m+n-2, m-1) unique paths
- **Catalan numbers:** Valid parentheses, BST arrangements

---

## 6. GCD and LCM

```javascript
// GCD (Greatest Common Divisor) — Euclidean Algorithm
function gcd(a, b) {
  while (b !== 0) {
    [a, b] = [b, a % b];
  }
  return a;
}

// LCM = (a × b) / GCD(a, b)
function lcm(a, b) {
  return (a * b) / gcd(a, b);
}

console.log(gcd(12, 8));  // 4
console.log(lcm(4, 6));   // 12
```

```python
import math
print(math.gcd(12, 8))  # 4
print(math.lcm(4, 6))   # 12
```

---

## 7. Arithmetic & Geometric Progressions

### Arithmetic Progression (AP)
Each term differs by a constant: `1, 3, 5, 7, 9` (d=2)

```
Sum of first n natural numbers = n(n+1)/2
Sum of AP = n/2 × (first + last)
```

```javascript
// Sum 1+2+3+...+n
function sumN(n) { return n * (n + 1) / 2; }
console.log(sumN(100)); // 5050

// This is why double loop = O(n²):
// n + (n-1) + (n-2) + ... + 1 = n(n+1)/2 ≈ n²/2 → O(n²)
```

### Geometric Progression (GP)
Each term is multiplied by a constant: `1, 2, 4, 8, 16` (r=2)

```
Sum of GP = a(rⁿ - 1)/(r - 1)
Sum of 1+2+4+...+2^(n-1) = 2ⁿ - 1
```

This is why: total nodes in a perfect binary tree of height h = 2^(h+1) - 1.

---

## 8. Prime Numbers (Sieve of Eratosthenes)

```javascript
function sieveOfEratosthenes(n) {
  let isPrime = new Array(n + 1).fill(true);
  isPrime[0] = isPrime[1] = false;

  for (let i = 2; i * i <= n; i++) {
    if (isPrime[i]) {
      for (let j = i * i; j <= n; j += i) {
        isPrime[j] = false;
      }
    }
  }

  return isPrime.reduce((primes, val, idx) => {
    if (val) primes.push(idx);
    return primes;
  }, []);
}

console.log(sieveOfEratosthenes(30));
// [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
// Time: O(n log log n), Space: O(n)
```

```python
def sieve(n):
    is_prime = [True] * (n + 1)
    is_prime[0] = is_prime[1] = False
    for i in range(2, int(n**0.5) + 1):
        if is_prime[i]:
            for j in range(i*i, n + 1, i):
                is_prime[j] = False
    return [i for i in range(n + 1) if is_prime[i]]
```

---

## 9. Quick Reference — Math for DSA

| Concept | Formula | DSA Usage |
|---------|---------|-----------|
| Sum 1..n | n(n+1)/2 | Complexity analysis |
| log₂(n) | — | Binary search, trees |
| 2ⁿ subsets | — | Backtracking |
| n! permutations | — | Brute force bound |
| GCD | Euclidean algo | Number theory problems |
| Modulo 10⁹+7 | (a%m + b%m)%m | Overflow prevention |
| Sieve | O(n log log n) | Prime detection |
| C(n,r) | n!/(r!(n-r)!) | Grid paths, counting |

---

## 10. Practice Problems

| Problem | Concept | Difficulty |
|---------|---------|-----------|
| Count trailing zeros in n! | Factors of 5 | Easy |
| Check if number is power of 2 | Bit / Math | Easy |
| GCD of two numbers | Euclidean | Easy |
| Count primes less than n | Sieve | Medium |
| Excel column number ↔ title | Base conversion | Medium |
| Power(x, n) | Fast exponentiation | Medium |
| Unique Paths in grid | Combinations | Medium |
| Count good numbers (LeetCode 1922) | Modular exponentiation | Medium |
