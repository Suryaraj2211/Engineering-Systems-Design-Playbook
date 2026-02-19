# Chapter 17 — Graphs: Basics

## Simple Explanation (ELI5)
A graph is like a social network — people (nodes) connected by friendships (edges). Unlike trees, graphs can have cycles, multiple paths, and no single "root."

## Technical Definition
A graph G = (V, E) consists of vertices V and edges E. Edges can be directed/undirected, weighted/unweighted.

---

## 1. Graph Types & Terminology

| Term | Meaning |
|------|---------|
| Vertex/Node | A point in the graph |
| Edge | Connection between two vertices |
| Directed | Edges have direction (A → B) |
| Undirected | Edges go both ways (A — B) |
| Weighted | Edges have cost/distance |
| Degree | Number of edges at a node (in-degree, out-degree for directed) |
| Path | Sequence of vertices connected by edges |
| Cycle | Path that starts and ends at same vertex |
| DAG | Directed Acyclic Graph |
| Connected | Every node reachable from every other (undirected) |

---

## 2. Representations

### Adjacency List (Most common)
```javascript
class Graph {
  constructor() { this.adj = new Map(); }
  addVertex(v) { if (!this.adj.has(v)) this.adj.set(v, []); }
  addEdge(u, v) { this.adj.get(u).push(v); this.adj.get(v).push(u); }
}
```

```python
from collections import defaultdict
graph = defaultdict(list)
graph[0].append(1)
graph[1].append(0)  # undirected
```

### Adjacency Matrix
```javascript
// matrix[i][j] = 1 if edge exists
let matrix = [[0,1,1,0],[1,0,0,1],[1,0,0,1],[0,1,1,0]];
```

| | Adj List | Adj Matrix |
|--|---------|------------|
| Space | O(V + E) | O(V²) |
| Check edge | O(V) | O(1) |
| Best for | Sparse | Dense |

---

## 3. BFS — O(V + E) ⭐

Level-by-level exploration. Uses **queue**.

```javascript
function bfs(graph, start) {
  let visited = new Set([start]);
  let queue = [start], result = [];
  while (queue.length) {
    let node = queue.shift();
    result.push(node);
    for (let neighbor of graph.adj.get(node) || []) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push(neighbor);
      }
    }
  }
  return result;
}
```

```python
from collections import deque
def bfs(graph, start):
    visited = {start}
    queue = deque([start])
    result = []
    while queue:
        node = queue.popleft()
        result.append(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    return result
```

### BFS Shortest Path (Unweighted)
```javascript
function shortestPath(graph, start, end) {
  let queue = [[start, [start]]];
  let visited = new Set([start]);
  while (queue.length) {
    let [node, path] = queue.shift();
    if (node === end) return path;
    for (let nb of graph.adj.get(node) || []) {
      if (!visited.has(nb)) {
        visited.add(nb);
        queue.push([nb, [...path, nb]]);
      }
    }
  }
  return null;
}
```

---

## 4. DFS — O(V + E) ⭐

Go as deep as possible, then backtrack. Uses **stack** (or recursion).

```javascript
function dfs(graph, start, visited = new Set()) {
  visited.add(start);
  let result = [start];
  for (let nb of graph.adj.get(start) || []) {
    if (!visited.has(nb)) result.push(...dfs(graph, nb, visited));
  }
  return result;
}

// Iterative DFS
function dfsIterative(graph, start) {
  let stack = [start], visited = new Set(), result = [];
  while (stack.length) {
    let node = stack.pop();
    if (visited.has(node)) continue;
    visited.add(node);
    result.push(node);
    for (let nb of graph.adj.get(node) || []) {
      if (!visited.has(nb)) stack.push(nb);
    }
  }
  return result;
}
```

### BFS vs DFS

| Feature | BFS | DFS |
|---------|-----|-----|
| Data structure | Queue | Stack / Recursion |
| Pattern | Level by level | As deep as possible |
| Shortest path | ✅ (unweighted) | ❌ |
| Space (worst) | O(V) | O(V) |
| Best for | Shortest path, level order | Cycle detection, topological sort |

---

## 5. Connected Components
```javascript
function countComponents(n, edges) {
  let graph = new Map();
  for (let i = 0; i < n; i++) graph.set(i, []);
  for (let [u, v] of edges) { graph.get(u).push(v); graph.get(v).push(u); }

  let visited = new Set(), count = 0;
  for (let i = 0; i < n; i++) {
    if (!visited.has(i)) {
      count++;
      let queue = [i];
      visited.add(i);
      while (queue.length) {
        let node = queue.shift();
        for (let nb of graph.get(node)) {
          if (!visited.has(nb)) { visited.add(nb); queue.push(nb); }
        }
      }
    }
  }
  return count;
}
```

## 6. Cycle Detection

### Undirected — DFS
```javascript
function hasCycle(graph, n) {
  let visited = new Set();
  function dfs(node, parent) {
    visited.add(node);
    for (let nb of graph.get(node) || []) {
      if (!visited.has(nb)) {
        if (dfs(nb, node)) return true;
      } else if (nb !== parent) return true;
    }
    return false;
  }
  for (let i = 0; i < n; i++)
    if (!visited.has(i) && dfs(i, -1)) return true;
  return false;
}
```

### Directed — Three Colors
```javascript
function hasCycleDirected(graph) {
  let white = new Set(graph.keys()), gray = new Set(), black = new Set();
  function dfs(node) {
    white.delete(node); gray.add(node);
    for (let nb of graph.get(node) || []) {
      if (gray.has(nb)) return true;
      if (white.has(nb) && dfs(nb)) return true;
    }
    gray.delete(node); black.add(node);
    return false;
  }
  for (let node of [...white]) if (dfs(node)) return true;
  return false;
}
```

## 7. Number of Islands (Grid as Graph)
```javascript
function numIslands(grid) {
  let count = 0, R = grid.length, C = grid[0].length;
  function dfs(r, c) {
    if (r < 0 || r >= R || c < 0 || c >= C || grid[r][c] === '0') return;
    grid[r][c] = '0';
    dfs(r+1,c); dfs(r-1,c); dfs(r,c+1); dfs(r,c-1);
  }
  for (let r = 0; r < R; r++)
    for (let c = 0; c < C; c++)
      if (grid[r][c] === '1') { count++; dfs(r, c); }
  return count;
}
```

---

## 8. Practice Problems

| Problem | Algorithm | Difficulty |
|---------|-----------|-----------|
| Number of Islands | DFS/BFS grid | Medium |
| Clone Graph | DFS + HashMap | Medium |
| Graph Valid Tree | DFS + cycle | Medium |
| Connected Components | BFS/DFS | Medium |
| Pacific Atlantic Water Flow | Multi-source DFS | Medium |
| Surrounded Regions | DFS from border | Medium |
| Walls and Gates | Multi-source BFS | Medium |
| 01 Matrix | BFS | Medium |
| Word Ladder | BFS | Hard |
