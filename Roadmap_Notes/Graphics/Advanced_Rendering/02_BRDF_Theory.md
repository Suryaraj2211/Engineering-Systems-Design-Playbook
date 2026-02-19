# Module 2 — BRDF Theory & The Rendering Equation

> This module derives the mathematical foundation underlying ALL rendering.
> Every concept from PBR, GI, SSR, and path tracing is a special case of what follows.

---

## 1. Radiometric Quantities — The Units of Light

Before deriving anything, we must define light's physical units precisely.

### 1.1 Radiant Flux (Φ) — Total Power
Total energy emitted by a light source per second.
**Units:** Watts (W).
**Example:** A 60W light bulb emits Φ = 60W of radiant energy.

### 1.2 Irradiance (E) — Power Arriving at a Surface
Radiant flux per unit area arriving at a surface.
$$E = \frac{d\Phi}{dA}$$
**Units:** W/m².
**Example:** Sunlight at noon: E ≈ 1000 W/m².

### 1.3 Radiance (L) — Power per Area per Direction
The most important quantity in rendering. It measures the light traveling along a specific ray.
$$L = \frac{d^2\Phi}{dA \cdot d\omega \cdot \cos\theta}$$
**Units:** W/(m²·sr).
This is what your fragment shader computes as `fragColor`.

### 1.4 Solid Angle (ω) — The "Size" of a Direction

```
SOLID ANGLE VISUALIZATION:
══════════════════════════
  Imagine standing at a surface point P.
  Look upward. The entire hemisphere of directions above you
  has a total solid angle of 2π steradians (sr).
  
  A tiny patch of sky 1° wide subtends ~0.0003 sr.
  The full moon subtends ~0.00007 sr.
  
  In a shader, we approximate the hemisphere integral
  by summing over N discrete sample directions,
  each covering ΔΩ = 2π/N steradians.
```

---

## 2. The BRDF — Definition and Properties

### 2.1 Formal Definition

The **Bidirectional Reflectance Distribution Function** answers:
*"Given light arriving from direction $\omega_i$, what fraction bounces toward direction $\omega_o$?"*

$$f_r(p, \omega_i, \omega_o) = \frac{dL_o(p, \omega_o)}{dE(p, \omega_i)} = \frac{dL_o(p, \omega_o)}{L_i(p, \omega_i) \cos\theta_i \, d\omega_i}$$

**Units:** 1/sr (inverse steradians).

### 2.2 The Three Laws Every BRDF Must Obey

**Law 1: Positivity** — $f_r \geq 0$ always. (A surface cannot absorb negative light.)

**Law 2: Helmholtz Reciprocity** — $f_r(\omega_i, \omega_o) = f_r(\omega_o, \omega_i)$
Swapping light and view gives the same result. (Light doesn't care which direction it travels.)

**Law 3: Energy Conservation** — $\int_{\Omega} f_r(\omega_i, \omega_o) \cos\theta_o \, d\omega_o \leq 1$
The total reflected light cannot exceed the incoming light.

---

## 3. The Reflectance Equation

For a single light source at direction $\omega_i$:
$$L_o(p, \omega_o) = f_r(p, \omega_i, \omega_o) \cdot L_i(p, \omega_i) \cdot \cos\theta_i$$

```glsl
// Direct translation to shader code:
vec3 Lo = BRDF(wi, wo) * Li * max(dot(N, L), 0.0);
//         f_r              L_i         cos(theta_i)
```

For N point lights:
$$L_o = \sum_{k=1}^{N} f_r(p, \omega_{i_k}, \omega_o) \cdot L_{i_k} \cdot \cos\theta_{i_k}$$

```glsl
vec3 Lo = vec3(0.0);
for (int k = 0; k < numLights; k++) {
    Lo += BRDF(L[k], V) * lightRadiance[k] * max(dot(N, L[k]), 0.0);
}
```

---

## 4. The Full Rendering Equation

For ALL light from ALL directions (including indirect bounces):

$$L_o(p, \omega_o) = L_e(p, \omega_o) + \int_{\Omega} f_r(p, \omega_i, \omega_o) \cdot L_i(p, \omega_i) \cdot \cos\theta_i \, d\omega_i$$

**The integral is over the entire hemisphere.** This means we must sum the BRDF contribution from every possible incoming direction — including indirect light bouncing off other surfaces.

```
RENDERING EQUATION BREAKDOWN:
═════════════════════════════
  L_o(p, ω_o)                    What we want: color of pixel
  
  = L_e(p, ω_o)                  Emission (self-glow, e.g., fire)
  
  + ∫_Ω                          Sum over ALL incoming directions:
      f_r(p, ω_i, ω_o)             BRDF: how much bounces our way
    × L_i(p, ω_i)                  Incoming light from that direction
    × cos(θ_i)                     Lambert's cosine attenuation
    dω_i                           Infinitesimal solid angle
    
  In practice:
    Direct lighting:    L_i is known (light position/color)
    Indirect lighting:  L_i is ITSELF L_o of another surface
                        → INFINITE RECURSION (solved by GI in Module 3)
```

---

## 5. Lambertian Diffuse — The Simplest BRDF

### 5.1 Derivation

A perfectly matte surface scatters light equally in all directions. The BRDF is constant:

$$f_{Lambert} = \frac{\rho}{\pi}$$

where $\rho$ = albedo (surface color).

**Why divide by π?**
The integral of $\cos\theta$ over the hemisphere equals $\pi$:
$$\int_\Omega \cos\theta \, d\omega = \pi$$

If $f_r = \rho$ (without dividing by π), then:
$$\int_\Omega \rho \cdot \cos\theta \, d\omega = \rho \cdot \pi$$

For ρ = 0.5, we'd reflect 1.57× the incoming light. That violates energy conservation!
Dividing by π normalizes: $\rho/\pi \cdot \pi = \rho \leq 1.0$ ✓

```glsl
// CORRECT Lambertian diffuse
vec3 diffuse = albedo / PI;

// WRONG (too bright by ~3.14×)
vec3 diffuse = albedo; // Missing normalization!
```

---

## 6. Specular Models — From Phong to Cook-Torrance

### 6.1 The Evolution of Specular BRDFs

```
SPECULAR MODEL EVOLUTION:
═════════════════════════
  1975: Phong        f_r = (R·V)^n        ← Not energy conserving
  1977: Blinn-Phong  f_r = (N·H)^n        ← Better, still not physical
  1982: Cook-Torrance f_r = DGF/4(N·V)(N·L)  ← Physically based!
  2007: GGX/Trowbridge-Reitz replaces Beckmann for D term
        → Current industry standard (Disney, UE4, Unity)
```

### 6.2 GGX Normal Distribution Function (D) — Detailed Derivation

We need: *"What fraction of microfacets have their normal aligned with $\mathbf{H}$?"*

The GGX distribution is defined as:

$$D_{GGX}(\mathbf{n}, \mathbf{h}, \alpha) = \frac{\alpha^2}{\pi \cdot \left((\mathbf{n} \cdot \mathbf{h})^2 \cdot (\alpha^2 - 1) + 1\right)^2}$$

**Step-by-step numerical example:**
```
INPUT: roughness = 0.5, NdotH = 0.95 (light nearly aligned with specular peak)

  Step 1: α = roughness² = 0.25
  Step 2: α² = 0.0625
  Step 3: NdotH² = 0.9025
  Step 4: inner = NdotH² × (α² - 1) + 1
                = 0.9025 × (0.0625 - 1.0) + 1
                = 0.9025 × (-0.9375) + 1
                = -0.8461 + 1
                = 0.1539
  Step 5: denom = π × (0.1539)²
                = π × 0.023685
                = 0.07441
  Step 6: D = 0.0625 / 0.07441 = 0.8398

  INTERPRETATION: At this angle, ~84% of microfacet normals are aligned.
  This is a moderately concentrated specular highlight.
```

### 6.3 Geometry Function (G) — Detailed Derivation

**Smith's method:** Calculate self-shadowing from the light's perspective AND self-masking from the viewer's perspective independently, then multiply.

$$G_1(\mathbf{n}, \mathbf{v}, k) = \frac{\mathbf{n} \cdot \mathbf{v}}{(\mathbf{n} \cdot \mathbf{v})(1 - k) + k}$$

where $k_{direct} = \frac{(\alpha + 1)^2}{8}$

**Step-by-step numerical example:**
```
INPUT: roughness = 0.5, NdotV = 0.3 (looking at surface at ~73° from normal)

  Step 1: α = roughness = 0.5 (NOT squared for k computation)
  Step 2: k = (0.5 + 1.0)² / 8 = (1.5)² / 8 = 2.25 / 8 = 0.28125
  Step 3: G1 = 0.3 / (0.3 × (1 - 0.28125) + 0.28125)
             = 0.3 / (0.3 × 0.71875 + 0.28125)
             = 0.3 / (0.21563 + 0.28125)
             = 0.3 / 0.49688
             = 0.604

  INTERPRETATION: At this grazing angle with moderate roughness,
  ~40% of microfacets are blocked (either shadowed by or masking each other).
```

### 6.4 Full Cook-Torrance: Putting It All Together

$$f_{Cook-Torrance} = \frac{D \cdot G \cdot F}{4 \cdot (\mathbf{n} \cdot \mathbf{l}) \cdot (\mathbf{n} \cdot \mathbf{v})}$$

**Complete numerical example for one pixel:**
```
INPUTS:
  albedo = (0.8, 0.2, 0.2) (red plastic)
  metallic = 0.0
  roughness = 0.5
  NdotL = 0.7, NdotV = 0.85, NdotH = 0.95, VdotH = 0.9

COMPUTE:
  F0 = mix(0.04, albedo, 0.0) = (0.04, 0.04, 0.04)
  
  D = GGX(N, H, 0.5) = 0.8398  (from example above)
  
  G = Smith(N, V, L, 0.5)
    G1_view  = GSchlick(0.85, 0.5) = 0.85 / (0.85 × 0.71875 + 0.28125)
             = 0.85 / 0.8922 = 0.9527
    G1_light = GSchlick(0.70, 0.5) = 0.70 / (0.70 × 0.71875 + 0.28125)
             = 0.70 / 0.7844 = 0.8925
    G = 0.9527 × 0.8925 = 0.8503
  
  F = Schlick(0.9, 0.04) = 0.04 + 0.96 × (1 - 0.9)^5
    = 0.04 + 0.96 × 0.00001 = 0.04001 (nearly F0 at this angle)
  
  Specular = (0.8398 × 0.8503 × 0.04001) / (4 × 0.7 × 0.85)
           = 0.02857 / 2.38
           = 0.01201
  
  kD = (1.0 - 0.04001) × (1.0 - 0.0) = 0.95999
  Diffuse = 0.95999 × (0.8, 0.2, 0.2) / π = (0.2444, 0.0611, 0.0611)
  
  FINAL = (Diffuse + Specular) × LightColor × NdotL
        = ((0.2444+0.01201), (0.0611+0.01201), (0.0611+0.01201)) × (1,1,1) × 0.7
        = (0.2564, 0.0731, 0.0731) × 0.7
        = (0.1795, 0.0512, 0.0512)  ← Dark red, mostly diffuse (low specular at this angle)
```

---

## 7. GPU Cost of BRDF Evaluation

| BRDF Component | ALU Operations | Texture Reads | Register Usage |
|----------------|---------------|---------------|----------------|
| `normalize(L + V)` → H | 4 ALU | 0 | 1 vec3 |
| `dot(N, H)`, `dot(N, V)`, etc. | 5 ALU | 0 | 5 floats |
| `DistributionGGX()` | 6 ALU + 1 `pow` | 0 | 3 floats |
| `GeometrySmith()` | 8 ALU | 0 | 3 floats |
| `fresnelSchlick()` | 4 ALU + 1 `pow` | 0 | 1 vec3 |
| Combine (D×G×F / denom) | 6 ALU | 0 | 1 vec3 |
| **TOTAL (per light)** | **~33 ALU + 2 `pow`** | **0** | **~12 registers** |
| **10 lights** | **~330 ALU** | **0** | **Same 12 regs** |

**Bottleneck Analysis:**
- At 1080p with 10 lights: 2M pixels × 330 ALU ≈ 660M operations.
- A mid-range GPU executes ~10 TFLOPS = 10,000M operations per second.
- Pure ALU time: 660M / 10,000M = 0.066ms (very fast!).
- **True bottleneck is texture bandwidth** (sampling albedo, normal, metallic, roughness) **not BRDF math.**

---

## 8. Common BRDF Mistakes & Debugging

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Missing `/PI` in diffuse | Everything 3.14× too bright | `diffuse = kD * albedo / PI` |
| Not squaring roughness for α | Materials too rough/glossy | `float a = roughness * roughness` in GGX |
| Using `NdotV` instead of `VdotH` in Fresnel | Wrong Fresnel angle | Fresnel uses `dot(V, H)` for microfacet model |
| `k` computation differs for IBL vs. direct | IBL specular too bright | Direct: k = (r+1)²/8. IBL: k = r²/2 |
| Denominator = 0 at edges | NaN → black/white fireflies | Use `max(4*NdotV*NdotL, 0.001)` |
