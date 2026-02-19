# Chapter 28 — Pattern Recognition for DSA Problems

## Why Patterns Matter
Most interview problems map to one of ~15 known patterns. Recognizing the pattern is **the hardest part** — once identified, the solution template is known.

---

## 1. Pattern Decision Tree

```
Given a problem, ask:
│
├─ Is it about a sorted array / search space?
│  └── Binary Search (Ch 11)
│
├─ Is it about subarrays / substrings?
│  ├── Fixed size? → Fixed Sliding Window (Ch 25)
│  ├── Variable size with condition? → Variable Sliding Window (Ch 25)
│  └── Sum equals K? → Prefix Sum + HashMap (Ch 27)
│
├─ Is it about pairs / triples in sorted data?
│  └── Two Pointer (Ch 25)
│
├─ Is it about order / next greater-smaller?
│  └── Monotonic Stack (Ch 26)
│
├─ Is it about a tree?
│  ├── Traversal / path? → DFS recursion (Ch 12)
│  ├── Level-by-level? → BFS (Ch 12)
│  └── BST property? → BST techniques (Ch 13)
│
├─ Is it about a graph / grid / connected components?
│  ├── Shortest path (unweighted)? → BFS (Ch 17)
│  ├── Shortest path (weighted)? → Dijkstra (Ch 18)
│  ├── Components / connectivity? → Union-Find / DFS (Ch 17-18)
│  └── Ordering / dependencies? → Topological Sort (Ch 18)
│
├─ Does it ask for all possible answers?
│  └── Backtracking (Ch 21)
│
├─ Does it ask for optimal value with overlapping subproblems?
│  └── Dynamic Programming (Ch 22-23)
│
├─ Can you make a local optimal choice?
│  └── Greedy (Ch 20)
│
├─ Is it about frequency / lookup?
│  └── HashMap / HashSet (Ch 9)
│
├─ Is it about k-th largest / smallest / top-k?
│  └── Heap / Priority Queue (Ch 14)
│
├─ Is it about prefixes / autocomplete?
│  └── Trie (Ch 15)
│
└── Does it involve bits / XOR?
   └── Bit Manipulation (Ch 24)
```

---

## 2. Pattern Quick Reference

| Pattern | Key Signal | Chapters |
|---------|-----------|----------|
| **Sliding Window** | Contiguous subarray/substring | 25 |
| **Two Pointer** | Sorted array, pairs, remove duplicates | 25 |
| **Fast & Slow Pointer** | Linked list cycle, middle | 6 |
| **Merge Intervals** | Overlapping intervals | 20 |
| **Cyclic Sort** | Array with nums in range [0,n] | 4 |
| **In-place Reversal** | Reverse linked list / sublist | 6 |
| **BFS** | Shortest path, level order | 12, 17 |
| **DFS** | Exhaust all paths, backtrack | 12, 17, 21 |
| **Two Heaps** | Median, top/bottom half | 14 |
| **Binary Search** | Sorted data, search boundaries | 11 |
| **Subsets/Permutations** | Generate all combinations | 21 |
| **Top K** | K largest/smallest/frequent | 14 |
| **Prefix Sum** | Range sum queries | 27 |
| **Monotonic Stack** | Next greater/smaller | 26 |
| **Dynamic Programming** | Overlapping subproblems | 22, 23 |
| **Knapsack** | Subset selection with constraint | 22 |
| **Trie** | String prefix matching | 15 |
| **Union-Find** | Connected components | 18 |
| **Topological Sort** | Task ordering / dependencies | 18 |
| **Bit Manipulation** | XOR tricks, single number | 24 |

---

## 3. Common Problem → Pattern Mapping

| Problem | Pattern |
|---------|---------|
| Two Sum | HashMap |
| Best Time Buy Stock | Greedy / DP |
| Valid Parentheses | Stack |
| Merge Intervals | Sort + Greedy |
| Number of Islands | DFS/BFS Grid |
| LRU Cache | DLL + HashMap |
| Longest Substring No Repeat | Sliding Window |
| Course Schedule | Topological Sort |
| Coin Change | DP (Knapsack) |
| Word Search | Backtracking |
| Median from Stream | Two Heaps |
| Meeting Rooms | Sort Intervals |
| Trapping Rain Water | Two Pointer / Monotonic Stack |
| Kth Largest | Min Heap |
| Word Ladder | BFS |

---

## 4. Difficulty Progression Strategy

### Easy (Pattern recognition practice)
Start with: Two Sum, Valid Parentheses, Merge Sorted Lists, Reverse List, Max Depth Tree, Climbing Stairs, Single Number

### Medium (Core patterns)
Focus on: 3Sum, LCA, Number of Islands, Course Schedule, Coin Change, Group Anagrams, Top K Frequent, House Robber

### Hard (Combine patterns)
Attempt after mastering medium: Merge K Lists, Trapping Rain Water, Alien Dictionary, Word Break II, Burst Balloons, Sliding Window Max
