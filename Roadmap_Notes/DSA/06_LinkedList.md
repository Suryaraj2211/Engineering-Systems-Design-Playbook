# Chapter 06 — Linked List

## Simple Explanation (ELI5)
Imagine a treasure hunt where each clue tells you where the next clue is. You can't jump to clue #5 directly — you must follow clue #1 → #2 → #3 → #4 → #5. That's a Linked List — each element knows only about the next one.

## Technical Definition
A linked list is a linear data structure where elements (nodes) are stored in **non-contiguous memory**, connected by **pointers**. Each node contains data and a reference to the next node.

## Why It's Needed
- O(1) insertion/deletion at the beginning (vs O(n) for arrays)
- Dynamic size — no wasted memory from pre-allocation
- Foundation for stacks, queues, hash tables (chaining), graphs

## Real-world Analogy
A **conga line** — each person holds the shoulders of the person in front. To add someone at the beginning, they just grab the front person. No one else needs to move.

---

## 1. Singly Linked List (SLL)

Each node has `data` + `next` pointer.

```
Head → [10|→] → [20|→] → [30|→] → [40|null]
```

### Node Structure
```javascript
class Node {
  constructor(value) {
    this.value = value;
    this.next = null;  // Pointer to next node
  }
}
```

```python
class Node:
    def __init__(self, value):
        self.value = value
        self.next = None
```

### Full Implementation

```javascript
class SinglyLinkedList {
  constructor() {
    this.head = null;
    this.size = 0;
  }

  // Prepend: O(1)
  prepend(value) {
    let node = new Node(value);
    node.next = this.head;
    this.head = node;
    this.size++;
  }

  // Append: O(n) — must traverse to end
  append(value) {
    let node = new Node(value);
    if (!this.head) { this.head = node; this.size++; return; }
    let curr = this.head;
    while (curr.next) curr = curr.next;
    curr.next = node;
    this.size++;
  }

  // Delete by value: O(n)
  delete(value) {
    if (!this.head) return false;
    if (this.head.value === value) {
      this.head = this.head.next;
      this.size--;
      return true;
    }
    let curr = this.head;
    while (curr.next) {
      if (curr.next.value === value) {
        curr.next = curr.next.next;
        this.size--;
        return true;
      }
      curr = curr.next;
    }
    return false;
  }

  // Search: O(n)
  search(value) {
    let curr = this.head;
    while (curr) {
      if (curr.value === value) return true;
      curr = curr.next;
    }
    return false;
  }

  // Print
  print() {
    let parts = [];
    let curr = this.head;
    while (curr) { parts.push(curr.value); curr = curr.next; }
    console.log(parts.join(" → ") + " → null");
  }
}
```

### Dry Run: Prepend(5) on [10 → 20 → null]
```
Before: Head → [10|→] → [20|null]

Step 1: Create node [5|null]
Step 2: node.next = head  →  [5|→] → [10|→] → [20|null]
Step 3: head = node

After: Head → [5|→] → [10|→] → [20|null]
```

---

## 2. Doubly Linked List (DLL)

Each node has `prev` + `data` + `next`. Allows traversal in **both directions**.

```
null ← [10] ⇄ [20] ⇄ [30] ⇄ [40] → null
        Head                    Tail
```

```javascript
class DoublyNode {
  constructor(value) {
    this.value = value;
    this.next = null;
    this.prev = null;
  }
}

class DoublyLinkedList {
  constructor() { this.head = null; this.tail = null; this.size = 0; }

  append(value) {
    let node = new DoublyNode(value);
    if (!this.head) { this.head = this.tail = node; }
    else {
      node.prev = this.tail;
      this.tail.next = node;
      this.tail = node;
    }
    this.size++;
  }

  prepend(value) {
    let node = new DoublyNode(value);
    if (!this.head) { this.head = this.tail = node; }
    else {
      node.next = this.head;
      this.head.prev = node;
      this.head = node;
    }
    this.size++;
  }

  // Delete: O(1) if you have the node reference!
  deleteNode(node) {
    if (node.prev) node.prev.next = node.next;
    else this.head = node.next;
    if (node.next) node.next.prev = node.prev;
    else this.tail = node.prev;
    this.size--;
  }
}
```

---

## 3. Circular Linked List

Last node points back to the head (forms a cycle).

```
[10] → [20] → [30] → [10] → ... (loops)
```

Used in: Round-robin scheduling, circular buffers, multiplayer game turns.

---

## 4. Array vs Linked List

| Feature | Array | Linked List |
|---------|-------|-------------|
| Access by index | O(1) ✅ | O(n) ❌ |
| Insert at start | O(n) | O(1) ✅ |
| Insert at end | O(1)* | O(1) with tail ✅ |
| Delete at start | O(n) | O(1) ✅ |
| Memory | Contiguous | Scattered |
| Cache performance | Excellent | Poor |
| Extra memory | None | 1-2 pointers per node |

---

## 5. Classic Problems

### Reverse Linked List — O(n) Time, O(1) Space

```javascript
function reverseList(head) {
  let prev = null, curr = head;
  while (curr) {
    let next = curr.next;
    curr.next = prev;
    prev = curr;
    curr = next;
  }
  return prev;
}
```

```python
def reverse_list(head):
    prev, curr = None, head
    while curr:
        curr.next, prev, curr = prev, curr, curr.next
    return prev
```

### Detect Cycle — Floyd's Tortoise & Hare

```javascript
function hasCycle(head) {
  let slow = head, fast = head;
  while (fast && fast.next) {
    slow = slow.next;
    fast = fast.next.next;
    if (slow === fast) return true;
  }
  return false;
}
```

### Find Middle Node

```javascript
function middleNode(head) {
  let slow = head, fast = head;
  while (fast && fast.next) {
    slow = slow.next;
    fast = fast.next.next;
  }
  return slow;
}
```

### Merge Two Sorted Lists

```javascript
function mergeTwoLists(l1, l2) {
  let dummy = new Node(0), curr = dummy;
  while (l1 && l2) {
    if (l1.value <= l2.value) { curr.next = l1; l1 = l1.next; }
    else { curr.next = l2; l2 = l2.next; }
    curr = curr.next;
  }
  curr.next = l1 || l2;
  return dummy.next;
}
```

### Remove Nth from End (Two Pointer Gap)

```javascript
function removeNthFromEnd(head, n) {
  let dummy = new Node(0);
  dummy.next = head;
  let fast = dummy, slow = dummy;
  for (let i = 0; i <= n; i++) fast = fast.next;
  while (fast) { slow = slow.next; fast = fast.next; }
  slow.next = slow.next.next;
  return dummy.next;
}
```

### LRU Cache (Doubly Linked List + HashMap)

```javascript
class LRUCache {
  constructor(capacity) {
    this.cap = capacity;
    this.map = new Map();  // key → node
    this.head = new DoublyNode(0); // dummy head
    this.tail = new DoublyNode(0); // dummy tail
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  _remove(node) {
    node.prev.next = node.next;
    node.next.prev = node.prev;
  }

  _addToFront(node) {
    node.next = this.head.next;
    node.prev = this.head;
    this.head.next.prev = node;
    this.head.next = node;
  }

  get(key) {
    if (!this.map.has(key)) return -1;
    let node = this.map.get(key);
    this._remove(node);
    this._addToFront(node);
    return node.val;
  }

  put(key, value) {
    if (this.map.has(key)) this._remove(this.map.get(key));
    let node = new DoublyNode(0);
    node.key = key; node.val = value;
    this._addToFront(node);
    this.map.set(key, node);
    if (this.map.size > this.cap) {
      let lru = this.tail.prev;
      this._remove(lru);
      this.map.delete(lru.key);
    }
  }
}
```

---

## 6. Common Mistakes
- Forgetting to handle `null` head
- Losing reference to next node during reversal
- Not using dummy node for edge cases (delete head, merge)
- Cycle in modifications → infinite loop

## 7. Edge Cases
- Empty list, single node, two nodes
- Deleting the head node
- Cycle present (infinite traversal if not detected)

## 8. When NOT to Use
- Need random access by index → Array
- Cache-friendly traversal needed → Array
- Simple stack/queue → Array-based is fine

---

## 9. Practice Problems

| Problem | Technique | Difficulty |
|---------|-----------|-----------|
| Reverse Linked List | Iterative/Recursive | Easy |
| Middle of Linked List | Slow/Fast | Easy |
| Linked List Cycle | Floyd's | Easy |
| Merge Two Sorted Lists | Two pointers | Easy |
| Remove Nth From End | Two pointer gap | Medium |
| Add Two Numbers | Carry simulation | Medium |
| Palindrome Linked List | Reverse half + compare | Medium |
| Reorder List | Find mid + reverse + merge | Medium |
| LRU Cache | DLL + HashMap | Medium |
| Merge K Sorted Lists | Min Heap / D&C | Hard |
| Reverse Nodes in K-Group | Iterative reversal | Hard |
