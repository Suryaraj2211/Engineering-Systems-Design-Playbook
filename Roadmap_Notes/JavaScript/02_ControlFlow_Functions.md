# JavaScript Roadmap: Part 2 — Control Flow & Functions

---

## 1. Conditionals

```javascript
// Standard if/else
if (score >= 90) {
    grade = "A";
} else if (score >= 80) {
    grade = "B";
} else {
    grade = "C";
}

// Ternary (for simple conditions — keep it readable!)
const status = age >= 18 ? "adult" : "minor";

// Switch (strict comparison ===)
switch (day) {
    case "Monday":
    case "Tuesday":
        console.log("Weekday");
        break;  // Without break, execution FALLS THROUGH to next case!
    case "Saturday":
        console.log("Weekend");
        break;
    default:
        console.log("Unknown");
}
```

---

## 2. Loops

```javascript
// for — when you know the count
for (let i = 0; i < 5; i++) {
    console.log(i); // 0, 1, 2, 3, 4
}

// for...of — iterate over VALUES (arrays, strings, maps, sets)
for (const item of [10, 20, 30]) {
    console.log(item); // 10, 20, 30
}

// for...in — iterate over KEYS (object properties)
// WARNING: for...in on arrays gives STRING indices, not numbers!
const obj = { a: 1, b: 2, c: 3 };
for (const key in obj) {
    console.log(key, obj[key]); // "a" 1, "b" 2, "c" 3
}

// while / do-while
let n = 0;
while (n < 3) { n++; }          // Checks BEFORE each iteration
do { n++; } while (n < 3);      // Checks AFTER (always runs at least once)

// Loop control
for (let i = 0; i < 10; i++) {
    if (i === 3) continue;  // Skip this iteration
    if (i === 7) break;     // Exit the loop entirely
    console.log(i);         // 0, 1, 2, 4, 5, 6
}
```

### Common Mistake: Closure in Loops
```javascript
// BUG: All callbacks print 5 (var is function-scoped, shared across iterations)
for (var i = 0; i < 5; i++) {
    setTimeout(() => console.log(i), 100); // 5, 5, 5, 5, 5
}

// FIX: Use let (block-scoped — creates a new i for each iteration)
for (let i = 0; i < 5; i++) {
    setTimeout(() => console.log(i), 100); // 0, 1, 2, 3, 4 ✓
}
```

---

## 3. Functions — The Deep Dive

### Function Declaration (Hoisted)
```javascript
greet("Surya"); // Works! Declarations are hoisted to the top of their scope.

function greet(name) {
    return `Hello, ${name}!`;
}
```

### Function Expression (NOT Hoisted)
```javascript
// greet("Surya"); // ERROR! Cannot access before initialization.
const greet = function(name) {
    return `Hello, ${name}!`;
};
```

### Arrow Functions
```javascript
const add = (a, b) => a + b;           // Implicit return (no braces)
const square = x => x * x;              // Single param: no parentheses needed
const log = () => console.log("Hi");    // No params: empty parens required
const multiLine = (a, b) => {           // Multi-line: braces + explicit return
    const sum = a + b;
    return sum * 2;
};

// CRITICAL DIFFERENCE: Arrow functions do NOT have their own `this`!
const obj = {
    name: "Surya",
    regularMethod() {
        console.log(this.name);          // "Surya" — `this` = obj
        setTimeout(function() {
            console.log(this.name);      // undefined! — `this` = window (or undefined in strict)
        }, 100);
        setTimeout(() => {
            console.log(this.name);      // "Surya" — arrow inherits `this` from outer scope ✓
        }, 100);
    }
};
```

### The `arguments` Object (Legacy — Know It, Don't Use It)
```javascript
// Every NON-ARROW function has a hidden `arguments` object:
function oldStyleSum() {
    console.log(arguments);        // [1, 2, 3] — looks like an array, but ISN'T one!
    console.log(arguments.length); // 3
    console.log(arguments[0]);     // 1
    
    // arguments is an "array-like" object — it has indices and .length, but NO array methods:
    // arguments.map(x => x * 2);  // TypeError: arguments.map is not a function!
    
    // Convert to real array:
    const arr1 = Array.from(arguments);     // Method 1
    const arr2 = [...arguments];            // Method 2
    const arr3 = Array.prototype.slice.call(arguments); // Method 3 (oldest)
    
    return arr1.reduce((sum, n) => sum + n, 0);
}
oldStyleSum(1, 2, 3); // 6

// Arrow functions do NOT have `arguments`:
const arrowFn = () => {
    // console.log(arguments); // ReferenceError: arguments is not defined
};

// MODERN REPLACEMENT — Use rest parameters instead:
function modernSum(...nums) {
    return nums.reduce((sum, n) => sum + n, 0); // nums IS a real array!
}
modernSum(1, 2, 3); // 6

// WHY arguments STILL MATTERS:
// 1. You'll see it in legacy code and libraries.
// 2. Interview question: "What is the arguments object?"
// 3. It reveals how JS internally passes parameters.
```

### Default Parameters, Rest, Spread
```javascript
function createUser(name, role = "viewer", ...permissions) {
    return { name, role, permissions };
}

createUser("Surya", "admin", "read", "write", "delete");
// { name: "Surya", role: "admin", permissions: ["read", "write", "delete"] }

createUser("Guest");
// { name: "Guest", role: "viewer", permissions: [] }
```

### Higher-Order Functions
```javascript
// A function that TAKES a function as an argument:
function repeat(fn, times) {
    for (let i = 0; i < times; i++) fn(i);
}
repeat(i => console.log(`Iteration ${i}`), 3);

// A function that RETURNS a function:
function multiplier(factor) {
    return (number) => number * factor;
}
const double = multiplier(2);
const triple = multiplier(3);
console.log(double(5));  // 10
console.log(triple(5));  // 15
```

---

## 4. `this` — JavaScript's Most Confusing Concept

```
HOW `this` IS DETERMINED:
═════════════════════════
  1. Global scope:         this = window (browser) / global (Node)
  2. Object method:        this = the object that CALLED the method
  3. Constructor (new):    this = the newly created object
  4. Arrow function:       this = inherited from enclosing scope (lexical)
  5. .call() / .apply():   this = whatever you pass as first argument
  6. .bind():              this = permanently bound to first argument
  
  RULE: `this` is determined by HOW a function is CALLED, not WHERE it's defined.
  (Except arrow functions — they lock `this` at DEFINITION time.)
```

```javascript
const user = {
    name: "Surya",
    greet() { return this.name; }
};

user.greet();              // "Surya" — called as object method
const fn = user.greet;    
fn();                      // undefined — called as standalone function!

// Fix with bind:
const bound = user.greet.bind(user);
bound();                   // "Surya" — permanently bound
```

### Function Borrowing — Using Methods From Other Objects

Function borrowing is when you **use a method from one object on a different object** using `call`, `apply`, or `bind`. This is powerful when objects share similar structure but don't share a prototype.

```javascript
const person = {
    fullName() {
        return `${this.firstName} ${this.lastName}`;
    }
};

const employee = {
    firstName: "Surya",
    lastName: "Raj",
    department: "Engineering"
};

// employee doesn't have fullName(), but we can BORROW it from person:
person.fullName.call(employee);   // "Surya Raj" — borrows fullName, sets this = employee
person.fullName.apply(employee);  // "Surya Raj" — same, just different arg passing syntax

// PRACTICAL EXAMPLE: Borrowing Array methods for array-like objects:
const nodeList = document.querySelectorAll("div"); // NodeList (NOT an array!)
// nodeList.map(...)  // TypeError!

// Borrow map from Array.prototype:
Array.prototype.map.call(nodeList, el => el.textContent); // Works!
// Or shorthand:
[].slice.call(nodeList);  // Converts to real array

// Even more practical — borrowing hasOwnProperty safely:
const obj = Object.create(null); // Object with NO prototype!
obj.key = "value";
// obj.hasOwnProperty("key"); // TypeError! No prototype = no inherited methods
Object.prototype.hasOwnProperty.call(obj, "key"); // true ✓ — borrowed safely

// call vs apply vs bind:
const greetFn = function(greeting, punctuation) {
    return `${greeting}, ${this.name}${punctuation}`;
};
const user = { name: "Surya" };

greetFn.call(user, "Hello", "!");       // "Hello, Surya!" — args individually
greetFn.apply(user, ["Hello", "!"]);    // "Hello, Surya!" — args as array
const boundGreet = greetFn.bind(user, "Hello"); // Returns NEW function with this locked
boundGreet("!");                         // "Hello, Surya!"
boundGreet("?");                         // "Hello, Surya?"

// MEMORY AID:
// Call  = Comma separated args      (C for Comma)
// Apply = Array of args             (A for Array)
// Bind  = Returns Bound function    (B for Bound, doesn't execute immediately)
```

---

## 5. Error Handling

```javascript
try {
    const data = JSON.parse(invalidJson);
} catch (error) {
    console.error("Parse failed:", error.message);
    // error.name:    "SyntaxError"
    // error.message: "Unexpected token..."
    // error.stack:   Full stack trace
} finally {
    // Runs ALWAYS — whether error occurred or not
    cleanup();
}

// Custom errors:
class ValidationError extends Error {
    constructor(field, message) {
        super(message);
        this.name = "ValidationError";
        this.field = field;
    }
}

throw new ValidationError("email", "Invalid email format");
```

---

## 6. Built-in Functions — Global Utility Functions

These functions are available globally without any object prefix. They handle parsing, encoding, and type checking.

### Number Parsing & Validation
```javascript
// parseInt — Parses a string into an INTEGER:
parseInt("42");          // 42
parseInt("42.9");        // 42   (truncates decimal, does NOT round)
parseInt("0xFF", 16);    // 255  (second arg = radix/base)
parseInt("111", 2);      // 7    (binary to decimal)
parseInt("42px");        // 42   (stops at first non-numeric character)
parseInt("hello");       // NaN  (no number found)
parseInt("");            // NaN

// GOTCHA: parseInt with leading zeros
parseInt("08");          // 8 (modern JS). In OLD browsers: 0 (treated as octal!)
// ALWAYS pass the radix: parseInt("08", 10) to be safe.

// parseFloat — Parses a string into a FLOAT:
parseFloat("3.14");      // 3.14
parseFloat("3.14.15");   // 3.14 (stops at second dot)
parseFloat("  42  ");    // 42   (trims whitespace)
parseFloat("0.1e2");     // 10   (handles scientific notation)

// Number() — Stricter conversion (no partial parsing):
Number("42px");          // NaN  (unlike parseInt which returns 42!)
Number("");              // 0    (unlike parseInt which returns NaN!)
Number(true);            // 1
Number(false);           // 0
Number(null);            // 0
Number(undefined);       // NaN
```

### NaN & Infinity Checking
```javascript
// isNaN vs Number.isNaN — THE TRAP:
isNaN("hello");          // true  — WRONG! Coerces "hello" to number first → NaN
isNaN("42");             // false — "42" coerces to 42, which is not NaN
isNaN(undefined);        // true  — undefined coerces to NaN

Number.isNaN("hello");   // false ✓ — No coercion! "hello" is a string, not NaN
Number.isNaN(NaN);       // true  ✓ — The ONLY value that returns true
Number.isNaN(0 / 0);    // true  ✓ — 0/0 is NaN

// RULE: ALWAYS use Number.isNaN() instead of isNaN().

// isFinite vs Number.isFinite:
isFinite("42");                // true  — coerces string to number first
Number.isFinite("42");         // false ✓ — no coercion, "42" is not a number
Number.isFinite(Infinity);     // false
Number.isFinite(-Infinity);    // false
Number.isFinite(42);           // true

// Number.isInteger:
Number.isInteger(42);          // true
Number.isInteger(42.0);        // true  (42.0 === 42 in JS)
Number.isInteger(42.5);        // false
```

### URI Encoding/Decoding (For URLs)
```javascript
// encodeURIComponent — Encode a PART of a URL (query params, values):
encodeURIComponent("hello world & goodbye"); // "hello%20world%20%26%20goodbye"
encodeURIComponent("surya@email.com");       // "surya%40email.com"

// decodeURIComponent — Reverse the encoding:
decodeURIComponent("hello%20world"); // "hello world"

// encodeURI — Encode a FULL URL (preserves :, /, ?, #, &):
encodeURI("https://example.com/search?q=hello world");
// "https://example.com/search?q=hello%20world" (only encodes the space)

// decodeURI — Reverse:
decodeURI("https://example.com/search?q=hello%20world");
// "https://example.com/search?q=hello world"

// WHEN TO USE WHICH:
// encodeURIComponent: For individual values (query params, form data)
// encodeURI:          For complete URLs where you want to keep structure intact
```

### Other Global Functions
```javascript
// eval — Executes a string as code (NEVER USE IN PRODUCTION — security risk!)
eval("2 + 2");           // 4
eval("alert('hacked')"); // XSS vulnerability! User input → code execution

// setTimeout / setInterval (technically Window methods, but globally available):
setTimeout(() => console.log("After 1s"), 1000);
const id = setInterval(() => console.log("Every 1s"), 1000);
clearInterval(id); // Stop the interval

// structuredClone — Deep copy any object (modern, handles circular refs):
const original = { a: 1, b: { c: 2 } };
const deep = structuredClone(original);
deep.b.c = 999;
console.log(original.b.c); // 2 — original is untouched ✓
```
