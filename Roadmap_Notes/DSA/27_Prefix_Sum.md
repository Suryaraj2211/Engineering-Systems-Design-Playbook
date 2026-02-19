# Chapter 27 — Prefix Sum

## Simple Explanation (ELI5)
Instead of adding up numbers in a range every time (slow), you **pre-calculate cumulative sums** once, then answer any range sum in O(1). Like keeping a running total in your checkbook — to find spending in any period, just subtract two totals.

---

## 1. Basic Prefix Sum

```javascript
// Build prefix array
function buildPrefix(arr) {
  let prefix = [0];
  for (let num of arr) prefix.push(prefix[prefix.length - 1] + num);
  return prefix;
}

// Sum of arr[l..r] = prefix[r+1] - prefix[l]
let arr = [2, 3, 1, 4, 5];
let prefix = buildPrefix(arr); // [0, 2, 5, 6, 10, 15]
console.log(prefix[4] - prefix[1]); // sum(arr[1..3]) = 3+1+4 = 8
```

```python
from itertools import accumulate
arr = [2, 3, 1, 4, 5]
prefix = [0] + list(accumulate(arr))
# [0, 2, 5, 6, 10, 15]
```

| Operation | Without Prefix | With Prefix |
|-----------|---------------|-------------|
| Build | — | O(n) once |
| Range Sum | O(n) per query | O(1) per query |

---

## 2. Subarray Sum Equals K (Prefix + HashMap)
```javascript
function subarraySum(nums, k) {
  let count = 0, prefix = 0;
  let map = new Map([[0, 1]]);
  for (let num of nums) {
    prefix += num;
    if (map.has(prefix - k)) count += map.get(prefix - k);
    map.set(prefix, (map.get(prefix) || 0) + 1);
  }
  return count;
}
```

---

## 3. Product Except Self (Prefix Product)
```javascript
function productExceptSelf(nums) {
  let n = nums.length, result = new Array(n).fill(1);
  let leftProduct = 1;
  for (let i = 0; i < n; i++) { result[i] = leftProduct; leftProduct *= nums[i]; }
  let rightProduct = 1;
  for (let i = n - 1; i >= 0; i--) { result[i] *= rightProduct; rightProduct *= nums[i]; }
  return result;
}
// No division, O(n) time, O(1) extra space
```

---

## 4. 2D Prefix Sum

```javascript
function build2DPrefix(matrix) {
  let m = matrix.length, n = matrix[0].length;
  let prefix = Array.from({length: m+1}, () => new Array(n+1).fill(0));
  for (let i = 1; i <= m; i++)
    for (let j = 1; j <= n; j++)
      prefix[i][j] = matrix[i-1][j-1] + prefix[i-1][j] + prefix[i][j-1] - prefix[i-1][j-1];
  return prefix;
}

// Sum of submatrix (r1,c1) to (r2,c2):
function regionSum(prefix, r1, c1, r2, c2) {
  return prefix[r2+1][c2+1] - prefix[r1][c2+1] - prefix[r2+1][c1] + prefix[r1][c1];
}
```

---

## 5. Difference Array (Range Update in O(1))

```javascript
// Add value to range [l, r]
function rangeUpdate(diff, l, r, val) {
  diff[l] += val;
  if (r + 1 < diff.length) diff[r + 1] -= val;
}

// Reconstruct array from difference array
function reconstruct(diff) {
  let arr = [diff[0]];
  for (let i = 1; i < diff.length; i++) arr.push(arr[i-1] + diff[i]);
  return arr;
}
```

---

## 6. XOR Prefix
```javascript
function xorQueries(arr, queries) {
  let prefix = [0];
  for (let num of arr) prefix.push(prefix[prefix.length - 1] ^ num);
  return queries.map(([l, r]) => prefix[r+1] ^ prefix[l]);
}
// XOR of range [l..r] = prefix[r+1] ^ prefix[l]
```

---

## 7. Practice Problems

| Problem | Variant | Difficulty |
|---------|---------|-----------|
| Range Sum Query (Immutable) | Basic Prefix | Easy |
| Subarray Sum Equals K | Prefix + HashMap | Medium |
| Product Except Self | Prefix/Suffix Product | Medium |
| Continuous Subarray Sum | Prefix + Modulo | Medium |
| Range Sum 2D (Immutable) | 2D Prefix | Medium |
| Corporate Flight Bookings | Difference Array | Medium |
| XOR Queries of Subarray | XOR Prefix | Medium |
| Count Sub Islands | Prefix + Grid | Medium |
