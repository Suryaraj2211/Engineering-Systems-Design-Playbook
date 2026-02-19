# Module 3 — Global Illumination Basics

> This module covers indirect lighting — the light that bounces off other surfaces
> before reaching the camera. Without GI, shaded areas are pitch black.

---

## 1. Direct vs. Indirect Lighting

### 1.1 The Problem with Direct-Only Rendering

```
DIRECT LIGHTING ONLY:
═════════════════════
  Sun → hits the floor → Fragment shader computes PBR → done.
  
  What about the ceiling?
  The sun NEVER directly hits the ceiling in an indoor scene.
  With direct-only lighting, the ceiling is pure black.
  
  In reality: Sun → hits floor → floor reflects warm light upward →
  ceiling receives soft, warm-tinted illumination.
  
  This "bounce" is INDIRECT lighting (Global Illumination).
```

### 1.2 Bounce Classification

| Bounce Level | Description | Visual Impact | Cost |
|-------------|-------------|--------------|------|
| Direct (0 bounces) | Light → Surface → Camera | Harsh shadows, black interiors | Fast (standard shader) |
| 1 bounce | Light → Surface A → Surface B → Camera | Color bleeding, soft fill | Moderate |
| 2 bounces | Light → A → B → C → Camera | Ambient warmth, caustics | Expensive |
| ∞ bounces | Full light transport convergence | Photorealistic | Offline only |

Real-time engines typically simulate **1 bounce** of indirect diffuse light and use screen-space tricks for the rest.

---

## 2. Light Transport — How Light Moves Through a Scene

### 2.1 The Path Notation

Researchers describe light paths using Heckbert notation:
- **L** = Light source
- **D** = Diffuse bounce
- **S** = Specular bounce (mirror reflection)
- **E** = Eye (camera)

```
COMMON LIGHT PATHS:
═══════════════════
  LDE      → Direct diffuse lighting (standard shader)
  LSE      → Direct specular (mirror reflection in PBR)
  LDDE     → 1-bounce diffuse GI (color bleeding)
  LDSE     → Diffuse light hitting a mirror → camera (caustics!)
  LSSE     → Light bouncing between two mirrors
  LD*E     → All diffuse bounces (what GI systems approximate)
```

### 2.2 Why Rasterization Cannot Solve GI

Rasterization processes triangles → pixels in a fixed pipeline. The Fragment Shader knows:
- Its own position, normal, albedo.
- The positions of light sources.

It does NOT know:
- What other surfaces exist in the scene (they've already been rasterized or not yet drawn).
- Whether another triangle 5 meters away is bouncing red light toward this pixel.

**To calculate indirect light, we need scene-wide information. Rasterization is fundamentally local.**

---

## 3. Monte Carlo Sampling — The Mathematical Tool

### 3.1 The Hemisphere Integral

To calculate indirect light at a surface point, we must integrate incoming radiance from all directions:

$$L_{indirect}(p) = \int_{\Omega} f_r(p, \omega_i, \omega_o) \cdot L_{bounce}(\omega_i) \cdot \cos\theta_i \, d\omega_i$$

This integral covers an infinite number of directions. We approximate it with N random samples:

$$L_{indirect}(p) \approx \frac{1}{N} \sum_{k=1}^{N} \frac{f_r \cdot L_{bounce}(\omega_k) \cdot \cos\theta_k}{p(\omega_k)}$$

### 3.2 Practical Example: Screen-Space GI Sampling

```glsl
// Approximate 1-bounce GI using screen-space hemisphere sampling
vec3 computeSSGI(vec3 fragPos, vec3 N) {
    vec3 indirectLight = vec3(0.0);
    int samples = 16;
    
    for (int i = 0; i < samples; i++) {
        // Generate random direction on hemisphere aligned with N
        vec3 sampleDir = cosineWeightedHemisphere(N, randomVec2(i));
        
        // March along this direction in screen space
        vec2 hitUV = screenSpaceRaymarch(fragPos, sampleDir);
        
        if (hitUV.x >= 0.0) {
            // We hit something! Sample its color from the previous frame
            vec3 hitColor = texture(previousFrameColor, hitUV).rgb;
            indirectLight += hitColor; // Cosine weighting built into sampling
        }
    }
    
    return indirectLight / float(samples);
}
```

### 3.3 The Noise Problem

With N = 1 sample, the pixel gets one random bounce direction. It might hit a bright red wall (pixel = red) or miss everything (pixel = black). Adjacent pixels get different random directions → speckled noise.

```
CONVERGENCE VS SAMPLE COUNT:
════════════════════════════
  N = 1      → Extreme noise (unusable without denoiser)
  N = 4      → Very noisy (visible grain)
  N = 16     → Moderate noise (acceptable with TAA)
  N = 64     → Mild noise
  N = 1024   → Nearly converged (offline quality)
  N = 16384  → Reference quality (Pixar movies)
  
  Real-time budget: N = 1-4 (then denoise temporally)
```

---

## 4. Ray Tracing vs. Rasterization — Architectural Comparison

| Feature | Rasterization | Ray Tracing |
|---------|--------------|-------------|
| Primary visibility | O(triangles) — very fast | O(log N) BVH lookup — slower |
| Hard shadows | Shadow map (approximation) | Shadow ray (exact) |
| Reflections | SSR (screen-space only) | Reflection ray (all geometry) |
| GI (indirect light) | Impossible natively | Natural (recursive) |
| Transparency | Requires sorting | Free (ray keeps going) |
| GPU hardware | Fixed-function rasterizer (free) | RT Cores (compute) |
| Cost (ms/frame) | 2-8ms (geometry bound) | 5-20ms (BVH traversal bound) |

**Modern Architecture:** Rasterize primary view, ray trace secondary effects.

---

## 5. Ambient Occlusion — The Cheapest GI Approximation

### 5.1 What AO Simulates

AO darkens areas where geometry occludes incoming ambient light (corners, crevices, under objects). It does NOT add colored light; it only multiplicatively darkens.

$$AO(p) = \frac{1}{\pi} \int_{\Omega} V(p, \omega) \cdot \cos\theta \, d\omega$$

where $V(p, \omega) = 0$ if a ray is blocked, $1$ if unblocked.

### 5.2 SSAO (Screen-Space Ambient Occlusion) Algorithm

```glsl
float computeSSAO(vec2 texCoords) {
    vec3 fragPos = texture(gPosition, texCoords).xyz;
    vec3 normal  = texture(gNormal, texCoords).xyz;
    
    // Random rotation to avoid banding patterns
    vec3 randomVec = texture(noiseTexture, texCoords * noiseScale).xyz;
    vec3 tangent   = normalize(randomVec - normal * dot(randomVec, normal));
    vec3 bitangent = cross(normal, tangent);
    mat3 TBN       = mat3(tangent, bitangent, normal);
    
    float occlusion = 0.0;
    int kernelSize = 32; // 32 hemisphere samples
    float radius = 0.5;  // 0.5 meters search radius
    
    for (int i = 0; i < kernelSize; i++) {
        // Sample positions distributed in a hemisphere
        vec3 samplePos = fragPos + TBN * ssaoKernel[i] * radius;
        
        // Project sample to screen space
        vec4 offset = uProjection * uView * vec4(samplePos, 1.0);
        offset.xyz /= offset.w;
        offset.xyz = offset.xyz * 0.5 + 0.5;
        
        // Read the actual depth at this screen position
        float sampleDepth = texture(gPosition, offset.xy).z;
        
        // If geometry exists CLOSER than our sample point, it's occluded
        float rangeCheck = smoothstep(0.0, 1.0, radius / abs(fragPos.z - sampleDepth));
        occlusion += (sampleDepth >= samplePos.z + 0.025 ? 1.0 : 0.0) * rangeCheck;
    }
    
    return 1.0 - (occlusion / float(kernelSize));
}
```

### 5.3 GPU Cost Analysis

| SSAO Parameter | Value | GPU Impact |
|---------------|-------|------------|
| Kernel size | 32 samples | 32 texture reads per pixel (bandwidth heavy!) |
| Resolution | Full-res (1080p) | 2M pixels × 32 reads = 64M texture reads |
| Noise texture | 4×4 | Minimal cost |
| Blur pass | 4×4 bilateral | Additional 16 reads per pixel |
| **Total bandwidth** | **~80M reads** | **~1.5ms (RTX 3070)** |

### 5.4 SSAO Artifacts & Fixes

| Artifact | Cause | Fix |
|----------|-------|-----|
| Bright halos around objects | Range check not applied | `smoothstep` falloff beyond search radius |
| Banding patterns | Deterministic sampling | Rotate kernel per pixel (noise texture) |
| AO on the sky / distant objects | No range limit | Check depth difference; skip if too large |
| Self-occlusion (flat surfaces show shadow) | Bias too small | Add `0.025` depth bias in comparison |

---

## 6. Screen-Space Global Illumination (SSGI)

### 6.1 Algorithm Overview

SSGI extends SSAO to also gather COLOR from hit surfaces, not just occlusion.

```
SSAO:  For each direction → did I hit geometry?   → Yes/No (scalar)
SSGI:  For each direction → did I hit geometry?   → If yes, what COLOR is it?

SSGI Pipeline:
  1. For each pixel, generate 8 random hemisphere directions.
  2. Screen-space raymarch each direction through the Hi-Z depth buffer.
  3. If a hit is found, sample the previous frame's LIT color at the hit UV.
  4. Weight by the BRDF and Lambert cosine.
  5. Average all contributions → indirect diffuse color for this pixel.
  6. Apply temporal accumulation to reduce noise across frames.
```

### 6.2 Limitations

```
SCREEN-SPACE PROBLEM:
═════════════════════
  Camera sees:    [Wall A]  [Gap]  [Wall B]
  
  SSGI for pixel on the floor between walls:
    Shoots ray LEFT  → Hits Wall A (sees its red color) → adds red indirect light ✓
    Shoots ray RIGHT → Hits Wall B (sees its blue color) → adds blue indirect light ✓
    Shoots ray UP    → Nothing in screen space → returns black (0,0,0) ✗
    
  If the ceiling is just above the camera frustum (not visible on screen),
  SSGI returns BLACK for upward rays, even though the ceiling should
  reflect warm light onto the floor.
  
  FUNDAMENTAL LIMITATION: Screen-space techniques can only see pixels
  that are currently visible on the screen.
```

---

## 7. Real-Time GI Approximation Strategies (Overview)

| Strategy | Quality | Cost | Limitation |
|----------|---------|------|------------|
| Constant Ambient (`vec3(0.03)`) | Terrible | 0ms | Not GI at all |
| SSAO (darkness only) | Low | ~1.5ms | No color bleeding, screen-space only |
| SSGI (screen-space) | Medium | ~3ms | Screen-space only, temporal artifacts |
| Irradiance Probes (baked) | Good (static) | ~0.5ms | Static scenes only |
| DDGI (ray-traced probes) | High | ~4ms | Requires RT hardware |
| Lumen (UE5 hybrid) | Excellent | ~6ms | Complex, multi-technique hybrid |
| Full Path Tracing | Reference | ~30ms+ | Requires denoiser, RT hardware |

---

## 8. Debugging GI Issues

| Symptom | Root Cause | Debugging Approach |
|---------|-----------|-------------------|
| Indoor rooms pitch black | No indirect lighting at all | Add environment probe or constant ambient as baseline |
| Color bleeding too strong | GI contribution not attenuated | Multiply indirect by 0.5-1.0 artistic factor |
| GI pops/flickers on camera movement | Screen-space data changes | Increase temporal blend factor (more history) |
| Light leaking through thin walls | SSGI rays pass through 1-pixel-thick geometry | Increase ray march step count or use world-space probes |
| Grainy noise in dark areas | Too few GI samples | Increase sample count or add bilateral blur pass |
