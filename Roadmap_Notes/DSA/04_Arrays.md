# Chapter 04 — Arrays

## Simple Explanation (ELI5)
An array is a **row of numbered boxes** sitting side by side. Each box holds one item, and you can jump directly to any box using its number (index).

## Technical Definition
An array is a **contiguous block of memory** storing elements of the same type, accessed by integer index in O(1) time.

## Why It's Needed
Arrays are the **most fundamental data structure**. Almost every other structure (stacks, queues, heaps, hash tables) is built on top of arrays.

## Real-world Analogy
A **row of lockers** in school. Each locker has a number (index). You can go straight to locker #42 without checking lockers #1 through #41.

---

## 1. How Arrays Work Internally

```
Memory addresses are contiguous:
Address:  [1000] [1004] [1008] [1012] [1016]
Values:     10     20     30     40     50
Index:       0      1      2      3      4

To access arr[3]: start_address + (3 × element_size) = 1000 + 12 = 1012
→ Direct jump = O(1)
```

### Static vs Dynamic Arrays

| Feature | Static Array | Dynamic Array (JS Array) |
|---------|-------------|-------------------------|
| Size | Fixed at creation | Grows automatically |
| Resize | Cannot | Doubles capacity when full |
| Insert at end | O(1) | O(1) amortized |
| Memory | Exact fit | May have unused capacity |

---

## 2. Array Operations & Complexity

| Operation | Method | Time | Why |
|-----------|--------|------|-----|
| Access | `arr[i]` | O(1) | Direct address calculation |
| Search | `indexOf(x)` | O(n) | Must scan all elements |
| Push (end) | `arr.push(x)` | O(1)* | Append at tail |
| Pop (end) | `arr.pop()` | O(1) | Remove from tail |
| Insert (start) | `arr.unshift(x)` | O(n) | Shift all elements right |
| Delete (start) | `arr.shift()` | O(n) | Shift all elements left |
| Insert (middle) | `arr.splice(i,0,x)` | O(n) | Shift elements right |
| Delete (middle) | `arr.splice(i,1)` | O(n) | Shift elements left |

### Dry Run: Inserting at Beginning

```
Before: [10, 20, 30, 40]    indices: 0  1  2  3

Insert 5 at index 0:
Step 1: Shift 40 → index 4    [10, 20, 30, _, 40] — wait, wrong order
Actually: shift happens right to left
Step 1: arr[4] = arr[3]       [10, 20, 30, 40, 40]
Step 2: arr[3] = arr[2]       [10, 20, 30, 30, 40]
Step 3: arr[2] = arr[1]       [10, 20, 20, 30, 40]
Step 4: arr[1] = arr[0]       [10, 10, 20, 30, 40]
Step 5: arr[0] = 5            [5, 10, 20, 30, 40]

4 shifts for 4 elements → O(n)
```

---

## 3. Essential Array Methods

```javascript
let arr = [1, 2, 3, 4, 5];

// --- Modification ---
arr.push(6);              // [1,2,3,4,5,6] — O(1)
arr.pop();                // [1,2,3,4,5] — O(1)
arr.unshift(0);           // [0,1,2,3,4,5] — O(n)
arr.shift();              // [1,2,3,4,5] — O(n)
arr.splice(2, 1);         // [1,2,4,5] — remove at index 2
arr.splice(2, 0, 3);      // [1,2,3,4,5] — insert 3 at index 2

// --- Non-mutating ---
let sliced = arr.slice(1, 3);  // [2, 3] — indices 1 to 2
let rev = arr.slice().reverse();
let merged = arr.concat([6,7]);
let found = arr.includes(3);   // true
let idx = arr.indexOf(4);      // 3

// --- Higher Order Methods ---
arr.forEach(x => console.log(x));
let doubled = arr.map(x => x * 2);
let evens = arr.filter(x => x % 2 === 0);
let sum = arr.reduce((acc, x) => acc + x, 0);
let has = arr.some(x => x > 4);
let all = arr.every(x => x > 0);
let first = arr.find(x => x > 3);
let sorted = arr.slice().sort((a,b) => a - b);
```

```python
arr = [1, 2, 3, 4, 5]

arr.append(6)          # [1,2,3,4,5,6]
arr.pop()              # [1,2,3,4,5]
arr.insert(0, 0)       # [0,1,2,3,4,5] — O(n)
arr.pop(0)             # [1,2,3,4,5] — O(n)

sliced = arr[1:3]      # [2, 3]
rev = arr[::-1]        # [5,4,3,2,1]
doubled = [x*2 for x in arr]
evens = [x for x in arr if x % 2 == 0]
total = sum(arr)
```

---

## 4. 2D Arrays (Matrices)

```javascript
let matrix = [
  [1, 2, 3],
  [4, 5, 6],
  [7, 8, 9]
];

// Access: matrix[row][col]
console.log(matrix[1][2]); // 6 (row 1, col 2)

// Traverse entire matrix
for (let i = 0; i < matrix.length; i++) {         // rows
  for (let j = 0; j < matrix[0].length; j++) {    // cols
    console.log(matrix[i][j]);
  }
}
// Time: O(m × n), Space: O(1)
```

```python
matrix = [[1,2,3], [4,5,6], [7,8,9]]
print(matrix[1][2])  # 6

# Create m×n matrix filled with 0
m, n = 3, 4
grid = [[0] * n for _ in range(m)]
# WRONG: [[0]*n] * m — creates shallow copies!
```

---

## 5. Key Array Patterns & Problems

### Kadane's Algorithm — Maximum Subarray Sum

```javascript
function maxSubArray(nums) {
  let maxSum = nums[0];
  let currentSum = nums[0];

  for (let i = 1; i < nums.length; i++) {
    currentSum = Math.max(nums[i], currentSum + nums[i]);
    maxSum = Math.max(maxSum, currentSum);
  }
  return maxSum;
}

// Dry run: [-2, 1, -3, 4, -1, 2, 1, -5, 4]
// i=0: cur=-2, max=-2
// i=1: cur=max(1, -2+1)=1, max=1
// i=2: cur=max(-3, 1-3)=-2, max=1
// i=3: cur=max(4, -2+4)=4, max=4
// i=4: cur=max(-1, 4-1)=3, max=4
// i=5: cur=max(2, 3+2)=5, max=5
// i=6: cur=max(1, 5+1)=6, max=6 ✅
// i=7: cur=max(-5, 6-5)=1, max=6
// i=8: cur=max(4, 1+4)=5, max=6
// Answer: 6 → subarray [4,-1,2,1]
```

```python
def max_sub_array(nums):
    max_sum = cur_sum = nums[0]
    for num in nums[1:]:
        cur_sum = max(num, cur_sum + num)
        max_sum = max(max_sum, cur_sum)
    return max_sum
```

### Rotate Array by K Positions

```javascript
function rotate(arr, k) {
  k = k % arr.length;
  function reverse(a, l, r) {
    while (l < r) { [a[l], a[r]] = [a[r], a[l]]; l++; r--; }
  }
  reverse(arr, 0, arr.length - 1);  // Reverse all
  reverse(arr, 0, k - 1);           // Reverse first k
  reverse(arr, k, arr.length - 1);  // Reverse rest
}
// [1,2,3,4,5,6,7] k=3 → [7,6,5,4,3,2,1] → [5,6,7,4,3,2,1] → [5,6,7,1,2,3,4]
// Time: O(n), Space: O(1)
```

### Dutch National Flag (Sort Colors)

```javascript
function sortColors(nums) {
  let lo = 0, mid = 0, hi = nums.length - 1;
  while (mid <= hi) {
    if (nums[mid] === 0) {
      [nums[lo], nums[mid]] = [nums[mid], nums[lo]];
      lo++; mid++;
    } else if (nums[mid] === 1) {
      mid++;
    } else {
      [nums[mid], nums[hi]] = [nums[hi], nums[mid]];
      hi--;
    }
  }
}
// Time: O(n), Space: O(1), single pass!
```

### Move Zeros to End

```javascript
function moveZeroes(nums) {
  let insertPos = 0;
  for (let i = 0; i < nums.length; i++) {
    if (nums[i] !== 0) {
      [nums[insertPos], nums[i]] = [nums[i], nums[insertPos]];
      insertPos++;
    }
  }
}
```

### Find Missing Number (0 to n)

```javascript
function missingNumber(nums) {
  let n = nums.length;
  let expectedSum = n * (n + 1) / 2;
  let actualSum = nums.reduce((a, b) => a + b, 0);
  return expectedSum - actualSum;
}
// OR use XOR: 0^1^2^...^n XOR all elements → missing number
```

---

## 6. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Using `shift()/unshift()` in loops | O(n) per call → O(n²) total | Use two-pointer or queue |
| `sort()` without comparator | Sorts as strings: `[10,9,8].sort()→[10,8,9]` | Always use `(a,b)=>a-b` |
| Off-by-one in loops | `i <= arr.length` → out of bounds | Use `i < arr.length` |
| Modifying array during forEach | Unpredictable behavior | Use for loop or filter |
| `new Array(n)` without fill | Creates sparse array with holes | Use `new Array(n).fill(0)` |

---

## 7. Edge Cases

- Empty array `[]`
- Single element `[5]`
- All same elements `[3,3,3]`
- Already sorted / reverse sorted
- Array with negative numbers
- k > array length (for rotation: use k % n)
- Integer overflow (use BigInt or modulo)

---

## 8. When NOT to Use Arrays

- Frequent insertions/deletions at the beginning → Use **Linked List**
- Need O(1) lookup by key → Use **Hash Map**
- Need ordered insertion/deletion → Use **BST** or **Heap**
- Data is sparse (mostly empty) → Use **Hash Map** or **Sparse Array**

---

## 9. Practice Problems

| Problem | Pattern | Difficulty |
|---------|---------|-----------|
| Max Subarray (Kadane's) | DP/Greedy | Medium |
| Rotate Array | Reverse trick | Medium |
| Move Zeroes | Two Pointer | Easy |
| Remove Duplicates (sorted) | Two Pointer | Easy |
| Best Time to Buy/Sell Stock | Greedy | Easy |
| Product Except Self | Prefix/Suffix | Medium |
| Container With Most Water | Two Pointer | Medium |
| Sort Colors | Dutch Flag | Medium |
| Merge Intervals | Sort + Sweep | Medium |
| Trapping Rain Water | Two Pointer / Stack | Hard |
| First Missing Positive | Index as hash | Hard |
