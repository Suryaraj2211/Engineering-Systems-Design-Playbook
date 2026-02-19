# Chapter 09 — Hashing (HashMap & HashSet)

## Simple Explanation (ELI5)
Imagine a library where every book has a specific shelf number calculated from its title. Instead of searching every shelf (O(n)), you compute the shelf number and go directly there (O(1)). That's hashing.

## Technical Definition
A hash table maps **keys to values** using a **hash function** that converts keys into array indices. Average O(1) for insert, lookup, and delete.

## Why It's Needed
Hashing converts O(n) search to O(1) average. It's the single most impactful optimization in DSA — used in ~30% of interview problems.

---

## 1. How Hashing Works

```
key "apple" → hash("apple") → 42 → store at index 42

┌─────────────────────────────────┐
│ Hash Function                    │
│ "apple" → 97+112+112+108+101    │
│         → 530 % 53 = 0          │
│ Store at index 0                 │
└─────────────────────────────────┘

Table: [0: (apple, $3)] [1: empty] ... [17: (banana, $5)] ...
```

### Hash Function Properties
1. **Deterministic:** Same key → always same output
2. **Uniform:** Distributes keys evenly across indices
3. **Fast:** Computes in O(1)

---

## 2. Collision Handling

When two keys hash to the **same index**:

### Chaining (Linked Lists at each index)
```
Index 0: [apple → $3] → [grape → $4]    ← collision handled
Index 1: [banana → $5]
Index 2: empty
```

### Open Addressing (Linear Probing)
```
hash("apple") → 5 (occupied) → try 6 (occupied) → try 7 (empty) → store at 7
```

| Method | Pros | Cons |
|--------|------|------|
| Chaining | Simple, handles any load | Extra memory for lists |
| Open Addressing | Cache-friendly | Clustering, complex deletion |

---

## 3. Hash Table Implementation

```javascript
class HashTable {
  constructor(size = 53) {
    this.table = new Array(size);
    this.size = size;
  }

  _hash(key) {
    let hash = 0, PRIME = 31;
    for (let i = 0; i < Math.min(key.length, 100); i++) {
      hash = (hash * PRIME + key.charCodeAt(i)) % this.size;
    }
    return hash;
  }

  set(key, value) {
    let idx = this._hash(key);
    if (!this.table[idx]) this.table[idx] = [];
    for (let pair of this.table[idx]) {
      if (pair[0] === key) { pair[1] = value; return; }
    }
    this.table[idx].push([key, value]);
  }

  get(key) {
    let idx = this._hash(key);
    if (!this.table[idx]) return undefined;
    for (let pair of this.table[idx]) {
      if (pair[0] === key) return pair[1];
    }
    return undefined;
  }

  delete(key) {
    let idx = this._hash(key);
    if (!this.table[idx]) return false;
    for (let i = 0; i < this.table[idx].length; i++) {
      if (this.table[idx][i][0] === key) {
        this.table[idx].splice(i, 1);
        return true;
      }
    }
    return false;
  }
}
```

---

## 4. JavaScript Map & Set

### Map (key → value)
```javascript
let map = new Map();
map.set("name", "Surya");
map.set(42, "answer");
console.log(map.get("name"));  // "Surya"
console.log(map.has(42));       // true
console.log(map.size);          // 2
map.delete(42);

for (let [k, v] of map) console.log(k, v);
```

### Set (unique values)
```javascript
let set = new Set([1, 1, 2, 3, 3]);
console.log(set);  // Set {1, 2, 3} — duplicates removed

set.add(4);
set.has(3);     // true
set.delete(3);

// Remove duplicates from array
let unique = [...new Set([1,1,2,3])];  // [1, 2, 3]
```

```python
# dict (hash map)
d = {"name": "Surya", "age": 25}
d["city"] = "Chennai"
print(d.get("name"))

# set
s = {1, 2, 3, 2, 1}  # {1, 2, 3}

# Counter
from collections import Counter
freq = Counter("abracadabra")  # Counter({'a':5,'b':2,'r':2,'c':1,'d':1})
```

---

## 5. Pattern: Frequency Counter

```javascript
function charFrequency(s) {
  let freq = {};
  for (let ch of s) freq[ch] = (freq[ch] || 0) + 1;
  return freq;
}
```

---

## 6. Classic Problems

### Two Sum — O(n)
```javascript
function twoSum(nums, target) {
  let map = new Map();
  for (let i = 0; i < nums.length; i++) {
    let complement = target - nums[i];
    if (map.has(complement)) return [map.get(complement), i];
    map.set(nums[i], i);
  }
}
```

### Group Anagrams
```javascript
function groupAnagrams(strs) {
  let map = new Map();
  for (let s of strs) {
    let key = s.split("").sort().join("");
    if (!map.has(key)) map.set(key, []);
    map.get(key).push(s);
  }
  return [...map.values()];
}
```

### Longest Consecutive Sequence — O(n)
```javascript
function longestConsecutive(nums) {
  let set = new Set(nums);
  let maxLen = 0;
  for (let num of set) {
    if (!set.has(num - 1)) {  // Only start counting from sequence start
      let len = 1;
      while (set.has(num + len)) len++;
      maxLen = Math.max(maxLen, len);
    }
  }
  return maxLen;
}
// [100, 4, 200, 1, 3, 2] → 4 (sequence: 1,2,3,4)
```

### Subarray Sum Equals K (Prefix Sum + HashMap)
```javascript
function subarraySum(nums, k) {
  let count = 0, prefix = 0;
  let map = new Map([[0, 1]]);
  for (let num of nums) {
    prefix += num;
    if (map.has(prefix - k)) count += map.get(prefix - k);
    map.set(prefix, (map.get(prefix) || 0) + 1);
  }
  return count;
}
```

---

## 7. Complexity

| Operation | Average | Worst |
|-----------|---------|-------|
| Insert | O(1) | O(n) |
| Lookup | O(1) | O(n) |
| Delete | O(1) | O(n) |

Worst case O(n) = all keys collide. Extremely rare with good hash function.

---

## 8. Common Mistakes
- Using object keys that aren't strings (objects as keys → use Map!)
- Forgetting that JS `{}` keys are always strings
- Not initializing frequency counter properly
- O(n) lookup in array when HashMap gives O(1)

## 9. When NOT to Use
- Need ordered data → BST / Sorted Array
- Need min/max quickly → Heap
- Small data set → array might be simpler and faster

---

## 10. Practice Problems

| Problem | Pattern | Difficulty |
|---------|---------|-----------|
| Two Sum | HashMap lookup | Easy |
| Contains Duplicate | HashSet | Easy |
| Valid Anagram | Frequency Counter | Easy |
| First Unique Character | Frequency Map | Easy |
| Group Anagrams | HashMap + Sort Key | Medium |
| Longest Substring No Repeat | HashMap + Window | Medium |
| Subarray Sum Equals K | Prefix + HashMap | Medium |
| Top K Frequent Elements | HashMap + Heap | Medium |
| Longest Consecutive Sequence | HashSet | Medium |
| LRU Cache | HashMap + DLL | Medium |
