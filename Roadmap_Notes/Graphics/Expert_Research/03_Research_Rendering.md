# Module 3 — Research-Level Rendering & Light Transport

> This module covers the mathematics and algorithmic foundations used in
> cutting-edge real-time rendering research (SIGGRAPH 2017–2025).
> Full derivations are provided so you can implement from scratch.

---

## 1. The Light Transport Equation — Full Mathematical Decomposition

### 1.1 The Equation

$L_o(p, \omega_o) = L_e(p, \omega_o) + \int_{\Omega} f_r(p, \omega_i, \omega_o) \cdot L_i(p, \omega_i) \cdot |\cos\theta_i| \, d\omega_i$

**Symbol-by-symbol breakdown:**

| Symbol | Type | Meaning | Units |
|--------|------|---------|-------|
| $L_o(p, \omega_o)$ | Output | Outgoing radiance from point $p$ toward camera direction $\omega_o$ | $W \cdot m^{-2} \cdot sr^{-1}$ |
| $L_e(p, \omega_o)$ | Known | Emitted radiance (self-glow, e.g., neon sign) | Same |
| $\int_{\Omega}$ | Operator | Integral over the entire hemisphere above the surface | — |
| $f_r$ | Material | The BRDF — probability that light from $\omega_i$ bounces to $\omega_o$ | $sr^{-1}$ |
| $L_i(p, \omega_i)$ | Unknown | Incoming radiance from direction $\omega_i$ (THIS is the recursive nightmare) | Same as $L_o$ |
| $|\cos\theta_i|$ | Geometry | Lambert's cosine law. Light at grazing angles is weaker. | dimensionless |
| $d\omega_i$ | Differential | An infinitesimal solid angle worth of directions | $sr$ |

### 1.2 Why This Is Impossible in Real-Time

To solve $L_o$ for a single pixel, you must solve $L_i$ for every direction on the hemisphere. But $L_i$ is ITSELF $L_o$ of another surface point. This creates infinite recursion:

```
RECURSIVE LIGHT TRANSPORT:
══════════════════════════
  Camera sees Pixel → needs L_o(surface A)
    L_o(A) depends on L_i from direction B
      L_i(B) is actually L_o(surface B)
        L_o(B) depends on L_i from direction C
          L_i(C) is L_o(surface C)
            L_o(C) depends on...
              (infinite recursion)

  In OFFLINE rendering (Pixar): Trace 5–10 bounce levels, 1024 rays/pixel.
  Rendering time: 10 minutes per frame.

  In REAL-TIME rendering (Games): We MUST terminate after 1-2 bounces
  with 1-4 rays per pixel, then DENOISE the garbage.
```

---

## 2. Monte Carlo Integration — Solving Integrals with Randomness

### 2.1 The Core Math

We cannot analytically solve $\int_\Omega f_r \cdot L_i \cdot \cos\theta \, d\omega$. Instead, we *estimate* it using random sampling:

$$\int_\Omega f(x) \, dx \approx \frac{1}{N} \sum_{i=1}^{N} \frac{f(x_i)}{p(x_i)}$$

where $p(x_i)$ is the Probability Density Function (PDF) of our chosen sampling strategy.

**Manual Calculation Example: Estimating Pi**

```
MONTE CARLO PI ESTIMATION:
══════════════════════════
  Problem: Estimate the area of a unit circle (π).
  
  Method: Randomly throw N darts at a 2×2 square.
  Count how many land inside the circle (distance from center < 1).
  
  N = 4 samples: x₁=(0.3,0.5) → inside ✓   x₂=(0.9,0.8) → outside ✗
                  x₃=(0.1,0.2) → inside ✓   x₄=(0.7,0.7) → inside ✓
  
  Hits / Total = 3/4 = 0.75
  Area of square = 4.0
  Estimated π = 0.75 × 4.0 = 3.0  (True: 3.14159)
  
  N = 100,000 samples: Estimated π ≈ 3.14052 (much closer!)
  
  KEY INSIGHT: More samples → less noise. But cost grows linearly.
  At 60fps, we can afford maybe 1-4 samples per pixel.
```

### 2.2 Variance — The Enemy

Variance is the mathematical measure of "noise." High variance = grainy image.

$\text{Variance} = E[X^2] - (E[X])^2$

If you take N=1 sample per pixel:
- Frame 1: The random ray hits a bright light → pixel value = 500,000
- Frame 2: The ray misses → pixel value = 0
- Frame 3: The ray hits → pixel value = 480,000
- Frame 4: The ray misses → pixel value = 0

The pixel rapidly alternates between blinding white and pure black. This is **intolerable noise**.

**Variance Reduction Strategy:** Don't choose ray directions randomly. Choose them **intelligently**.

---

## 3. Importance Sampling — Directing Rays Toward What Matters

### 3.1 The Math

If we sample uniformly ($p(x) = \frac{1}{2\pi}$ for a hemisphere), most rays miss the light source entirely.

If instead we choose $p(x)$ to be proportional to the integrand $f(x)$, the fraction $\frac{f(x_i)}{p(x_i)}$ becomes nearly constant, and variance drops to near-zero!

**Example: BRDF Importance Sampling for GGX**

For a surface with roughness $\alpha$, the GGX normal distribution peaks sharply around the surface normal. We sample microfacet normals $\mathbf{h}$ with probability proportional to $D(\mathbf{h})$:

```glsl
// Generate a microfacet normal 'H' using GGX importance sampling
// xi1, xi2 are uniform random numbers in [0, 1]
vec3 importanceSampleGGX(vec2 Xi, vec3 N, float roughness) {
    float a = roughness * roughness;
    
    // Spherical coordinates of the sampled half-vector
    float phi = 2.0 * PI * Xi.x;
    
    // This is the KEY equation: cosTheta is biased toward the normal
    // for smooth surfaces (small alpha), creating tight sampling
    float cosTheta = sqrt((1.0 - Xi.y) / (1.0 + (a*a - 1.0) * Xi.y));
    float sinTheta = sqrt(1.0 - cosTheta * cosTheta);
    
    // Convert from spherical to Cartesian (tangent space)
    vec3 H;
    H.x = cos(phi) * sinTheta;
    H.y = sin(phi) * sinTheta;
    H.z = cosTheta;
    
    // Convert from tangent space to world space using TBN matrix
    vec3 up = abs(N.z) < 0.999 ? vec3(0.0, 0.0, 1.0) : vec3(1.0, 0.0, 0.0);
    vec3 tangent   = normalize(cross(up, N));
    vec3 bitangent = cross(N, tangent);
    
    vec3 sampleVec = tangent * H.x + bitangent * H.y + N * H.z;
    return normalize(sampleVec);
}

// The corresponding PDF for this sampling strategy:
float pdfGGX(float NdotH, float roughness, float VdotH) {
    float a = roughness * roughness;
    float D = DistributionGGX(NdotH, a);
    return (D * NdotH) / (4.0 * VdotH);
}
```

### 3.2 Visualizing Importance Sampling

```
UNIFORM SAMPLING (1000 rays on hemisphere):
═══════════════════════════════════════════
  Rays spread everywhere equally.
  Only ~5 rays actually hit the tiny area light.
  995 rays contribute 0 radiance. Wasted compute.
  
          ╱ ╲  ← Area Light (tiny)
  ● ● ● ● ● ● ● ● ● ●  ← Rays (uniform, mostly miss)
  ─────────────────────   ← Surface

BRDF IMPORTANCE SAMPLING (1000 rays, clustered around reflection):
══════════════════════════════════════════════════════════════════
  Rays concentrated around the mirror reflection direction.
  ~400 rays hit the area light. Much more signal per ray!
  
          ╱ ╲  ← Area Light
       ●●●●●●●           ← Rays (clustered near reflection)
  ─────────────────────   ← Surface
```

---

## 4. Multiple Importance Sampling (MIS) — The Veach Power Heuristic

### 4.1 The Problem MIS Solves

BRDF sampling clusters rays around the reflection lobe. But if the light source is far from the reflection direction, all rays miss!

Light sampling shoots rays directly at the light source. But if the BRDF lobe is narrow (glossy surface), the math evaluates $f_r \approx 0$ for those directions!

**Neither strategy alone is optimal for all surface/light combinations.**

### 4.2 The Math (Balance Heuristic)

Take one sample from each strategy. Combine using the power heuristic:

$$w_{brdf}(x) = \frac{p_{brdf}(x)^2}{p_{brdf}(x)^2 + p_{light}(x)^2}$$
$$w_{light}(x) = \frac{p_{light}(x)^2}{p_{brdf}(x)^2 + p_{light}(x)^2}$$

$$L_{combined} = \frac{f(x_{brdf}) \cdot w_{brdf}(x_{brdf})}{p_{brdf}(x_{brdf})} + \frac{f(x_{light}) \cdot w_{light}(x_{light})}{p_{light}(x_{light})}$$

### 4.3 Shader Implementation

```glsl
vec3 evaluateMIS(vec3 N, vec3 V, vec3 albedo, float roughness, vec3 F0,
                 vec3 lightPos, vec3 lightColor, float lightRadius) {
    vec3 result = vec3(0.0);
    
    // --- Strategy 1: BRDF Sampling ---
    vec2 xi1 = hammersley(sampleIndex, totalSamples);
    vec3 H = importanceSampleGGX(xi1, N, roughness);
    vec3 L_brdf = reflect(-V, H);
    
    float NdotL_brdf = max(dot(N, L_brdf), 0.0);
    if (NdotL_brdf > 0.0) {
        // Check if this BRDF-sampled direction actually hits the light
        vec3 Li = traceRay(surfacePos, L_brdf); // Returns radiance if light is hit
        
        float pdf_brdf = pdfGGX(max(dot(N, H), 0.0), roughness, max(dot(V, H), 0.0));
        float pdf_light = pdfLightSampling(L_brdf, lightPos, lightRadius);
        
        float w_brdf = (pdf_brdf * pdf_brdf) / (pdf_brdf * pdf_brdf + pdf_light * pdf_light);
        
        vec3 f = evaluateBRDF(N, V, L_brdf, albedo, roughness, F0);
        result += f * Li * NdotL_brdf * w_brdf / max(pdf_brdf, 0.0001);
    }
    
    // --- Strategy 2: Light Sampling ---
    vec3 L_light = samplePointOnLight(lightPos, lightRadius, xi1);
    vec3 dirToLight = normalize(L_light - surfacePos);
    
    float NdotL_light = max(dot(N, dirToLight), 0.0);
    if (NdotL_light > 0.0 && !traceShadowRay(surfacePos, dirToLight)) {
        float pdf_light2 = pdfLightSampling(dirToLight, lightPos, lightRadius);
        float pdf_brdf2 = pdfGGX(max(dot(N, normalize(V + dirToLight)), 0.0), roughness,
                                  max(dot(V, normalize(V + dirToLight)), 0.0));
        
        float w_light = (pdf_light2 * pdf_light2) / (pdf_brdf2 * pdf_brdf2 + pdf_light2 * pdf_light2);
        
        vec3 f2 = evaluateBRDF(N, V, dirToLight, albedo, roughness, F0);
        vec3 Li2 = lightColor / dot(L_light - surfacePos, L_light - surfacePos); // inverse square
        result += f2 * Li2 * NdotL_light * w_light / max(pdf_light2, 0.0001);
    }
    
    return result;
}
```

---

## 5. ReSTIR — Reservoir-Based Spatiotemporal Importance Resampling

### 5.1 The Problem

A scene has 100,000 dynamic emissive lights (neon signs, candles, cars). Even with MIS, evaluating all lights per pixel at 1spp costs ~100ms (100,000 shadow rays!).

### 5.2 The Reservoir Algorithm (Weighted Reservoir Sampling)

The core insight is a 1985 statistics algorithm called **Weighted Reservoir Sampling (WRS)**:

```
RESERVOIR SAMPLING:
═══════════════════
  Goal: Pick 1 item from a stream of N items, weighted by importance,
        using only O(1) memory (you don't store all N items).

  Initialize: reservoir = { sample: null, weightSum: 0.0, M: 0 }

  For each candidate light i (i = 1 to 100,000):
    weight_i = targetPDF(light_i) / sourcePDF(light_i)
    reservoir.weightSum += weight_i
    reservoir.M += 1
    
    if (random() < weight_i / reservoir.weightSum):
        reservoir.sample = light_i   // Replace the stored sample!
  
  After processing ALL 100,000 lights:
    reservoir.sample contains the single most important light
    with MATHEMATICALLY CORRECT probability proportional to its weight!
    
  Cost: O(N) PDF evaluations but only O(1) memory.
        No shadow rays during selection!
```

### 5.3 The Spatio-Temporal Reuse (The Innovation)

Raw WRS still requires iterating 100,000 lights. ReSTIR makes it practical by **reusing reservoirs across pixels and frames**:

```
RESTIR PIPELINE (Per Frame):
════════════════════════════
  STEP 1: Initial Candidates (per pixel)
    Each pixel randomly tests 32 lights (out of 100,000).
    Builds a local reservoir containing the "best" light.
    Cost: 32 PDF evaluations per pixel. No rays traced yet.

  STEP 2: Temporal Reuse
    Each pixel finds its corresponding pixel in the PREVIOUS frame
    (using motion vectors, exactly like TAA reprojection).
    It MERGES the previous frame's reservoir with its current one.
    
    combinedReservoir = merge(currentReservoir, previousReservoir);
    
    After 10 frames, each pixel has effectively tested
    32 × 10 = 320 candidates while only paying for 32 per frame!

  STEP 3: Spatial Reuse
    Each pixel looks at 5 random neighbor pixels on the current screen.
    It merges THEIR reservoirs with its own.
    
    for (int i = 0; i < 5; i++) {
        neighbor = randomNeighborPixel();
        if (geometrySimilar(center, neighbor)) {  // Same surface?
            combinedReservoir = merge(combined, neighbor.reservoir);
        }
    }
    
    Now each pixel has effectively sampled 320 × 5 = 1,600 candidates!

  STEP 4: Final Shading
    Trace exactly ONE shadow ray toward the reservoir's chosen light.
    Evaluate the FULL BRDF for just that one light.
    
    Cost: 1 shadow ray per pixel. That's it.
    
  RESULT: 100,000 lights rendered with ~1 ray per pixel.
```

### 5.4 GPU Cost Analysis

| Stage | Cost (RTX 3080, 1080p) | Bottleneck |
|-------|----------------------|------------|
| Initial candidate sampling (32 lights) | ~0.5ms | ALU (PDF math) |
| Motion vector reprojection (temporal) | ~0.2ms | Bandwidth (reading history) |
| Spatial reuse (5 neighbors) | ~0.8ms | Bandwidth (random neighbor reads) |
| Final shadow ray trace | ~1.5ms | Compute (BVH traversal) |
| **TOTAL** | **~3.0ms** | |

---

## 6. Denoising — Reconstructing Images from 1spp Noise

### 6.1 SVGF (Spatiotemporal Variance-Guided Filtering)

SVGF runs a multi-pass spatial blur that respects geometric boundaries.

**The A-Trous Wavelet Transform:**
Instead of Gaussian blur (which requires a massive kernel at high radii), A-Trous performs multiple passes with geometrically increasing step sizes:

```
A-TROUS FILTER (5 iterations):
═══════════════════════════════
  Pass 0: Step size = 1   (blur radius ~2 pixels)
  Pass 1: Step size = 2   (blur radius ~4 pixels)
  Pass 2: Step size = 4   (blur radius ~8 pixels)
  Pass 3: Step size = 8   (blur radius ~16 pixels)
  Pass 4: Step size = 16  (blur radius ~32 pixels)
  
  Each pass only samples 5×5 = 25 texels (with the step offset).
  Total: 5 × 25 = 125 texture reads.
  Effective blur radius: 32 pixels!
  
  Equivalent Gaussian would need a 65×65 kernel = 4,225 reads!
```

### 6.2 Edge-Stopping Functions (The Critical Detail)

The blur MUST NOT cross geometric edges. We weight each neighbor by geometric similarity:

```glsl
// SVGF A-Trous Denoiser (single pass)
vec3 denoise(ivec2 pixelCoord, int stepSize) {
    vec3 centerColor  = texelFetch(noisyImage, pixelCoord, 0).rgb;
    vec3 centerNormal = texelFetch(gNormal, pixelCoord, 0).rgb;
    float centerDepth = texelFetch(gDepth, pixelCoord, 0).r;
    float centerLuminanceVariance = texelFetch(varianceBuffer, pixelCoord, 0).r;
    
    float sigmaLuminance = 4.0 * sqrt(max(centerLuminanceVariance, 0.0)) + 0.0001;
    
    vec3 sumColor = vec3(0.0);
    float sumWeight = 0.0;
    
    for (int dy = -2; dy <= 2; dy++) {
        for (int dx = -2; dx <= 2; dx++) {
            ivec2 offset = ivec2(dx, dy) * stepSize;
            ivec2 sampleCoord = pixelCoord + offset;
            
            vec3 sampleColor  = texelFetch(noisyImage, sampleCoord, 0).rgb;
            vec3 sampleNormal = texelFetch(gNormal, sampleCoord, 0).rgb;
            float sampleDepth = texelFetch(gDepth, sampleCoord, 0).r;
            
            // --- EDGE-STOPPING WEIGHT 1: Depth ---
            // Large depth difference = different object. DO NOT BLUR.
            float wDepth = exp(-abs(centerDepth - sampleDepth) / 
                             (max(abs(centerDepth), 0.0001) * 0.1));
            
            // --- EDGE-STOPPING WEIGHT 2: Normal ---
            // Different normal direction = geometric edge. DO NOT BLUR.
            float wNormal = pow(max(0.0, dot(centerNormal, sampleNormal)), 128.0);
            
            // --- EDGE-STOPPING WEIGHT 3: Luminance ---
            // Variance-guided: blur more in noisy areas, less in clean areas
            float lCenter = dot(centerColor, vec3(0.2126, 0.7152, 0.0722));
            float lSample = dot(sampleColor, vec3(0.2126, 0.7152, 0.0722));
            float wLuminance = exp(-abs(lCenter - lSample) / sigmaLuminance);
            
            float weight = wDepth * wNormal * wLuminance * kernel5x5[dx+2][dy+2];
            
            sumColor  += sampleColor * weight;
            sumWeight += weight;
        }
    }
    
    return sumColor / max(sumWeight, 0.0001);
}
```

### 6.3 Debugging Denoiser Artifacts

| Artifact | Visual | Root Cause | Fix |
|----------|--------|-----------|-----|
| Over-blurring (textures look like wax) | Loss of high-frequency detail | Edge-stop thresholds too loose | Tighten `phiNormal` and `phiDepth`; reduce blur iterations |
| Ghosting (trails behind moving objects) | Smeared duplicates of moving characters | Temporal accumulation using stale history | Reduce temporal blend alpha; add motion-based rejection |
| Boiling / Shimmering | Noisy swim patterns on static surfaces | Temporal history is rejected too aggressively | Increase temporal blend alpha for static pixels (detect via motion vectors) |
| Dark edges on geometry boundaries | Black halos around objects | Over-aggressive depth edge-stopping | Use gradient-based depth comparison instead of absolute difference |

---

## 7. Neural Rendering Systems Overview

### 7.1 NeRF (Neural Radiance Fields) — Architecture Summary

```
NERF ARCHITECTURE:
══════════════════
  Input: 3D coordinate (x, y, z) + viewing direction (θ, φ)
         │
         ▼
  ┌──────────────────────────┐
  │  MLP (Multi-Layer        │
  │  Perceptron)             │
  │  8 layers × 256 neurons  │
  │  ~1.2 million parameters │
  └──────────┬───────────────┘
             │
             ▼
  Output: RGB color + Density (σ)
  
  RENDERING:
  For each pixel, shoot a ray. Sample 64 points along the ray.
  Query the MLP 64 times. Alpha-composite the results front-to-back.
  
  COST: ~30 seconds per frame at 800×800 (UNUSABLE for real-time).
```

### 7.2 3D Gaussian Splatting — The Real-Time Alternative

Instead of querying an MLP, represent the scene as millions of 3D Gaussian blobs:

```
GAUSSIAN SPLATTING:
══════════════════
  Each Gaussian has:
    - Position (x, y, z)         3 floats
    - Covariance (3×3 symmetric) 6 floats (defines shape/orientation)
    - Opacity (α)                1 float
    - Color (SH coefficients)    48 floats (Spherical Harmonics for view-dependent color)
  
  RENDERING:
  1. Project each 3D Gaussian onto 2D screen space (becomes an ellipse).
  2. Sort all Gaussians by depth (front-to-back).
  3. Alpha-blend the 2D ellipses per-pixel using standard rasterization.
  
  COST: 100+ fps at 1080p on a mid-range GPU!
  
  WHY IT WORKS: Rasterization of 2D ellipses is trivially parallelizable.
  No neural network inference, no raymarching, just standard GPU rasterization.
```
