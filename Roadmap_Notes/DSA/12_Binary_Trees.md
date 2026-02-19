# Chapter 12 — Binary Trees

## Simple Explanation (ELI5)
A tree is like an upside-down family tree. One person at the top (root), each person has at most 2 children (left and right), and people at the bottom have no children (leaves).

## Technical Definition
A binary tree is a hierarchical data structure where each node has at most **two children** (left, right). Traversal follows DFS (Inorder/Preorder/Postorder) or BFS (Level Order).

---

## 1. Node Structure
```javascript
class TreeNode {
  constructor(val) {
    this.val = val;
    this.left = null;
    this.right = null;
  }
}
```

```python
class TreeNode:
    def __init__(self, val=0):
        self.val = val
        self.left = None
        self.right = None
```

---

## 2. Traversals ⭐

```javascript
// INORDER: Left → Root → Right
function inorder(root, result = []) {
  if (!root) return result;
  inorder(root.left, result);
  result.push(root.val);
  inorder(root.right, result);
  return result;
}

// PREORDER: Root → Left → Right
function preorder(root, result = []) {
  if (!root) return result;
  result.push(root.val);
  preorder(root.left, result);
  preorder(root.right, result);
  return result;
}

// POSTORDER: Left → Right → Root
function postorder(root, result = []) {
  if (!root) return result;
  postorder(root.left, result);
  postorder(root.right, result);
  result.push(root.val);
  return result;
}

// LEVEL ORDER (BFS)
function levelOrder(root) {
  if (!root) return [];
  let result = [], queue = [root];
  while (queue.length) {
    let level = [], size = queue.length;
    for (let i = 0; i < size; i++) {
      let node = queue.shift();
      level.push(node.val);
      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }
    result.push(level);
  }
  return result;
}
```

### Iterative Inorder (using stack)
```javascript
function inorderIterative(root) {
  let result = [], stack = [], curr = root;
  while (curr || stack.length) {
    while (curr) { stack.push(curr); curr = curr.left; }
    curr = stack.pop();
    result.push(curr.val);
    curr = curr.right;
  }
  return result;
}
```

---

## 3. Classic Problems

### Max Depth
```javascript
function maxDepth(root) {
  if (!root) return 0;
  return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}
```

### Invert Tree
```javascript
function invertTree(root) {
  if (!root) return null;
  [root.left, root.right] = [root.right, root.left];
  invertTree(root.left);
  invertTree(root.right);
  return root;
}
```

### Symmetric Tree
```javascript
function isSymmetric(root) {
  function mirror(a, b) {
    if (!a && !b) return true;
    if (!a || !b) return false;
    return a.val === b.val && mirror(a.left, b.right) && mirror(a.right, b.left);
  }
  return !root || mirror(root.left, root.right);
}
```

### Diameter
```javascript
function diameter(root) {
  let max = 0;
  function height(node) {
    if (!node) return 0;
    let l = height(node.left), r = height(node.right);
    max = Math.max(max, l + r);
    return 1 + Math.max(l, r);
  }
  height(root);
  return max;
}
```

### Path Sum
```javascript
function hasPathSum(root, sum) {
  if (!root) return false;
  if (!root.left && !root.right) return sum === root.val;
  return hasPathSum(root.left, sum - root.val) || hasPathSum(root.right, sum - root.val);
}
```

### Lowest Common Ancestor
```javascript
function lca(root, p, q) {
  if (!root || root === p || root === q) return root;
  let left = lca(root.left, p, q);
  let right = lca(root.right, p, q);
  if (left && right) return root;
  return left || right;
}
```

### Construct from Preorder + Inorder
```javascript
function buildTree(preorder, inorder) {
  if (!preorder.length) return null;
  let root = new TreeNode(preorder[0]);
  let mid = inorder.indexOf(preorder[0]);
  root.left = buildTree(preorder.slice(1, mid + 1), inorder.slice(0, mid));
  root.right = buildTree(preorder.slice(mid + 1), inorder.slice(mid + 1));
  return root;
}
```

### Serialize / Deserialize
```javascript
function serialize(root) {
  if (!root) return "null";
  return root.val + "," + serialize(root.left) + "," + serialize(root.right);
}

function deserialize(data) {
  let vals = data.split(","), i = 0;
  function build() {
    if (vals[i] === "null") { i++; return null; }
    let node = new TreeNode(parseInt(vals[i++]));
    node.left = build();
    node.right = build();
    return node;
  }
  return build();
}
```

---

## 4. Practice Problems

| Problem | Concept | Difficulty |
|---------|---------|-----------|
| Max Depth | DFS | Easy |
| Invert Tree | DFS | Easy |
| Symmetric Tree | DFS | Easy |
| Same Tree | DFS | Easy |
| Path Sum | DFS | Easy |
| Level Order | BFS | Medium |
| Diameter | DFS | Easy |
| LCA | DFS | Medium |
| Right Side View | BFS | Medium |
| Build from Pre+In | Divide & Conquer | Medium |
| Binary Tree Max Path Sum | DFS | Hard |
| Serialize/Deserialize | DFS/BFS | Hard |
