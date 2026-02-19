# Chapter 05 — Strings

## Simple Explanation (ELI5)
A string is just **a sequence of characters** — letters, numbers, symbols — stored in order. Think of it as an array of characters with special rules.

## Technical Definition
A string is an **immutable** (in most languages) sequence of characters, typically stored as an array of character codes (UTF-16 in JavaScript, Unicode in Python).

## Why It's Needed
Almost every application deals with text: user input, file paths, URLs, parsing data, matching patterns, searching documents.

## Real-world Analogy
A **train** with labeled carriages. Each carriage (character) has a position, and you can't swap carriages without building a new train (immutability).

---

## 1. Strings Are Immutable

```javascript
let s = "hello";
s[0] = "H";        // Does nothing! Strings are immutable in JS
console.log(s);     // "hello" — unchanged

// Must create a new string
let updated = "H" + s.slice(1);  // "Hello"
```

```python
s = "hello"
# s[0] = "H"  # TypeError! Strings are immutable in Python
updated = "H" + s[1:]  # "Hello"
```

**Important implication:** String concatenation in a loop creates **n new strings** → O(n²).

```javascript
// BAD — O(n²)
let result = "";
for (let i = 0; i < n; i++) {
  result += "a";  // Creates new string each time!
}

// GOOD — O(n)
let parts = [];
for (let i = 0; i < n; i++) {
  parts.push("a");
}
let result = parts.join("");
```

---

## 2. Essential String Methods

```javascript
let s = "Hello, World!";

// --- Access ---
s.length;              // 13
s[0];                  // "H"
s.charAt(0);           // "H"
s.charCodeAt(0);       // 72 (ASCII/UTF-16 code)
String.fromCharCode(72); // "H"

// --- Search ---
s.indexOf("World");     // 7
s.lastIndexOf("l");     // 10
s.includes("Hello");    // true
s.startsWith("Hello");  // true
s.endsWith("!");        // true

// --- Extract ---
s.slice(0, 5);         // "Hello"
s.slice(-6);           // "orld!"
s.substring(7, 12);    // "World"

// --- Transform ---
s.toLowerCase();       // "hello, world!"
s.toUpperCase();       // "HELLO, WORLD!"
s.trim();              // removes whitespace from both ends
s.replace("World", "DSA"); // "Hello, DSA!"
s.replaceAll("l", "L"); // "HeLLo, WorLd!"

// --- Split / Join ---
"a,b,c".split(",");    // ["a", "b", "c"]
["a","b","c"].join("-"); // "a-b-c"

// --- String ↔ Array ---
let chars = [...s];     // Array of characters
let arr = s.split("");  // Same thing
let back = chars.join(""); // Back to string

// --- Repeat ---
"ab".repeat(3);         // "ababab"
```

```python
s = "Hello, World!"

len(s)              # 13
s[0]                # "H"
ord("H")            # 72 (Unicode code point)
chr(72)             # "H"

s.find("World")     # 7
s.lower()           # "hello, world!"
s.upper()           # "HELLO, WORLD!"
s.strip()           # trim whitespace
s.replace("World", "DSA")
s.split(", ")       # ["Hello", "World!"]
", ".join(["a","b"]) # "a, b"
s[::-1]             # "!dlroW ,olleH" (reverse)
s.isalpha()         # False (has comma, space)
s.isdigit()         # False
s.isalnum()         # False
```

---

## 3. Character Code Tricks

```javascript
// Check if character is lowercase letter
function isLower(ch) { return ch >= 'a' && ch <= 'z'; }

// Convert letter to index (a=0, b=1, ..., z=25)
function charToIndex(ch) { return ch.charCodeAt(0) - 'a'.charCodeAt(0); }

// Frequency array for lowercase letters
function charFrequency(s) {
  let freq = new Array(26).fill(0);
  for (let ch of s) {
    freq[ch.charCodeAt(0) - 97]++; // 97 = 'a'
  }
  return freq;
}
```

---

## 4. Classic String Problems

### Valid Palindrome

```javascript
function isPalindrome(s) {
  s = s.toLowerCase().replace(/[^a-z0-9]/g, '');
  let l = 0, r = s.length - 1;
  while (l < r) {
    if (s[l] !== s[r]) return false;
    l++; r--;
  }
  return true;
}
// Time: O(n), Space: O(n) for cleaned string
```

### Valid Anagram

```javascript
function isAnagram(s, t) {
  if (s.length !== t.length) return false;
  let freq = new Array(26).fill(0);
  for (let i = 0; i < s.length; i++) {
    freq[s.charCodeAt(i) - 97]++;
    freq[t.charCodeAt(i) - 97]--;
  }
  return freq.every(f => f === 0);
}
```

```python
def is_anagram(s, t):
    return sorted(s) == sorted(t)  # O(n log n)
    # OR: Counter(s) == Counter(t)  # O(n)
```

### Longest Palindromic Substring (Expand from Center)

```javascript
function longestPalindrome(s) {
  let start = 0, maxLen = 0;

  function expand(l, r) {
    while (l >= 0 && r < s.length && s[l] === s[r]) {
      if (r - l + 1 > maxLen) {
        start = l;
        maxLen = r - l + 1;
      }
      l--; r++;
    }
  }

  for (let i = 0; i < s.length; i++) {
    expand(i, i);     // Odd length palindromes
    expand(i, i + 1); // Even length palindromes
  }

  return s.substring(start, start + maxLen);
}
// Time: O(n²), Space: O(1)
```

### Longest Common Prefix

```javascript
function longestCommonPrefix(strs) {
  if (strs.length === 0) return "";
  let prefix = strs[0];

  for (let i = 1; i < strs.length; i++) {
    while (strs[i].indexOf(prefix) !== 0) {
      prefix = prefix.slice(0, -1);
      if (prefix === "") return "";
    }
  }
  return prefix;
}
// ["flower","flow","flight"] → "fl"
```

---

## 5. Pattern Matching Algorithms

### Brute Force — O(n × m)

```javascript
function bruteForceSearch(text, pattern) {
  for (let i = 0; i <= text.length - pattern.length; i++) {
    let match = true;
    for (let j = 0; j < pattern.length; j++) {
      if (text[i + j] !== pattern[j]) { match = false; break; }
    }
    if (match) return i;
  }
  return -1;
}
```

### KMP Algorithm — O(n + m)

**Key idea:** When mismatch occurs, don't restart from scratch. Use the **LPS (Longest Prefix Suffix) array** to know where to resume.

```javascript
function kmpSearch(text, pattern) {
  let lps = buildLPS(pattern);
  let i = 0, j = 0;

  while (i < text.length) {
    if (text[i] === pattern[j]) {
      i++; j++;
      if (j === pattern.length) return i - j; // Found!
    } else if (j > 0) {
      j = lps[j - 1]; // Don't reset j to 0, use LPS
    } else {
      i++;
    }
  }
  return -1;
}

function buildLPS(pattern) {
  let lps = new Array(pattern.length).fill(0);
  let len = 0, i = 1;

  while (i < pattern.length) {
    if (pattern[i] === pattern[len]) {
      len++;
      lps[i] = len;
      i++;
    } else if (len > 0) {
      len = lps[len - 1];
    } else {
      lps[i] = 0;
      i++;
    }
  }
  return lps;
}

// LPS for "AAACAAAA":
// Pattern: A A A C A A A A
// LPS:     0 1 2 0 1 2 3 3
```

### Rabin-Karp — O(n + m) average

Uses **rolling hash** to compare pattern with substrings.

```javascript
function rabinKarp(text, pattern) {
  let m = pattern.length, n = text.length;
  let base = 256, mod = 1e9 + 7;

  let patternHash = 0, textHash = 0, power = 1;

  // Calculate hash of pattern and first window
  for (let i = 0; i < m; i++) {
    patternHash = (patternHash * base + pattern.charCodeAt(i)) % mod;
    textHash = (textHash * base + text.charCodeAt(i)) % mod;
    if (i > 0) power = (power * base) % mod;
  }

  for (let i = 0; i <= n - m; i++) {
    if (patternHash === textHash) {
      // Hash match → verify character by character
      if (text.substring(i, i + m) === pattern) return i;
    }
    if (i < n - m) {
      // Slide window: remove leftmost, add new rightmost
      textHash = ((textHash - text.charCodeAt(i) * power) * base
                  + text.charCodeAt(i + m)) % mod;
      if (textHash < 0) textHash += mod;
    }
  }
  return -1;
}
```

---

## 6. String Pattern Comparison

| Algorithm | Time | Space | Best For |
|-----------|------|-------|----------|
| Brute Force | O(n × m) | O(1) | Short patterns |
| KMP | O(n + m) | O(m) | Guaranteed linear |
| Rabin-Karp | O(n + m) avg | O(1) | Multiple pattern search |
| Built-in `indexOf` | O(n × m) worst | O(1) | General purpose |

---

## 7. Common Mistakes

| Mistake | Fix |
|---------|-----|
| String concat in loop → O(n²) | Use array + join |
| Forgetting strings are immutable | Always create new string |
| Not handling empty string `""` | Check `s.length === 0` |
| Comparing with `==` (loose) | Use `===` always |
| Not handling Unicode (emojis, non-Latin) | Use `[...s]` to get true characters |

---

## 8. Edge Cases

- Empty string `""`
- Single character `"a"`
- All same characters `"aaaa"`
- String with spaces, punctuation, unicode
- Case sensitivity (`"A" !== "a"`)
- Very long strings (need O(n) solution, not O(n²))

---

## 9. Practice Problems

| Problem | Pattern | Difficulty |
|---------|---------|-----------|
| Valid Palindrome | Two Pointer | Easy |
| Valid Anagram | Frequency Count | Easy |
| First Unique Character | Hash Map | Easy |
| Longest Common Prefix | Iteration | Easy |
| Longest Palindromic Substring | Expand from Center | Medium |
| Group Anagrams | HashMap + Sort Key | Medium |
| Longest Substring No Repeat | Sliding Window | Medium |
| String to Integer (atoi) | Parsing | Medium |
| Minimum Window Substring | Sliding Window | Hard |
| Edit Distance | 2D DP | Hard |
| KMP Pattern Matching | KMP | Medium |
| Implement strStr() | Pattern Match | Easy |
