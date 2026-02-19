# Engineering Documentation Syllabus

This directory contains 83 highly detailed architectural whitepapers and implementation guides. 

This is not a learning resource. It is a systematic teardown of four major software engineering domains, designed to train engineers to think in terms of memory layouts, compilation phases, hardware execution, and Big-O efficiency.

## The Problem-Solving Tracks

### JavaScript: The Runtime Domain
**The Problem:** Managing asynchronous execution without multi-threading.
**The Focus:** Deconstructing the V8 compiler (AST, Ignition, TurboFan). Analyzing how the Event Loop queues execution phases to prevent UI blocking. Explaining the inherent dangers of dynamic typing (coercion) and how to architect memory-safe applications using strict closures and WeakMaps to prevent garbage collection leaks.

### TypeScript: The Compile-Time Domain
**The Problem:** Enforcing rigid algebraic data contracts across decentralized teams to prevent runtime exceptions.
**The Focus:** Leveraging the compiler as a mathematical proof engine. This track ignores simple annotations and focuses on advanced type-level programming: Generic Constraints, Conditional Types (`infer`), and Exhaustiveness Checking via the `never` type to ensure that architectural edge cases are caught before the code ever reaches a browser or server.

### Data Structures & Algorithms: The Optimization Domain
**The Problem:** Processing massive datasets where inefficient operations lead to exponential time and infrastructure costs.
**The Focus:** Mathematical performance analysis. Analyzing the exact memory layouts of continuous arrays versus pointer-based nodes. Implementing caching strategies via Dynamic Programming, managing priority queues for scheduling (Heaps), and analyzing graph search algorithms (Dijkstra, Topological Sort) for network routing and dependency resolution.

### Graphics Engineering: The Hardware Domain
**The Problem:** Executing millions of complex mathematical operations concurrently within a strict 16-millisecond frame budget.
**The Focus:** Raw GPU architecture. Understanding Single Instruction, Multiple Threads (SIMT). Implementing low-level graphics APIs (WebGL, WebGPU) from scratch. Calculating the physics of light (PBR, Cook-Torrance BRDF) and analyzing the memory bandwidth tradeoffs of Forward versus Deferred Rendering architectures.

## Documentation Navigation

The documentation is sequentially numbered (e.g., `01_...`, `02_...`). In an engineering context, the earlier modules define the raw hardware constraints and language physics, while the later modules build complex architectural systems on top of those constraints.
