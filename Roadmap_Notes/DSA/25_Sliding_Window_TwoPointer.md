# Chapter 25 — Sliding Window & Two Pointer

## Simple Explanation (ELI5)
Sliding Window: Imagine looking through a window on a train — the view shifts as the train moves, but you only see a fixed portion at a time.
Two Pointer: Two bookmarks in a book, moving toward each other or in the same direction based on conditions.

---

## 1. Fixed-Size Sliding Window

```javascript
// Max sum of k consecutive elements
function maxSumSubarray(arr, k) {
  let sum = 0, maxSum = 0;
  for (let i = 0; i < k; i++) sum += arr[i]; // Initial window
  maxSum = sum;
  for (let i = k; i < arr.length; i++) {
    sum += arr[i] - arr[i - k]; // Slide: add right, remove left
    maxSum = Math.max(maxSum, sum);
  }
  return maxSum;
}
// Time: O(n), not O(n×k)!
```

---

## 2. Variable-Size Sliding Window

### Longest Substring Without Repeating Characters
```javascript
function lengthOfLongestSubstring(s) {
  let map = new Map(), left = 0, maxLen = 0;
  for (let right = 0; right < s.length; right++) {
    if (map.has(s[right])) left = Math.max(left, map.get(s[right]) + 1);
    map.set(s[right], right);
    maxLen = Math.max(maxLen, right - left + 1);
  }
  return maxLen;
}
```

```python
def length_of_longest_substring(s):
    seen = {}
    left = max_len = 0
    for right, ch in enumerate(s):
        if ch in seen:
            left = max(left, seen[ch] + 1)
        seen[ch] = right
        max_len = max(max_len, right - left + 1)
    return max_len
```

### Minimum Window Substring
```javascript
function minWindow(s, t) {
  let need = new Map();
  for (let c of t) need.set(c, (need.get(c)||0) + 1);
  let have = 0, required = need.size;
  let left = 0, minLen = Infinity, minStart = 0;
  let window = new Map();

  for (let right = 0; right < s.length; right++) {
    let c = s[right];
    window.set(c, (window.get(c)||0) + 1);
    if (need.has(c) && window.get(c) === need.get(c)) have++;

    while (have === required) {
      if (right - left + 1 < minLen) { minLen = right - left + 1; minStart = left; }
      let lc = s[left];
      window.set(lc, window.get(lc) - 1);
      if (need.has(lc) && window.get(lc) < need.get(lc)) have--;
      left++;
    }
  }
  return minLen === Infinity ? "" : s.substring(minStart, minStart + minLen);
}
```

### Minimum Size Subarray Sum
```javascript
function minSubArrayLen(target, nums) {
  let left = 0, sum = 0, minLen = Infinity;
  for (let right = 0; right < nums.length; right++) {
    sum += nums[right];
    while (sum >= target) {
      minLen = Math.min(minLen, right - left + 1);
      sum -= nums[left++];
    }
  }
  return minLen === Infinity ? 0 : minLen;
}
```

---

## 3. Two Pointer Patterns

### Opposite Direction (sorted array)
```javascript
function twoSum(arr, target) {
  let l = 0, r = arr.length - 1;
  while (l < r) {
    let sum = arr[l] + arr[r];
    if (sum === target) return [l, r];
    else if (sum < target) l++;
    else r--;
  }
  return [-1, -1];
}
```

### Three Sum
```javascript
function threeSum(nums) {
  nums.sort((a,b) => a-b);
  let result = [];
  for (let i = 0; i < nums.length - 2; i++) {
    if (i > 0 && nums[i] === nums[i-1]) continue;
    let l = i+1, r = nums.length-1;
    while (l < r) {
      let sum = nums[i] + nums[l] + nums[r];
      if (sum === 0) {
        result.push([nums[i], nums[l], nums[r]]);
        while (l < r && nums[l] === nums[l+1]) l++;
        while (l < r && nums[r] === nums[r-1]) r--;
        l++; r--;
      } else if (sum < 0) l++;
      else r--;
    }
  }
  return result;
}
```

### Container With Most Water
```javascript
function maxArea(height) {
  let l = 0, r = height.length - 1, max = 0;
  while (l < r) {
    max = Math.max(max, Math.min(height[l], height[r]) * (r - l));
    if (height[l] < height[r]) l++;
    else r--;
  }
  return max;
}
```

### Trapping Rain Water
```javascript
function trap(height) {
  let l = 0, r = height.length - 1, lMax = 0, rMax = 0, water = 0;
  while (l < r) {
    if (height[l] < height[r]) {
      lMax = Math.max(lMax, height[l]);
      water += lMax - height[l];
      l++;
    } else {
      rMax = Math.max(rMax, height[r]);
      water += rMax - height[r];
      r--;
    }
  }
  return water;
}
```

---

## 4. When to Use Which

| Clue | Technique |
|------|-----------|
| "Contiguous subarray/substring" | Sliding Window |
| "Sorted array, find pair" | Two Pointer |
| "Minimum/maximum window" | Variable Sliding Window |
| "Sum equals target (sorted)" | Two Pointer |
| "Remove duplicates in-place" | Two Pointer (same direction) |

---

## 5. Practice Problems

| Problem | Technique | Difficulty |
|---------|-----------|-----------|
| Max Sum Subarray of Size K | Fixed Window | Easy |
| Longest Substring No Repeat | Variable Window | Medium |
| Minimum Window Substring | Variable Window | Hard |
| Two Sum II (sorted) | Two Pointer | Medium |
| Three Sum | Sort + Two Pointer | Medium |
| Container With Most Water | Two Pointer | Medium |
| Trapping Rain Water | Two Pointer | Hard |
| Remove Duplicates Sorted | Same-direction | Easy |
| Longest Repeating Character Replacement | Window | Medium |
| Fruits Into Baskets | Window | Medium |
