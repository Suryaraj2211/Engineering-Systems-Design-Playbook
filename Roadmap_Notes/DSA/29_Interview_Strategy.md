# Chapter 29 — Interview Strategy & Revision Checklist

## 1. How to Approach Any DSA Problem in an Interview

```
Step 1: UNDERSTAND
  → Read the problem twice
  → Identify inputs, outputs, constraints
  → Ask clarifying questions
  → Write down examples (including edge cases)

Step 2: PLAN
  → Identify the pattern (Ch 28)
  → Think of brute force first
  → Optimize: Can I use a HashMap? Sort? Binary Search?
  → State your approach to the interviewer BEFORE coding

Step 3: CODE
  → Write clean, readable code
  → Use meaningful variable names
  → Talk through your code as you write

Step 4: VERIFY
  → Dry run with your example
  → Check edge cases
  → State time and space complexity
```

---

## 2. Time Management (45-minute interview)

| Phase | Time | Activity |
|-------|------|----------|
| Understand | 3-5 min | Read, ask questions, examples |
| Plan | 5-7 min | Identify pattern, discuss approach |
| Code | 15-20 min | Implement solution |
| Debug | 5-7 min | Dry run, fix bugs |
| Optimize | 5-7 min | Discuss complexity, optimize if possible |

---

## 3. What Interviewers Actually Evaluate

| Skill | What They Look For |
|-------|-------------------|
| Problem Solving | Can you break down the problem? |
| Communication | Do you explain your thinking? |
| Coding | Clean, correct, efficient code |
| Testing | Do you check edge cases? Do you dry run? |
| Optimization | Can you identify time/space tradeoffs? |

---

## 4. 12-Week Study Plan

| Week | Topics | Problems/Day |
|------|--------|-------------|
| 1 | Ch 01-03: Math, Complexity, Recursion | 2-3 Easy |
| 2 | Ch 04-05: Arrays, Strings | 3-4 Easy/Medium |
| 3 | Ch 06-08: LinkedList, Stack, Queue | 3-4 Easy/Medium |
| 4 | Ch 09: Hashing + Ch 25: Sliding Window | 3-4 Medium |
| 5 | Ch 10-11: Sorting, Searching | 3-4 Medium |
| 6 | Ch 12-13: Trees, BST | 3-4 Medium |
| 7 | Ch 14-16: Heaps, Tries, Advanced Trees | 2-3 Medium |
| 8 | Ch 17-18: Graphs | 3-4 Medium |
| 9 | Ch 20-21: Greedy, Backtracking | 3-4 Medium |
| 10 | Ch 22-23: DP | 2-3 Medium/Hard |
| 11 | Ch 24, 26, 27: Bits, Monotonic, Prefix | 3-4 Mixed |
| 12 | Ch 28-29: Patterns, Mock Interviews | Full sessions |

---

## 5. Revision Checklist (Before Interview Day)

### Data Structures — Can You Implement?
- [ ] Array operations and patterns
- [ ] Linked List (reverse, cycle detection, merge)
- [ ] Stack (valid parentheses, monotonic)
- [ ] Queue (BFS, circular)
- [ ] HashMap (Two Sum, frequency counting)
- [ ] Min/Max Heap
- [ ] Trie (insert, search, prefix)
- [ ] Graph (adjacency list, BFS, DFS)
- [ ] Binary Tree (traversals, LCA, diameter)
- [ ] BST (insert, delete, validate)

### Algorithms — Can You Apply?
- [ ] Binary Search (all 4 templates)
- [ ] Merge Sort & Quick Sort
- [ ] BFS & DFS (graph and grid)
- [ ] Dijkstra's Algorithm
- [ ] Topological Sort
- [ ] Union-Find
- [ ] Kadane's Algorithm
- [ ] Quick Select

### Patterns — Can You Recognize?
- [ ] Sliding Window (fixed and variable)
- [ ] Two Pointer (same/opposite direction)
- [ ] Prefix Sum
- [ ] Frequency Counter
- [ ] Monotonic Stack
- [ ] Backtracking template
- [ ] DP: 1D, 2D, Knapsack
- [ ] Bit manipulation tricks

### Complexity — Can You Analyze?
- [ ] Big O for all common sorts
- [ ] HashMap: O(1) avg, O(n) worst
- [ ] BST: O(log n) avg, O(n) worst
- [ ] Heap: insert/extract O(log n)
- [ ] BFS/DFS: O(V + E)
- [ ] Binary Search: O(log n)

---

## 6. Complexity Cheatsheet

| Data Structure | Access | Search | Insert | Delete |
|---------------|--------|--------|--------|--------|
| Array | O(1) | O(n) | O(n) | O(n) |
| Stack/Queue | O(n) | O(n) | O(1) | O(1) |
| Linked List | O(n) | O(n) | O(1) | O(1) |
| HashMap | — | O(1)* | O(1)* | O(1)* |
| BST Balanced | — | O(log n) | O(log n) | O(log n) |
| Heap | — | O(n) | O(log n) | O(log n) |
| Trie | — | O(L) | O(L) | O(L) |

| Algorithm | Time | Space |
|-----------|------|-------|
| Binary Search | O(log n) | O(1) |
| Merge Sort | O(n log n) | O(n) |
| Quick Sort | O(n log n)* | O(log n) |
| DFS/BFS | O(V + E) | O(V) |
| Dijkstra | O((V+E) log V) | O(V) |
| Topological Sort | O(V + E) | O(V) |

\* = average case

---

## 7. Top 50 Must-Do Problems

### Arrays & Strings (10)
1. Two Sum
2. Best Time to Buy Stock
3. Product Except Self
4. Maximum Subarray
5. Container With Most Water
6. 3Sum
7. Group Anagrams
8. Longest Substring No Repeat
9. Valid Palindrome
10. Merge Intervals

### Linked Lists (5)
11. Reverse Linked List
12. Merge Two Sorted Lists
13. Linked List Cycle
14. Remove Nth from End
15. LRU Cache

### Trees (8)
16. Max Depth
17. Invert Tree
18. Level Order Traversal
19. Validate BST
20. Kth Smallest in BST
21. LCA
22. Serialize/Deserialize
23. Binary Tree Max Path Sum

### Graphs (5)
24. Number of Islands
25. Course Schedule
26. Clone Graph
27. Word Ladder
28. Network Delay Time

### DP (8)
29. Climbing Stairs
30. House Robber
31. Coin Change
32. LIS
33. Longest Common Subsequence
34. Edit Distance
35. Word Break
36. Unique Paths

### Stack/Queue (4)
37. Valid Parentheses
38. Min Stack
39. Daily Temperatures
40. Largest Rectangle Histogram

### Heap (3)
41. Kth Largest Element
42. Top K Frequent
43. Find Median from Stream

### Other Patterns (7)
44. Sort Colors
45. Trapping Rain Water
46. Sliding Window Maximum
47. Subarray Sum Equals K
48. Missing Number
49. Implement Trie
50. Word Search

---

## 8. How to Handle "I'm Stuck"

1. **Re-read the problem** — you might have missed a constraint
2. **Think of a simpler version** — can you solve for n=2 or n=3?
3. **Draw it out** — visualize with diagrams
4. **Think about the pattern** — use the decision tree (Ch 28)
5. **Try brute force first** — then optimize
6. **Ask for a hint** — interviewers expect this sometimes!

> **Remember:** It's not about solving every problem — it's about showing how you **think** and **communicate**.
