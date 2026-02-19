# Chapter 21 — Backtracking

## Simple Explanation (ELI5)
Backtracking is trying every path in a maze — when you hit a dead end, you **go back** to the last junction and try a different path. You explore all possibilities but cut off bad paths early (pruning).

## Technical Definition
A systematic method to explore all potential solutions by building candidates incrementally and **abandoning ("backtracking")** a candidate as soon as it's determined to be invalid.

---

## 1. Template

```javascript
function backtrack(candidates, path, result) {
  if (path is a valid solution) {
    result.push([...path]);  // Clone path!
    return;
  }
  for (let choice of candidates) {
    if (choice is invalid) continue;  // PRUNE
    path.push(choice);        // CHOOSE
    backtrack(remaining, path, result);  // EXPLORE
    path.pop();                // UN-CHOOSE (backtrack)
  }
}
```

---

## 2. Subsets (Power Set)
```javascript
function subsets(nums) {
  let result = [];
  function bt(start, path) {
    result.push([...path]);
    for (let i = start; i < nums.length; i++) {
      path.push(nums[i]);
      bt(i + 1, path);
      path.pop();
    }
  }
  bt(0, []);
  return result;
}
// [1,2,3] → [[], [1], [1,2], [1,2,3], [1,3], [2], [2,3], [3]]
```

```python
def subsets(nums):
    result = []
    def bt(start, path):
        result.append(path[:])
        for i in range(start, len(nums)):
            path.append(nums[i])
            bt(i + 1, path)
            path.pop()
    bt(0, [])
    return result
```

---

## 3. Permutations
```javascript
function permute(nums) {
  let result = [];
  function bt(path, used) {
    if (path.length === nums.length) { result.push([...path]); return; }
    for (let i = 0; i < nums.length; i++) {
      if (used[i]) continue;
      used[i] = true;
      path.push(nums[i]);
      bt(path, used);
      path.pop();
      used[i] = false;
    }
  }
  bt([], new Array(nums.length).fill(false));
  return result;
}
```

---

## 4. Combination Sum
```javascript
function combinationSum(candidates, target) {
  let result = [];
  function bt(start, path, remaining) {
    if (remaining === 0) { result.push([...path]); return; }
    for (let i = start; i < candidates.length; i++) {
      if (candidates[i] > remaining) continue;
      path.push(candidates[i]);
      bt(i, path, remaining - candidates[i]);  // i not i+1 (can reuse)
      path.pop();
    }
  }
  candidates.sort((a,b) => a-b);
  bt(0, [], target);
  return result;
}
```

---

## 5. N-Queens
```javascript
function solveNQueens(n) {
  let result = [], board = Array.from({length: n}, () => '.'.repeat(n));
  let cols = new Set(), diag1 = new Set(), diag2 = new Set();

  function bt(row) {
    if (row === n) { result.push([...board]); return; }
    for (let col = 0; col < n; col++) {
      if (cols.has(col) || diag1.has(row-col) || diag2.has(row+col)) continue;
      cols.add(col); diag1.add(row-col); diag2.add(row+col);
      board[row] = board[row].substring(0,col) + 'Q' + board[row].substring(col+1);
      bt(row + 1);
      board[row] = board[row].substring(0,col) + '.' + board[row].substring(col+1);
      cols.delete(col); diag1.delete(row-col); diag2.delete(row+col);
    }
  }
  bt(0);
  return result;
}
```

---

## 6. Sudoku Solver
```javascript
function solveSudoku(board) {
  function isValid(board, r, c, num) {
    for (let i = 0; i < 9; i++) {
      if (board[r][i] === num || board[i][c] === num) return false;
      let br = 3*Math.floor(r/3) + Math.floor(i/3);
      let bc = 3*Math.floor(c/3) + i % 3;
      if (board[br][bc] === num) return false;
    }
    return true;
  }
  function solve() {
    for (let r = 0; r < 9; r++) {
      for (let c = 0; c < 9; c++) {
        if (board[r][c] === '.') {
          for (let num = 1; num <= 9; num++) {
            if (isValid(board, r, c, String(num))) {
              board[r][c] = String(num);
              if (solve()) return true;
              board[r][c] = '.';
            }
          }
          return false;
        }
      }
    }
    return true;
  }
  solve();
}
```

---

## 7. Generate Parentheses
```javascript
function generateParenthesis(n) {
  let result = [];
  function bt(path, open, close) {
    if (path.length === 2 * n) { result.push(path); return; }
    if (open < n) bt(path + '(', open + 1, close);
    if (close < open) bt(path + ')', open, close + 1);
  }
  bt('', 0, 0);
  return result;
}
```

---

## 8. Practice Problems

| Problem | Type | Difficulty |
|---------|------|-----------|
| Subsets | Generate | Medium |
| Subsets II (with dups) | Deduplicate | Medium |
| Permutations | Generate | Medium |
| Permutations II (with dups) | Deduplicate | Medium |
| Combination Sum | Reusable | Medium |
| Generate Parentheses | Constraint backtrack | Medium |
| Letter Combinations of Phone | Generate | Medium |
| Palindrome Partitioning | Partition | Medium |
| N-Queens | Constraint | Hard |
| Sudoku Solver | Constraint | Hard |
| Word Search | Grid DFS | Medium |
