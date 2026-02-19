# Chapter 14 — Heaps & Priority Queues

## The Problem This Solves
You need to repeatedly find the minimum (or maximum) element from a dynamic collection. A sorted array gives O(1) min but O(n) insert. An unsorted array gives O(1) insert but O(n) min. A Heap gives **O(log n) for both** — the optimal tradeoff.

---

## 1. What Is a Heap?

A Heap is a **complete binary tree** stored in a **flat array** where every parent satisfies the heap property:
- **Min-Heap:** Parent <= both children (root = smallest)
- **Max-Heap:** Parent >= both children (root = largest)

```
MIN-HEAP:                        ARRAY LAYOUT:
       1                         Index:  0  1  2  3  4  5  6
      / \                        Value: [1, 3, 5, 7, 8, 9, 6]
     3   5
    / \ / \                      Parent of i: Math.floor((i - 1) / 2)
   7  8 9  6                     Left child:  2 * i + 1
                                 Right child: 2 * i + 2
```

**Why an array?** No pointers needed. A binary tree with N nodes needs N pointer pairs (2N extra space). An array needs zero extra space. The parent/child relationships are pure math.

---

## 2. Core Operations — Step by Step

### Insert (Bubble Up / Sift Up)
Add element to the end, then swap upward until heap property is restored.

```javascript
insert(val) {
    this.heap.push(val);
    let i = this.heap.length - 1;
    while (i > 0) {
        let parent = Math.floor((i - 1) / 2);
        if (this.heap[parent] <= this.heap[i]) break; // Min-heap satisfied
        [this.heap[parent], this.heap[i]] = [this.heap[i], this.heap[parent]];
        i = parent;
    }
}
```

**Manual Walkthrough — Insert 2 into [1, 3, 5, 7, 8, 9, 6]:**
```
Step 1: Push 2 → [1, 3, 5, 7, 8, 9, 6, 2]   (index 7)
Step 2: Parent of 7 = index 3 (value 7). 2 < 7? YES → swap
        [1, 3, 5, 2, 8, 9, 6, 7]
Step 3: Parent of 3 = index 1 (value 3). 2 < 3? YES → swap
        [1, 2, 5, 3, 8, 9, 6, 7]
Step 4: Parent of 1 = index 0 (value 1). 2 < 1? NO → STOP
Result: [1, 2, 5, 3, 8, 9, 6, 7]  ✓ valid min-heap
```

### Extract Min (Bubble Down / Sift Down)
Remove root, move last element to root, then swap downward with the smaller child.

```javascript
extractMin() {
    if (this.heap.length === 0) return null;
    const min = this.heap[0];
    const last = this.heap.pop();
    if (this.heap.length > 0) {
        this.heap[0] = last;
        this._siftDown(0);
    }
    return min;
}

_siftDown(i) {
    const n = this.heap.length;
    while (true) {
        let smallest = i;
        let left = 2 * i + 1;
        let right = 2 * i + 2;
        if (left < n && this.heap[left] < this.heap[smallest]) smallest = left;
        if (right < n && this.heap[right] < this.heap[smallest]) smallest = right;
        if (smallest === i) break;
        [this.heap[i], this.heap[smallest]] = [this.heap[smallest], this.heap[i]];
        i = smallest;
    }
}
```

**Manual Walkthrough — Extract Min from [1, 2, 5, 3, 8, 9, 6, 7]:**
```
Step 1: Save root (1). Move last element (7) to root.
        [7, 2, 5, 3, 8, 9, 6]
Step 2: Children of 0: left=2, right=5. Smallest child = 2 (index 1). 7 > 2? YES → swap
        [2, 7, 5, 3, 8, 9, 6]
Step 3: Children of 1: left=3, right=8. Smallest child = 3 (index 3). 7 > 3? YES → swap
        [2, 3, 5, 7, 8, 9, 6]
Step 4: Children of 3: left=index 7 (out of bounds) → STOP
Result: [2, 3, 5, 7, 8, 9, 6]  ✓ valid min-heap. Returned 1.
```

---

## 3. Heapify — Build a Heap in O(n)

**Naive approach:** Insert N elements one by one = O(n log n).
**Optimal approach:** Start from the last parent and sift down = **O(n)**.

```javascript
function buildHeap(arr) {
    // Start from the last non-leaf node and sift down each
    for (let i = Math.floor(arr.length / 2) - 1; i >= 0; i--) {
        siftDown(arr, i, arr.length);
    }
}
```

**Why O(n) and not O(n log n)?** Most nodes are near the bottom of the tree and only sift down 1-2 levels. The sum of work across all levels is a geometric series that converges to O(n).

---

## 4. Heap Sort

Uses a max-heap to sort in ascending order:
1. Build a max-heap from the array — O(n)
2. Repeatedly extract the max and place it at the end — O(n log n)

```javascript
function heapSort(arr) {
    const n = arr.length;
    // Build max-heap
    for (let i = Math.floor(n / 2) - 1; i >= 0; i--) {
        siftDown(arr, i, n);
    }
    // Extract elements one by one
    for (let end = n - 1; end > 0; end--) {
        [arr[0], arr[end]] = [arr[end], arr[0]]; // Move max to end
        siftDown(arr, 0, end);                     // Restore heap on reduced array
    }
}

function siftDown(arr, i, size) {
    while (true) {
        let largest = i, left = 2*i+1, right = 2*i+2;
        if (left < size && arr[left] > arr[largest]) largest = left;
        if (right < size && arr[right] > arr[largest]) largest = right;
        if (largest === i) break;
        [arr[i], arr[largest]] = [arr[largest], arr[i]];
        i = largest;
    }
}
```

**Properties:** O(n log n) guaranteed (no worst-case degradation like QuickSort). In-place (no extra memory). Not stable.

---

## 5. Priority Queue (Application of Heap)

A priority queue is an abstract data structure where each element has a priority. The highest-priority element is always dequeued first. Heaps are the standard implementation.

```javascript
class PriorityQueue {
    constructor(comparator = (a, b) => a - b) {
        this.heap = [];
        this.cmp = comparator;
    }
    
    enqueue(val) { this.heap.push(val); this._bubbleUp(this.heap.length - 1); }
    dequeue()    { return this._extract(); }
    peek()       { return this.heap[0]; }
    get size()   { return this.heap.length; }
    
    // ... (bubbleUp and siftDown as shown above, using this.cmp for comparison)
}

// Usage: Dijkstra's shortest path, task scheduling, merge K sorted lists
const pq = new PriorityQueue((a, b) => a.distance - b.distance);
pq.enqueue({ node: "A", distance: 5 });
pq.enqueue({ node: "B", distance: 2 });
pq.dequeue(); // { node: "B", distance: 2 } — smallest distance first
```

---

## 6. Complexity Reference

| Operation | Time | Notes |
|-----------|------|-------|
| Insert | O(log n) | Bubble up at most log n levels |
| Extract Min/Max | O(log n) | Sift down at most log n levels |
| Peek (get min/max) | O(1) | Root element |
| Build Heap | O(n) | Bottom-up heapify |
| Heap Sort | O(n log n) | In-place, not stable |
| Decrease Key | O(log n) | Update value + bubble up |

---

## 7. Practice Problems

| Problem | Concept | Difficulty |
|---------|---------|------------|
| Kth Largest Element | Min-heap of size K | Medium |
| Merge K Sorted Lists | Min-heap of K heads | Hard |
| Top K Frequent Elements | Heap + HashMap | Medium |
| Find Median from Data Stream | Two heaps (max + min) | Hard |
| Task Scheduler | Max-heap + cooldown | Medium |
| Reorganize String | Max-heap greedy | Medium |
