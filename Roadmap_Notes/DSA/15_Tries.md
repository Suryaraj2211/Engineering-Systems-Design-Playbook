# Chapter 15 — Tries (Prefix Trees)

## The Problem This Solves
Searching for a word in a sorted array is O(L log n) where L = word length. A HashMap gives O(L) exact lookup but cannot answer "what words start with 'app'?" efficiently. A Trie gives **O(L) insert, search, AND prefix matching** — the only structure that solves all three.

---

## 1. How a Trie Works

Each node stores one character. Paths from root to marked nodes form complete words. Shared prefixes share the same path (massive space savings for dictionaries).

```
Words: "app", "apple", "ant", "car", "card"

Root
├── a
│   ├── p
│   │   └── p ← "app" ✓
│   │       └── l
│   │           └── e ← "apple" ✓
│   └── n
│       └── t ← "ant" ✓
└── c
    └── a
        └── r ← "car" ✓
            └── d ← "card" ✓

Note: "app" and "apple" share the path a→p→p (3 nodes saved).
```

---

## 2. Implementation

```javascript
class TrieNode {
    constructor() {
        this.children = {};  // Map: character → TrieNode
        this.isEnd = false;  // Marks a complete word
        this.count = 0;      // How many words pass through this node (for prefix counting)
    }
}

class Trie {
    constructor() { this.root = new TrieNode(); }
    
    insert(word) {
        let node = this.root;
        for (const ch of word) {
            if (!node.children[ch]) node.children[ch] = new TrieNode();
            node = node.children[ch];
            node.count++;
        }
        node.isEnd = true;
    }
    
    search(word) {
        const node = this._traverse(word);
        return node !== null && node.isEnd;
    }
    
    startsWith(prefix) {
        return this._traverse(prefix) !== null;
    }
    
    // Count how many words have this prefix
    countPrefix(prefix) {
        const node = this._traverse(prefix);
        return node ? node.count : 0;
    }
    
    // Autocomplete: return all words starting with prefix
    autocomplete(prefix) {
        const node = this._traverse(prefix);
        if (!node) return [];
        const results = [];
        this._dfs(node, prefix, results);
        return results;
    }
    
    _traverse(str) {
        let node = this.root;
        for (const ch of str) {
            if (!node.children[ch]) return null;
            node = node.children[ch];
        }
        return node;
    }
    
    _dfs(node, path, results) {
        if (node.isEnd) results.push(path);
        for (const [ch, child] of Object.entries(node.children)) {
            this._dfs(child, path + ch, results);
        }
    }
}
```

### Manual Walkthrough — Insert "app" then "apple"
```
Insert "app":
  root → 'a' (create) → 'p' (create) → 'p' (create, mark isEnd=true)
  
Insert "apple":
  root → 'a' (exists) → 'p' (exists) → 'p' (exists) → 'l' (create) → 'e' (create, isEnd=true)
  
  Shared nodes: a, p, p (3 nodes reused instead of duplicated)
```

---

## 3. Memory Analysis

**Naive Trie (using object/map per node):**
- Each node stores a map of children. For English lowercase, at most 26 entries.
- N words of average length L = **O(N * L)** nodes total.
- If words share heavy prefixes (e.g., dictionary), actual nodes << N * L.

**Array-based Trie (faster, more memory):**
```javascript
class ArrayTrieNode {
    constructor() {
        this.children = new Array(26).fill(null); // Fixed 26-slot array
        this.isEnd = false;
    }
}
// Faster lookups (array index vs. hash), but wastes memory on sparse nodes.
// Use for: competitive programming where speed > memory.
```

**Compressed Trie (Patricia Trie / Radix Tree):**
Merges single-child chains into one node. "apple" stored in 3 nodes instead of 5:
```
Uncompressed:   a → p → p → l → e   (5 nodes)
Compressed:     "ap" → "p" → "le"    (3 nodes, edges store strings)
```
Use for: IP routing tables, large dictionaries where memory matters.

---

## 4. Real-World Applications

| Application | How Trie Solves It |
|-------------|--------------------|
| Phone autocomplete | `autocomplete("hel")` → ["hello", "help", "helmet"] in O(prefix + results) |
| Spell checker | Insert dictionary, `search(word)` returns false = misspelled |
| IP routing (Longest Prefix Match) | Binary trie on IP bits, find longest matching prefix for routing |
| DNS resolution | Reversed domain names in trie for fast hierarchical lookup |
| Boggle / Word Search | DFS on grid with trie pruning (skip invalid prefixes early) |

---

## 5. Trie vs. HashMap vs. Sorted Array

| Operation | Trie | HashMap | Sorted Array |
|-----------|------|---------|--------------|
| Exact search | O(L) | O(L) | O(L log n) |
| Prefix search | O(L) | O(n) scan all | O(L log n) + scan |
| Insert | O(L) | O(L) amortized | O(n) shift |
| Autocomplete | O(L + results) | Impossible efficiently | O(L log n + results) |
| Sorted iteration | Natural (DFS) | Must sort keys | Natural |
| Memory | High (pointers) | Moderate | Low |

---

## 6. Practice Problems

| Problem | Concept | Difficulty |
|---------|---------|------------|
| Implement Trie | Core insert/search/startsWith | Medium |
| Word Search II | Trie + backtracking on grid | Hard |
| Design Autocomplete System | Trie + frequency sorting | Hard |
| Replace Words | Shortest prefix replacement | Medium |
| Word Dictionary (`.` wildcard) | Trie + DFS with wildcard | Medium |
| Longest Word in Dictionary | Trie + BFS (build word by word) | Medium |
