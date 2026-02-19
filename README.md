# Systems Engineering: From Algorithms to GPU Architecture

This repository is a comprehensive engineering reference documenting major problem domains across the hardware and software stack. It is not a collection of tutorials. It is a structured breakdown of **what breaks in real systems, why it breaks, and what the correct architectural solution is** for each domain.

This documentation was built to systematically close gaps in engineering knowledge — identifying where bottlenecks exist across runtime execution, type safety, algorithmic efficiency, and GPU hardware — and documenting the exact solution for each.

---

## What This Repository Covers

### JavaScript — Solving the Single-Threaded Concurrency Problem
JavaScript runs on a single thread, yet modern applications demand thousands of concurrent network requests, timers, and UI updates without freezing. This track documents exactly how the V8 Engine, the Event Loop, and the Microtask/Macrotask queue architecture solve this problem. It also covers the memory pitfalls (closures retaining DOM nodes, reference vs. value semantics) that cause production memory leaks and the exact patterns (Debounce, Throttle, WeakMap) that prevent them.

**When to use what:**
- Event Loop architecture solves non-blocking I/O on a single thread
- Closures solve private state encapsulation without classes
- Promises / async-await solve callback hell and error propagation
- WeakMap solves metadata caching without blocking garbage collection
- Event Delegation solves the "1000 listeners" DOM memory problem


---

### TypeScript — Solving the Large-Scale Refactoring Problem
When a codebase grows beyond a single developer, dynamic typing becomes a liability. A renamed property silently breaks 40 files. A function receives `null` where it expects an object and crashes in production at 2 AM. TypeScript's compiler eliminates these entire classes of bugs at build time. This track documents how to use the type system not just for annotations, but as a compile-time proof engine — Discriminated Unions make invalid states unrepresentable, Generics eliminate code duplication, and Conditional Types compute new types algebraically.

**When to use what:**
- Discriminated Unions solve the "forgot to handle a new case" bug (exhaustive checking)
- Generics solve writing the same function for 10 different types
- Mapped Types solve transforming entire interfaces (e.g., making all fields optional or readonly)
- `unknown` over `any` solves unsafe external data (API responses, user input)
- The Result type pattern solves unpredictable try/catch error handling


---

### Data Structures & Algorithms — Solving the Scalability Problem
An O(n^2) algorithm works fine on 100 items. On 10 million items, it takes 11 days. The difference between a junior and a senior engineer is knowing that switching to an O(n log n) approach reduces that to 0.2 seconds. This track documents 30 modules covering every major data structure and algorithmic paradigm, with the specific problem each one is designed to solve.

**When to use what:**
- HashMap solves O(1) key lookups (vs. O(n) linear search)
- Binary Search Tree solves ordered data with O(log n) insert/search/delete
- Heap / Priority Queue solves "find the minimum/maximum" in O(log n)
- Trie solves prefix-based autocomplete search
- BFS solves shortest unweighted path; Dijkstra solves shortest weighted path
- Dynamic Programming solves overlapping subproblems by caching partial results
- Sliding Window solves contiguous subarray problems in O(n) instead of O(n^2)
- Backtracking solves constraint satisfaction (Sudoku, N-Queens, permutations)


---

### Graphics Engineering — Solving the Real-Time Rendering Problem
Rendering a single photorealistic frame requires tracing millions of light rays, computing physics-based material interactions, and writing results to millions of pixels. Doing this 60 times per second on consumer hardware is one of the hardest engineering problems in computing. This track documents 39 modules across three phases — from raw matrix math and GPU hardware architecture to production rendering techniques (PBR, Deferred Shading, TAA) and cutting-edge research (NeRF, Gaussian Splatting, DLSS).

**When to use what:**
- Forward Rendering solves simple scenes; Deferred Rendering solves scenes with hundreds of lights
- Instanced Drawing solves rendering 10,000 identical objects without 10,000 draw calls
- Compute Shaders solve arbitrary parallel computation (particle physics, post-processing)
- PBR (Cook-Torrance BRDF) solves physically accurate material rendering
- Shadow Mapping solves real-time shadow generation; CSM solves shadow quality at distance
- TAA solves jagged edges by accumulating subpixel samples across frames
- HDR + Tone Mapping solves the limited dynamic range of displays
- WebGPU solves the state-machine overhead and lack of compute in WebGL


---

## Why This Exists

This repository is the result of continuous, rigorous study and practical implementation — consolidating academic theory, official documentation, and hands-on system building. To build an unshakable understanding of these problem domains, every topic was broken down to its lowest-level implementation: manually calculating matrix multiplications, tracing Event Loop execution order, and tracking GPU warp scheduling.

The result is 83 modules, each explaining not just *how* something works, but *what specific engineering failure it was designed to prevent*.
