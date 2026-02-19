# Chapter 20 — Greedy Algorithms

## The Problem This Solves
Some optimization problems have the property that making the locally best choice at each step leads to the globally optimal solution. Greedy algorithms exploit this by never reconsidering past decisions, resulting in O(n log n) or O(n) solutions where brute force would be exponential.

---

## 1. The Greedy Strategy

```
GREEDY TEMPLATE:
════════════════
  1. Sort the input by some criteria (key design decision)
  2. Iterate through the sorted input
  3. At each step, make the locally optimal choice
  4. Never backtrack or reconsider previous choices

  WARNING: Greedy only works when the "greedy choice property" holds.
  If it doesn't, the algorithm gives WRONG answers (use DP instead).
```

### When Does Greedy Work?

Two conditions must hold:
1. **Greedy Choice Property:** A locally optimal choice is part of a globally optimal solution.
2. **Optimal Substructure:** An optimal solution contains optimal solutions to its subproblems.

**How to verify:** Use the Exchange Argument — assume there exists an optimal solution that doesn't include the greedy choice, then show you can swap in the greedy choice without worsening the result.

---

## 2. Activity Selection (The Classic Greedy Problem)

**Problem:** Given N activities with start and end times, find the maximum number of non-overlapping activities.

**Greedy Choice:** Always pick the activity that finishes earliest (sort by end time).

```javascript
function activitySelection(activities) {
    // Sort by end time (THE critical design choice)
    activities.sort((a, b) => a.end - b.end);
    
    const selected = [activities[0]];
    let lastEnd = activities[0].end;
    
    for (let i = 1; i < activities.length; i++) {
        if (activities[i].start >= lastEnd) {
            selected.push(activities[i]);
            lastEnd = activities[i].end;
        }
    }
    return selected;
}
```

**Manual Walkthrough:**
```
Activities: [(1,4), (3,5), (0,6), (5,7), (8,9), (5,9)]
Sorted by end: [(1,4), (3,5), (0,6), (5,7), (8,9), (5,9)]

Step 1: Pick (1,4). lastEnd = 4.
Step 2: (3,5) starts at 3 < 4. SKIP (overlaps).
Step 3: (0,6) starts at 0 < 4. SKIP.
Step 4: (5,7) starts at 5 >= 4. PICK. lastEnd = 7.
Step 5: (8,9) starts at 8 >= 7. PICK. lastEnd = 9.
Step 6: (5,9) starts at 5 < 9. SKIP.

Result: [(1,4), (5,7), (8,9)] — 3 activities. OPTIMAL. ✓
```

**Why earliest-end-time works (Exchange Argument):** If any optimal solution picks an activity A that ends later than our greedy pick G, we can swap A for G without creating a conflict (G ends earlier, so it can't overlap with the next activity). Therefore, greedy is always at least as good.

---

## 3. Fractional Knapsack

**Problem:** Given items with weight and value, maximize value in a knapsack with limited capacity. Items can be split (fractions allowed).

**Greedy Choice:** Sort by value-to-weight ratio (value/weight), pick highest ratio first.

```javascript
function fractionalKnapsack(items, capacity) {
    // Sort by value/weight ratio (descending)
    items.sort((a, b) => (b.value / b.weight) - (a.value / a.weight));
    
    let totalValue = 0;
    let remaining = capacity;
    
    for (const item of items) {
        if (remaining >= item.weight) {
            totalValue += item.value;      // Take the whole item
            remaining -= item.weight;
        } else {
            totalValue += item.value * (remaining / item.weight); // Take a fraction
            break; // Knapsack is full
        }
    }
    return totalValue;
}
```

**Note:** Fractional Knapsack is Greedy. **0/1 Knapsack** (no splitting) is NOT greedy — it requires Dynamic Programming.

---

## 4. Huffman Coding — Optimal Prefix-Free Compression

**Problem:** Encode characters using variable-length binary codes such that total encoded size is minimized. Frequent characters get short codes; rare characters get long codes.

```javascript
function huffmanCoding(frequencies) {
    // Use a min-heap (priority queue sorted by frequency)
    const pq = new MinHeap();
    
    // Step 1: Create a leaf node for each character
    for (const [char, freq] of Object.entries(frequencies)) {
        pq.insert({ char, freq, left: null, right: null });
    }
    
    // Step 2: Repeatedly merge the two lowest-frequency nodes
    while (pq.size > 1) {
        const left = pq.extractMin();
        const right = pq.extractMin();
        pq.insert({
            char: null,
            freq: left.freq + right.freq,
            left,
            right
        });
    }
    
    return pq.peek(); // Root of the Huffman tree
}
```

**Manual Walkthrough — Frequencies: A=5, B=9, C=12, D=13, E=16, F=45:**
```
Step 1: Merge A(5) + B(9) = AB(14)
Step 2: Merge C(12) + D(13) = CD(25)
Step 3: Merge AB(14) + E(16) = ABE(30)
Step 4: Merge CD(25) + ABE(30) = CDABE(55)
Step 5: Merge F(45) + CDABE(55) = Root(100)

Resulting codes:
  F = 0          (1 bit — most frequent)
  C = 100        (3 bits)
  D = 101        (3 bits)
  A = 1100       (4 bits — least frequent)
  B = 1101       (4 bits)
  E = 111        (3 bits)
```

---

## 5. Interval Scheduling — Merge Overlapping Intervals

```javascript
function mergeIntervals(intervals) {
    intervals.sort((a, b) => a[0] - b[0]); // Sort by start time
    
    const merged = [intervals[0]];
    for (let i = 1; i < intervals.length; i++) {
        const last = merged[merged.length - 1];
        if (intervals[i][0] <= last[1]) {
            last[1] = Math.max(last[1], intervals[i][1]); // Extend
        } else {
            merged.push(intervals[i]); // No overlap
        }
    }
    return merged;
}
// [[1,3],[2,6],[8,10],[15,18]] → [[1,6],[8,10],[15,18]]
```

---

## 6. Greedy vs. Dynamic Programming

| Criteria | Greedy | Dynamic Programming |
|----------|--------|---------------------|
| Decision | Locally optimal, never reconsider | Consider ALL options, pick best |
| Speed | Usually O(n log n) | Usually O(n^2) or O(n * W) |
| Correctness | Only when greedy property holds | Always correct |
| Backtracking | Never | Implicitly (via overlapping subproblems) |
| Example | Activity Selection, Huffman | 0/1 Knapsack, Edit Distance |

**Rule:** Try Greedy first (faster). If a counterexample exists where greedy fails, use DP.

---

## 7. Practice Problems

| Problem | Greedy Strategy | Difficulty |
|---------|----------------|------------|
| Activity Selection | Sort by end time | Medium |
| Fractional Knapsack | Sort by value/weight | Medium |
| Minimum Platforms | Sweep line on start/end | Medium |
| Job Sequencing | Sort by profit, use deadline slots | Medium |
| Huffman Coding | Min-heap merge | Medium |
| Merge Intervals | Sort by start, extend | Medium |
| Jump Game | Track farthest reachable | Medium |
| Gas Station | Circular greedy | Medium |
