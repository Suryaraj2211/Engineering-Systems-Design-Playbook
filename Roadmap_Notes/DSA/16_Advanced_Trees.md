# Chapter 16 — Advanced Trees (AVL, Red-Black, Segment, Fenwick)

## 1. AVL Tree (Self-Balancing BST)

### Why?
BST degrades to O(n) with sorted insertions. AVL trees maintain **balance factor** (-1, 0, 1) via rotations, guaranteeing O(log n).

### Balance Factor = height(left) - height(right)
```
Unbalanced (right-heavy):   After Left Rotation:
    10 (bf=-2)                   20
      \                         /  \
       20 (bf=-1)             10    30
         \
          30
```

### Four Rotation Types
```
LL → Right Rotate
RR → Left Rotate
LR → Left Rotate child, then Right Rotate
RL → Right Rotate child, then Left Rotate
```

### Right Rotation (LL case)
```javascript
function rightRotate(y) {
  let x = y.left;
  let T2 = x.right;
  x.right = y;
  y.left = T2;
  // Update heights
  y.height = 1 + Math.max(getHeight(y.left), getHeight(y.right));
  x.height = 1 + Math.max(getHeight(x.left), getHeight(x.right));
  return x;
}
```

---

## 2. Red-Black Tree (Concept)

Used in Java TreeMap, C++ std::map, Linux kernel.

### Rules:
1. Every node is Red or Black
2. Root is always Black
3. No two consecutive Red nodes
4. Every path from root to null has same number of Black nodes

**Guarantees O(log n)** for all operations with fewer rotations than AVL.

---

## 3. Segment Tree (Range Queries + Point Updates)

### Why?
| Operation | Array | Prefix Sum | Segment Tree |
|-----------|-------|-----------|-------------|
| Range Query | O(n) | O(1) | O(log n) |
| Point Update | O(1) | O(n) | O(log n) |

Segment Tree is best when you need **both** fast queries AND fast updates.

```javascript
class SegmentTree {
  constructor(arr) {
    this.n = arr.length;
    this.tree = new Array(4 * this.n).fill(0);
    this._build(arr, 0, 0, this.n - 1);
  }

  _build(arr, node, start, end) {
    if (start === end) { this.tree[node] = arr[start]; return; }
    let mid = Math.floor((start + end) / 2);
    this._build(arr, 2*node+1, start, mid);
    this._build(arr, 2*node+2, mid+1, end);
    this.tree[node] = this.tree[2*node+1] + this.tree[2*node+2];
  }

  query(l, r, node = 0, start = 0, end = this.n - 1) {
    if (r < start || end < l) return 0;
    if (l <= start && end <= r) return this.tree[node];
    let mid = Math.floor((start + end) / 2);
    return this.query(l, r, 2*node+1, start, mid) +
           this.query(l, r, 2*node+2, mid+1, end);
  }

  update(idx, val, node = 0, start = 0, end = this.n - 1) {
    if (start === end) { this.tree[node] = val; return; }
    let mid = Math.floor((start + end) / 2);
    if (idx <= mid) this.update(idx, val, 2*node+1, start, mid);
    else this.update(idx, val, 2*node+2, mid+1, end);
    this.tree[node] = this.tree[2*node+1] + this.tree[2*node+2];
  }
}

let st = new SegmentTree([1, 3, 5, 7, 9, 11]);
console.log(st.query(1, 3));  // 15 (3+5+7)
st.update(2, 10);
console.log(st.query(1, 3));  // 20 (3+10+7)
```

---

## 4. Fenwick Tree (Binary Indexed Tree — BIT)

Simpler and more memory-efficient than Segment Tree. Supports **prefix sum queries** and **point updates** in O(log n).

```javascript
class FenwickTree {
  constructor(n) { this.tree = new Array(n + 1).fill(0); this.n = n; }

  update(i, delta) {
    for (i++; i <= this.n; i += i & (-i))
      this.tree[i] += delta;
  }

  prefixSum(i) {
    let sum = 0;
    for (i++; i > 0; i -= i & (-i))
      sum += this.tree[i];
    return sum;
  }

  rangeSum(l, r) {
    return this.prefixSum(r) - (l > 0 ? this.prefixSum(l - 1) : 0);
  }
}
```

```python
class FenwickTree:
    def __init__(self, n):
        self.tree = [0] * (n + 1)
        self.n = n

    def update(self, i, delta):
        i += 1
        while i <= self.n:
            self.tree[i] += delta
            i += i & (-i)

    def prefix_sum(self, i):
        i += 1
        s = 0
        while i > 0:
            s += self.tree[i]
            i -= i & (-i)
        return s
```

### Key Trick: `i & (-i)` gives the **lowest set bit**
```
12 = 1100 → lowest bit = 0100 = 4
10 = 1010 → lowest bit = 0010 = 2
```

---

## 5. Comparison

| Feature | BST | AVL | Red-Black | Segment | Fenwick |
|---------|-----|-----|-----------|---------|---------|
| Search | O(log n)* | O(log n) | O(log n) | — | — |
| Insert | O(log n)* | O(log n) | O(log n) | — | — |
| Range Query | — | — | — | O(log n) | O(log n) |
| Point Update | — | — | — | O(log n) | O(log n) |
| Space | O(n) | O(n) | O(n) | O(4n) | O(n) |

\* Average case; worst case O(n) for unbalanced BST

---

## 6. Practice Problems

| Problem | Structure | Difficulty |
|---------|----------|-----------|
| Range Sum Query Mutable | Segment/Fenwick | Medium |
| Count Smaller After Self | Segment/Fenwick | Hard |
| Range Sum Query 2D | 2D Fenwick | Hard |
| Count Inversions | Merge Sort / Fenwick | Hard |
