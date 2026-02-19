# Chapter 11 — Searching Algorithms

## Simple Explanation (ELI5)
Searching = finding a specific item. Linear search checks every item. Binary search halves the search space each time — like looking up a word in a dictionary by opening to the middle.

---

## 1. Linear Search — O(n)
```javascript
function linearSearch(arr, target) {
  for (let i = 0; i < arr.length; i++) {
    if (arr[i] === target) return i;
  }
  return -1;
}
```
No sorting required. Use when data is unsorted and small.

---

## 2. Binary Search — O(log n) ⭐

**Prerequisite:** Array must be **sorted**.

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
```

```python
def binary_search(arr, target):
    lo, hi = 0, len(arr) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if arr[mid] == target: return mid
        elif arr[mid] < target: lo = mid + 1
        else: hi = mid - 1
    return -1
```

### Dry Run: Search for 7 in [1, 3, 5, 7, 9, 11]
```
Step 1: lo=0 hi=5 mid=2 arr[2]=5 < 7 → lo=3
Step 2: lo=3 hi=5 mid=4 arr[4]=9 > 7 → hi=3
Step 3: lo=3 hi=3 mid=3 arr[3]=7 = 7 → FOUND at index 3 ✅
```

---

## 3. Binary Search Templates

### Template 1: Exact Match
```javascript
while (lo <= hi) {
  let mid = lo + Math.floor((hi - lo) / 2);
  if (arr[mid] === target) return mid;
  else if (arr[mid] < target) lo = mid + 1;
  else hi = mid - 1;
}
```

### Template 2: Lower Bound (first >= target)
```javascript
function lowerBound(arr, target) {
  let lo = 0, hi = arr.length;
  while (lo < hi) {
    let mid = lo + Math.floor((hi - lo) / 2);
    if (arr[mid] < target) lo = mid + 1;
    else hi = mid;
  }
  return lo;
}
```

### Template 3: Upper Bound (first > target)
```javascript
function upperBound(arr, target) {
  let lo = 0, hi = arr.length;
  while (lo < hi) {
    let mid = lo + Math.floor((hi - lo) / 2);
    if (arr[mid] <= target) lo = mid + 1;
    else hi = mid;
  }
  return lo;
}
```

### Template 4: Search on Answer
```javascript
// "Find minimum X such that condition(X) is true"
function searchOnAnswer(lo, hi) {
  while (lo < hi) {
    let mid = lo + Math.floor((hi - lo) / 2);
    if (condition(mid)) hi = mid;     // mid might be answer
    else lo = mid + 1;                // too small
  }
  return lo;
}
```

---

## 4. BS Variations

### Search in Rotated Sorted Array
```javascript
function searchRotated(arr, target) {
  let lo = 0, hi = arr.length - 1;
  while (lo <= hi) {
    let mid = Math.floor((lo + hi) / 2);
    if (arr[mid] === target) return mid;
    if (arr[lo] <= arr[mid]) {
      if (target >= arr[lo] && target < arr[mid]) hi = mid - 1;
      else lo = mid + 1;
    } else {
      if (target > arr[mid] && target <= arr[hi]) lo = mid + 1;
      else hi = mid - 1;
    }
  }
  return -1;
}
```

### Find First and Last Position
```javascript
function searchRange(arr, target) {
  let first = lowerBound(arr, target);
  if (first === arr.length || arr[first] !== target) return [-1, -1];
  return [first, upperBound(arr, target) - 1];
}
```

### Find Peak Element
```javascript
function findPeakElement(nums) {
  let lo = 0, hi = nums.length - 1;
  while (lo < hi) {
    let mid = Math.floor((lo + hi) / 2);
    if (nums[mid] > nums[mid + 1]) hi = mid;
    else lo = mid + 1;
  }
  return lo;
}
```

### Koko Eating Bananas (BS on Answer)
```javascript
function minEatingSpeed(piles, h) {
  let lo = 1, hi = Math.max(...piles);
  while (lo < hi) {
    let mid = Math.floor((lo + hi) / 2);
    let hours = piles.reduce((s, p) => s + Math.ceil(p / mid), 0);
    if (hours <= h) hi = mid;
    else lo = mid + 1;
  }
  return lo;
}
```

### Search in 2D Matrix
```javascript
function searchMatrix(matrix, target) {
  let m = matrix.length, n = matrix[0].length;
  let lo = 0, hi = m * n - 1;
  while (lo <= hi) {
    let mid = Math.floor((lo + hi) / 2);
    let val = matrix[Math.floor(mid / n)][mid % n];
    if (val === target) return true;
    else if (val < target) lo = mid + 1;
    else hi = mid - 1;
  }
  return false;
}
```

---

## 5. Common Mistakes
- Using `(lo + hi) / 2` → overflow risk. Use `lo + (hi - lo) / 2`
- Wrong loop condition: `lo <= hi` vs `lo < hi` depends on template
- Not handling edge case: target not in array
- Forgetting array must be sorted

## 6. When NOT to Use Binary Search
- Unsorted data (sort first → O(n log n) overhead)
- Linked list (no random access → O(n) per step)
- Small array (linear search is simpler and equally fast)

---

## 7. Practice Problems

| Problem | Pattern | Difficulty |
|---------|---------|-----------|
| Binary Search | Exact match | Easy |
| Search Insert Position | Lower Bound | Easy |
| First Bad Version | BS on Answer | Easy |
| Find First and Last Position | Lower/Upper Bound | Medium |
| Search Rotated Array | Modified BS | Medium |
| Find Peak Element | BS | Medium |
| Koko Eating Bananas | BS on Answer | Medium |
| Search 2D Matrix | Flattened BS | Medium |
| Find Min in Rotated Array | Modified BS | Medium |
| Median of Two Sorted Arrays | BS | Hard |
| Split Array Largest Sum | BS on Answer | Hard |
