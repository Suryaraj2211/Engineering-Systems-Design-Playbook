# Module 9 — Rendering Architecture & Professional Practice

> This final module ties everything together: how to architect a renderer,
> checklists for optimization and debugging, and progressive projects.

---

## 1. Rendering Architecture — The Complete Pipeline

### 1.1 The Professional Renderer Pipeline (Pass Order)

```
FRAME EXECUTION ORDER:
══════════════════════
  ┌───────────────────────────────────┐
  │ 1. SHADOW PASS                    │ Render depth from each light
  │    - CSM cascades (3× render)     │ GPU: Geometry bound
  │    - Point light cubemaps         │ Cost: ~2ms
  └─────────────┬─────────────────────┘
                │
  ┌─────────────▼─────────────────────┐
  │ 2. DEPTH PRE-PASS                  │ Render depth only (no color)
  │    - Populates Hi-Z buffer for SSR │ Eliminates overdraw in G-Buffer
  │                                    │ Cost: ~0.3ms
  └─────────────┬─────────────────────┘
                │
  ┌─────────────▼─────────────────────┐
  │ 3. G-BUFFER PASS                   │ Write albedo, normals, PBR params
  │    - MRT output (3-4 targets)      │ GPU: Bandwidth bound (writing MRTs)
  │    - Motion vectors                │ Cost: ~1.5ms
  └─────────────┬─────────────────────┘
                │
  ┌─────────────▼─────────────────────┐
  │ 4. SSAO/GTAO                       │ Ambient occlusion from depth
  │    - Half-res compute              │ GPU: Bandwidth (depth reads)
  │    - Bilateral blur                │ Cost: ~0.8ms
  └─────────────┬─────────────────────┘
                │
  ┌─────────────▼─────────────────────┐
  │ 5. LIGHTING PASS                   │ PBR + all lights + shadows
  │    - Read G-Buffer + shadow maps   │ GPU: ALU (BRDF math) + Bandwidth
  │    - Output HDR color (RGBA16F)    │ Cost: ~2ms
  └─────────────┬─────────────────────┘
                │
  ┌─────────────▼─────────────────────┐
  │ 6. SSR (Screen-Space Reflections)  │ Hi-Z raymarch
  │    - Quarter-res → bilateral up    │ GPU: Bandwidth
  │                                    │ Cost: ~0.6ms
  └─────────────┬─────────────────────┘
                │
  ┌─────────────▼─────────────────────┐
  │ 7. TRANSPARENT PASS (Forward)      │ Glass, particles, water
  │    - Sorted back-to-front          │ GPU: Overdraw, alpha blending
  │    - Separate from deferred        │ Cost: Variable
  └─────────────┬─────────────────────┘
                │
  ┌─────────────▼─────────────────────┐
  │ 8. POST-PROCESSING (all in HDR)    │
  │    a. Bloom (extract + blur chain) │ ~1ms
  │    b. Auto-exposure                │ ~0.1ms
  │    c. Tone mapping (ACES)          │ ~0.1ms
  │    d. TAA                          │ ~0.5ms
  │    e. Gamma correction (sRGB out)  │ ~0.05ms
  └─────────────┬─────────────────────┘
                │
                ▼
          FINAL 8-BIT sRGB OUTPUT TO MONITOR
          
  TOTAL: ~9ms = 111fps at 1080p (RTX 3070)
```

---

## 2. Optimization Checklist

### Memory & Bandwidth
- [ ] G-Buffer uses packed formats (ORM in 1 texture, octahedral normals)?
- [ ] World positions reconstructed from depth (no position buffer)?
- [ ] Shadow maps use `D16` format (not `D32F`) where precision allows?
- [ ] SSAO/SSR computed at half-resolution with bilateral upsample?
- [ ] Textures use compressed formats (BC7/ASTC) where possible?
- [ ] Mipmaps generated for all textures (prevents shimmering + saves bandwidth)?

### Rendering
- [ ] Depth pre-pass enabled (eliminates overdraw in G-Buffer)?
- [ ] Draw calls sorted by material (minimize pipeline state changes)?
- [ ] Instancing used for repeated objects (trees, particles)?
- [ ] Frustum culling on CPU before submission (skip off-screen objects)?
- [ ] Small objects culled by screen-space size (skip sub-pixel objects)?
- [ ] LOD system for mesh complexity at distance?

### Shaders
- [ ] All dot products clamped (`max(dot(N,V), 0.001)`)?
- [ ] All divisions protected against zero (`max(denominator, 0.0001)`)?
- [ ] No redundant calculations (cache `NdotL`, `NdotV`, `NdotH`)?
- [ ] Texture samples minimized (pack ORM, use atlas)?
- [ ] `mediump` used for intermediate values where precision isn't critical?

### Post-Processing
- [ ] Bloom operates on HDR data (before tone mapping)?
- [ ] Tone mapping is the LAST operation before gamma?
- [ ] Gamma correction applied ONCE (not zero times, not twice)?
- [ ] TAA has proper motion vectors for ALL moving objects?

---

## 3. Debugging Checklist

When something looks wrong, systematically isolate the problem:

### Step 1: Isolate the Pass
```
Disable ALL post-processing. Does the problem persist?
  YES → Problem is in geometry, G-Buffer, or lighting pass.
  NO  → Problem is in post-processing. Enable one pass at a time.
```

### Step 2: Visualize G-Buffer Channels
```
Switch debug view to show each G-Buffer channel independently:
  Albedo:    Should look like a flat-shaded, unlit scene.
  Normals:   Should smoothly transition across surfaces (no black gaps).
  Metallic:  Should be binary (0 or 1) for most materials.
  Roughness: Should vary smoothly. Pure 0.0 = suspicious.
  Depth:     Should show correct distance relationships.
```

### Step 3: Check Lighting
```
Reduce to 1 directional light. No shadows. No AO. No SSR.
  Does the material look correct under simple lighting?
  YES → Problem is in secondary effects (shadows, AO, SSR, etc.).
  NO  → Problem is in PBR BRDF or material data.
```

### Step 4: Check Color Pipeline
```
  Output raw HDR color (no tone mapping, no gamma):
    Are values > 1.0 present? (They should be for lit areas)
    Are values negative? (They should NEVER be → NaN bug)
    
  Apply tone mapping only:
    Does the range compress to [0, 1] correctly?
    
  Apply gamma only:
    Does the image get appropriately brighter?
```

---

## 4. Mini-Projects (5 — Increasing Difficulty)

### Project 1: PBR Material Viewer (1 week)
Render 5×5 grid of spheres. Each row varies metallic (0→1). Each column varies roughness (0→1). Add a rotating environment light. Implement full Cook-Torrance with the code from Module 1.
**Skills tested:** BRDF implementation, F0 computation, energy conservation.

### Project 2: Deferred Renderer (2 weeks)
Build a 2-pass deferred renderer with 3-channel G-Buffer + depth reconstruction. Support 50 dynamic point lights with light volumes (icosphere meshes). Add an SSAO pass with 4×4 noise texture.
**Skills tested:** MRT setup, G-Buffer encoding, light accumulation.

### Project 3: Shadow System (2 weeks)
Add 3-cascade CSM for the directional light. Implement slope-scale bias, 5×5 PCF, and cascade transition blending. Add screen-space contact shadows for small-scale detail.
**Skills tested:** Light-space projection, depth comparison, cascade management.

### Project 4: Full Post-Processing Stack (2 weeks)
Add HDR framebuffer (RGBA16F). Implement: Bright pass extraction → 6-level bloom → ACES tone mapping → auto-exposure. Add TAA with Halton jitter, motion vectors, and neighborhood clamping.
**Skills tested:** Multi-pass rendering, mipmap chain, temporal stability.

### Project 5 (Capstone): PBR + Deferred Renderer in WebGPU (4 weeks)
Build the COMPLETE pipeline from this module's Section 1:
- Depth pre-pass → G-Buffer → GTAO → Deferred Lighting → SSR → Forward Transparency → Bloom → Exposure → ACES → TAA → Final Output.
- Support GLTF model loading with PBR materials.
- Include a debug overlay panel showing per-pass GPU timing.
- Target: 60fps with 100 lights on a mid-range GPU.
**Skills tested:** Full pipeline architecture, resource management, performance tuning.

---

## 5. Performance Budget Template

Use this template to track your frame budget:

```
TARGET: 16.67ms per frame (60fps) at 1080p
═══════════════════════════════════════════
  Pass                  Budget(ms)  Actual(ms)  Status
  ──────────────────    ──────────  ──────────  ──────
  Shadow Maps           2.0         ___         [ ]
  Depth Pre-pass        0.5         ___         [ ]
  G-Buffer              2.0         ___         [ ]
  SSAO/GTAO             1.0         ___         [ ]
  Lighting              3.0         ___         [ ]
  SSR                   1.0         ___         [ ]
  Transparency          1.5         ___         [ ]
  Bloom                 1.0         ___         [ ]
  TAA                   0.5         ___         [ ]
  Tone Map + Gamma      0.2         ___         [ ]
  Headroom              3.97        ___         [ ]
  ──────────────────    ──────────  ──────────
  TOTAL                 16.67       ___
  
  If ANY pass exceeds budget by 2×:
    1. Check texture resolution (halve G-Buffer resolution if needed).
    2. Check light count (cull lights aggressively).
    3. Check shader complexity (profile with GPU timestamp queries).
```
