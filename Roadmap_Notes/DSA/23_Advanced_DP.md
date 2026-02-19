# Chapter 23 — Advanced Dynamic Programming

## Topics Beyond Basic DP
This chapter covers DP patterns seen in harder interview and competitive programming problems.

---

## 1. DP on Strings — Distinct Subsequences
```javascript
function numDistinct(s, t) {
  let m = s.length, n = t.length;
  let dp = Array.from({length: m+1}, () => new Array(n+1).fill(0));
  for (let i = 0; i <= m; i++) dp[i][0] = 1;
  for (let i = 1; i <= m; i++)
    for (let j = 1; j <= n; j++)
      dp[i][j] = dp[i-1][j] + (s[i-1] === t[j-1] ? dp[i-1][j-1] : 0);
  return dp[m][n];
}
```

---

## 2. State Machine DP — Buy & Sell Stock with Cooldown
```javascript
function maxProfit(prices) {
  let hold = -prices[0], sold = 0, rest = 0;
  for (let i = 1; i < prices.length; i++) {
    let prevHold = hold, prevSold = sold;
    hold = Math.max(hold, rest - prices[i]);
    sold = hold + prices[i]; // Actually: prevHold + prices[i]
    rest = Math.max(rest, prevSold);
    // Correct version:
    sold = prevHold + prices[i];
  }
  return Math.max(sold, rest);
}
```

### Stock with K Transactions
```javascript
function maxProfitK(k, prices) {
  if (k >= prices.length / 2) {
    let profit = 0;
    for (let i = 1; i < prices.length; i++)
      profit += Math.max(0, prices[i] - prices[i-1]);
    return profit;
  }
  let dp = Array.from({length: k+1}, () => new Array(prices.length).fill(0));
  for (let t = 1; t <= k; t++) {
    let maxDiff = -prices[0];
    for (let d = 1; d < prices.length; d++) {
      dp[t][d] = Math.max(dp[t][d-1], prices[d] + maxDiff);
      maxDiff = Math.max(maxDiff, dp[t-1][d] - prices[d]);
    }
  }
  return dp[k][prices.length - 1];
}
```

---

## 3. Interval DP — Burst Balloons
```javascript
function maxCoins(nums) {
  nums = [1, ...nums, 1];
  let n = nums.length;
  let dp = Array.from({length: n}, () => new Array(n).fill(0));

  for (let len = 2; len < n; len++) {
    for (let left = 0; left < n - len; left++) {
      let right = left + len;
      for (let k = left + 1; k < right; k++) {
        dp[left][right] = Math.max(dp[left][right],
          dp[left][k] + dp[k][right] + nums[left]*nums[k]*nums[right]);
      }
    }
  }
  return dp[0][n-1];
}
// Time: O(n³), Space: O(n²)
```

---

## 4. Bitmask DP — Traveling Salesman (TSP concept)
```javascript
// Visit all n cities exactly once, minimize total distance
function tsp(dist, n) {
  let ALL = (1 << n) - 1;
  let dp = Array.from({length: 1 << n}, () => new Array(n).fill(Infinity));
  dp[1][0] = 0; // Start at city 0

  for (let mask = 1; mask <= ALL; mask++) {
    for (let u = 0; u < n; u++) {
      if (!(mask & (1 << u))) continue;
      for (let v = 0; v < n; v++) {
        if (mask & (1 << v)) continue;
        let newMask = mask | (1 << v);
        dp[newMask][v] = Math.min(dp[newMask][v], dp[mask][u] + dist[u][v]);
      }
    }
  }

  let ans = Infinity;
  for (let u = 0; u < n; u++)
    ans = Math.min(ans, dp[ALL][u] + dist[u][0]);
  return ans;
}
// Time: O(2ⁿ × n²), Space: O(2ⁿ × n)
// Works for n ≤ 20
```

---

## 5. DP on Trees — House Robber III
```javascript
function rob(root) {
  function dfs(node) {
    if (!node) return [0, 0]; // [rob this node, skip this node]
    let left = dfs(node.left), right = dfs(node.right);
    let robThis = node.val + left[1] + right[1];
    let skipThis = Math.max(...left) + Math.max(...right);
    return [robThis, skipThis];
  }
  return Math.max(...dfs(root));
}
```

---

## 6. Digit DP — Count Numbers with Property
Count numbers from 1 to N with a specific digit property (e.g., no repeated digits).

```javascript
// Count numbers from 1 to n where no digit repeats
function countSpecialNumbers(n) {
  let digits = String(n).split('').map(Number);
  // ... (complex implementation with tight bound tracking)
  // Key idea: dp[position][mask][tight][started]
}
// Used in competitive programming for digit-constraint problems
```

---

## 7. DP Pattern Summary

| Pattern | Example Problems | Key State |
|---------|-----------------|-----------|
| 1D Linear | Fibonacci, Stairs, House Robber | dp[i] |
| 1D with choices | Coin Change, Jump Game | dp[i] with inner loop |
| 2D Grid | Unique Paths, Min Path Sum | dp[i][j] |
| 2D String | LCS, Edit Distance | dp[i][j] on two strings |
| Knapsack | 0/1 Knapsack, Subset Sum | dp[i][w] item × capacity |
| Interval | Burst Balloons, Matrix Chain | dp[l][r] left × right |
| State Machine | Stock problems | dp[i][state] |
| Bitmask | TSP, Assign tasks | dp[mask][i] |
| Tree DP | House Robber III | DFS returns tuple |

---

## 8. Practice Problems

| Problem | Pattern | Difficulty |
|---------|---------|-----------|
| Best Time to Buy Stock Cooldown | State Machine | Medium |
| Best Time to Buy Stock IV | State Machine | Hard |
| Burst Balloons | Interval DP | Hard |
| Regular Expression Matching | 2D DP | Hard |
| Wildcard Matching | 2D DP | Hard |
| Longest Valid Parentheses | 1D DP / Stack | Hard |
| House Robber III | Tree DP | Medium |
| Palindrome Partitioning II | Interval DP | Hard |
| Distinct Subsequences | String DP | Hard |
| Interleaving String | 2D DP | Medium |
