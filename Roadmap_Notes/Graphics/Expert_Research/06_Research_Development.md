# Module 6 — Original Research Development

> This module teaches the methodology of conducting rendering research:
> reading papers, designing experiments, benchmarking, and publishing.

---

## 1. How to Read SIGGRAPH Papers (Systematic Method)

### 1.1 The 4-Pass Reading Strategy

Academic papers are intimidating. They contain dense LaTeX, obscure notation, and assume familiarity with a decade of prior work. Never read them linearly.

```
PASS 1: TRIAGE (5 minutes)
══════════════════════════
  Read ONLY:
    - Title
    - Abstract
    - All figures and their captions
    - The comparison images at the end
  
  DECISION: "Is the visual improvement worth implementing?"
  If the paper shows a 5% improvement over existing tech: SKIP.
  If the paper shows a fundamental new capability: CONTINUE.

PASS 2: UNDERSTANDING (30 minutes)
═══════════════════════════════════
  Read ONLY:
    - Section 1 (Introduction) — WHY previous methods fail
    - Section 2 (Related Work) — WHAT other approaches exist
    - The Algorithm Overview / Pipeline Figure
    - Section "Results" — Performance numbers (ms, quality metrics)
  
  Skip all math derivations. Focus on the HIGH-LEVEL PIPELINE.
  
  After this pass, you should be able to explain the technique
  to a colleague in 3 sentences.

PASS 3: IMPLEMENTATION DETAILS (2 hours)
════════════════════════════════════════
  Now read:
    - The implementation section
    - Pseudocode (if provided)
    - Buffer formats, texture dimensions, workgroup sizes
    - Any supplementary code or GitHub links
  
  Translate the math into mental GLSL/WGSL:
    ∫ → for loop
    Σ → += accumulator
    ∇ → finite differences (dFdx, dFdy)
    E[X] → average over N samples

PASS 4: DEEP MATH (Only if implementing)
═════════════════════════════════════════
  Only now read the full mathematical derivation.
  Verify step-by-step that each equation flows into the next.
  Check: Does the final equation match the pseudocode?
```

### 1.2 Notation Cheat Sheet for Graphics Papers

| Symbol | Graphics Meaning | GLSL/WGSL Equivalent |
|--------|-----------------|---------------------|
| $\omega_o$ | View direction (toward camera) | `normalize(cameraPos - fragPos)` |
| $\omega_i$ | Incoming light direction | `normalize(lightPos - fragPos)` |
| $\mathbf{n}$ or $\hat{n}$ | Surface normal | `normalize(vNormal)` |
| $\mathbf{h}$ | Halfway vector | `normalize(V + L)` |
| $f_r$ | BRDF | `evaluateBRDF()` |
| $L_i$, $L_o$ | Incoming / Outgoing radiance | `vec3 radiance` |
| $\alpha$ | Roughness (usually squared) | `roughness * roughness` |
| $\Omega$ | Hemisphere of directions | Loop over sample directions |
| $p(\omega)$ or pdf | Probability density function | `float pdf` |
| $\langle \cdot \rangle$ | Clamped dot product | `max(dot(a,b), 0.0)` |
| $\otimes$ | Component-wise multiply | `a * b` (vec3 multiply) |
| $\sigma$ | Standard deviation / density | `float sigma` |
| $\nabla$ | Gradient / derivative | `dFdx()`, `dFdy()` |

---

## 2. Extracting Implementation Ideas — The Approximation Mindset

### 2.1 The Framework for Real-Time Approximation

Every research paper presents a technique that takes N seconds offline. Your job is to compress it to M milliseconds. Here is a systematic framework:

```
THE APPROXIMATION LADDER:
═════════════════════════
  RESEARCH ALGORITHM                     YOUR REAL-TIME HACK
  ─────────────────                     ───────────────────
  1024 samples per pixel          →     1 sample + temporal accumulation
  Full scene BVH traversal        →     Screen-space depth buffer march
  Exact path tracing (10 bounces) →     1 bounce + SH probe interpolation
  Analytically precise integral   →     Pre-baked LUT texture lookup
  Complex polynomial function     →     Rational Padé approximant
  Per-pixel Monte Carlo sampling  →     Shared spatial neighbor reuse (ReSTIR)
  Full 3D volumetric evaluation   →     2D heightmap/distance field proxy
```

### 2.2 Case Study: Approximating Subsurface Scattering

```
RESEARCH (2001): Jensen's BSSRDF model
  - For translucent materials (skin, milk, marble)
  - Shoots thousands of rays INSIDE the surface
  - Simulates light scattering through tissue
  - Cost: minutes per pixel

REAL-TIME HACK (Screen-Space SSS):
  1. Render the scene normally (Deferred).
  2. In a post-processing pass, detect "skin" pixels via material ID.
  3. For each skin pixel, perform a Gaussian blur in screen-space,
     BUT weight the blur by DEPTH DIFFERENCE:
     
     // Only blur with neighbors at similar depth (same face, not ear behind nose)
     float depthDiff = abs(centerDepth - neighborDepth);
     float weight = exp(-depthDiff * depthDiff / (2.0 * sigma * sigma));
     
  4. Blend the blurred result into the diffuse channel.
  
  COST: ~0.3ms. Looks 90% as good as the research technique.
```

---

## 3. Designing Rendering Experiments

### 3.1 The Controlled Environment

```
EXPERIMENTAL SETUP CHECKLIST:
═════════════════════════════
  1. SCENE: Use standard test scenes with known properties:
     - Cornell Box (diffuse-only, closed room, 1 area light)
     - Sponza (architectural, hard shadows, thin geometry)
     - San Miguel (vegetation, complex occlusion)
     - Dragon/Bunny (smooth curved geometry, subsurface)
  
  2. CAMERA: Lock the camera to 3 fixed positions.
     Never compare screenshots from different angles!
  
  3. LIGHTING: Use physically accurate light values.
     Directional light: 100,000 lux (sunlight)
     Point light: 800 lumens (60W bulb)
     Environment: HDRI probe (provide the .hdr filename for reproduction)
  
  4. BASELINE: Always render a "Ground Truth" reference.
     Path trace at 4096 spp with no denoising.
     This is the image your technique must approach.
  
  5. METRICS: Use established quality metrics:
     - PSNR (Peak Signal-to-Noise Ratio): Higher is better. > 30dB = good.
     - SSIM (Structural Similarity): 0 to 1. > 0.95 = excellent.
     - FLIP (perceptual difference): Lower is better.
```

### 3.2 A/B Comparison Implementation

```javascript
// Split-screen comparison with draggable slider
class RenderComparison {
    constructor(canvas, groundTruthTexture, experimentTexture) {
        this.slider = 0.5; // 50% split
        
        // The comparison shader
        this.shaderCode = `
            @fragment
            fn main(@location(0) uv: vec2f) -> @location(0) vec4f {
                let groundTruth = textureSample(gtTex, sampler, uv);
                let experiment  = textureSample(expTex, sampler, uv);
                
                if (uv.x < sliderPosition) {
                    return groundTruth;
                } else {
                    return experiment;
                }
            }
        `;
    }
    
    // Error visualization mode
    renderErrorMap() {
        // Amplify the difference by 10× and display as a heatmap
        // fragColor = abs(groundTruth - experiment) * 10.0;
        // Red = large error, Black = perfect match
    }
}
```

---

## 4. Benchmarking & GPU Profiling Methodology

### 4.1 The Correct Way to Measure GPU Time

```
WRONG:
  const start = performance.now();
  renderPass();
  const end = performance.now();
  // This measures CPU TIME, not GPU time!
  // The GPU may still be processing for another 5ms after this returns.

RIGHT:
  Use GPU Timestamp Queries (see Module 4 for WebGPU implementation).
  
  Additionally, measure STABILITY, not just single-frame timing:
  
  Run the benchmark for 1000 frames.
  Discard the first 100 frames (JIT warmup, shader compilation cache).
  Report:
    - Mean: average ms
    - P95:  95th percentile ms (worst 5% of frames)
    - P99:  99th percentile ms (worst 1% — the "hitches")
    - Min:  best-case ms
    - Std:  standard deviation
```

### 4.2 Benchmark Reporting Template

```markdown
## Results: [Technique Name]

| Pass           | Mean (ms) | P95 (ms) | P99 (ms) | GPU Bottleneck |
|----------------|-----------|----------|----------|----------------|
| Shadow Map     |     0.82  |    0.91  |    1.05  | Vertex (geometry) |
| G-Buffer       |     1.45  |    1.52  |    1.68  | Bandwidth (MRT write) |
| **[My Tech]**  |   **2.10**|  **2.35**|  **2.80**| **ALU (BRDF math)** |
| Lighting       |     1.80  |    1.95  |    2.20  | Bandwidth (G-Buffer read) |
| Post-Process   |     0.60  |    0.65  |    0.72  | Bandwidth (full-screen) |
| **TOTAL**      |   **6.77**|  **7.38**|  **8.45**| |

**Quality Metrics (vs 4096spp Ground Truth):**
| Metric | Value |
|--------|-------|
| PSNR   | 32.4 dB |
| SSIM   | 0.967 |
| FLIP   | 0.042 |

**Hardware:** NVIDIA RTX 3070, 1920×1080, Driver 535.104
**Scene:** Sponza, 1 directional light + 32 point lights
```

---

## 5. Writing Technical Blog Posts

### 5.1 The Optimal Blog Post Structure

```
ENGAGING TECHNICAL BLOG POST TEMPLATE:
══════════════════════════════════════
  TITLE: "Real-Time Caustics in WebGPU: A Photon Splatting Approach"
  
  SECTION 1: THE HOOK (3 sentences + 1 GIF/video)
    Show the beautiful final result immediately.
    "Here's what we're building. It runs at 60fps in a browser."
    [Embed: caustics_final.mp4]
  
  SECTION 2: THE PROBLEM (1 paragraph + comparison image)
    "Traditional screen-space caustics produce [ugly artifacts].
     Here's why the existing approach fails:"
    [Image: comparison_artifacts.png]
  
  SECTION 3: PRIOR ART (bullet list)
    "Previous approaches include:
     - Backward ray tracing (Wyman, 2008) — too slow
     - Screen-space approximation (Sousa, 2011) — artifacts
     - Our approach builds on photon mapping (Jensen, 1996)"
  
  SECTION 4: OUR APPROACH (diagrams + key equations)
    Clear pipeline diagram.
    The 2-3 most important equations, each followed by
    "In English, this means..."
  
  SECTION 5: IMPLEMENTATION (code snippets)
    Show the actual WGSL/GLSL code.
    Annotate EVERY line with comments.
    Readers should be able to copy-paste and run it.
  
  SECTION 6: RESULTS (performance table + comparison)
    Side-by-side: Ground Truth vs Our Method vs Previous Method
    Include GPU timing data.
  
  SECTION 7: LIMITATIONS & FUTURE WORK
    Be honest about what doesn't work.
    "Our method fails when [edge case]. Future work could address..."
  
  FOOTER: GitHub link to minimal reproducible demo
```

---

## 6. Open-Sourcing Experimental Engines

### 6.1 The "Minimal Reproducible Demo" Standard

```
IDEAL OPEN-SOURCE RENDERING EXPERIMENT:
═══════════════════════════════════════
  Repository structure:
  ├── index.html          (Single HTML file, no framework)
  ├── main.js             (All logic, < 1000 lines)
  ├── shaders/
  │   ├── gbuffer.wgsl
  │   ├── lighting.wgsl
  │   └── caustics.wgsl   (YOUR novel contribution)
  ├── assets/
  │   ├── scene.glb
  │   └── hdri.hdr
  ├── README.md
  │   ├── What this does (1 sentence)
  │   ├── Screenshot / GIF
  │   ├── How to run (must be: "open index.html")
  │   ├── How it works (3 paragraphs)
  │   └── Performance data
  └── LICENSE (MIT)
  
  RULES:
  ✅ Zero NPM dependencies. Zero build step.
  ✅ Works by opening index.html in Chrome Canary.
  ✅ Includes a pre-built .glb scene (don't make users find assets).
  ✅ Runs on a mid-range GPU (don't require an RTX 4090).
  ❌ No 50,000-line engine framework.
  ❌ No proprietary assets or shaders.
```
