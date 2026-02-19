# The Innovator's Blueprint (Expanded)

> Your complete 5-year career trajectory, foundational paper reading list,
> 10 annotated experimental projects, engine architecture template, and checklists.

---

## 1. The 5-Year Mastery Roadmap

### Year 1: Systems Engineering & Data-Oriented Design
- **Goal:** Stop thinking about "drawing triangles." Start thinking about "moving bytes."
- **Core Skills:**
  - Build a custom Linear Allocator and Pool Allocator in JavaScript/TypeScript.
  - Refactor an existing renderer to achieve **zero `new`/`malloc` calls per frame**.
  - Convert all rendering data from Array-of-Structs (AoS) to Struct-of-Arrays (SoA) for GPU cache efficiency.
  - Profile the CPU-side rendering loop in Chrome DevTools. Identify and eliminate all garbage collection pauses.
- **Milestone:** Your render loop runs with 0 GC pauses and 0 dynamic allocations.
- **Study:** "Data-Oriented Design" by Richard Fabian. Mike Acton's "Data-Oriented Design and C++" (CppCon 2014 talk).

### Year 2: Frame Graph Architecture & GPU Pipeline Mastery
- **Goal:** Decouple rendering logic from execution. Build a compiler, not a renderer.
- **Core Skills:**
  - Implement a Directed Acyclic Graph (DAG) that represents your rendering passes.
  - Implement automated transient resource aliasing (dead textures reuse VRAM addresses).
  - Implement automatic pipeline barrier insertion based on read/write dependencies.
  - Build async compute overlap between independent passes.
- **Milestone:** Adding a new post-processing effect to your engine is a 5-line declaration, not a 200-line code change.
- **Study:** Yuriy O'Donnell's "FrameGraph: Extensible Rendering Architecture in Frostbite" (GDC 2017).

### Year 3: Advanced Light Transport & Stochastic Rendering
- **Goal:** Embrace noise. Master variance reduction. Build denoisers.
- **Core Skills:**
  - Implement Importance Sampling for the GGX BRDF (Module 3's shader code).
  - Build a full SVGF A-Trous denoiser with edge-stopping functions.
  - Implement a basic ReSTIR pipeline for many-light sampling.
  - Profile shaders at the instruction level using NVIDIA Nsight (SASS assembly view).
- **Milestone:** Achieve stable 60fps with a computationally heavy stochastic effect (caustics, volumetric fog) using only 1spp + temporal accumulation + denoising.
- **Study:** Schied et al. "SVGF" (2017), Bitterli et al. "ReSTIR" (2020).

### Year 4: Neural Rendering & ML Integration
- **Goal:** Make neural networks a first-class citizen in your rendering pipeline.
- **Core Skills:**
  - Train a tiny Auto-Encoder in PyTorch to denoise a static noisy image.
  - Export the model to ONNX format.
  - Write a WebGPU compute shader (or use ONNX Runtime Web) to run inference on the GPU.
  - Replace the mathematical SVGF denoiser with the trained neural denoiser. Compare quality/performance.
- **Milestone:** Your engine has a toggle: "Mathematical Denoiser" vs "Neural Denoiser." The neural version produces better quality at comparable speed.
- **Study:** Chaitanya et al. "Interactive Reconstruction of Monte Carlo Image Sequences" (2017).

### Year 5: Original Contribution & Public Impact
- **Goal:** Give back to the community. Publish original work.
- **Core Skills:**
  - Identify an inefficiency or artifact in a published SIGGRAPH technique.
  - Design and implement a mathematical approximation that runs 3-5× faster.
  - Build a minimal, standalone WebGPU demo (zero dependencies, < 1000 lines).
  - Write a comprehensive technical blog post with performance comparisons.
  - Present at a local game dev meetup or graphics conference.
- **Milestone:** Your open-source demo has 100+ GitHub stars. Another developer has cited or used your technique.
- **Study:** Read the latest SIGGRAPH proceedings. Submit to JCGT, I3D, or HPG.

---

## 2. Research Reading Order (Annotated Foundational Papers)

Read these in order. Each builds on the previous.

| # | Paper | Year | Key Contribution | Effort |
|---|-------|------|-----------------|--------|
| 1 | "Physically Based Shading at Disney" (Burley) | 2012 | Unified the PBR material model. All modern engines use this. | ★★☆ |
| 2 | "Real Shading in Unreal Engine 4" (Karis) | 2013 | How to approximate Disney BRDF for real-time. Split-sum, pre-filtered env maps. | ★★☆ |
| 3 | "Moving Frostbite to PBR" (Lagarde, de Rousiers) | 2014 | Production PBR pipeline. Area lights, IES profiles, LUT-based BRDF integration. | ★★★ |
| 4 | "Volumetric Fog & Lighting" (Wronski) | 2014 | Froxel-based volumetric scattering using 3D textures and compute shaders. | ★★★ |
| 5 | "FrameGraph" (O'Donnell) | 2017 | Declarative rendering architecture used in Frostbite (Battlefield). | ★★☆ |
| 6 | "SVGF" (Schied et al.) | 2017 | Edge-aware A-Trous denoising for 1spp path-traced images. The denoising bible. | ★★★★ |
| 7 | "ReSTIR" (Bitterli et al.) | 2020 | Sample millions of lights with 1 ray/pixel via reservoir resampling. | ★★★★ |
| 8 | "A Survey on Baking Neural Radiance Fields" | 2022 | How to convert slow NeRFs into fast, renderable formats. | ★★★ |
| 9 | "3D Gaussian Splatting" (Kerbl et al.) | 2023 | Replaces polygons with differentiable Gaussians. Real-time photorealism. | ★★★★ |
| 10 | "Generative AI for Rendering" (various) | 2024+ | Neural texture synthesis, learned materials, AI-driven LOD. Active research area. | ★★★★★ |

---

## 3. Experimental Project List (10 Projects with Detailed Specifications)

### Project 1: The PBR Material Comparator
**Goal:** Render 10 spheres, each with different metallic/roughness values. Add slider controls.
**Key Learning:** The exact visual response of Cook-Torrance under varying parameters.
**GPU Focus:** ALU-bound (pure BRDF math). Profile register usage.
**Estimated Time:** 1 week.

### Project 2: Compute-Only Software Rasterizer
**Goal:** Project and rasterize triangles using `@compute` shaders and `atomicMin` depth testing.
**Key Learning:** Understanding when hardware rasterization loses to software (sub-pixel triangles).
**GPU Focus:** Atomic contention on the depth buffer. Measure throughput vs triangle size.
**Estimated Time:** 2 weeks.

### Project 3: GPU-Driven Frustum Culling
**Goal:** Move frustum culling entirely to a Compute Shader. Output a packed `indirectDrawBuffer`.
**Key Learning:** `drawIndexedIndirect()`, GPU-driven rendering, eliminating CPU bottleneck.
**GPU Focus:** Subgroup ballot operations for efficient compaction.
**Estimated Time:** 1 week.

### Project 4: SVGF Edge-Avoiding Denoiser
**Goal:** Implement the full A-Trous wavelet filter with depth/normal/luminance edge-stopping.
**Key Learning:** How denoising allows 1spp path tracing to look clean. Trade-off: blur vs detail.
**GPU Focus:** Bandwidth-heavy (25 texture reads per pass × 5 passes).
**Estimated Time:** 2 weeks.

### Project 5: Cascaded Shadow Maps with Smooth Transitions
**Goal:** Implement 3-cascade CSM with cross-cascade blending to eliminate visible transitions.
**Key Learning:** Orthographic projection fitting, cascade distance tuning, depth precision.
**GPU Focus:** Measuring shadow map resolution waste (how many texels are unused per cascade).
**Estimated Time:** 2 weeks.

### Project 6: Volumetric Fog (Froxel-Based)
**Goal:** Slice the camera frustum into a 160×90×64 3D grid. Inject shadow data. Raymarch and compose.
**Key Learning:** 3D texture creation, exponential fog scattering math, temporal jitter.
**GPU Focus:** 3D texture bandwidth. Measure L2 cache pressure from random 3D sampling.
**Estimated Time:** 3 weeks.

### Project 7: Screen-Space Reflections (Hi-Z Raymarching)
**Goal:** Build a Hierarchical-Z buffer. March reflected rays through it at progressively finer LODs.
**Key Learning:** Hi-Z construction (compute mipchain), ray-depth intersection, fallback to cubemaps.
**GPU Focus:** Bandwidth (reading Hi-Z mips randomly). Measure cache miss rate.
**Estimated Time:** 2 weeks.

### Project 8: Temporal Anti-Aliasing (From Scratch)
**Goal:** Implement Halton jitter, motion vector generation, history reprojection, and neighborhood clamping.
**Key Learning:** Sub-pixel sampling theory, ghosting prevention, integration with post-processing.
**GPU Focus:** History buffer bandwidth cost. Measure quality vs temporal blend factor.
**Estimated Time:** 2 weeks.

### Project 9: ReSTIR Direct Lighting (Many Lights)
**Goal:** Place 10,000 point lights randomly. Implement the full reservoir sampling pipeline.
**Key Learning:** Weighted reservoir sampling, MIS weights, spatial/temporal reuse correctness proofs.
**GPU Focus:** Compare 10,000 lights with brute-force loop vs ReSTIR. Measure speedup.
**Estimated Time:** 4 weeks.

### Project 10: Neural Denoiser Inference in WebGPU
**Goal:** Export a pre-trained 4-layer Conv2D denoiser to weight arrays. Run inference in compute shaders.
**Key Learning:** ML inference on GPU without frameworks. Quantization, activation functions in WGSL.
**GPU Focus:** Compute throughput. Compare Tensor Core efficiency (if available) vs standard ALU.
**Estimated Time:** 4 weeks.

---

## 4. Engine Architecture Blueprint Template

```
THE PROFESSIONAL RENDERER ARCHITECTURE:
═══════════════════════════════════════

  ┌─────────────────────────────────────────────────────┐
  │                    APPLICATION LAYER                  │
  │  ECS, Gameplay, Physics, Animation, Audio            │
  │  Knows NOTHING about WebGPU/Vulkan/Metal             │
  │  Output: "Render Proxy" = flat ArrayBuffer of data   │
  └────────────────────────┬────────────────────────────┘
                           │ Transfer (WebWorker)
  ┌────────────────────────▼────────────────────────────┐
  │                    FRAME GRAPH COMPILER               │
  │  Input: Active settings (bloom? shadows? RT?)        │
  │  Output: Compiled DAG of passes                      │
  │  Actions: Memory aliasing, barrier insertion          │
  │           Dead pass elimination, async compute split  │
  └────────────────────────┬────────────────────────────┘
                           │
  ┌────────────────────────▼────────────────────────────┐
  │                    COMMAND DISPATCHER                  │
  │  Iterates compiled DAG                               │
  │  Calls: setPipeline, setBindGroup, draw, dispatch    │
  │  Submits: commandEncoder.finish() → queue.submit()   │
  └────────────────────────┬────────────────────────────┘
                           │
  ┌────────────────────────▼────────────────────────────┐
  │                    SHADER LIBRARY                     │
  │  Modular WGSL/GLSL chunks:                           │
  │    BRDF.wgsl, Shadows.wgsl, SSAO.wgsl, Bloom.wgsl  │
  │  Assembled at runtime via string includes             │
  │  Permutation system: #ifdef HAS_NORMAL_MAP           │
  └────────────────────────┬────────────────────────────┘
                           │
  ┌────────────────────────▼────────────────────────────┐
  │                    RESOURCE MANAGER                   │
  │  Texture Atlas, Buffer Pool, Pipeline Cache          │
  │  Async asset streaming (Virtual Texture pages)        │
  │  Hot shader reloading (dev mode only)                 │
  └─────────────────────────────────────────────────────┘
```

---

## 5. Innovation Checklist

Before publishing or shipping a rendering technique:

### Physical Correctness
- [ ] Light intensity is measured in physically accurate units (Lux, Lumens, Candela)?
- [ ] Energy is conserved? (kD + kS ≤ 1.0 for all materials?)
- [ ] Colors are processed in Linear space? (sRGB only at final output?)
- [ ] The technique handles edge cases? (NdotV = 0, roughness = 0, roughness = 1?)

### Temporal Stability
- [ ] No flickering when the camera is perfectly still?
- [ ] No ghosting trails on fast-moving objects?
- [ ] Frame-rate independent? (deltaTime used correctly in all animations?)

### Robustness
- [ ] Works at multiple resolutions (720p, 1080p, 4K)?
- [ ] Works with 1 light AND 1000 lights?
- [ ] Degrades gracefully on weaker hardware (no crashes, just lower quality)?
- [ ] No NaN propagation? (All divisions protected by `max(denom, 0.001)`?)

---

## 6. Performance Analysis Checklist

Before claiming "our technique runs at X ms":

### Measurement Validity
- [ ] GPU time measured via Timestamp Queries (not `performance.now()`)?
- [ ] Warmup frames discarded (first 100 frames excluded from statistics)?
- [ ] Results include Mean, P95, P99, and Standard Deviation?
- [ ] Tested on at least 2 different GPU vendors (NVIDIA + AMD or Intel)?

### Optimization Verification
- [ ] Zero VRAM allocations during the render loop?
- [ ] Compute workgroup sizes tuned for target hardware (32/64 threads)?
- [ ] Texture reads are spatially/temporally coherent (no random UV access)?
- [ ] Common math results (dot products, normalize) cached and reused?
- [ ] Texture channels packed efficiently (ORM map, not separate textures)?
- [ ] Resolution-dependent costs scale linearly (not quadratically)?

### Comparison Fairness
- [ ] Compared against the BEST existing alternative (not a straw man)?
- [ ] Both techniques tested on the SAME scene, camera, lighting?
- [ ] Quality metrics (PSNR, SSIM) reported alongside performance numbers?
