# Chapter 00 — DSA Roadmap & Overview

## What is DSA?

**Data Structures** = Ways to organize and store data efficiently.
**Algorithms** = Step-by-step procedures to solve problems using that data.

Together, DSA teaches you **how to think about problems** and solve them efficiently.

---

## Complete Learning Roadmap

```
 MODULE 1: FOUNDATIONS
 ┌─────────────────────────────────────────────────┐
 │  Math Basics → Time Complexity → Recursion       │
 └──────────────────────┬──────────────────────────┘
                        ▼
 MODULE 2: LINEAR DATA STRUCTURES
 ┌─────────────────────────────────────────────────┐
 │  Arrays → Strings → Linked Lists → Stack → Queue │
 │  → Hashing                                       │
 └──────────────────────┬──────────────────────────┘
                        ▼
 MODULE 3: SORTING & SEARCHING
 ┌─────────────────────────────────────────────────┐
 │  All Sorting Algorithms → Binary Search Mastery   │
 └──────────────────────┬──────────────────────────┘
                        ▼
 MODULE 4: TREES
 ┌─────────────────────────────────────────────────┐
 │  Binary Tree → BST → Heaps → Tries → AVL/Seg    │
 └──────────────────────┬──────────────────────────┘
                        ▼
 MODULE 5: GRAPHS
 ┌─────────────────────────────────────────────────┐
 │  Representations → BFS/DFS → Shortest Path → MST │
 └──────────────────────┬──────────────────────────┘
                        ▼
 MODULE 6: ALGORITHM PARADIGMS
 ┌─────────────────────────────────────────────────┐
 │  D&C → Greedy → Backtracking → DP → Advanced DP  │
 └──────────────────────┬──────────────────────────┘
                        ▼
 MODULE 7: TECHNIQUES & PATTERNS
 ┌─────────────────────────────────────────────────┐
 │  Bit Manipulation → Sliding Window → Two Pointer  │
 │  → Monotonic Stack → Prefix Sum                   │
 └──────────────────────┬──────────────────────────┘
                        ▼
 MODULE 8: INTERVIEW PREPARATION
 ┌─────────────────────────────────────────────────┐
 │  Pattern Recognition → Interview Strategy         │
 └─────────────────────────────────────────────────┘
```

---

## Chapter Index

### Module 1: Foundations
| Ch | Topic | Key Concepts |
|----|-------|-------------|
| 01 | [Math Foundations](./01_Math_Foundations.md) | Log, powers, modular math, combinatorics |
| 02 | [Time & Space Complexity](./02_Time_Space_Complexity.md) | Big O, Omega, Theta, amortized analysis |
| 03 | [Recursion](./03_Recursion.md) | Call stack, base case, tree recursion, tail recursion |

### Module 2: Linear Data Structures
| Ch | Topic | Key Concepts |
|----|-------|-------------|
| 04 | [Arrays](./04_Arrays.md) | 1D, 2D, dynamic, Kadane's, rotation |
| 05 | [Strings](./05_Strings.md) | Immutability, pattern matching, KMP, Rabin-Karp |
| 06 | [Linked List](./06_LinkedList.md) | SLL, DLL, Circular, Floyd's cycle |
| 07 | [Stack](./07_Stack.md) | LIFO, monotonic stack, expression parsing |
| 08 | [Queue](./08_Queue.md) | FIFO, circular, deque, priority queue |
| 09 | [Hashing](./09_Hashing.md) | Hash functions, collisions, Map/Set |

### Module 3: Sorting & Searching
| Ch | Topic | Key Concepts |
|----|-------|-------------|
| 10 | [Sorting Algorithms](./10_Sorting.md) | All sorts, stability, comparison table |
| 11 | [Searching Algorithms](./11_Searching.md) | Binary search templates, search on answer |

### Module 4: Trees
| Ch | Topic | Key Concepts |
|----|-------|-------------|
| 12 | [Binary Trees](./12_Binary_Trees.md) | Traversals, construction, classic problems |
| 13 | [BST](./13_BST.md) | Insert/delete, validation, floor/ceil |
| 14 | [Heaps & Priority Queue](./14_Heaps_PriorityQueue.md) | Heapify, k-way merge, median stream |
| 15 | [Tries](./15_Tries.md) | Insert/search, autocomplete, word search |
| 16 | [Advanced Trees](./16_Advanced_Trees.md) | AVL, Red-Black, Segment Tree, Fenwick |

### Module 5: Graphs
| Ch | Topic | Key Concepts |
|----|-------|-------------|
| 17 | [Graphs — Basics](./17_Graphs_Basics.md) | Representations, BFS, DFS, cycle detection |
| 18 | [Graphs — Advanced](./18_Graphs_Advanced.md) | Dijkstra, Bellman-Ford, MST, Topo Sort, DSU |

### Module 6: Algorithm Paradigms
| Ch | Topic | Key Concepts |
|----|-------|-------------|
| 19 | [Divide & Conquer](./19_Divide_and_Conquer.md) | Merge sort analysis, quick select, master theorem |
| 20 | [Greedy Algorithms](./20_Greedy.md) | Proof of correctness, intervals, Huffman |
| 21 | [Backtracking](./21_Backtracking.md) | Template, pruning, N-Queens, Sudoku |
| 22 | [Dynamic Programming](./22_Dynamic_Programming.md) | Memo/Tab, 1D/2D DP, Knapsack, LCS |
| 23 | [Advanced DP](./23_Advanced_DP.md) | Bitmask, interval, DP on trees, state machine |

### Module 7: Techniques & Patterns
| Ch | Topic | Key Concepts |
|----|-------|-------------|
| 24 | [Bit Manipulation](./24_Bit_Manipulation.md) | Operators, XOR tricks, subset enumeration |
| 25 | [Sliding Window & Two Pointer](./25_Sliding_Window_TwoPointer.md) | Fixed/variable window, templates |
| 26 | [Monotonic Stack & Queue](./26_Monotonic_Stack_Queue.md) | Next greater, largest rectangle, sliding max |
| 27 | [Prefix Sum](./27_Prefix_Sum.md) | 1D/2D prefix, difference array, XOR prefix |

### Module 8: Interview Preparation
| Ch | Topic | Key Concepts |
|----|-------|-------------|
| 28 | [Pattern Recognition](./28_Pattern_Recognition.md) | 15+ patterns, decision tree |
| 29 | [Interview Strategy](./29_Interview_Strategy.md) | Study plan, revision checklist, cheatsheets |

---

## How to Use This Documentation

1. **Follow the order** — Modules build on each other
2. **Type every code example** yourself — muscle memory matters
3. **Do practice problems** after each chapter (Easy → Medium → Hard)
4. **Revisit** — Read each chapter at least twice
5. **After all chapters**, use Chapter 28 (Pattern Recognition) to tie everything together
6. **Before interviews**, use Chapter 29 (Interview Strategy) for revision

---

## Recommended Study Plan

| Week | Modules | Focus |
|------|---------|-------|
| 1 | Module 1 | Foundations, math, complexity |
| 2-3 | Module 2 | All linear data structures |
| 4 | Module 3 | Sorting & searching mastery |
| 5-6 | Module 4 | Trees deep-dive |
| 7 | Module 5 | Graphs |
| 8-9 | Module 6 | Algorithm paradigms (hardest) |
| 10 | Module 7 | Techniques & patterns |
| 11-12 | Module 8 + Review | Interview prep + revision |
