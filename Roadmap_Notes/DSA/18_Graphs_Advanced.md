# Chapter 18 — Graphs: Advanced Algorithms

## 1. Topological Sort (DAG only)

Order vertices so every edge goes from earlier to later. Used for: build systems, course prerequisites, task scheduling.

### Kahn's Algorithm (BFS-based)
```javascript
function topologicalSort(numNodes, edges) {
  let graph = new Map(), inDegree = new Array(numNodes).fill(0);
  for (let i = 0; i < numNodes; i++) graph.set(i, []);
  for (let [u, v] of edges) { graph.get(u).push(v); inDegree[v]++; }

  let queue = [];
  for (let i = 0; i < numNodes; i++) if (inDegree[i] === 0) queue.push(i);

  let order = [];
  while (queue.length) {
    let node = queue.shift();
    order.push(node);
    for (let nb of graph.get(node)) {
      inDegree[nb]--;
      if (inDegree[nb] === 0) queue.push(nb);
    }
  }
  return order.length === numNodes ? order : []; // Empty = cycle exists
}
```

```python
from collections import deque
def topo_sort(n, edges):
    graph = [[] for _ in range(n)]
    in_deg = [0] * n
    for u, v in edges:
        graph[u].append(v)
        in_deg[v] += 1
    queue = deque([i for i in range(n) if in_deg[i] == 0])
    order = []
    while queue:
        node = queue.popleft()
        order.append(node)
        for nb in graph[node]:
            in_deg[nb] -= 1
            if in_deg[nb] == 0: queue.append(nb)
    return order if len(order) == n else []
```

---

## 2. Dijkstra's Algorithm — Shortest Path (Weighted, Non-negative)

```javascript
function dijkstra(graph, start, n) {
  let dist = new Array(n).fill(Infinity);
  dist[start] = 0;
  let pq = [[0, start]]; // [distance, node]

  while (pq.length) {
    pq.sort((a,b) => a[0] - b[0]); // Simple PQ (use MinHeap for efficiency)
    let [d, u] = pq.shift();
    if (d > dist[u]) continue;
    for (let [v, w] of graph[u] || []) {
      if (dist[u] + w < dist[v]) {
        dist[v] = dist[u] + w;
        pq.push([dist[v], v]);
      }
    }
  }
  return dist;
}

// graph[u] = [[v, weight], ...]
```

```python
import heapq
def dijkstra(graph, start, n):
    dist = [float('inf')] * n
    dist[start] = 0
    pq = [(0, start)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]: continue
        for v, w in graph[u]:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                heapq.heappush(pq, (dist[v], v))
    return dist
```

**Time: O((V+E) log V)** with MinHeap. Doesn't work with **negative weights**.

---

## 3. Bellman-Ford — Handles Negative Weights

```javascript
function bellmanFord(n, edges, start) {
  let dist = new Array(n).fill(Infinity);
  dist[start] = 0;
  for (let i = 0; i < n - 1; i++) {
    for (let [u, v, w] of edges) {
      if (dist[u] !== Infinity && dist[u] + w < dist[v])
        dist[v] = dist[u] + w;
    }
  }
  // Detect negative cycle
  for (let [u, v, w] of edges)
    if (dist[u] !== Infinity && dist[u] + w < dist[v]) return null; // Negative cycle!
  return dist;
}
// Time: O(V × E)
```

---

## 4. Floyd-Warshall — All Pairs Shortest Path

```javascript
function floydWarshall(n, edges) {
  let dist = Array.from({length: n}, () => new Array(n).fill(Infinity));
  for (let i = 0; i < n; i++) dist[i][i] = 0;
  for (let [u, v, w] of edges) dist[u][v] = w;

  for (let k = 0; k < n; k++)
    for (let i = 0; i < n; i++)
      for (let j = 0; j < n; j++)
        dist[i][j] = Math.min(dist[i][j], dist[i][k] + dist[k][j]);
  return dist;
}
// Time: O(V³), Space: O(V²)
```

---

## 5. Union-Find (Disjoint Set Union)

```javascript
class UnionFind {
  constructor(n) {
    this.parent = Array.from({length: n}, (_,i) => i);
    this.rank = new Array(n).fill(0);
  }
  find(x) {
    if (this.parent[x] !== x) this.parent[x] = this.find(this.parent[x]);
    return this.parent[x];
  }
  union(x, y) {
    let rx = this.find(x), ry = this.find(y);
    if (rx === ry) return false;
    if (this.rank[rx] < this.rank[ry]) this.parent[rx] = ry;
    else if (this.rank[rx] > this.rank[ry]) this.parent[ry] = rx;
    else { this.parent[ry] = rx; this.rank[rx]++; }
    return true;
  }
  connected(x, y) { return this.find(x) === this.find(y); }
}
// Nearly O(1) per operation with path compression + union by rank
```

---

## 6. Minimum Spanning Tree (MST)

### Kruskal's (Sort edges + Union-Find)
```javascript
function kruskal(n, edges) {
  edges.sort((a, b) => a[2] - b[2]);
  let uf = new UnionFind(n), mst = [], total = 0;
  for (let [u, v, w] of edges) {
    if (uf.union(u, v)) { mst.push([u, v, w]); total += w; }
  }
  return { mst, total };
}
// Time: O(E log E)
```

### Prim's (Grow tree from start)
```javascript
function prim(graph, n) {
  let visited = new Set(), pq = [[0, 0]], total = 0; // [weight, node]
  while (visited.size < n && pq.length) {
    pq.sort((a,b) => a[0] - b[0]);
    let [w, u] = pq.shift();
    if (visited.has(u)) continue;
    visited.add(u);
    total += w;
    for (let [v, weight] of graph[u])
      if (!visited.has(v)) pq.push([weight, v]);
  }
  return total;
}
```

---

## 7. Shortest Path Comparison

| Algorithm | Weights | Negative? | Time | Space |
|-----------|---------|-----------|------|-------|
| BFS | Unweighted | — | O(V+E) | O(V) |
| Dijkstra | Non-negative | ❌ | O((V+E)logV) | O(V) |
| Bellman-Ford | Any | ✅ (detects cycles) | O(VE) | O(V) |
| Floyd-Warshall | Any | ✅ | O(V³) | O(V²) |

---

## 8. Practice Problems

| Problem | Algorithm | Difficulty |
|---------|-----------|-----------|
| Course Schedule | Topological Sort | Medium |
| Course Schedule II | Topological Sort | Medium |
| Network Delay Time | Dijkstra | Medium |
| Cheapest Flights K Stops | Bellman-Ford / BFS | Medium |
| Redundant Connection | Union-Find | Medium |
| Accounts Merge | Union-Find | Medium |
| Min Cost to Connect All Points | MST (Kruskal/Prim) | Medium |
| Alien Dictionary | Topological Sort | Hard |
| Word Ladder | BFS | Hard |
| Swim in Rising Water | Dijkstra/BS+BFS | Hard |
