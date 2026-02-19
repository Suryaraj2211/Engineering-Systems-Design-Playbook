# JavaScript Roadmap: Part 1 — Introduction & Fundamentals

**Goal:** Understand how JavaScript actually works under the hood, not just the syntax.

---

## 1. What Is JavaScript?

JavaScript is a **dynamic, weakly-typed, prototype-based, single-threaded** language with a **non-blocking event loop**. It runs in browsers (V8/SpiderMonkey/JavaScriptCore) and on servers (Node.js).

```
HOW JS EXECUTES:
════════════════
  Source Code (.js)
       ↓
  Parser (creates AST — Abstract Syntax Tree)
       ↓
  Interpreter (Ignition in V8) → Bytecode (runs immediately)
       ↓
  If a function is called MANY times ("hot"):
       ↓
  JIT Compiler (TurboFan in V8) → Optimized Machine Code (runs 10-100× faster)
       ↓
  If assumptions break (e.g., type changes):
       ↓
  DEOPTIMIZATION → Falls back to interpreter (performance cliff!)
```

---

## 2. Variables: var, let, const

### var (Legacy — Avoid in Modern Code)
```javascript
var x = 10;
var x = 20;     // No error! Re-declaration allowed. This causes bugs.
console.log(x); // 20

// var is FUNCTION-scoped, not block-scoped:
if (true) {
    var leaked = "I'm visible outside!";
}
console.log(leaked); // "I'm visible outside!" — var LEAKS out of blocks!
```

### let (Modern — Use for Reassignable Variables)
```javascript
let y = 10;
y = 20;          // OK — reassignment allowed
// let y = 30;   // ERROR! Re-declaration in same scope is NOT allowed

if (true) {
    let blocked = "I stay inside";
}
// console.log(blocked); // ERROR! let is BLOCK-scoped
```

### const (Modern — Use for Everything Else)
```javascript
const z = 10;
// z = 20;        // ERROR! Cannot reassign const

// BUT: const objects/arrays CAN be MUTATED (the reference is constant, not the value):
const user = { name: "Surya" };
user.name = "Raj";     // ALLOWED! We didn't reassign `user`, just changed its content.
user.age = 25;         // ALLOWED! Adding new properties is fine.
// user = {};          // ERROR! Can't point `user` to a different object.

const arr = [1, 2, 3];
arr.push(4);            // ALLOWED! arr still points to the same array.
// arr = [5, 6];        // ERROR! Can't reassign.
```

### The Rule
```
ALWAYS use const.
Use let ONLY when you need to reassign.
NEVER use var.
```

---

## 3. Data Types

### Primitives (Immutable, Stored by Value)
```javascript
// 7 primitive types:
typeof 42;           // "number"  (64-bit float — ALL numbers are floats!)
typeof "hello";      // "string"
typeof true;         // "boolean"
typeof undefined;    // "undefined" (variable declared but not assigned)
typeof null;         // "object"    ← FAMOUS BUG from 1995, never fixed!
typeof Symbol();     // "symbol"    (unique identifiers)
typeof 9007199254740991n; // "bigint" (arbitrary precision integers)

// NUMBER GOTCHAS:
0.1 + 0.2;           // 0.30000000000000004 (IEEE 754 float precision!)
Number.MAX_SAFE_INTEGER; // 9007199254740991 (2^53 - 1)
9007199254740992 === 9007199254740993; // true! (UNSAFE — beyond precision)

// Fix: Use Math.round, toFixed, or BigInt for large integers.
(0.1 * 10 + 0.2 * 10) / 10; // 0.3 ✓ (integer math, then divide)
```

### Reference Types (Mutable, Stored by Reference)
```javascript
const a = { x: 1 };
const b = a;        // b and a point to THE SAME object in memory
b.x = 999;
console.log(a.x);   // 999 — a was also modified!

// To create a TRUE COPY:
const c = { ...a };             // Shallow copy (spread operator)
const d = structuredClone(a);   // Deep copy (nested objects too)
```

---

## 4. Type Coercion — JavaScript's Most Dangerous Feature

```javascript
// Implicit coercion (JS "helpfully" converts types):
"5" + 3;      // "53"   (number → string, because + is also concatenation!)
"5" - 3;      // 2      (string → number, because - is ONLY math)
"5" * "3";    // 15     (both → number)
true + true;  // 2      (true → 1)
[] + [];       // ""     (both → "")
[] + {};       // "[object Object]"
{} + [];       // 0      (depends on context!)

// EQUALITY CHAOS:
"0" == false;  // true  (coercion!)
"0" === false; // false (strict — no coercion)
null == undefined; // true  (special case)
null === undefined; // false

// THE RULE: ALWAYS use === and !== (strict equality). NEVER use == or !=.
```

### Equality Algorithms — Under the Hood

The ECMAScript spec defines **4 equality algorithms**. Knowing them helps you predict exactly what JS will do:

```
ALGORITHM COMPARISON:
═════════════════════
  Algorithm            Triggered By           NaN === NaN?    +0 === -0?
  ─────────────────────────────────────────────────────────────────────
  IsLooselyEqual       ==                     false           true
  IsStrictlyEqual      ===                    false           true
  SameValue            Object.is()            true ✓          false ✓
  SameValueZero        Map/Set key lookup     true ✓          true
```

```javascript
// IsStrictlyEqual (===) — No type coercion, but has quirks:
NaN === NaN;        // false! (the ONLY value not equal to itself)
+0 === -0;          // true   (they ARE different in IEEE 754!)

// SameValue — Object.is() fixes both quirks:
Object.is(NaN, NaN);  // true ✓ (correctly identifies NaN)
Object.is(+0, -0);    // false ✓ (correctly distinguishes signed zeros)

// SameValueZero — Used internally by Map, Set, Array.includes():
// Same as SameValue, except +0 and -0 are treated as equal.
new Set([+0, -0]).size;    // 1 (SameValueZero treats them as same key)
[NaN].includes(NaN);       // true! (includes uses SameValueZero, not ===)
[NaN].indexOf(NaN);        // -1!   (indexOf uses ===, so NaN is never found)

// PRACTICAL TAKEAWAY:
// Use === for 99% of comparisons.
// Use Object.is() when you need to distinguish +0/-0 or check NaN.
// Know that Map/Set/includes handle NaN correctly (SameValueZero).
```

---

## 5. Hoisting — The Invisible Code Rearrangement

Before ANY code runs, JavaScript "hoists" (moves) declarations to the top of their scope. But **only the declaration, not the initialization.**

```javascript
// WHAT YOU WRITE:                    WHAT THE ENGINE SEES:
console.log(x);     // undefined      var x;              // Declaration hoisted!
var x = 5;                            console.log(x);     // undefined (not yet assigned)
console.log(x);     // 5              x = 5;
                                      console.log(x);     // 5
```

### var Hoisting (Initialized as `undefined`)
```javascript
console.log(a);  // undefined — var is hoisted AND initialized to undefined
var a = 10;
console.log(a);  // 10

// This is WHY var is dangerous — it silently gives you undefined instead of erroring.
```

### let & const Hoisting (Temporal Dead Zone)
```javascript
// let and const ARE hoisted, but they are NOT initialized.
// The gap between hoisting and declaration is called the "Temporal Dead Zone" (TDZ).

// console.log(b); // ReferenceError: Cannot access 'b' before initialization
let b = 20;        // TDZ ends here — b is now initialized
console.log(b);    // 20

// Same for const:
// console.log(c); // ReferenceError!
const c = 30;
```

### Function Hoisting (Fully Hoisted!)
```javascript
// Function DECLARATIONS are hoisted entirely — you can call them before they appear:
sayHi();  // "Hi!" ✓ — works because the ENTIRE function is hoisted

function sayHi() {
    console.log("Hi!");
}

// Function EXPRESSIONS are NOT hoisted (they follow var/let/const rules):
// greet(); // TypeError: greet is not a function (var) or ReferenceError (let/const)
var greet = function() { console.log("Hello!"); };
```

### Class Hoisting (NOT Hoisted in Practice)
```javascript
// Classes are hoisted but NOT initialized (like let/const — TDZ applies):
// const dog = new Animal(); // ReferenceError!
class Animal { }
const dog = new Animal();    // Works ✓
```

### The Complete Hoisting Table
```
HOISTING BEHAVIOR:
══════════════════
  Declaration          Hoisted?    Initialized?    TDZ?     Can Use Before Declaration?
  ────────────────────────────────────────────────────────────────────────────────────
  var                  YES         YES (undefined) NO       YES (value is undefined)
  let                  YES         NO              YES      NO (ReferenceError)
  const                YES         NO              YES      NO (ReferenceError)
  function decl.       YES         YES (full fn)   NO       YES (fully works ✓)
  function expr.       Follows var/let/const rules of the variable it's assigned to
  class                YES         NO              YES      NO (ReferenceError)

  RULE: Always declare at the top of your scope. Don't rely on hoisting.
```

---

## 6. Strings

```javascript
// Template literals (backticks) — the modern way:
const name = "Surya";
const greeting = `Hello, ${name}! 2 + 2 = ${2 + 2}`; // "Hello, Surya! 2 + 2 = 4"

// Essential string methods:
"Hello World".length;            // 11
"Hello World".toUpperCase();     // "HELLO WORLD"
"Hello World".includes("World"); // true
"Hello World".indexOf("World");  // 6
"Hello World".slice(0, 5);       // "Hello" (start, end-exclusive)
"Hello World".split(" ");        // ["Hello", "World"]
"  spaces  ".trim();             // "spaces"
"Hello".padStart(10, "-");       // "-----Hello"
"abc".repeat(3);                 // "abcabcabc"
"Hello World".replace("World", "JS"); // "Hello JS"
"Hello World".replaceAll("l", "L");   // "HeLLo WorLd"

// Strings are IMMUTABLE:
let s = "hello";
s[0] = "H";       // Silently fails! s is still "hello"
s = "H" + s.slice(1); // "Hello" — creates a NEW string
```

---

## 7. Operators

```javascript
// Nullish coalescing (??) — use default ONLY if null/undefined:
const val = null ?? "default";      // "default"
const val2 = 0 ?? "default";       // 0 (zero is NOT nullish!)
const val3 = "" ?? "default";      // "" (empty string is NOT nullish!)
// Compare with || which treats 0, "", false as falsy:
const val4 = 0 || "default";       // "default" — WRONG if 0 is a valid value!

// Optional chaining (?.) — safe property access:
const user = { address: { city: "Chennai" } };
user?.address?.city;    // "Chennai"
user?.phone?.number;    // undefined (no error, even though phone doesn't exist)

// Spread operator (...):
const arr1 = [1, 2, 3];
const arr2 = [...arr1, 4, 5]; // [1, 2, 3, 4, 5]
const obj1 = { a: 1, b: 2 };
const obj2 = { ...obj1, c: 3 }; // { a: 1, b: 2, c: 3 }

// Destructuring:
const [first, second, ...rest] = [1, 2, 3, 4, 5]; // first=1, second=2, rest=[3,4,5]
const { name: userName, age = 25 } = { name: "Surya" }; // userName="Surya", age=25 (default)
```

---

## 8. Bitwise Operators — Working at the Binary Level

Bitwise operators treat numbers as **32-bit integers** and operate on individual bits. Rare in daily code, but essential for flags, permissions, and low-level optimization.

```javascript
// BINARY BASICS:
// Decimal 5 = Binary 0101
// Decimal 3 = Binary 0011

// AND (&) — Both bits must be 1:
5 & 3;    // 1     (0101 & 0011 = 0001)

// OR (|) — Either bit can be 1:
5 | 3;    // 7     (0101 | 0011 = 0111)

// XOR (^) — Exactly ONE bit must be 1:
5 ^ 3;    // 6     (0101 ^ 0011 = 0110)

// NOT (~) — Flip all bits (also adds 1 and negates):
~5;       // -6    (~x = -(x+1))
~-1;      // 0     (useful trick: ~arr.indexOf(x) is truthy if found)

// LEFT SHIFT (<<) — Shift bits left (multiply by 2^n):
5 << 1;   // 10    (0101 << 1 = 1010) — same as 5 * 2
5 << 3;   // 40    — same as 5 * 8 (2^3)

// RIGHT SHIFT (>>) — Shift bits right (divide by 2^n, keeps sign):
20 >> 2;  // 5     — same as Math.floor(20 / 4)
-20 >> 2; // -5    — preserves the sign bit

// UNSIGNED RIGHT SHIFT (>>>) — Shift right, fills with 0 (no sign):
-1 >>> 0; // 4294967295 (all 32 bits set to 1, interpreted as unsigned)
```

### Real-World Use Cases
```javascript
// 1. PERMISSION FLAGS (like Unix file permissions):
const READ    = 0b001;  // 1
const WRITE   = 0b010;  // 2
const EXECUTE = 0b100;  // 4

let userPerms = READ | WRITE;          // Combine: 0b011 (3)
const canRead = (userPerms & READ) !== 0;   // Check: true
userPerms = userPerms ^ WRITE;          // Toggle WRITE off: 0b001 (1)
userPerms = userPerms & ~EXECUTE;       // Remove EXECUTE: 0b001 (1)

// 2. FAST FLOOR (for positive numbers only):
const floored = 9.7 | 0;   // 9 (truncates decimal — faster than Math.floor)
const floored2 = ~~9.7;    // 9 (double NOT — same effect)

// 3. SWAP WITHOUT TEMP VARIABLE:
let a = 5, b = 3;
a ^= b; b ^= a; a ^= b;  // Now a=3, b=5
```

---

## 9. BigInt Operators — Arbitrary Precision Math

BigInt lets you work with integers **larger than 2^53-1** (Number.MAX_SAFE_INTEGER). BigInt operators work like regular operators, but with restrictions.

```javascript
// Creating BigInts:
const big1 = 9007199254740993n;    // Suffix with 'n'
const big2 = BigInt("9007199254740993"); // From string
const big3 = BigInt(42);           // From number (must be integer)

// Arithmetic — works like normal, but BOTH operands must be BigInt:
10n + 20n;     // 30n
100n - 50n;    // 50n
10n * 20n;     // 200n
20n / 3n;      // 6n  (truncates! No decimals in BigInt — like integer division)
20n % 3n;      // 2n
2n ** 100n;    // 1267650600228229401496703205376n (huge number, no problem!)

// Comparison — CAN compare BigInt with Number:
10n === 10;    // false! (different types — strict equality)
10n == 10;     // true   (loose equality does type coercion)
10n > 5;       // true   (comparison operators work across types)
10n < 20;      // true

// CANNOT MIX BigInt and Number in arithmetic:
// 10n + 5;    // TypeError: Cannot mix BigInt and other types!
// FIX:
10n + BigInt(5);     // 15n
Number(10n) + 5;     // 15 (but loses precision for huge numbers!)

// Bitwise operators work with BigInt too:
5n & 3n;    // 1n
5n | 3n;    // 7n
5n << 2n;   // 20n

// WHAT DOESN'T WORK:
// Math.sqrt(16n);  // TypeError! Math functions don't accept BigInt
// +10n;            // TypeError! Unary + doesn't work on BigInt
// JSON.stringify(10n); // TypeError! BigInt is not serializable to JSON
```

---

## 10. Comma Operator — Evaluate Multiple Expressions

The comma operator evaluates **multiple expressions** left-to-right and returns the **last one**. Rarely used directly, but shows up in `for` loops and minified code.

```javascript
// Basic: Evaluates all expressions, returns the LAST value:
const result = (1 + 2, 3 + 4, 5 + 6);  // result = 11 (only last expression returned)

// Most common use — multiple variables in a for loop:
for (let i = 0, j = 10; i < j; i++, j--) {
    console.log(i, j); // 0 10, 1 9, 2 8, 3 7, 4 6
}

// In arrow functions (avoid — hard to read):
const sideEffectAndReturn = (x) => (console.log(x), x * 2);
sideEffectAndReturn(5); // Logs: 5, Returns: 10

// CAUTION: Don't confuse with commas in declarations, arguments, or arrays.
// These are NOT the comma operator:
let a = 1, b = 2;               // Variable declaration list
fn(1, 2, 3);                    // Function arguments
const arr = [1, 2, 3];          // Array literal
```

---

## 11. Built-in Objects — JavaScript's Standard Library

These objects are available globally without importing anything.

### Math (No Constructor — All Static Methods)
```javascript
Math.PI;              // 3.141592653589793
Math.E;               // 2.718281828459045

Math.round(4.5);      // 5   (standard rounding)
Math.ceil(4.1);       // 5   (always rounds UP)
Math.floor(4.9);      // 4   (always rounds DOWN)
Math.trunc(4.9);      // 4   (removes decimal — same as floor for positives)
Math.trunc(-4.9);     // -4  (NOT -5! trunc ≠ floor for negatives)

Math.abs(-42);        // 42  (absolute value)
Math.max(1, 5, 3);    // 5
Math.min(1, 5, 3);    // 1
Math.pow(2, 10);      // 1024 (same as 2 ** 10)
Math.sqrt(144);       // 12
Math.cbrt(27);        // 3   (cube root)
Math.log2(1024);      // 10

// Random number between min (inclusive) and max (exclusive):
function randomInt(min, max) {
    return Math.floor(Math.random() * (max - min)) + min;
}
randomInt(1, 7);  // Random dice roll: 1, 2, 3, 4, 5, or 6
```

### Date
```javascript
const now = new Date();           // Current date/time
const specific = new Date(2025, 0, 15); // Jan 15, 2025 (months are 0-indexed!)
const fromString = new Date("2025-01-15T10:30:00");

now.getFullYear();   // 2025
now.getMonth();      // 0-11 (0 = January! Add 1 for human-readable)
now.getDate();       // 1-31 (day of month)
now.getDay();        // 0-6  (0 = Sunday)
now.getHours();      // 0-23
now.getTime();       // Milliseconds since Jan 1, 1970 (Unix epoch)

// Date math:
Date.now();                              // Current timestamp (ms)
new Date(Date.now() + 86400000);        // Tomorrow (add 24h in ms)

// Formatting:
now.toLocaleDateString("en-IN");        // "15/1/2025" (locale-dependent)
now.toISOString();                      // "2025-01-15T10:30:00.000Z" (standard)

// GOTCHA: Date is MUTABLE. Use libraries (date-fns, Temporal API) for serious date work.
const d = new Date();
d.setFullYear(2030);  // Modifies d in place! No new Date created.
```

### Error Objects
```javascript
// Built-in error types:
new Error("Generic error");         // Base type
new TypeError("Wrong type");        // Type mismatch (e.g., calling non-function)
new RangeError("Out of range");     // Value out of allowed range
new ReferenceError("Not defined");  // Accessing undefined variable
new SyntaxError("Bad syntax");      // Invalid code structure
new URIError("Bad URI");            // Malformed URI
new EvalError("Eval failed");       // eval() related (rare)

// All errors have:
const err = new TypeError("x is not a function");
err.name;      // "TypeError"
err.message;   // "x is not a function"
err.stack;     // Full stack trace (non-standard but universally supported)
```
