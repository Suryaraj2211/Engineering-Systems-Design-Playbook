# Module 1 — Rendering System Innovation (Deep Dive)

> This module assumes mastery of PBR, G-Buffers, and standard shadow/post-processing pipelines.
> We focus exclusively on architectural decisions, trade-off analysis, and systems-level innovation.

---

## 1. The End of Pure Rasterization — Why Hybrid Pipelines Exist

### 1.1 The Fundamental Limitation of Screen-Space Effects
Every screen-space hack (SSR, SSAO, SSGI) shares one fatal flaw: they can only see what the camera sees. Reflections of off-screen objects disappear. Ambient occlusion from geometry behind the camera is invisible. Shadows from objects outside the frustum don't exist.

For two decades, engineers tolerated these artifacts because the alternative (ray tracing) was 100x slower. In 2018, NVIDIA introduced dedicated **RT Cores** — fixed-function BVH traversal silicon that accelerates ray-scene intersection by 10x compared to software compute tracing.

### 1.2 The Hybrid Architecture Decision Matrix

The architect's first decision: **which effects should be rasterized, and which should be ray traced?**

| Effect | Rasterization Cost | Ray Tracing Cost | Quality Winner | Recommendation |
|--------|-------------------|-----------------|----------------|----------------|
| Primary Visibility (Depth/Normals) | ~1ms (extremely fast) | ~4ms (wasteful for coherent rays) | Tie | **Rasterize** |
| Hard Shadows (Directional) | ~0.5ms (Shadow Map) | ~2ms (1 ray/pixel) | Ray Tracing (no acne/bias) | **Rasterize** (cost wins) |
| Soft Shadows (Area Lights) | ~2ms (PCSS hack) | ~3ms (stochastic) | Ray Tracing (physically correct penumbra) | **Ray Trace** |
| Reflections (Glossy) | ~2ms (SSR, with artifacts) | ~4ms (1spp + denoise) | Ray Tracing (no screen-edge cutoff) | **Ray Trace** |
| Ambient Occlusion | ~1.5ms (SSAO/HBAO) | ~2ms (short rays) | Ray Tracing (no halo artifacts) | **Either** (SSAO is "good enough") |
| Global Illumination (1 bounce) | ~3ms (SSGI, heavy artifacts) | ~6ms (DDGI probes) | Ray Tracing (color bleeding, no screen limit) | **Ray Trace** (via probes) |

### 1.3 The Bandwidth vs Compute Trade-off

This is the single most important architectural constraint in modern rendering.

**Bandwidth-bound operations:** Reading/writing large textures (G-Buffers, Shadow Maps, History Buffers). The GPU's ALU cores sit idle, waiting for VRAM to deliver data.

**Compute-bound operations:** Heavy mathematical operations per pixel (Cook-Torrance BRDF, ray-BVH intersection). VRAM is idle; ALU cores are saturated.

```
BANDWIDTH BOTTLENECK EXAMPLE:
─────────────────────────────
A Deferred G-Buffer at 4K resolution:
  Position:  3840 × 2160 × 4 channels × 2 bytes (RGBA16F) = 63 MB
  Normal:    3840 × 2160 × 4 channels × 2 bytes (RGBA16F) = 63 MB
  Albedo:    3840 × 2160 × 4 channels × 1 byte  (RGBA8)   = 32 MB
  Depth:     3840 × 2160 × 1 channel  × 4 bytes (D32F)    = 32 MB
                                                    TOTAL  = 190 MB

  Writing these in Pass 1:    190 MB write
  Reading them in Pass 2:     190 MB read
  TOTAL BANDWIDTH PER FRAME:  380 MB × 60fps = 22.8 GB/s

  RTX 4070 VRAM Bandwidth:    504 GB/s
  Percentage consumed by G-Buffer alone: ~4.5%

  NOW ADD: Shadow Cascades (3×), SSAO buffer, SSR History,
           Bloom mip chain, TAA History, Motion Vectors...
  REAL bandwidth consumption: 30-50% of total GPU bandwidth!
```

**The Architect's Decision:**
- If bandwidth is the bottleneck → reduce texture resolution, pack channels, use compressed formats (BC/ASTC).
- If compute is the bottleneck → reduce ray counts, simplify BRDF math, use LUT approximations.
- The ideal pipeline keeps both ALU and bandwidth utilization at ~70-80% simultaneously.

---

## 2. Real-Time GI Strategies — Architectural Comparison

### 2.1 DDGI (Dynamic Diffuse Global Illumination)

**Architecture:**
1. Place a 3D grid of probes (e.g., 24×12×24 = 6,912 probes) throughout the scene.
2. Each frame, each probe shoots N rays (e.g., 128 rays) using hardware RT Cores.
3. The incoming radiance from each ray is encoded into an **Octahedral Map** (a 2D unwrapping of a sphere) stored per-probe.
4. During the camera's lighting pass, each pixel samples the 8 nearest probes via trilinear interpolation.

**The Hysteresis (Temporal Stability) Math:**
```glsl
// Blend new radiance with old radiance to prevent flickering
// alpha controls how fast the probe "learns" new light
float alpha = 0.97; // 97% old data, 3% new data per frame
vec3 newRadiance = traceNewRays();
vec3 stableRadiance = mix(newRadiance, previousRadiance, alpha);
```

**GPU Cost Breakdown:**
| Stage | Cost (RTX 3070, 1080p) | Bottleneck |
|-------|----------------------|------------|
| Ray generation (6912 probes × 128 rays) | ~2.5ms | Compute (BVH traversal) |
| Octahedral irradiance update | ~0.3ms | Bandwidth (texture write) |
| Probe sampling during lighting | ~0.8ms | Bandwidth (texture read) |
| **TOTAL** | **~3.6ms** | |

**Common Mistakes & Debugging:**
- **Light Leaking:** A probe placed inside a thin wall receives light from both sides, bleeding indoor light outdoors. *Fix:* Implement probe relocation — detect when a probe is inside geometry (all rays hit within 0.1m) and push it outward along the surface normal.
- **Temporal Lag:** With `alpha = 0.97`, it takes ~100 frames for a probe to fully adapt to a new light source. If a player tosses a torch, the room stays dark for 1.5 seconds. *Fix:* Detect large variance changes and temporarily reduce alpha to 0.5 for rapid adaptation.

### 2.2 Screen-Space GI (SSGI) — The Cheap Alternative

**Architecture (Simplified):**
1. For each screen pixel, shoot 4-8 short rays in random hemisphere directions.
2. Instead of tracing against BVH, raymarch against the Hi-Z (Hierarchical Depth) buffer.
3. If a ray hits something, sample the previous frame's lit color at that hit point.
4. Average the contributions for approximate 1-bounce diffuse GI.

**Why it fails at edges:** The Depth Buffer only stores the closest surface. If a ray passes behind a visible object, it cannot detect the occluded geometry. This creates bright "halos" around objects where GI should be blocked.

**GPU Cost:** ~1.5ms (bandwidth-heavy, reading Hi-Z + Color repeatedly).

### 2.3 Lumen (Unreal Engine 5) — The Hybrid Genius

Lumen dynamically selects between multiple GI strategies per-pixel:
- **Close to camera:** Software ray tracing against Signed Distance Fields (SDFs) for fast, accurate short-range bounces.
- **Medium range:** Screen-space tracing against the Hi-Z buffer.
- **Far range:** Hardware ray tracing against simplified proxy meshes.
- **Fallback:** Radiance cache probes (similar to DDGI).

*Innovation Insight:* Lumen proves that no single GI technique is optimal everywhere. **The best architecture is a heterogeneous dispatcher** that selects the cheapest viable algorithm per-pixel based on distance and surface properties.

---

## 3. Frame Graph Optimization — Advanced Strategies

### 3.1 Resource Lifetime Analysis & Memory Aliasing

```
FRAME GRAPH DEPENDENCY EXAMPLE:
═══════════════════════════════

Pass 1: Shadow Map     → Writes [ShadowTex]
Pass 2: G-Buffer       → Writes [PosTex, NrmTex, AlbTex]
Pass 3: SSAO           → Reads  [PosTex, NrmTex]  → Writes [AOTex]
Pass 4: Lighting       → Reads  [All G-Buffer + ShadowTex + AOTex] → Writes [HDRColor]
Pass 5: Bloom Extract  → Reads  [HDRColor] → Writes [BrightTex]
Pass 6: Bloom Blur     → Reads  [BrightTex] → Writes [BloomTex]
Pass 7: Composite      → Reads  [HDRColor + BloomTex] → Writes [FinalColor]

LIFETIME ANALYSIS:
  ShadowTex:  alive [Pass 1 → Pass 4]    dead after Pass 4
  AOTex:      alive [Pass 3 → Pass 4]    dead after Pass 4
  BrightTex:  alive [Pass 5 → Pass 6]    dead after Pass 6
  BloomTex:   alive [Pass 6 → Pass 7]    dead after Pass 7

ALIASING OPPORTUNITY:
  ShadowTex dies at Pass 4. BrightTex is born at Pass 5.
  → Allocate BrightTex at the SAME physical VRAM address as ShadowTex!
  → Saves: 3840×2160×4bytes = 32MB of VRAM!
```

### 3.2 Automatic Barrier Insertion
In Vulkan/WebGPU, before a texture transitions from "Write" to "Read", you must insert a pipeline barrier (telling the GPU to flush caches). The Frame Graph automatically calculates these:

```javascript
// Pseudocode: Frame Graph barrier injection
for (const pass of compiledPasses) {
    for (const input of pass.reads) {
        if (input.lastWrittenBy !== pass) {
            insertBarrier({
                resource: input,
                oldUsage: 'RENDER_ATTACHMENT',    // Was written to
                newUsage: 'TEXTURE_BINDING',      // Will be sampled
                // GPU flushes L2 cache for this texture
            });
        }
    }
}
```

### 3.3 Async Compute Overlap
If Pass A (Shadow Rendering, Graphics Queue) and Pass B (SSAO Compute, Compute Queue) have zero data dependencies, the Frame Graph schedules them on **separate hardware queues**. The GPU physically executes both simultaneously.

**When this helps:** SSAO is bandwidth-heavy (reading depth textures). Shadow rendering is geometry-heavy (vertex processing). They use different hardware units, so overlap is nearly free.

**When this hurts:** If both passes compete for the same L2 cache bandwidth, they evict each other's data, and both run *slower* than sequential. Always profile before assuming async compute helps!

---

## 4. Multi-Platform Abstraction Strategies

### 4.1 The Backend Architecture Pattern

Never abstract the API surface. Abstract the **Data Descriptors**.

```
APPLICATION LAYER (Platform-Agnostic)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  PipelineDescriptor {
      vertexShader: "mesh.vert",
      fragmentShader: "pbr.frag",
      depthTest: true,
      blendMode: ALPHA_BLEND,
      vertexLayout: [POSITION_F32x3, NORMAL_F32x3, UV_F32x2]
  }
        │
        ▼
BACKEND LAYER (Platform-Specific)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐
  │ Vulkan      │  │  WebGPU      │  │  Metal      │
  │ Backend     │  │  Backend     │  │  Backend    │
  └──────┬──────┘  └──────┬───────┘  └──────┬──────┘
         │               │                  │
   VkPipeline    GPURenderPipeline   MTLRenderPipeline
```

### 4.2 Shader Cross-Compilation
You write shaders in ONE language (e.g., WGSL or a custom DSL).
A build-time transpiler (like Naga, SPIRV-Cross, or Tint) converts:
- WGSL → SPIR-V (for Vulkan)
- WGSL → MSL (for Metal)
- WGSL → GLSL (for WebGL fallback)

**Critical Gotcha:** Different backends have different `uniform` alignment rules. Metal requires 16-byte alignment for all struct members. Vulkan uses `std140` layout. Your shader cross-compiler must automatically inject padding bytes!

---

## 5. Experimental Innovation Project

### Build a Compute-Only Software Rasterizer in WebGPU

**Why?** Unreal Engine 5's Nanite bypasses hardware rasterization for tiny triangles because when triangles are smaller than 2×2 pixels, the hardware rasterizer's fixed-function 2×2 Quad overhead makes it *slower* than software math.

**Architecture:**
```wgsl
// Step 1: Vertex Projection (Compute Shader)
@compute @workgroup_size(64)
fn projectVertices(@builtin(global_invocation_id) id: vec3u) {
    let vertex = vertices[id.x];
    let clipPos = viewProjMatrix * vec4f(vertex.position, 1.0);
    let ndc = clipPos.xyz / clipPos.w;
    
    // Convert NDC [-1,1] to screen pixels [0, width]
    let screenX = u32((ndc.x * 0.5 + 0.5) * f32(screenWidth));
    let screenY = u32((ndc.y * -0.5 + 0.5) * f32(screenHeight));
    
    projectedVerts[id.x] = vec2u(screenX, screenY);
}

// Step 2: Atomic Depth Test (Compute Shader)
// Use atomicMin on a u32 depth buffer to resolve visibility
@compute @workgroup_size(64)
fn rasterizeTriangles(@builtin(global_invocation_id) id: vec3u) {
    let triIdx = id.x;
    let v0 = projectedVerts[indices[triIdx * 3 + 0]];
    let v1 = projectedVerts[indices[triIdx * 3 + 1]];
    let v2 = projectedVerts[indices[triIdx * 3 + 2]];
    
    // Bounding box of the triangle on screen
    let minX = min(v0.x, min(v1.x, v2.x));
    let maxX = max(v0.x, max(v1.x, v2.x));
    // ... iterate pixels in bounding box ...
    // For each pixel: edge function test, then atomicMin depth
    
    let depthAsUint = bitcast<u32>(interpolatedDepth);
    atomicMin(&depthBuffer[pixelIndex], depthAsUint);
}
```

**Performance Target:** Beat the hardware rasterizer on scenes with 10 million sub-pixel triangles. Profile using WebGPU Timestamp Queries.

---

## 6. Debugging Strategies for Hybrid Pipelines

| Symptom | Likely Cause | Debugging Approach |
|---------|-------------|-------------------|
| Reflections pop in/out at screen edges | SSR → RT fallback transition is abrupt | Add a 10% screen-edge fade zone blending SSR into cubemap fallback |
| GI has bright spots that dance/flicker | DDGI probe hysteresis too low | Visualize probe irradiance as colored spheres in the debug view |
| Shadow quality suddenly degrades at distance | CSM cascade boundary visible | Render cascade indices as false-color overlay (Red=0, Green=1, Blue=2) |
| Frame time spikes every 4th frame | Async compute fence stall | Check GPU timeline in RenderDoc for unexpected sync points |
| Memory usage grows unboundedly | Frame Graph not reclaiming transient resources | Log every `createTexture` call; count should be 0 after initialization |
