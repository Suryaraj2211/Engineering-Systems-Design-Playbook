# Chapter 13 — Binary Search Tree (BST)

## Simple Explanation (ELI5)
A BST is a binary tree with a strict rule: for every node, **all values in the left subtree are smaller** and **all values in the right subtree are larger**. This makes searching as fast as binary search — O(log n).

---

## 1. BST Property
```
        8
       / \
      3   10
     / \    \
    1   6    14
       / \   /
      4   7 13

For node 8: left subtree {1,3,4,6,7} < 8 < right subtree {10,13,14} ✅
Inorder traversal → sorted: [1, 3, 4, 6, 7, 8, 10, 13, 14]
```

---

## 2. BST Operations

```javascript
class BST {
  constructor() { this.root = null; }

  // Insert: O(log n) avg, O(n) worst
  insert(val) {
    this.root = this._insert(this.root, val);
  }
  _insert(node, val) {
    if (!node) return new TreeNode(val);
    if (val < node.val) node.left = this._insert(node.left, val);
    else node.right = this._insert(node.right, val);
    return node;
  }

  // Search: O(log n) avg
  search(val) {
    let curr = this.root;
    while (curr) {
      if (val === curr.val) return curr;
      curr = val < curr.val ? curr.left : curr.right;
    }
    return null;
  }

  // Delete: O(log n) avg
  delete(val) {
    this.root = this._delete(this.root, val);
  }
  _delete(node, val) {
    if (!node) return null;
    if (val < node.val) node.left = this._delete(node.left, val);
    else if (val > node.val) node.right = this._delete(node.right, val);
    else {
      if (!node.left) return node.right;       // 0 or 1 child
      if (!node.right) return node.left;        // 1 child
      let succ = node.right;                    // 2 children: inorder successor
      while (succ.left) succ = succ.left;
      node.val = succ.val;
      node.right = this._delete(node.right, succ.val);
    }
    return node;
  }

  // Inorder → sorted output
  inorder(node = this.root, result = []) {
    if (!node) return result;
    this.inorder(node.left, result);
    result.push(node.val);
    this.inorder(node.right, result);
    return result;
  }
}
```

```python
class BST:
    def __init__(self):
        self.root = None

    def insert(self, val):
        self.root = self._insert(self.root, val)

    def _insert(self, node, val):
        if not node: return TreeNode(val)
        if val < node.val: node.left = self._insert(node.left, val)
        else: node.right = self._insert(node.right, val)
        return node

    def search(self, val):
        curr = self.root
        while curr:
            if val == curr.val: return curr
            curr = curr.left if val < curr.val else curr.right
        return None
```

---

## 3. Validate BST
```javascript
function isValidBST(root, min = -Infinity, max = Infinity) {
  if (!root) return true;
  if (root.val <= min || root.val >= max) return false;
  return isValidBST(root.left, min, root.val) &&
         isValidBST(root.right, root.val, max);
}
```

---

## 4. Classic BST Problems

### Kth Smallest Element
```javascript
function kthSmallest(root, k) {
  let stack = [], curr = root;
  while (curr || stack.length) {
    while (curr) { stack.push(curr); curr = curr.left; }
    curr = stack.pop();
    k--;
    if (k === 0) return curr.val;
    curr = curr.right;
  }
}
```

### LCA in BST — O(log n)
```javascript
function lcaBST(root, p, q) {
  while (root) {
    if (p.val < root.val && q.val < root.val) root = root.left;
    else if (p.val > root.val && q.val > root.val) root = root.right;
    else return root;
  }
}
```

### Floor & Ceil in BST
```javascript
function floor(root, key) {
  let result = null;
  while (root) {
    if (root.val === key) return root.val;
    if (root.val < key) { result = root.val; root = root.right; }
    else root = root.left;
  }
  return result;
}

function ceil(root, key) {
  let result = null;
  while (root) {
    if (root.val === key) return root.val;
    if (root.val > key) { result = root.val; root = root.left; }
    else root = root.right;
  }
  return result;
}
```

---

## 5. BST Complexity

| Operation | Average | Worst (skewed) |
|-----------|---------|----------------|
| Search | O(log n) | O(n) |
| Insert | O(log n) | O(n) |
| Delete | O(log n) | O(n) |

Worst case = inserting sorted data → becomes a linked list. Fix with **balanced BSTs** (Chapter 16).

---

## 6. Practice Problems

| Problem | Concept | Difficulty |
|---------|---------|-----------|
| Validate BST | Range check | Medium |
| Kth Smallest in BST | Inorder | Medium |
| LCA of BST | BST property | Medium |
| Convert Sorted Array to BST | Divide & Conquer | Easy |
| Delete Node in BST | Successor logic | Medium |
| BST Iterator | Stack | Medium |
| Recover BST (swapped nodes) | Inorder | Medium |
