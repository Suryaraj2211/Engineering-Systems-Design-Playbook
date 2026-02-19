# JavaScript Roadmap: Part 7 — Extras & Deep Dives

---

## 1. Regular Expressions (RegExp)

```javascript
// Syntax: /pattern/flags
const emailRegex = /^[\w.-]+@[\w.-]+\.\w{2,}$/i;
emailRegex.test("surya@example.com"); // true
emailRegex.test("invalid@");          // false

// Common patterns:
/\d+/           // One or more digits
/[a-zA-Z]+/     // One or more letters
/^hello/        // Starts with "hello"
/world$/        // Ends with "world"
/colou?r/       // Matches "color" or "colour" (u is optional)
/\b\w+\b/g      // All words (word boundary)

// String methods with regex:
"Hello World".match(/\w+/g);        // ["Hello", "World"]
"2024-01-15".replace(/(\d{4})-(\d{2})-(\d{2})/, "$3/$2/$1"); // "15/01/2024"
"a1b2c3".split(/\d/);               // ["a", "b", "c", ""]
```

---

## 2. Web Workers (Multi-Threading)

```javascript
// JavaScript is single-threaded. Heavy computation blocks the UI.
// Web Workers run JS in a SEPARATE THREAD (no DOM access).

// main.js
const worker = new Worker("worker.js");
worker.postMessage({ data: hugeArray }); // Send data to worker
worker.onmessage = (e) => {
    console.log("Result:", e.data);       // Receive result from worker
};

// worker.js
self.onmessage = (e) => {
    const result = heavyComputation(e.data);
    self.postMessage(result); // Send result back to main thread
};
```

---

## 3. Performance Patterns

```javascript
// DEBOUNCE: Wait until user STOPS doing something for N ms
function debounce(fn, delay) {
    let timer;
    return (...args) => {
        clearTimeout(timer);
        timer = setTimeout(() => fn(...args), delay);
    };
}
const debouncedSearch = debounce(query => fetchResults(query), 300);
input.addEventListener("input", e => debouncedSearch(e.target.value));

// THROTTLE: Execute at most once every N ms
function throttle(fn, limit) {
    let inThrottle = false;
    return (...args) => {
        if (!inThrottle) {
            fn(...args);
            inThrottle = true;
            setTimeout(() => inThrottle = false, limit);
        }
    };
}
window.addEventListener("scroll", throttle(handleScroll, 100));

// MEMOIZATION: Cache function results
function memoize(fn) {
    const cache = new Map();
    return (...args) => {
        const key = JSON.stringify(args);
        if (cache.has(key)) return cache.get(key);
        const result = fn(...args);
        cache.set(key, result);
        return result;
    };
}
const expensiveFn = memoize((n) => { /* heavy computation */ return n * n; });
```

---

## 4. Design Patterns in JavaScript

```javascript
// SINGLETON: Only one instance ever created
class Database {
    static #instance = null;
    static getInstance() {
        if (!Database.#instance) Database.#instance = new Database();
        return Database.#instance;
    }
}

// OBSERVER: Pub/Sub pattern (events)
class EventEmitter {
    #listeners = {};
    
    on(event, callback) {
        (this.#listeners[event] ??= []).push(callback);
    }
    
    emit(event, ...args) {
        (this.#listeners[event] ?? []).forEach(cb => cb(...args));
    }
    
    off(event, callback) {
        this.#listeners[event] = this.#listeners[event]?.filter(cb => cb !== callback);
    }
}

// FACTORY: Create objects without specifying exact class
function createShape(type) {
    switch (type) {
        case "circle": return new Circle();
        case "square": return new Square();
        default: throw new Error(`Unknown shape: ${type}`);
    }
}
```

---

## 5. Memory Management

```javascript
// JavaScript has automatic garbage collection (Mark-and-Sweep).
// But YOU can still cause memory leaks:

// LEAK 1: Forgotten event listeners
element.addEventListener("click", handler); // If element is removed from DOM but
                                            // handler references outer variables → LEAK
// FIX: element.removeEventListener("click", handler);

// LEAK 2: Closures holding large data
function processData() {
    const hugeArray = new Array(1000000).fill("x");
    return () => hugeArray.length; // Closure keeps hugeArray alive forever!
}
// FIX: Set references to null when done, or restructure to not close over large data.

// LEAK 3: setInterval without clearInterval
const id = setInterval(update, 1000);
// FIX: clearInterval(id) when no longer needed.
```
