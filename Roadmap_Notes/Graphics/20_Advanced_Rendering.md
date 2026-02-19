# Chapter 20 — Advanced Rendering Techniques

> This chapter provides concise summaries of advanced techniques.
> For complete implementations, see the `Advanced_Rendering/` modules.

---

## 1. Shadow Mapping (Quick Reference)

```
TWO-PASS SHADOW MAPPING:
════════════════════════
  Pass 1: Render depth from LIGHT's perspective → depth texture (shadow map)
  Pass 2: Render scene from CAMERA
           For each fragment:
             Project fragment position into light space
             Compare depth vs shadow map
             If fragment.depth > shadowMap.depth → fragment is in shadow

  KEY TECHNIQUES:
  - Bias: Prevents shadow acne (self-shadowing artifact)
  - PCF (5×5): Soft shadow edges (25 depth comparisons per pixel)
  - CSM (Cascaded): Multiple shadow maps for near/mid/far distances
  
  Full implementation → Advanced_Rendering/04_Advanced_Shadows.md
```

---

## 2. Deferred Rendering (Quick Reference)

```
  Pass 1: Write surface data to G-Buffer (albedo, normal, roughness, depth)
  Pass 2: Full-screen lighting pass reads G-Buffer, computes PBR per-pixel
  
  Advantage: No overdraw lighting × zero lighting for invisible pixels
  Disadvantage: No MSAA, no transparency in deferred pass, high bandwidth
  
  Full implementation → Advanced_Rendering/05_Deferred_Rendering.md
```

---

## 3. Instancing (Drawing 100,000 Objects)

```
WITHOUT INSTANCING: 100,000 trees × 1 draw call = 100,000 draw calls
  CPU cost: 100,000 × 0.1ms = 10 SECONDS per frame. Unplayable.

WITH INSTANCING: 1 draw call with 100,000 instance transforms
  CPU cost: 1 × 0.1ms = 0.1ms. 
  GPU renders all 100,000 trees in parallel.

USE FOR: Trees, grass blades, particles, buildings, asteroids.
```

---

## 4. Post-Processing Pipeline

```
POST-PROCESSING ORDER (CORRECT):
═════════════════════════════════
  1. Render scene → HDR framebuffer (RGBA16F)
  2. Bloom: Extract bright → downsample → upsample → add
  3. Auto-exposure: Calculate average luminance → adjust exposure
  4. Tone mapping: HDR → [0,1] using ACES
  5. TAA: Blend with history buffer
  6. Gamma correction: Linear → sRGB
  7. Output to screen
  
  Full implementation:
    → Advanced_Rendering/07_TAA.md
    → Advanced_Rendering/08_HDR_Rendering.md
```

---

## 5. HDR Pipeline (Quick Reference)

```
LDR (WRONG):  Render to 8-bit buffer → values > 1.0 are CLAMPED → lost forever
HDR (CORRECT): Render to 16-bit buffer → post-process → tone map → 8-bit output

  Full implementation → Advanced_Rendering/08_HDR_Rendering.md
```

---

## 6. PBR Pipeline (Quick Reference)

```
COOK-TORRANCE:
  f(l,v) = kD × (albedo/π) + kS × (D × G × F) / (4 × NdotV × NdotL)
  
  D = GGX microfacet distribution (roughness controls highlight shape)
  G = Smith geometry (self-shadowing of micro-surfaces)
  F = Fresnel-Schlick (more reflection at grazing angles)
  
  Metallic workflow: metallic=0 → dielectric (plastic), metallic=1 → metal
  F0 = mix(vec3(0.04), albedo, metallic)
  
  Full implementation → Advanced_Rendering/01_PBR_Theory.md
```
