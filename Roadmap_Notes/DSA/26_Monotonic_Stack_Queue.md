# Chapter 26 — Monotonic Stack & Monotonic Queue

## The Problem This Solves
"For each element, find the next greater element" is O(n^2) with brute force (check every element to the right). A Monotonic Stack solves this in **O(n)** by maintaining elements in sorted order and discarding irrelevant candidates. Similarly, "find the maximum in every sliding window of size K" is O(nK) with brute force; a Monotonic Queue (Deque) does it in **O(n)**.

---

## 1. Monotonic Stack — Core Concept

A stack where elements are always in sorted order (either strictly increasing or strictly decreasing). When a new element violates the order, we pop elements until the order is restored.

```
MONOTONIC DECREASING STACK (for "Next Greater Element"):
════════════════════════════════════════════════════════
  For each element:
    1. While stack top <= current element: POP (current is the "next greater" for popped)
    2. Push current element onto stack
    
  Elements remaining in the stack at the end have NO next greater element.
```

---

## 2. Next Greater Element — Complete Walkthrough

**Problem:** For each element in an array, find the first element to its right that is strictly greater. Return -1 if none exists.

```javascript
function nextGreaterElement(nums) {
    const n = nums.length;
    const result = new Array(n).fill(-1);
    const stack = []; // Stores INDICES, not values
    
    for (let i = 0; i < n; i++) {
        // Pop all elements that current element is greater than
        while (stack.length && nums[stack[stack.length - 1]] < nums[i]) {
            const idx = stack.pop();
            result[idx] = nums[i]; // nums[i] is the next greater for nums[idx]
        }
        stack.push(i);
    }
    return result;
}
```

**Manual Walkthrough — [2, 1, 2, 4, 3]:**
```
i=0, val=2: Stack empty. Push 0.           Stack: [0]        Result: [-1,-1,-1,-1,-1]
i=1, val=1: 1 < 2? No pop. Push 1.        Stack: [0,1]      Result: [-1,-1,-1,-1,-1]
i=2, val=2: 2 > nums[1]=1? YES → pop 1, result[1]=2.
            2 > nums[0]=2? NO. Push 2.    Stack: [0,2]      Result: [-1, 2,-1,-1,-1]
i=3, val=4: 4 > nums[2]=2? YES → pop 2, result[2]=4.
            4 > nums[0]=2? YES → pop 0, result[0]=4.
            Stack empty. Push 3.           Stack: [3]        Result: [ 4, 2, 4,-1,-1]
i=4, val=3: 3 > nums[3]=4? NO. Push 4.    Stack: [3,4]      Result: [ 4, 2, 4,-1,-1]

Remaining in stack: indices 3,4 → no next greater → stay as -1.
Final result: [4, 2, 4, -1, -1] ✓
```

---

## 3. Next Smaller Element (Monotonic Increasing Stack)

Same concept, but maintain an increasing stack. Pop when current element is SMALLER.

```javascript
function nextSmallerElement(nums) {
    const n = nums.length;
    const result = new Array(n).fill(-1);
    const stack = [];
    
    for (let i = 0; i < n; i++) {
        while (stack.length && nums[stack[stack.length - 1]] > nums[i]) {
            result[stack.pop()] = nums[i];
        }
        stack.push(i);
    }
    return result;
}
```

---

## 4. Largest Rectangle in Histogram — The Classic Problem

**Problem:** Given an array of bar heights, find the largest rectangular area under the histogram.

```javascript
function largestRectangle(heights) {
    const stack = []; // Monotonic increasing stack of indices
    let maxArea = 0;
    const n = heights.length;
    
    for (let i = 0; i <= n; i++) {
        const h = i === n ? 0 : heights[i]; // Sentinel: force final pop
        
        while (stack.length && heights[stack[stack.length - 1]] > h) {
            const height = heights[stack.pop()];
            const width = stack.length ? i - stack[stack.length - 1] - 1 : i;
            maxArea = Math.max(maxArea, height * width);
        }
        stack.push(i);
    }
    return maxArea;
}
```

**Manual Walkthrough — heights = [2, 1, 5, 6, 2, 3]:**
```
i=0, h=2: Push 0.                         Stack: [0]
i=1, h=1: Pop 0 (height=2, width=1). Area=2.   Stack: []
           Push 1.                         Stack: [1]
i=2, h=5: Push 2.                         Stack: [1,2]
i=3, h=6: Push 3.                         Stack: [1,2,3]
i=4, h=2: Pop 3 (height=6, width=4-2-1=1). Area=6.
           Pop 2 (height=5, width=4-1-1=2). Area=10. ← MAXIMUM
           Push 4.                         Stack: [1,4]
i=5, h=3: Push 5.                         Stack: [1,4,5]
i=6, h=0: Pop 5 (height=3, width=6-4-1=1). Area=3.
           Pop 4 (height=2, width=6-1-1=4). Area=8.
           Pop 1 (height=1, width=6).       Area=6.

Answer: 10 ✓ (rectangle of height 5, width 2, at indices 2-3)
```

---

## 5. Monotonic Queue — Sliding Window Maximum

**Problem:** Given an array and window size K, find the maximum value in every window of size K.

```javascript
function slidingWindowMax(nums, k) {
    const deque = []; // Stores INDICES. Front = index of current max.
    const result = [];
    
    for (let i = 0; i < nums.length; i++) {
        // Remove elements outside the window
        while (deque.length && deque[0] <= i - k) {
            deque.shift();
        }
        
        // Remove elements smaller than current (they can never be the max)
        while (deque.length && nums[deque[deque.length - 1]] <= nums[i]) {
            deque.pop();
        }
        
        deque.push(i);
        
        // Window is fully formed starting at index k-1
        if (i >= k - 1) {
            result.push(nums[deque[0]]); // Front of deque = current max
        }
    }
    return result;
}
```

**Manual Walkthrough — nums = [1, 3, -1, -3, 5, 3, 6, 7], k = 3:**
```
i=0, val=1:  Deque: [0]        (window not full yet)
i=1, val=3:  Pop 0 (1<3). Deque: [1]
i=2, val=-1: Deque: [1,2]      Window [1,3,-1] → max = nums[1] = 3
i=3, val=-3: Deque: [1,2,3]    Window [3,-1,-3] → max = nums[1] = 3
i=4, val=5:  Remove index 1 (outside window). Pop 3,2 (both<5).
             Deque: [4]        Window [-1,-3,5] → max = nums[4] = 5
i=5, val=3:  Deque: [4,5]      Window [-3,5,3] → max = nums[4] = 5
i=6, val=6:  Pop 5 (3<6), pop 4 (5<6). Deque: [6]
             Window [5,3,6] → max = nums[6] = 6
i=7, val=7:  Pop 6 (6<7). Deque: [7]
             Window [3,6,7] → max = nums[7] = 7

Result: [3, 3, 5, 5, 6, 7] ✓
```

---

## 6. When to Use What

| Pattern | Data Structure | Key Signal in Problem |
|---------|---------------|----------------------|
| Next Greater Element | Monotonic Decreasing Stack | "find the next larger value to the right" |
| Next Smaller Element | Monotonic Increasing Stack | "find the next smaller value to the right" |
| Largest Rectangle | Monotonic Increasing Stack | "maximize area under constraints" |
| Sliding Window Max/Min | Monotonic Deque | "find max/min in every window of size K" |
| Sum of Subarray Minimums | Monotonic Stack + contribution | "sum of min of all subarrays" |

---

## 7. Practice Problems

| Problem | Pattern | Difficulty |
|---------|---------|------------|
| Next Greater Element I | Monotonic Stack + HashMap | Easy |
| Next Greater Element II (Circular) | Stack + 2N loop | Medium |
| Daily Temperatures | Next Greater (index diff) | Medium |
| Largest Rectangle in Histogram | Increasing Stack | Hard |
| Maximal Rectangle (2D) | Histogram per row | Hard |
| Sliding Window Maximum | Monotonic Deque | Hard |
| Sum of Subarray Minimums | Stack + contribution counting | Medium |
| Trapping Rain Water | Two approaches: Stack or Two-Pointer | Hard |
