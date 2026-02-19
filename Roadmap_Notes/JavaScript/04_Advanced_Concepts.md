# JavaScript Roadmap: Part 4 — Advanced Concepts

---

## 1. Asynchronous JavaScript

### The Event Loop — How JS Handles Concurrency
```
EVENT LOOP ARCHITECTURE:
════════════════════════
  ┌─────────────┐     ┌──────────────┐
  │  CALL STACK  │     │   WEB APIs   │
  │ (sync code)  │────►│ setTimeout   │
  │              │     │ fetch         │
  │ main()       │     │ DOM events   │
  └──────┬───────┘     └──────┬───────┘
         │                    │
         │   ┌────────────────▼───────────────┐
         │   │        TASK QUEUES              │
         │   │                                  │
         │   │  Microtask Queue (PRIORITY):     │
         │   │    Promise.then, queueMicrotask  │
         │   │                                  │
         │   │  Macrotask Queue:                │
         │   │    setTimeout, setInterval,      │
         │   │    I/O callbacks                 │
         │   └────────────────┬───────────────┘
         │                    │
         └◄───EVENT LOOP──────┘
         
  ORDER: Stack empties → ALL microtasks → 1 macrotask → ALL microtasks → repeat
```

### Execution Order Quiz
```javascript
console.log("1");
setTimeout(() => console.log("2"), 0);  // Macrotask
Promise.resolve().then(() => console.log("3")); // Microtask
console.log("4");

// Output: 1, 4, 3, 2
// WHY: 1 and 4 are synchronous (call stack).
//      3 is a microtask (runs before ANY macrotasks).
//      2 is a macrotask (runs last, even with 0ms delay).
```

### Promises — Complete Pattern
```javascript
// Creating a promise:
function fetchUser(id) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            if (id > 0) resolve({ id, name: "Surya" });
            else reject(new Error("Invalid ID"));
        }, 1000);
    });
}

// Consuming:
fetchUser(1)
    .then(user => console.log(user.name))
    .catch(err => console.error(err.message))
    .finally(() => console.log("Done")); // Runs always

// Promise combinators:
Promise.all([p1, p2, p3]);    // Wait for ALL. Fails if ANY fails.
Promise.allSettled([p1, p2]);  // Wait for ALL. Never fails. Returns status for each.
Promise.race([p1, p2]);       // Returns FIRST to settle (resolve OR reject).
Promise.any([p1, p2]);        // Returns FIRST to RESOLVE. Fails only if ALL fail.
```

### Async/Await — Clean Async Code
```javascript
async function getUser() {
    try {
        const response = await fetch("/api/user/1");
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        const user = await response.json();
        return user;
    } catch (error) {
        console.error("Failed:", error.message);
        return null;
    }
}

// Parallel execution (DON'T await sequentially when operations are independent!):
// BAD — sequential (takes 2 seconds):
const a = await fetchA(); // 1 second
const b = await fetchB(); // 1 second

// GOOD — parallel (takes 1 second):
const [a, b] = await Promise.all([fetchA(), fetchB()]);
```

---

## 2. Closures — Deep Understanding

A closure is a function that **remembers its outer scope** even after the outer function has returned.

```javascript
function createCounter() {
    let count = 0; // This variable is "closed over"
    
    return {
        increment: () => ++count,
        decrement: () => --count,
        getCount: () => count
    };
}

const counter = createCounter();
counter.increment(); // 1
counter.increment(); // 2
counter.decrement(); // 1
counter.getCount();  // 1

// `count` is private! No way to access it directly from outside.
// This is the MODULE PATTERN — encapsulation without classes.
```

### Common Interview Question: Closure in Loops
```javascript
function createFunctions() {
    const fns = [];
    for (var i = 0; i < 3; i++) {
        fns.push(() => i); // All closures share the SAME `i` (var is function-scoped)
    }
    return fns;
}
createFunctions().map(fn => fn()); // [3, 3, 3] — all see i=3 after loop ends!

// FIX 1: Use let (block-scoped — each iteration gets its own `i`)
for (let i = 0; i < 3; i++) { fns.push(() => i); } // [0, 1, 2] ✓

// FIX 2: IIFE (Immediately Invoked Function Expression)
for (var i = 0; i < 3; i++) {
    fns.push((function(j) { return () => j; })(i)); // [0, 1, 2] ✓
}
```

---

## 3. Prototypes & Classes

### Prototype Chain
```javascript
const animal = { eats: true };
const rabbit = Object.create(animal); // rabbit's prototype = animal
rabbit.jumps = true;

rabbit.eats;  // true (found on prototype chain, not on rabbit directly)
rabbit.jumps; // true (found on rabbit directly)

// The chain: rabbit → animal → Object.prototype → null
```

### Classes (ES6 — Syntactic Sugar Over Prototypes)
```javascript
class Animal {
    #sound; // Private field (#)
    
    constructor(name, sound) {
        this.name = name;
        this.#sound = sound;
    }
    
    speak() {
        return `${this.name} says ${this.#sound}`;
    }
    
    static create(name, sound) { // Static method — called on class, not instances
        return new Animal(name, sound);
    }
}

class Dog extends Animal {
    constructor(name) {
        super(name, "Woof"); // MUST call super() before using `this`
    }
    
    fetch(item) {
        return `${this.name} fetches ${item}`;
    }
}

const dog = new Dog("Rex");
dog.speak();       // "Rex says Woof"
dog.fetch("ball"); // "Rex fetches ball"
// dog.#sound;     // SyntaxError! Private fields are truly private.
```

---

## 4. Iterators & Generators

```javascript
// Generator function (produces values lazily, on demand)
function* range(start, end) {
    for (let i = start; i < end; i++) {
        yield i; // Pauses here, returns i, resumes on next call
    }
}

const gen = range(0, 3);
gen.next(); // { value: 0, done: false }
gen.next(); // { value: 1, done: false }
gen.next(); // { value: 2, done: false }
gen.next(); // { value: undefined, done: true }

// Works with for...of:
for (const n of range(10, 15)) {
    console.log(n); // 10, 11, 12, 13, 14
}

// Infinite generator (produces values forever!)
function* fibonacci() {
    let a = 0, b = 1;
    while (true) {
        yield a;
        [a, b] = [b, a + b];
    }
}
```

---

## 5. Proxy & Reflect

```javascript
const handler = {
    get(target, prop) {
        console.log(`Reading ${prop}`);
        return prop in target ? target[prop] : `Property ${prop} not found`;
    },
    set(target, prop, value) {
        if (typeof value !== "string") throw new TypeError("Value must be a string");
        target[prop] = value;
        return true;
    }
};

const user = new Proxy({}, handler);
user.name = "Surya";     // set trap fires
console.log(user.name);  // get trap fires: "Reading name" → "Surya"
console.log(user.foo);   // "Property foo not found"
// user.age = 25;         // TypeError: Value must be a string

// Use cases: Validation, logging, watchers (Vue 3 reactivity!), default values
```
