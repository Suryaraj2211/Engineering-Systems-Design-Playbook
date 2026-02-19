# Module 2 — GPU Architecture & Hardware Deep Dive

> This module explains how GPU silicon physically executes your shaders,
> and how understanding this hardware transforms your architectural decisions.

---

## 1. The SIMT Execution Model (Single Instruction, Multiple Threads)

### 1.1 Warps and Wavefronts — The Atomic Unit of Execution

A CPU executes threads independently. A GPU **cannot**.

The fundamental execution unit is a **Warp** (NVIDIA, 32 threads) or **Wavefront** (AMD, 32 or 64 threads depending on architecture).

Every thread in a Warp shares the same Program Counter (Instruction Pointer). When the GPU executes `float x = dot(N, L);`, all 32 threads execute `dot()` simultaneously on 32 different sets of data. This is SIMD (Single Instruction, Multiple Data) at the thread level.

```
WARP EXECUTION MODEL:
══════════════════════
                  Thread 0   Thread 1   Thread 2   ...  Thread 31
                  ────────   ────────   ────────        ─────────
Cycle 1 (MUL):   N0.x*L0.x  N1.x*L1.x  N2.x*L2.x      N31.x*L31.x
Cycle 2 (MAD):   +N0.y*L0.y +N1.y*L1.y +N2.y*L2.y      +N31.y*L31.y
Cycle 3 (MAD):   +N0.z*L0.z +N1.z*L1.z +N2.z*L2.z      +N31.z*L31.z
                  ────────   ────────   ────────        ─────────
                  Result 0   Result 1   Result 2         Result 31

ALL 32 THREADS EXECUTE THE EXACT SAME INSTRUCTION EACH CYCLE.
```

### 1.2 The Divergence Penalty — The Silent Killer

When a Warp encounters a branch (`if/else`), the GPU cannot split. It must execute BOTH paths.

```glsl
// DIVERGENT SHADER (BAD)
if (material.isMetallic) {
    // Path A: 20 instructions (specular-only metals)
    color = computeMetalBRDF(N, V, L, F0);
} else {
    // Path B: 35 instructions (diffuse + specular dielectrics)
    color = computeDielectricBRDF(N, V, L, albedo, F0);
}
```

**What the GPU actually does:**

```
DIVERGENT EXECUTION TIMELINE:
═════════════════════════════
  Thread  0: material.isMetallic = TRUE
  Thread  1: material.isMetallic = FALSE
  Thread  2: material.isMetallic = TRUE
  ...
  Thread 31: material.isMetallic = FALSE

  Step 1: Execute Path A (20 instructions) for ALL 32 threads.
          Threads 1, 31, etc. (FALSE) execute but DISCARD results.
          
  Step 2: Execute Path B (35 instructions) for ALL 32 threads.
          Threads 0, 2, etc. (TRUE) execute but DISCARD results.

  TOTAL COST: 20 + 35 = 55 instructions (instead of max 35)
  WASTED WORK: ~40% of ALU cycles produce discarded data
```

**Architectural Solutions:**
1. **Sort draw calls by material type.** If all 32 pixels in a Warp are metallic, the GPU takes Path A in 20 instructions (no divergence).
2. **Use branchless math.** Replace `if(metallic)` with `mix()`:
```glsl
// BRANCHLESS (GOOD) — Always executes both, blends via math
vec3 metalColor = computeMetalBRDF(N, V, L, F0);
vec3 dielectricColor = computeDielectricBRDF(N, V, L, albedo, F0);
color = mix(dielectricColor, metalColor, metallic);
// Cost: 55 instructions ALWAYS, but NO divergence penalty.
// This is only better when divergence is frequent.
// If 99% of pixels are dielectric, the branch is actually faster!
```

### 1.3 Subgroup (Wave) Intrinsics — Communication Within a Warp

Threads within the same Warp can communicate directly via ultra-fast **Subgroup Operations** without touching memory.

```wgsl
// WGSL Subgroup Operations (WebGPU)
// Find the minimum depth across all 32 threads in the Warp
let minDepthInWarp = subgroupMin(fragmentDepth);

// Sum up all light contributions across the Warp
let totalLight = subgroupAdd(lightContribution);

// Broadcast thread 0's value to all other threads
let sharedValue = subgroupBroadcastFirst(myValue);
```

**Why this matters architecturally:**
- Traditional reduction (finding the minimum of 1024 values) requires writing to shared memory, synchronizing, reading back — 3 memory operations.
- Subgroup reduction does it in **1 clock cycle** with zero memory traffic.
- Professional engines use subgroup ops for: frustum culling vote (all threads vote "visible" or "invisible"), light list compaction, and prefix sum calculations for indirect draw buffers.

---

## 2. The GPU Memory Hierarchy — Every Byte Counts

### 2.1 The Full Memory Stack

```
MEMORY HIERARCHY (NVIDIA RTX 4080):
════════════════════════════════════
  ┌─────────────────────────────────────────────────────────┐
  │  Registers (L0)                                         │
  │  Per-thread: 255 max (32-bit each)                      │
  │  Latency: 0 cycles (FREE)                               │
  │  Total per SM: 65,536 registers                         │
  ├─────────────────────────────────────────────────────────┤
  │  Shared Memory / L1 Cache                               │
  │  Per SM: 128 KB (configurable split with L1)            │
  │  Latency: ~5 cycles                                     │
  │  Bandwidth: ~128 bytes/cycle                            │
  │  Accessible via: var<workgroup> in WGSL                 │
  ├─────────────────────────────────────────────────────────┤
  │  L2 Cache                                               │
  │  Global: 64 MB (shared across ALL SMs)                  │
  │  Latency: ~200 cycles                                   │
  │  Bandwidth: ~6 TB/s internal                            │
  ├─────────────────────────────────────────────────────────┤
  │  VRAM (Global Memory)                                   │
  │  Total: 16 GB GDDR6X                                    │
  │  Latency: ~400-800 cycles                               │
  │  Bandwidth: 717 GB/s                                    │
  ├─────────────────────────────────────────────────────────┤
  │  System RAM (via PCIe 4.0 x16)                          │
  │  Latency: ~10,000+ cycles                               │
  │  Bandwidth: ~28 GB/s                                    │
  │  NEVER read back to CPU in real-time!                   │
  └─────────────────────────────────────────────────────────┘
```

### 2.2 Register Pressure & Occupancy — The Critical Bottleneck

The GPU hides VRAM latency (~500 cycles) by rapidly switching between Warps. While Warp A waits for a texture fetch, Warp B executes math. While Warp B waits, Warp C runs.

**But there's a finite number of registers per Streaming Multiprocessor (SM).**

```
OCCUPANCY CALCULATION EXAMPLE:
══════════════════════════════
  SM has 65,536 registers total.
  Max Warps per SM: 48 (NVIDIA Ada Lovelace)
  
  Case A: Shader uses 32 registers per thread
    Registers per Warp: 32 × 32 threads = 1,024
    Max Warps that fit: 65,536 / 1,024 = 64 → capped at 48
    OCCUPANCY: 48/48 = 100% ✅ (GPU can fully hide latency)
  
  Case B: Shader uses 128 registers per thread  
    Registers per Warp: 128 × 32 = 4,096
    Max Warps that fit: 65,536 / 4,096 = 16
    OCCUPANCY: 16/48 = 33% ⚠️ (GPU stalls waiting for memory)
  
  Case C: Shader uses 256 registers per thread (SPILL!)
    Registers per Warp: 256 × 32 = 8,192
    Max Warps that fit: 65,536 / 8,192 = 8
    OCCUPANCY: 8/48 = 17% ❌ (catastrophic stalling)
    ADDITIONALLY: 256 > 255 max. Extra registers "spill" to VRAM.
    Every variable access now costs 400 cycles instead of 0!
```

**How to reduce register usage:**
1. Reduce temporary variables. Reuse `vec3 temp` instead of declaring `vec3 a, b, c, d, e`.
2. Move complex functions to separate shader stages (e.g., compute BRDF in a separate pass).
3. Use `mediump` (16-bit float) where full precision isn't needed (UV coordinates, colors).

### 2.3 Memory Coalescing — The Warp Reads Together

When 32 threads in a Warp read from a `StorageBuffer`, the GPU hardware attempts to merge (coalesce) the 32 individual requests into a single wide memory transaction.

```
COALESCED ACCESS (FAST):
════════════════════════
  Thread 0  reads buffer[0]     ┐
  Thread 1  reads buffer[1]     │ GPU merges into ONE
  Thread 2  reads buffer[2]     │ 128-byte memory
  ...                           │ transaction
  Thread 31 reads buffer[31]    ┘
  
  COST: 1 memory transaction (~400 cycles)

UNCOALESCED ACCESS (SLOW):
══════════════════════════
  Thread 0  reads buffer[0]        → 1 transaction
  Thread 1  reads buffer[1000]     → 1 transaction  
  Thread 2  reads buffer[42]       → 1 transaction
  ...                              
  Thread 31 reads buffer[7777]     → 1 transaction
  
  COST: up to 32 separate memory transactions (~12,800 cycles)
  PERFORMANCE: 32× SLOWER!
```

**Architectural Rule:** Always structure `StorageBuffer` data so that consecutive threads read consecutive memory addresses. If your data is an Array of Structs (AoS), convert it to Struct of Arrays (SoA):

```
AoS (BAD for GPU):                  SoA (GOOD for GPU):
struct Particle {                   positions: Float32Array (all X,Y,Z packed)
  position: vec3,                   velocities: Float32Array (all VX,VY,VZ packed)  
  velocity: vec3,                   colors: Uint32Array (all colors packed)
  color: u32,
}
particles: Particle[]               Thread 0 reads positions[0], Thread 1 reads positions[1]
                                    → PERFECTLY COALESCED
Thread 0 reads particles[0].pos
Thread 1 reads particles[1].pos     
→ Stride of 28 bytes between reads (scattered!)
```

---

## 3. The Texture Cache & Spatial Locality

### 3.1 Morton Code (Z-Order Curve) Layout

Textures are NOT stored linearly in VRAM (row by row). They are stored in a **Z-Order Curve** (Morton Code) pattern that interleaves X and Y bits:

```
LINEAR MEMORY LAYOUT (how you THINK textures are stored):
  Row 0: [0,0] [1,0] [2,0] [3,0] [4,0] [5,0] [6,0] [7,0]
  Row 1: [0,1] [1,1] [2,1] [3,1] [4,1] [5,1] [6,1] [7,1]
  Row 2: [0,2] [1,2] [2,2] [3,2] [4,2] [5,2] [6,2] [7,2]

MORTON CODE LAYOUT (how textures are ACTUALLY stored):
  [0,0] [1,0] [0,1] [1,1]  |  [2,0] [3,0] [2,1] [3,1]  | ...
  └──── 2×2 Block ─────┘     └──── 2×2 Block ─────┘

  Adjacent pixels in 2D are adjacent in VRAM memory!
  The GPU's L1 texture cache loads entire 2D blocks, not rows.
```

**Why this matters:**
- Fragment shaders process pixels in 2×2 "Quads" (for `dFdx`/`dFdy` derivative calculation).
- Because Morton Code stores 2×2 blocks contiguously, when the Quad reads `texture(albedo, uv)`, all 4 threads hit the **same cache line**. 
- If a Compute Shader reads a texture at random UV coordinates per thread (e.g., for SSR raymarching), it defeats the 2D cache entirely. Performance plummets.

### 3.2 Mipmap Level Selection — Hardware Derivatives

The GPU selects which Mipmap level to sample by comparing the UV coordinates of adjacent pixels in the 2×2 Quad:

```glsl
// The GPU implicitly calculates these for EVERY texture() call:
float dUdx = dFdx(uv.x); // How much does U change per pixel horizontally
float dVdy = dFdy(uv.y); // How much does V change per pixel vertically

// If the UV changes rapidly (object is far away, many texels per pixel)
// → Select a SMALLER mipmap (blurrier, fewer cache misses)
// If the UV barely changes (object fills the screen)
// → Select the FULL resolution mipmap
```

**Critical Consequence:** This is why `texture()` calls inside `if/else` branches or Compute Shaders sometimes produce artifacts — the 2×2 Quad may not exist, so derivatives are undefined! You must use `textureSampleLevel()` (explicit LOD) instead.

---

## 4. Shader Compiler Optimization Behavior

### 4.1 What the Driver Does to Your Shaders

You write WGSL/GLSL. The driver's shader compiler transforms it through multiple stages before the GPU sees it:

```
YOUR WGSL CODE
     │
     ▼
┌─────────────┐
│  Tint/Naga  │  Frontend: Parse WGSL → IR (Intermediate Representation)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  SPIR-V     │  Platform-independent bytecode
└──────┬──────┘
       │
       ▼
┌──────────────────┐
│  NVIDIA Compiler │  Proprietary! Converts SPIR-V to native GPU ISA (SASS)
│  (hidden in      │  Applies: Dead Code Elimination, Loop Unrolling,
│   driver .dll)   │  Register Allocation, Instruction Scheduling,
└──────┬───────────┘  Constant Folding, Function Inlining
       │
       ▼
┌─────────────┐
│  GPU SASS   │  Native machine code that actually hits the silicon
│  microcode  │
└─────────────┘
```

### 4.2 Optimizations the Compiler Makes (and What It Can't Do)

**What it WILL optimize:**
```glsl
// BEFORE (your code):
float a = 3.14159;
float b = 2.0;
float c = a * b;        // Constant! Compiler folds to c = 6.28318
vec3 unused = vec3(0.0); // Dead code. Compiler removes entirely.

// AFTER (compiled):
// The GPU literally never executes the multiply or the vec3 allocation.
```

**What it CANNOT optimize:**
```glsl
// The compiler cannot hoist texture fetches out of loops:
for (int i = 0; i < 16; i++) {
    vec3 sample = texture(myTex, uv + offsets[i]).rgb; // 16 fetches!
    // The compiler cannot know if the texture changes between iterations
    // (even though it doesn't). It must fetch 16 times.
}

// The compiler cannot restructure your algorithm:
// If you wrote O(N²) when O(N log N) exists, the compiler won't fix it.
```

### 4.3 Instruction-Level Parallelism (ILP)

Modern GPU ALUs can execute multiple independent instructions simultaneously within a single thread using superscalar pipelines:

```glsl
// LOW ILP (Serial Dependencies):
float a = dot(N, L);      // Instruction 1
float b = a * roughness;  // Instruction 2 DEPENDS on a (must wait!)
float c = b + 0.5;        // Instruction 3 DEPENDS on b (must wait!)
// Pipeline: [1]→[2]→[3] = 3 cycles minimum

// HIGH ILP (Independent Instructions):
float a = dot(N, L);       // Instruction 1
float b = dot(N, V);       // Instruction 2 (INDEPENDENT! Runs in parallel with 1)
float c = dot(N, H);       // Instruction 3 (INDEPENDENT! Runs in parallel with 1 & 2)
// Pipeline: [1,2,3 simultaneously] = 1 cycle!
```

**Architectural Takeaway:** When writing performance-critical shader code, calculate ALL independent `dot()` products and `texture()` fetches before using any of their results. This maximizes ILP.

---

## 5. Driver-Level Bottlenecks

### 5.1 Pipeline State Object (PSO) Compilation Stalls

In Vulkan/WebGPU, creating a `RenderPipeline` triggers the driver's shader compiler. This can take **50-500 milliseconds** (blocking the main thread!).

**The Stutter Problem:**
If a player encounters a new material for the first time (e.g., enters a water area), the engine creates a new Pipeline. The game freezes for 200ms. This is the infamous "shader compilation stutter" that plagues many PC games.

**Solutions:**
1. **Pipeline Cache / Warm-up:** At game startup, create ALL possible Pipeline combinations (opaque, transparent, shadow, depth-only, skinned, instanced). This shifts the cost to the loading screen.
2. **Async Pipeline Compilation:** WebGPU's `device.createRenderPipelineAsync()` compiles the pipeline on a background thread, returning a Promise. Display a fallback material until compilation completes.

### 5.2 Draw Call Overhead

Every `renderPass.draw()` call requires the CPU to:
1. Validate the current pipeline state.
2. Write commands into a Command Buffer.
3. Flush the Command Buffer to the GPU driver.

```
DRAW CALL BUDGET:
═════════════════
  A modern CPU can submit ~5,000-10,000 draw calls per frame at 60fps.
  
  A naive renderer drawing 50,000 objects individually = IMPOSSIBLE.
  
  Solutions:
  1. Instancing: Draw 1,000 identical trees with 1 draw call.
     renderPass.draw(vertexCount, instanceCount=1000);
  
  2. Indirect Drawing: The GPU writes the draw parameters itself!
     A Compute Shader fills a buffer with (vertexCount, instanceCount, firstVertex, firstInstance).
     renderPass.drawIndirect(indirectBuffer, offset);
     The CPU submits 1 draw call. The GPU decides what to draw.
  
  3. Mesh Shaders (not yet in WebGPU): The GPU generates geometry
     entirely on-chip, eliminating the CPU from the geometry pipeline.
```

---

## 6. Hardware-Aware Debugging & Profiling

### 6.1 Key Hardware Counters (What to Measure)

| Counter | Meaning | Healthy Range | Action if Unhealthy |
|---------|---------|---------------|---------------------|
| SM Occupancy | % of max Warps active | >50% | Reduce register count, reduce shared memory |
| ALU Utilization | % of cycles doing math | >60% for compute-heavy shaders | If low → bandwidth bound; reduce texture reads |
| Texture L1 Hit Rate | % of texture reads served from L1 | >80% | If low → random UV access patterns; restructure |
| L2 Hit Rate | % of memory reads served from L2 | >60% | If low → working set too large; reduce texture resolution |
| VRAM Throughput | GB/s actually consumed | <80% of max | If saturated → pack textures, use compressed formats |
| Register Spill Count | Registers that overflowed to VRAM | 0 | If >0 → CRITICAL! Simplify shader immediately |
| Warp Stall Reasons | Why Warps are paused | Varies | "Memory Dependency" = bandwidth bound; "Execution Dependency" = ALU bound |

### 6.2 Tools for GPU Profiling

| Tool | Platform | Best For |
|------|----------|----------|
| NVIDIA Nsight Graphics | NVIDIA GPUs | Frame capture, shader profiling at SASS level, pipeline state inspection |
| AMD Radeon GPU Profiler | AMD GPUs | Wavefront occupancy, cache hit rates, instruction timing |
| Apple GPU Profiler (Instruments) | Apple M-series | Tile memory utilization, bandwidth per pass |
| RenderDoc | Cross-platform | API call inspection, resource state tracking, mesh visualization |
| PIX (Windows) | DirectX 12 | GPU timing, pipeline statistics |
| `performance.now()` / Timestamp Queries | WebGPU | Per-pass GPU timing (not CPU timing!) |

### 6.3 Debugging Example: Finding a Bandwidth Bottleneck

```
SYMPTOM: Fragment shader runs at 8ms instead of expected 2ms.
         GPU ALU utilization is only 15%. 

DIAGNOSIS STEPS:
1. Check L2 Hit Rate → 22% (terrible!)
2. Check Texture L1 Hit Rate → 12% (disaster!)
3. Identify the shader → It's the SSAO pass.

ROOT CAUSE:
  SSAO samples 64 random depth buffer locations per pixel.
  Random access patterns defeat both L1 and L2 caches.
  The GPU spends 85% of its time WAITING for VRAM data.

FIX:
  Option A: Reduce SSAO to 16 samples (4× less bandwidth).
  Option B: Use a spatially coherent sampling pattern (Hilbert curve).
  Option C: Render SSAO at half resolution (4× fewer pixels to process).
  Option D: Switch to HBAO+ which uses screen-space horizon angles
            (sequential depth reads along lines, highly cache-friendly).
```
