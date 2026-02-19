# JavaScript Roadmap: Part 3 — Data Structures

---

## 1. Arrays — Complete Reference

```javascript
const arr = [1, 2, 3, 4, 5];

// MUTATING methods (change the original array):
arr.push(6);          // Add to end → [1,2,3,4,5,6]
arr.pop();            // Remove from end → returns 6
arr.unshift(0);       // Add to start → [0,1,2,3,4,5]
arr.shift();          // Remove from start → returns 0
arr.splice(2, 1);     // Remove 1 element at index 2 → [1,2,4,5]
arr.splice(2, 0, 3);  // Insert 3 at index 2 → [1,2,3,4,5]
arr.sort((a, b) => a - b); // Sort numerically (DEFAULT sort is string-based!)
arr.reverse();        // Reverse in place

// NON-MUTATING methods (return a new array):
arr.slice(1, 3);      // [2, 3] — extract from index 1 to 3 (exclusive)
arr.concat([6, 7]);   // [1,2,3,4,5,6,7]
arr.flat();           // Flatten nested arrays: [[1,2],[3]] → [1,2,3]
arr.includes(3);      // true
arr.indexOf(3);       // 2 (first occurrence, -1 if not found)
arr.findIndex(x => x > 3); // 3 (index of first match)
arr.find(x => x > 3);      // 4 (value of first match)
```

### The Big 4: map, filter, reduce, forEach
```javascript
const nums = [1, 2, 3, 4, 5];

// MAP: Transform every element → new array
const doubled = nums.map(n => n * 2);         // [2, 4, 6, 8, 10]

// FILTER: Keep elements that pass a test → new array
const evens = nums.filter(n => n % 2 === 0);  // [2, 4]

// REDUCE: Accumulate all elements into a single value
const sum = nums.reduce((acc, n) => acc + n, 0); // 15
// Step by step: acc=0+1=1, acc=1+2=3, acc=3+3=6, acc=6+4=10, acc=10+5=15

// FOREACH: Execute side effect for each element (returns nothing)
nums.forEach(n => console.log(n)); // Prints: 1, 2, 3, 4, 5

// Chaining:
const result = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
    .filter(n => n % 2 === 0)    // [2, 4, 6, 8, 10]
    .map(n => n * n)             // [4, 16, 36, 64, 100]
    .reduce((sum, n) => sum + n, 0); // 220
```

### Common Gotcha: Sort Without Compare Function
```javascript
[10, 9, 2, 1, 100].sort();
// Returns: [1, 10, 100, 2, 9] — WRONG! Default sort is LEXICOGRAPHIC (string-based)

[10, 9, 2, 1, 100].sort((a, b) => a - b);
// Returns: [1, 2, 9, 10, 100] — Correct numeric sort ✓
```

---

## 2. Objects

```javascript
const user = {
    name: "Surya",
    age: 25,
    greet() { return `Hi, I'm ${this.name}`; }
};

// Access
user.name;           // "Surya" (dot notation)
user["name"];        // "Surya" (bracket notation — use for dynamic keys)

// Modify
user.role = "admin"; // Add new property
delete user.age;     // Remove property

// Check existence
"name" in user;                  // true
user.hasOwnProperty("name");    // true (doesn't check prototype chain)

// Iteration
Object.keys(user);     // ["name", "role", "greet"]
Object.values(user);   // ["Surya", "admin", ƒ]
Object.entries(user);  // [["name","Surya"], ["role","admin"], ["greet",ƒ]]

// Computed property names
const key = "color";
const obj = { [key]: "red" }; // { color: "red" }

// Object.freeze (truly immutable — but SHALLOW)
const frozen = Object.freeze({ a: 1, b: { c: 2 } });
// frozen.a = 99;    // Silently fails (or throws in strict mode)
frozen.b.c = 99;     // WORKS! Freeze is shallow. Nested objects are NOT frozen.
```

---

## 3. Map & Set

### Map (Better than Objects for Key-Value Pairs)
```javascript
const map = new Map();
map.set("name", "Surya");
map.set(42, "forty-two");          // ANY type as key! (object keys are always strings)
map.set({ id: 1 }, "object key"); // Even objects as keys!

map.get("name");    // "Surya"
map.has(42);        // true
map.size;           // 3
map.delete(42);

// Iteration (preserves insertion order — objects don't guarantee this!)
for (const [key, value] of map) {
    console.log(key, value);
}

// When to use Map vs Object:
// Map: Frequent additions/deletions, non-string keys, need .size, ordered iteration
// Object: Static shape, JSON serialization, destructuring, prototype methods
```

### Set (Unique Values Only)
```javascript
const set = new Set([1, 2, 3, 3, 3]); // {1, 2, 3} — duplicates removed!
set.add(4);
set.has(3);     // true
set.delete(3);
set.size;       // 3

// Remove duplicates from array:
const unique = [...new Set([1, 1, 2, 3, 3, 4])]; // [1, 2, 3, 4]
```

---

## 4. WeakMap & WeakSet (Garbage-Collection Friendly)

```javascript
// WeakMap: Keys MUST be objects. Keys are weakly held (can be garbage collected).
const cache = new WeakMap();
let element = document.getElementById("btn");
cache.set(element, { clicks: 0 });

element = null; // The DOM element is garbage collected.
// The WeakMap entry is AUTOMATICALLY removed! No memory leak.

// Use case: Caching metadata for DOM elements without preventing GC.
```

---

## 5. JSON

```javascript
const obj = { name: "Surya", age: 25, skills: ["JS", "TS"] };

// Serialize (object → string)
const str = JSON.stringify(obj);
// '{"name":"Surya","age":25,"skills":["JS","TS"]}'

// Pretty print (with indentation)
JSON.stringify(obj, null, 2);

// Parse (string → object)
const parsed = JSON.parse(str);

// GOTCHAS:
// Functions, undefined, and Symbols are SILENTLY DROPPED by JSON.stringify.
JSON.stringify({ fn: () => {}, val: undefined }); // '{}'
// Dates become strings: JSON.parse can't reconstruct Date objects automatically.
// Circular references throw an error.
```

---

## 6. Typed Arrays — Binary Data at C-level Speed

Typed Arrays give JavaScript **direct access to raw binary memory**, like arrays in C. They're essential for WebGL, audio processing, file I/O, and network protocols.

### The Architecture: ArrayBuffer → View
```
TYPED ARRAY ARCHITECTURE:
═════════════════════════
  ArrayBuffer (raw bytes — you can't read/write directly)
       ↓
  TypedArray View (interprets bytes as specific number types)
  
  Think of it like:
  ArrayBuffer = a block of RAM (just bytes: 0101 1010 1100...)
  TypedArray  = a lens that reads those bytes as int8, float32, etc.
```

### Creating Typed Arrays
```javascript
// METHOD 1: Create from length (pre-allocated, filled with zeros)
const int8 = new Int8Array(4);       // 4 bytes: [0, 0, 0, 0]
const float32 = new Float32Array(3); // 12 bytes (4 bytes per float): [0, 0, 0]

// METHOD 2: Create from values
const pixels = new Uint8Array([255, 0, 128, 255]); // RGBA pixel

// METHOD 3: Create from ArrayBuffer (shared memory!)
const buffer = new ArrayBuffer(16);         // 16 raw bytes
const int32View = new Int32Array(buffer);    // Interprets as 4 × 32-bit integers
const uint8View = new Uint8Array(buffer);    // Interprets same bytes as 16 × 8-bit unsigned

int32View[0] = 257;
console.log(uint8View[0]); // 1  — same memory, different interpretation!
console.log(uint8View[1]); // 1  (257 = 0x00000101 in little-endian)
```

### All Typed Array Types
```
TYPE                BYTES   RANGE                        USE CASE
─────────────────────────────────────────────────────────────────
Int8Array           1       -128 to 127                  Signed bytes
Uint8Array          1       0 to 255                     Pixel data, binary files
Uint8ClampedArray   1       0 to 255 (clamped)           Canvas ImageData
Int16Array          2       -32768 to 32767              Audio samples (16-bit)
Uint16Array         2       0 to 65535                   Unicode chars
Int32Array          4       -2^31 to 2^31-1              General integers
Uint32Array         4       0 to 2^32-1                  Color values (ARGB)
Float32Array        4       ±3.4 × 10^38                 WebGL vertices, 3D graphics
Float64Array        8       ±1.8 × 10^308                Scientific computing
BigInt64Array       8       -2^63 to 2^63-1              Large integers
BigUint64Array      8       0 to 2^64-1                  Large unsigned integers
```

### Key Differences from Regular Arrays
```javascript
const regular = [1, 2, 3];
const typed = new Int32Array([1, 2, 3]);

// 1. FIXED SIZE — cannot grow or shrink:
// typed.push(4);      // TypeError! No push/pop/splice on typed arrays
// typed.length = 10;  // Silently fails. Length is immutable.

// 2. SINGLE TYPE ONLY — values are coerced:
const u8 = new Uint8Array([256, -1, 3.9]);
console.log(u8); // [0, 255, 3] — 256 overflows to 0, -1 wraps to 255, 3.9 truncates

// 3. Uint8ClampedArray clamps instead of wrapping:
const clamped = new Uint8ClampedArray([256, -1, 3.9]);
console.log(clamped); // [255, 0, 4] — clamped to 0-255 range, rounds 3.9 to 4

// 4. SAME array methods (except mutating ones):
typed.map(x => x * 2);     // Int32Array [2, 4, 6]
typed.filter(x => x > 1);  // Int32Array [2, 3]
typed.reduce((a, b) => a + b, 0); // 6
typed.forEach(x => console.log(x));
typed.slice(1, 3);          // Int32Array [2, 3]
typed.includes(2);          // true
```

### Real-World: Reading Binary Files
```javascript
// Reading a file as binary data in the browser:
const response = await fetch("/image.png");
const buffer = await response.arrayBuffer();
const bytes = new Uint8Array(buffer);

// Check PNG signature (first 8 bytes):
console.log(bytes.slice(0, 8));
// [137, 80, 78, 71, 13, 10, 26, 10] = PNG header ✓

// DataView — for mixed-type binary structures (like file headers):
const view = new DataView(buffer);
view.getUint8(0);        // Read 1 byte at offset 0
view.getUint16(4, true); // Read 2 bytes at offset 4 (true = little-endian)
view.getFloat32(8);      // Read 4-byte float at offset 8
```
