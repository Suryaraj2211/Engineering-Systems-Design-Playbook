# Chapter 24 — Bit Manipulation

## The Problem This Solves
Certain operations that seem to require O(n) space or O(log n) time can be done in O(1) time and O(1) space using bitwise operations. The CPU executes bitwise instructions in a single clock cycle — faster than any arithmetic equivalent. This is critical for competitive programming, embedded systems, and graphics programming (where bit packing is used for color/depth buffers).

---

## 1. Binary Number System — The Foundation

```
Decimal → Binary conversion:
  13 in binary:
    13 ÷ 2 = 6 remainder 1
     6 ÷ 2 = 3 remainder 0
     3 ÷ 2 = 1 remainder 1
     1 ÷ 2 = 0 remainder 1
    Read bottom-up: 1101

  13 = 1×2³ + 1×2² + 0×2¹ + 1×2⁰
     = 8 + 4 + 0 + 1 = 13 ✓
```

### Two's Complement (How Computers Store Negative Numbers)
```
For 8-bit integers:
   5 = 00000101
  -5 = ?
  
  Step 1: Invert all bits:  11111010
  Step 2: Add 1:            11111011
  So -5 = 11111011

Verification: 00000101 + 11111011 = 100000000 (overflow discarded = 0) ✓

Range of 8-bit signed: -128 to 127
Range of 32-bit signed: -2,147,483,648 to 2,147,483,647
```

---

## 2. Bitwise Operators — Complete Reference

| Operator | Symbol | Rule | Example (5=0101, 3=0011) |
|----------|--------|------|--------------------------|
| AND | `&` | Both bits must be 1 | `5 & 3 = 0001 = 1` |
| OR | `\|` | At least one bit is 1 | `5 \| 3 = 0111 = 7` |
| XOR | `^` | Exactly one bit is 1 | `5 ^ 3 = 0110 = 6` |
| NOT | `~` | Flip every bit | `~5 = ...1010 = -6` |
| Left Shift | `<<` | Shift bits left (multiply by 2^n) | `5 << 1 = 1010 = 10` |
| Right Shift | `>>` | Shift bits right (divide by 2^n) | `5 >> 1 = 0010 = 2` |

---

## 3. Essential Bit Tricks (With Proofs)

```javascript
// CHECK ODD/EVEN — the last bit determines parity
n & 1;  // 1 = odd, 0 = even
// Why: Even numbers always have 0 as their last bit. AND with 1 isolates it.

// POWER OF 2? — powers of 2 have exactly one set bit
n > 0 && (n & (n - 1)) === 0;
// Why: 8 = 1000, 7 = 0111. 1000 & 0111 = 0000. Only works for single-bit numbers.

// GET BIT at position p
(n >> p) & 1;
// Shift the target bit to position 0, then AND with 1 to isolate it.

// SET BIT at position p (force to 1)
n | (1 << p);
// Create a mask with 1 at position p, OR it in.

// CLEAR BIT at position p (force to 0)
n & ~(1 << p);
// Create a mask with 0 at position p (invert 1<<p), AND it in.

// TOGGLE BIT at position p (flip)
n ^ (1 << p);
// XOR with 1 flips the bit. XOR with 0 keeps it.

// LOWEST SET BIT (isolate the rightmost 1)
n & (-n);
// 12 = 1100, -12 = 0100. Result: 0100 = 4 (position 2).

// CLEAR LOWEST SET BIT
n & (n - 1);
// 12 = 1100, 11 = 1011. Result: 1000 = 8.

// COUNT SET BITS (Brian Kernighan's Algorithm) — O(number of set bits)
function countBits(n) {
    let count = 0;
    while (n) {
        n &= n - 1; // Clears the lowest set bit each iteration
        count++;
    }
    return count;
}
// 13 = 1101 → 1100 → 1000 → 0000. count = 3 ✓

// SWAP WITHOUT TEMPORARY VARIABLE
a ^= b;
b ^= a;
a ^= b;
// After: a has original b, b has original a.
```

---

## 4. XOR Properties — The Most Useful Operator

```
FUNDAMENTAL PROPERTIES:
  a ^ a = 0          (any number XOR itself = 0)
  a ^ 0 = a          (XOR with 0 = identity)
  a ^ b = b ^ a      (commutative)
  (a ^ b) ^ c = a ^ (b ^ c)  (associative)
```

### Single Number — Find the non-duplicate in O(1) space
```javascript
// Every element appears twice except one. Find it.
function singleNumber(nums) {
    return nums.reduce((xor, n) => xor ^ n, 0);
}
// [4, 1, 2, 1, 2] → 4^1^2^1^2 = (1^1)^(2^2)^4 = 0^0^4 = 4

// Manual trace:
// 0 ^ 4 = 4
// 4 ^ 1 = 5
// 5 ^ 2 = 7
// 7 ^ 1 = 6  (1 cancels)
// 6 ^ 2 = 4  (2 cancels) → answer is 4 ✓
```

### Missing Number — Find the missing element from [0, n]
```javascript
function missingNumber(nums) {
    let xor = nums.length; // Start with n
    for (let i = 0; i < nums.length; i++) {
        xor ^= i ^ nums[i]; // XOR expected index with actual value
    }
    return xor;
}
// [0, 1, 3] → length=3, xor = 3^0^0^1^1^2^3 = 2
```

---

## 5. Bitmask Subset Enumeration

A bitmask of N bits can represent any subset of N elements. Bit i being set means element i is included.

```javascript
function allSubsets(nums) {
    const n = nums.length;
    const result = [];
    
    for (let mask = 0; mask < (1 << n); mask++) {
        const subset = [];
        for (let i = 0; i < n; i++) {
            if (mask & (1 << i)) subset.push(nums[i]);
        }
        result.push(subset);
    }
    return result;
}

// nums = [a, b, c] → 2^3 = 8 subsets:
// mask=000: []
// mask=001: [a]
// mask=010: [b]
// mask=011: [a,b]
// mask=100: [c]
// mask=101: [a,c]
// mask=110: [b,c]
// mask=111: [a,b,c]
```

### Bitmask DP — Traveling Salesman Problem (TSP)
```javascript
// dp[mask][i] = minimum cost to visit all cities in 'mask', ending at city 'i'
// mask is a bitmask where bit j = 1 means city j has been visited

function tsp(dist, n) {
    const ALL = (1 << n) - 1;
    const dp = Array.from({ length: 1 << n }, () => new Array(n).fill(Infinity));
    dp[1][0] = 0; // Start at city 0
    
    for (let mask = 1; mask <= ALL; mask++) {
        for (let u = 0; u < n; u++) {
            if (!(mask & (1 << u)) || dp[mask][u] === Infinity) continue;
            for (let v = 0; v < n; v++) {
                if (mask & (1 << v)) continue; // Already visited
                const newMask = mask | (1 << v);
                dp[newMask][v] = Math.min(dp[newMask][v], dp[mask][u] + dist[u][v]);
            }
        }
    }
    
    // Find minimum cost to visit all cities and return to start
    let ans = Infinity;
    for (let u = 1; u < n; u++) {
        ans = Math.min(ans, dp[ALL][u] + dist[u][0]);
    }
    return ans;
}
// Complexity: O(2^n * n^2) — exponential but vastly better than O(n!) brute force
```

---

## 6. Practice Problems

| Problem | Bit Trick | Difficulty |
|---------|-----------|------------|
| Single Number | XOR cancellation | Easy |
| Number of 1 Bits | Brian Kernighan | Easy |
| Missing Number | XOR with expected | Easy |
| Power of Two | n & (n-1) == 0 | Easy |
| Reverse Bits | Shift + OR | Easy |
| Counting Bits (0..n) | DP + lowest bit | Easy |
| Single Number II (3x) | Bit counting mod 3 | Medium |
| Subsets via Bitmask | Enumerate all masks | Medium |
| Sum Without + Operator | XOR + carry (AND<<1) | Medium |
| Minimum Flips for OR | Bit comparison | Medium |
