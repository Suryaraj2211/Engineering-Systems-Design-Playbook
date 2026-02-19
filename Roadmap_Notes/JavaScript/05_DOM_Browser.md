# JavaScript Roadmap: Part 5 — DOM & Browser APIs

---

## 1. DOM (Document Object Model) — The Bridge Between JS and HTML

The browser parses HTML into a **tree of nodes**. JavaScript can read and modify this tree in real time.

```javascript
// SELECTING ELEMENTS
document.getElementById("header");         // Single element by ID (fastest)
document.querySelector(".card");            // First match by CSS selector
document.querySelectorAll(".card");         // NodeList of ALL matches
document.getElementsByClassName("card");   // Live HTMLCollection (updates automatically)

// querySelector vs getElementById:
// querySelector: More flexible (any CSS selector), slightly slower.
// getElementById: Fastest, but only works with IDs.
```

---

## 2. Manipulating the DOM

```javascript
const el = document.querySelector("#app");

// Content
el.textContent = "Hello";         // Sets text (safe, escapes HTML)
el.innerHTML = "<b>Hello</b>";    // Sets HTML (DANGEROUS if user input! → XSS attack)

// Attributes
el.setAttribute("data-id", "42");
el.getAttribute("data-id");       // "42"
el.dataset.id;                    // "42" (shorthand for data-* attributes)
el.classList.add("active");
el.classList.remove("active");
el.classList.toggle("active");
el.classList.contains("active");  // true/false

// Styles
el.style.color = "red";           // Inline style (use sparingly!)
el.style.backgroundColor = "#333"; // camelCase, not kebab-case

// Creating & inserting elements
const card = document.createElement("div");
card.className = "card";
card.textContent = "New Card";
document.body.appendChild(card);           // Add as last child
document.body.prepend(card);               // Add as first child
el.insertAdjacentHTML("beforeend", "<p>Injected</p>"); // More options
el.remove();                               // Remove from DOM
```

---

## 3. Events

```javascript
const btn = document.querySelector("#btn");

// addEventListener (preferred — supports multiple handlers)
btn.addEventListener("click", (event) => {
    console.log("Clicked!", event.target);    // The element that was clicked
    console.log("Coordinates:", event.clientX, event.clientY);
});

// Event delegation (ONE handler on parent, handles ALL children)
document.querySelector(".list").addEventListener("click", (e) => {
    if (e.target.matches(".list-item")) {
        console.log("Clicked item:", e.target.textContent);
    }
});
// WHY: Adding individual handlers to 1000 list items = 1000 listeners = memory waste.
//      One handler on the parent + checking e.target = 1 listener. Much better.

// Bubbling vs Capturing
// Events bubble UP by default: child → parent → grandparent → document
// Use e.stopPropagation() to prevent bubbling.
// Third argument `true` = capture phase (top-down).
btn.addEventListener("click", handler, { capture: true, once: true });

// Common events:
// Mouse: click, dblclick, mousedown, mouseup, mousemove, mouseenter, mouseleave
// Keyboard: keydown, keyup (use e.key, NOT e.keyCode which is deprecated)
// Form: submit, change, input, focus, blur
// Window: load, DOMContentLoaded, resize, scroll
```

---

## 4. LocalStorage & SessionStorage

```javascript
// localStorage: Persists until manually cleared (survives browser restart)
localStorage.setItem("theme", "dark");
localStorage.getItem("theme");       // "dark"
localStorage.removeItem("theme");
localStorage.clear();                 // Remove ALL

// sessionStorage: Cleared when the tab is closed
sessionStorage.setItem("token", "abc123");

// STORING OBJECTS (must serialize!)
localStorage.setItem("user", JSON.stringify({ name: "Surya", age: 25 }));
const user = JSON.parse(localStorage.getItem("user"));

// LIMITS: ~5-10MB per origin. Synchronous (blocks main thread). Strings only.
```

---

## 5. Fetch API — Network Requests

```javascript
// GET request
async function getUser(id) {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return await response.json();
}

// POST request
async function createUser(data) {
    const response = await fetch("/api/users", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(data)
    });
    return await response.json();
}

// AbortController (cancel requests)
const controller = new AbortController();
fetch("/api/data", { signal: controller.signal })
    .catch(err => {
        if (err.name === "AbortError") console.log("Request cancelled");
    });
setTimeout(() => controller.abort(), 5000); // Cancel after 5 seconds
```

---

## 6. XMLHTTPRequest — The Legacy Network API

Before `fetch()` existed, **XMLHTTPRequest (XHR)** was the ONLY way to make HTTP requests from JavaScript. You'll still encounter it in legacy code, jQuery's `$.ajax`, and some Edge cases where Fetch can't be used.

```javascript
// Basic GET Request with XHR:
const xhr = new XMLHttpRequest();
xhr.open("GET", "/api/users/1", true); // true = async (always use async!)

xhr.onload = function() {
    if (xhr.status >= 200 && xhr.status < 300) {
        const data = JSON.parse(xhr.responseText);
        console.log("Success:", data);
    } else {
        console.error("HTTP Error:", xhr.status);
    }
};

xhr.onerror = function() {
    console.error("Network error — request failed entirely");
};

xhr.send(); // Actually sends the request
```

### POST Request with XHR
```javascript
const xhr = new XMLHttpRequest();
xhr.open("POST", "/api/users", true);
xhr.setRequestHeader("Content-Type", "application/json");

xhr.onload = function() {
    if (xhr.status === 201) {
        console.log("Created:", JSON.parse(xhr.responseText));
    }
};

xhr.send(JSON.stringify({ name: "Surya", role: "admin" }));
```

### XHR Events & Progress Tracking
```javascript
const xhr = new XMLHttpRequest();
xhr.open("GET", "/api/large-file", true);

// XHR has progress events that Fetch doesn't (one advantage!):
xhr.onprogress = function(event) {
    if (event.lengthComputable) {
        const percent = (event.loaded / event.total) * 100;
        console.log(`Downloaded: ${percent.toFixed(1)}%`);
    }
};

xhr.onreadystatechange = function() {
    // readyState values:
    // 0 = UNSENT        — open() not called yet
    // 1 = OPENED        — open() called
    // 2 = HEADERS_RECEIVED — response headers received
    // 3 = LOADING       — response body is being received
    // 4 = DONE          — request complete
    if (xhr.readyState === 4 && xhr.status === 200) {
        console.log("Done:", xhr.responseText);
    }
};

xhr.send();
```

### XHR vs Fetch — Why Fetch Won
```
FEATURE              XHR                          FETCH
─────────────────────────────────────────────────────────
API Style            Callback-based               Promise-based (async/await) ✓
Syntax               Verbose (~10 lines)           Clean (~3 lines) ✓
Streaming            No                           Yes (ReadableStream) ✓
Upload Progress      Yes ✓ (xhr.upload.onprogress) No (workaround needed)
Download Progress    Yes ✓ (xhr.onprogress)        Limited (ReadableStream)
Abort                xhr.abort()                  AbortController ✓
Service Workers      Not available                Available ✓
CORS                 Complex                      Simple ✓

VERDICT: Use Fetch for everything new. Use XHR only when you need
         upload/download progress tracking in legacy environments.
```

---

## 7. Performance Tips

| Cause | Impact | Fix |
|-------|--------|-----|
| Frequent DOM reads/writes | Layout thrashing (reflow storm) | Batch DOM operations, use DocumentFragment |
| `innerHTML` in a loop | Entire DOM re-parsed each iteration | Build string first, set innerHTML once |
| No event delegation | 1000 listeners for 1000 items | One listener on parent with `e.target` check |
| Unremoved event listeners | Memory leak | Remove listeners on cleanup |
| Synchronous `localStorage` | Blocks main thread on large data | Use IndexedDB for large datasets |
