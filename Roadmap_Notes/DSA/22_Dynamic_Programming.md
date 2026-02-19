# Chapter 22 — Dynamic Programming

## Simple Explanation (ELI5)
DP is "smart recursion" — instead of solving the same subproblem again and again, you **remember the answer** the first time and reuse it. Like writing your homework answers on a whiteboard so you never redo the same calculation.

## Technical Definition
An optimization technique for problems with **overlapping subproblems** and **optimal substructure**. Two approaches: top-down (memoization) and bottom-up (tabulation).

---

## 1. DP Framework
1. **Define state:** What does `dp[i]` represent?
2. **Recurrence relation:** How to compute `dp[i]` from smaller states?
3. **Base case:** Starting values
4. **Order:** Which direction to iterate?
5. **Answer:** Which cell has the final answer?

---

## 2. 1D Problems

### Fibonacci
```javascript
// Bottom-up — O(n) time, O(1) space
function fib(n) {
  if (n <= 1) return n;
  let a = 0, b = 1;
  for (let i = 2; i <= n; i++) { [a, b] = [b, a + b]; }
  return b;
}
```

### Climbing Stairs
```javascript
function climbStairs(n) {
  if (n <= 2) return n;
  let a = 1, b = 2;
  for (let i = 3; i <= n; i++) { [a, b] = [b, a + b]; }
  return b;
}
```

### House Robber
```javascript
function rob(nums) {
  let prev2 = 0, prev1 = 0;
  for (let num of nums) {
    let curr = Math.max(prev1, prev2 + num);
    prev2 = prev1; prev1 = curr;
  }
  return prev1;
}
// dp[i] = max(dp[i-1], dp[i-2] + nums[i])
```

### Coin Change
```javascript
function coinChange(coins, amount) {
  let dp = new Array(amount + 1).fill(Infinity);
  dp[0] = 0;
  for (let i = 1; i <= amount; i++)
    for (let coin of coins)
      if (coin <= i) dp[i] = Math.min(dp[i], dp[i - coin] + 1);
  return dp[amount] === Infinity ? -1 : dp[amount];
}
```

### Longest Increasing Subsequence
```javascript
function lengthOfLIS(nums) {
  let dp = new Array(nums.length).fill(1);
  for (let i = 1; i < nums.length; i++)
    for (let j = 0; j < i; j++)
      if (nums[j] < nums[i]) dp[i] = Math.max(dp[i], dp[j] + 1);
  return Math.max(...dp);
}
// O(n²). O(n log n) possible with binary search + patience sorting.
```

---

## 3. 2D Problems

### Unique Paths
```javascript
function uniquePaths(m, n) {
  let dp = Array.from({length: m}, () => new Array(n).fill(1));
  for (let i = 1; i < m; i++)
    for (let j = 1; j < n; j++)
      dp[i][j] = dp[i-1][j] + dp[i][j-1];
  return dp[m-1][n-1];
}
```

### Longest Common Subsequence
```javascript
function lcs(text1, text2) {
  let m = text1.length, n = text2.length;
  let dp = Array.from({length: m+1}, () => new Array(n+1).fill(0));
  for (let i = 1; i <= m; i++)
    for (let j = 1; j <= n; j++)
      dp[i][j] = text1[i-1] === text2[j-1]
        ? dp[i-1][j-1] + 1
        : Math.max(dp[i-1][j], dp[i][j-1]);
  return dp[m][n];
}
```

### Edit Distance
```javascript
function minDistance(w1, w2) {
  let m = w1.length, n = w2.length;
  let dp = Array.from({length: m+1}, (_,i) =>
    Array.from({length: n+1}, (_,j) => i === 0 ? j : j === 0 ? i : 0));
  for (let i = 1; i <= m; i++)
    for (let j = 1; j <= n; j++)
      dp[i][j] = w1[i-1] === w2[j-1]
        ? dp[i-1][j-1]
        : 1 + Math.min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1]);
  return dp[m][n];
}
```

---

## 4. Knapsack

### 0/1 Knapsack
```javascript
function knapsack(weights, values, cap) {
  let n = weights.length;
  let dp = Array.from({length: n+1}, () => new Array(cap+1).fill(0));
  for (let i = 1; i <= n; i++)
    for (let w = 0; w <= cap; w++)
      dp[i][w] = weights[i-1] <= w
        ? Math.max(dp[i-1][w], dp[i-1][w-weights[i-1]] + values[i-1])
        : dp[i-1][w];
  return dp[n][cap];
}
```

### Unbounded Knapsack
```javascript
function unboundedKnapsack(weights, values, cap) {
  let dp = new Array(cap + 1).fill(0);
  for (let w = 1; w <= cap; w++)
    for (let i = 0; i < weights.length; i++)
      if (weights[i] <= w)
        dp[w] = Math.max(dp[w], dp[w - weights[i]] + values[i]);
  return dp[cap];
}
```

---

## 5. Word Break
```javascript
function wordBreak(s, dict) {
  let dp = new Array(s.length + 1).fill(false);
  dp[0] = true;
  let words = new Set(dict);
  for (let i = 1; i <= s.length; i++)
    for (let j = 0; j < i; j++)
      if (dp[j] && words.has(s.substring(j, i))) { dp[i] = true; break; }
  return dp[s.length];
}
```

---

## 6. Top-Down vs Bottom-Up
| | Memoization | Tabulation |
|--|-------------|------------|
| Approach | Recursive + cache | Iterative + table |
| Solves | Only needed states | All states |
| Stack overflow | Possible | No |
| Intuition | More natural | Requires careful ordering |
| Space optimization | Harder | Easier |

---

## 7. Practice Problems

| Problem | Type | Difficulty |
|---------|------|-----------|
| Climbing Stairs | 1D | Easy |
| House Robber | 1D | Medium |
| Coin Change | 1D | Medium |
| LIS | 1D | Medium |
| Unique Paths | 2D Grid | Medium |
| LCS | 2D String | Medium |
| Edit Distance | 2D String | Medium |
| 0/1 Knapsack | Knapsack | Medium |
| Word Break | String DP | Medium |
| Partition Equal Subset Sum | Knapsack variant | Medium |
| Palindrome Substring | Expand/DP | Medium |
| Decode Ways | 1D | Medium |
| Maximum Product Subarray | 1D | Medium |
| Target Sum | Knapsack | Medium |
