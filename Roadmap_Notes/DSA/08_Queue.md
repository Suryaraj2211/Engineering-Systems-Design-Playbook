# Chapter 08 — Queue

## Simple Explanation (ELI5)
A queue is a line at a ticket counter — the first person who arrives is the first one served. **FIFO: First In, First Out.**

## Technical Definition
A queue is an abstract data type where elements are added at the rear (enqueue) and removed from the front (dequeue), both in O(1).

## Real-world Analogy
Movie ticket line, printer queue, YouTube buffering — everything processes in the order it arrived.

---

## 1. Operations & Complexity

| Operation | Description | Time |
|-----------|-------------|------|
| `enqueue(x)` | Add to rear | O(1) |
| `dequeue()` | Remove from front | O(1) |
| `front()/peek()` | View front element | O(1) |
| `isEmpty()` | Check if empty | O(1) |

---

## 2. Implementations

### Efficient Queue (Object-based)
```javascript
class Queue {
  constructor() { this.items = {}; this.head = 0; this.tail = 0; }
  enqueue(x) { this.items[this.tail] = x; this.tail++; }
  dequeue() {
    if (this.isEmpty()) return null;
    let val = this.items[this.head];
    delete this.items[this.head]; this.head++;
    return val;
  }
  front() { return this.items[this.head]; }
  isEmpty() { return this.tail === this.head; }
  size() { return this.tail - this.head; }
}
// All O(1) — no shifting!
```

```python
from collections import deque
q = deque()
q.append(1)    # enqueue
q.popleft()    # dequeue — O(1)
```

> **Warning:** Using array `shift()` is O(n)! Use object-based queue or `deque`.

---

## 3. Circular Queue

Fixed-size array where rear wraps around to the front.

```javascript
class CircularQueue {
  constructor(k) {
    this.arr = new Array(k);
    this.cap = k; this.front = 0; this.rear = -1; this.size = 0;
  }
  enqueue(x) {
    if (this.isFull()) return false;
    this.rear = (this.rear + 1) % this.cap;
    this.arr[this.rear] = x;
    this.size++;
    return true;
  }
  dequeue() {
    if (this.isEmpty()) return null;
    let val = this.arr[this.front];
    this.front = (this.front + 1) % this.cap;
    this.size--;
    return val;
  }
  isEmpty() { return this.size === 0; }
  isFull() { return this.size === this.cap; }
}
```

---

## 4. Deque (Double-Ended Queue)

Add/remove from **both ends** in O(1).

```javascript
class Deque {
  constructor() { this.items = {}; this.head = 0; this.tail = -1; }
  addFront(x) { this.head--; this.items[this.head] = x; }
  addRear(x) { this.tail++; this.items[this.tail] = x; }
  removeFront() { let v = this.items[this.head]; delete this.items[this.head]; this.head++; return v; }
  removeRear() { let v = this.items[this.tail]; delete this.items[this.tail]; this.tail--; return v; }
  isEmpty() { return this.tail < this.head; }
}
```

```python
from collections import deque
d = deque()
d.appendleft(1)  # add front
d.append(2)      # add rear
d.popleft()      # remove front
d.pop()          # remove rear
# All O(1)
```

---

## 5. Queue Applications

### BFS Traversal (Level Order)
```javascript
function bfs(graph, start) {
  let visited = new Set([start]);
  let queue = [start], result = [];
  while (queue.length) {
    let node = queue.shift();
    result.push(node);
    for (let neighbor of graph[node]) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push(neighbor);
      }
    }
  }
  return result;
}
```

### Rotting Oranges (Multi-source BFS)
```javascript
function orangesRotting(grid) {
  let queue = [], fresh = 0;
  let rows = grid.length, cols = grid[0].length;

  for (let r = 0; r < rows; r++)
    for (let c = 0; c < cols; c++) {
      if (grid[r][c] === 2) queue.push([r, c, 0]);
      if (grid[r][c] === 1) fresh++;
    }

  let maxTime = 0;
  let dirs = [[1,0],[-1,0],[0,1],[0,-1]];

  while (queue.length) {
    let [r, c, time] = queue.shift();
    for (let [dr, dc] of dirs) {
      let nr = r + dr, nc = c + dc;
      if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] === 1) {
        grid[nr][nc] = 2;
        fresh--;
        queue.push([nr, nc, time + 1]);
        maxTime = Math.max(maxTime, time + 1);
      }
    }
  }
  return fresh === 0 ? maxTime : -1;
}
```

### Implement Stack Using Queues & vice versa

(See Chapter 07 for cross-implementations)

---

## 6. Common Mistakes
- Using `Array.shift()` → O(n) per call. Use object-based queue or deque.
- Confusing FIFO (queue) with LIFO (stack)
- BFS with stack instead of queue → that's DFS!

## 7. When NOT to Use
- Need LIFO → Stack
- Need by-priority access → Priority Queue / Heap

---

## 8. Practice Problems

| Problem | Type | Difficulty |
|---------|------|-----------|
| Implement Queue using Stacks | Design | Easy |
| Design Circular Queue | Design | Medium |
| Rotting Oranges | BFS | Medium |
| Walls and Gates | BFS | Medium |
| Sliding Window Maximum | Deque | Hard |
| Number of Islands | BFS/DFS | Medium |
| First Non-Repeating Char Stream | Queue + Hash | Medium |
| Task Scheduler | Queue + Greedy | Medium |
