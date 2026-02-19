# Module 1 — Physically Based Rendering (PBR) Theory

> This module explains WHY traditional lighting is wrong, the physics behind PBR,
> and provides complete shader implementations with GPU cost analysis.

---

## 1. Why Traditional Phong/Blinn-Phong Fails

### 1.1 The Three Fatal Flaws

**Flaw 1: Energy Creation**
Phong calculates diffuse and specular independently and sums them:
```glsl
// PHONG (WRONG — creates energy out of nothing)
vec3 diffuse  = kD * max(dot(N, L), 0.0) * lightColor;
vec3 specular = kS * pow(max(dot(R, V), 0.0), shininess) * lightColor;
vec3 result   = diffuse + specular; // Can exceed input light energy!
```
If `kD = 0.8` and `kS = 0.5`, the surface reflects `1.3×` the incoming light energy. It literally creates photons from nothing. Under a bright HDR environment, Phong materials glow radioactively.

**Flaw 2: Shininess Is Not a Physical Property**
The `shininess` exponent is an arbitrary artistic number. `shininess = 32` means nothing in physics. You cannot derive it from surface measurements, and changing the lighting environment forces artists to re-tune every material.

**Flaw 3: No View-Angle Dependency (Missing Fresnel)**
Look at a wooden desk straight down — it appears matte. Look at it at a shallow grazing angle — it becomes reflective like a mirror. Phong completely ignores this fundamental optical behavior.

### 1.2 What PBR Guarantees
PBR materials look correct under ANY lighting condition because they obey measurable physics:
1. **Energy Conservation:** Light out ≤ Light in. Always.
2. **Microfacet Model:** Surface roughness is physically defined, not artistically tuned.
3. **Fresnel Reflection:** All materials become mirrors at grazing angles. Automatically.
4. **Reciprocity:** The BRDF gives the same result if you swap the light and view directions.

---

## 2. Energy Conservation — The Mathematical Foundation

### 2.1 The Core Constraint

When a photon hits a surface, it can only do two things:
1. **Reflect** off the surface (specular). This fraction is called $k_S$.
2. **Refract** into the surface (diffuse). This fraction is called $k_D$.

The constraint: $k_S + k_D \leq 1.0$

```glsl
// PBR Energy Conservation (CORRECT)
vec3 kS = fresnelSchlick(max(dot(H, V), 0.0), F0);
vec3 kD = vec3(1.0) - kS;  // Whatever isn't reflected is refracted

// Metals absorb ALL refracted light internally (no diffuse!)
kD *= (1.0 - metallic);
```

### 2.2 Step-by-Step Derivation

```
ENERGY BUDGET PER PHOTON:
═════════════════════════
  Incoming light energy:  1.0 (normalized)
  
  Case 1: Plastic (F0 = 0.04, metallic = 0.0)
    Fresnel at normal incidence: kS = 0.04 (4% reflected)
    kD = 1.0 - 0.04 = 0.96 (96% enters material → diffuse)
    Fresnel at grazing angle:    kS ≈ 1.0 (100% reflected!)
    kD = 1.0 - 1.0 = 0.0 (0% enters → pure mirror at edges)
  
  Case 2: Gold (F0 = (1.0, 0.71, 0.29), metallic = 1.0)
    Fresnel at normal: kS = (1.0, 0.71, 0.29) (Gold-tinted reflection!)
    kD = (1.0 - kS) * (1.0 - 1.0) = (0.0, 0.0, 0.0) (ZERO diffuse)
    All light is either reflected as gold or absorbed. No matte color.
```

---

## 3. Microfacet Theory — Surface Roughness Explained

### 3.1 The Physical Model

At microscopic scale, every surface is made of millions of tiny perfect mirrors (microfacets). Each microfacet has its own tiny normal vector $\mathbf{m}$.

```
SMOOTH SURFACE (Roughness = 0.05):
══════════════════════════════════
  All microfacets point straight up (aligned with macro normal N)
  
        N (macro normal)
        ↑
  ══════╤══════╤══════╤══════╤══════  ← Nearly flat micro-surface
        ↑      ↑      ↑      ↑
        m₁     m₂     m₃     m₄        All m ≈ N
  
  RESULT: Light reflects in a tight cone → sharp mirror highlight

ROUGH SURFACE (Roughness = 0.9):
════════════════════════════════
  Microfacets point in random directions
  
        N (macro normal)
        ↑
  ═╲═╱═══╲══╱═╲═══╱═╲══╱═  ← Jagged micro-surface
   ↗  ↖    ↗  ↖  ↗    ↖  ↗
   m₁  m₂  m₃  m₄ m₅  m₆         m vectors scattered randomly
  
  RESULT: Light scatters in all directions → large, dim highlight
  
  CRITICAL: Total reflected energy is IDENTICAL in both cases.
  Roughness redistributes the energy, it doesn't create or destroy it.
```

### 3.2 The Halfway Vector (H)

Only microfacets whose normal $\mathbf{m}$ perfectly equals the **Halfway Vector** $\mathbf{H}$ can reflect light from $\mathbf{L}$ toward $\mathbf{V}$.

$$\mathbf{H} = \frac{\mathbf{L} + \mathbf{V}}{||\mathbf{L} + \mathbf{V}||}$$

```glsl
vec3 H = normalize(L + V);
// Only microfacets aligned with H contribute to specular
// The NDF (D term) tells us what FRACTION of microfacets are aligned
```

---

## 4. The Metallic Workflow — Deep Physics

### 4.1 Dielectrics vs. Conductors

| Property | Dielectric (Non-Metal) | Conductor (Metal) |
|----------|----------------------|-------------------|
| Examples | Plastic, wood, skin, water | Gold, iron, copper, aluminum |
| F0 (Base Reflectivity) | 0.02 – 0.05 (colorless) | 0.5 – 1.0 (colored, tinted) |
| Diffuse Component | YES (colored by albedo) | NO (absorbed internally) |
| Specular Color | White / neutral | Tinted (gold = yellow specular) |
| Subsurface Scattering | Possible (skin, wax) | Never |

### 4.2 F0 Values Table (Measured Physical Data)

```
REAL-WORLD F0 VALUES (from measured IOR):
═════════════════════════════════════════
  Material        F0 (Linear RGB)              IOR
  ────────        ──────────────               ───
  Water           (0.02, 0.02, 0.02)           1.33
  Plastic         (0.04, 0.04, 0.04)           1.50
  Glass           (0.04, 0.04, 0.04)           1.52
  Diamond         (0.17, 0.17, 0.17)           2.42
  Iron            (0.56, 0.57, 0.58)           2.95
  Gold            (1.00, 0.71, 0.29)           0.47 + 2.42i
  Copper          (0.95, 0.64, 0.54)           0.27 + 3.41i
  Silver          (0.97, 0.96, 0.91)           0.18 + 3.64i
  Aluminum        (0.91, 0.92, 0.92)           1.35 + 7.47i
```

### 4.3 Shader: Computing F0 from the Metallic Workflow

```glsl
// F0 computation — the bridge between art and physics
vec3 computeF0(vec3 albedo, float metallic) {
    // Non-metals: F0 is always ~0.04 (physics constant)
    // Metals: F0 IS the albedo color (the specular IS the color)
    vec3 F0 = vec3(0.04);
    F0 = mix(F0, albedo, metallic);
    return F0;
    
    // When metallic = 0.0: F0 = (0.04, 0.04, 0.04) → colorless specular
    // When metallic = 1.0: F0 = albedo → gold reflects gold, copper reflects copper
}
```

---

## 5. The Fresnel Equation — Schlick's Approximation

### 5.1 The Physical Phenomenon

All surfaces become perfect mirrors at grazing angles. This is not artistic — it is a consequence of electromagnetic wave theory (Maxwell's equations applied to the boundary between two media with different refractive indices).

### 5.2 Full Derivation of Schlick's Approximation

The exact Fresnel equations involve complex numbers and separate calculations for S-polarization and P-polarization. Schlick observed that the Fresnel curve can be approximated by a simple 5th-power polynomial:

$$F(\theta) = F_0 + (1 - F_0)(1 - \cos\theta)^5$$

**Derivation of why the exponent is 5:**

```
FRESNEL CURVE ANALYSIS:
═══════════════════════
  θ = angle between View and surface normal
  cosθ = dot(N, V) (or dot(H, V) for microfacets)
  
  At θ = 0° (looking straight down): cosθ = 1.0
    F(0°) = F0 + (1-F0)(1-1)^5 = F0 + 0 = F0  ✓
  
  At θ = 90° (grazing angle): cosθ = 0.0
    F(90°) = F0 + (1-F0)(1-0)^5 = F0 + (1-F0) = 1.0  ✓
    100% reflection at grazing angle!
  
  At θ = 60° (moderate angle): cosθ = 0.5
    F(60°) = 0.04 + 0.96 * (0.5)^5 = 0.04 + 0.96 * 0.03125 = 0.07
    Only 7% reflective at 60° for plastic. Still mostly matte.
  
  At θ = 85° (near-grazing): cosθ ≈ 0.087
    F(85°) = 0.04 + 0.96 * (0.913)^5 = 0.04 + 0.96 * 0.634 = 0.65
    65% reflective! The surface is now mostly a mirror!
  
  The power of 5 was chosen by Schlick to minimize error vs. exact Fresnel
  across all dielectric IOR values (error < 1%).
```

### 5.3 Shader Implementation

```glsl
vec3 fresnelSchlick(float cosTheta, vec3 F0) {
    return F0 + (1.0 - F0) * pow(clamp(1.0 - cosTheta, 0.0, 1.0), 5.0);
}

// Roughness-corrected version for Image-Based Lighting (IBL)
// Prevents overly bright Fresnel on rough surfaces
vec3 fresnelSchlickRoughness(float cosTheta, vec3 F0, float roughness) {
    return F0 + (max(vec3(1.0 - roughness), F0) - F0) *
           pow(clamp(1.0 - cosTheta, 0.0, 1.0), 5.0);
}
```

---

## 6. The Cook-Torrance BRDF — Complete Breakdown

### 6.1 The Full Equation

$$f_r = k_D \frac{albedo}{\pi} + k_S \frac{D \cdot G \cdot F}{4 \cdot (\mathbf{n} \cdot \mathbf{l}) \cdot (\mathbf{n} \cdot \mathbf{v})}$$

| Term | Name | What It Computes |
|------|------|-----------------|
| $k_D \frac{albedo}{\pi}$ | Lambertian Diffuse | Matte, scattered light. Divided by $\pi$ for energy conservation. |
| $D$ | Normal Distribution (NDF) | Fraction of microfacets aligned with $\mathbf{H}$ |
| $G$ | Geometry Function | Fraction of microfacets not blocked by neighbors |
| $F$ | Fresnel | Fraction of light reflected (vs refracted) at this angle |
| $4 (n \cdot l)(n \cdot v)$ | Normalization | Prevents the math from producing values > 1.0 |

### 6.2 D Term: GGX/Trowbridge-Reitz Distribution

This function returns how many microfacets have their normal aligned with $\mathbf{H}$.

$$D_{GGX}(\mathbf{n}, \mathbf{h}, \alpha) = \frac{\alpha^2}{\pi \cdot ((n \cdot h)^2 \cdot (\alpha^2 - 1) + 1)^2}$$

where $\alpha = roughness^2$ (the squaring makes artist-facing roughness more perceptually linear).

```glsl
float DistributionGGX(vec3 N, vec3 H, float roughness) {
    float a  = roughness * roughness;        // Alpha (squared roughness)
    float a2 = a * a;                        // Alpha squared
    float NdotH  = max(dot(N, H), 0.0);
    float NdotH2 = NdotH * NdotH;
    
    float denom = (NdotH2 * (a2 - 1.0) + 1.0);
    denom = PI * denom * denom;
    
    return a2 / max(denom, 0.0000001); // Prevent division by zero
}
```

**Behavior analysis:**
```
GGX DISTRIBUTION OUTPUT:
════════════════════════
  roughness = 0.05, NdotH = 1.0 (perfect alignment):
    a = 0.0025, a2 = 0.00000625
    denom = π × (1.0 × (0.00000625 - 1) + 1)^2 ≈ π × 1.0
    D ≈ 0.00000625 / π ≈ 0.000002 (TINY for most directions)
    BUT at NdotH = 1.0 exactly: denominator → π × a2² ≈ very small
    D → HUGE spike (concentrated reflection)
    
  roughness = 0.9, NdotH = 1.0:
    a = 0.81, a2 = 0.6561
    D = 0.6561 / (π × 1.0) ≈ 0.21 (spread across many directions)
```

### 6.3 G Term: Smith's Geometry Function (Schlick-GGX)

Microfacets can block each other. At grazing angles, tiny peaks shadow the valleys behind them. The G term calculates what fraction of microfacets are both visible to the light AND visible to the camera.

$$G_{SchlickGGX}(n, v, k) = \frac{n \cdot v}{(n \cdot v)(1 - k) + k}$$

where $k = \frac{(\alpha + 1)^2}{8}$ for direct lighting, or $k = \frac{\alpha^2}{2}$ for IBL.

For the full answer, we need $G_{Smith}$ which combines shadowing AND masking:
$$G_{Smith}(\mathbf{n}, \mathbf{v}, \mathbf{l}, k) = G_{SchlickGGX}(\mathbf{n}, \mathbf{v}, k) \cdot G_{SchlickGGX}(\mathbf{n}, \mathbf{l}, k)$$

```glsl
float GeometrySchlickGGX(float NdotV, float roughness) {
    float r = roughness + 1.0;
    float k = (r * r) / 8.0; // k for direct lighting
    return NdotV / (NdotV * (1.0 - k) + k);
}

float GeometrySmith(vec3 N, vec3 V, vec3 L, float roughness) {
    float NdotV = max(dot(N, V), 0.0);
    float NdotL = max(dot(N, L), 0.0);
    float ggx2 = GeometrySchlickGGX(NdotV, roughness); // View direction
    float ggx1 = GeometrySchlickGGX(NdotL, roughness); // Light direction
    return ggx1 * ggx2;
}
```

### 6.4 Complete PBR Shader (Unified Implementation)

```glsl
vec3 calculatePBR(vec3 N, vec3 V, vec3 L, vec3 albedo,
                  float metallic, float roughness, vec3 lightColor) {
    vec3 H = normalize(V + L);
    
    // Compute F0 (base reflectivity)
    vec3 F0 = mix(vec3(0.04), albedo, metallic);
    
    // Cook-Torrance components
    float D = DistributionGGX(N, H, roughness);
    float G = GeometrySmith(N, V, L, roughness);
    vec3  F = fresnelSchlick(max(dot(H, V), 0.0), F0);
    
    // Specular BRDF
    vec3 numerator   = D * G * F;
    float denominator = 4.0 * max(dot(N, V), 0.0) * max(dot(N, L), 0.0) + 0.0001;
    vec3 specular = numerator / denominator;
    
    // Energy conservation
    vec3 kS = F;
    vec3 kD = (vec3(1.0) - kS) * (1.0 - metallic);
    
    // Lambertian diffuse (divided by PI for energy conservation)
    vec3 diffuse = kD * albedo / PI;
    
    // Final lighting
    float NdotL = max(dot(N, L), 0.0);
    return (diffuse + specular) * lightColor * NdotL;
}

// Main fragment shader
void main() {
    vec3 Lo = vec3(0.0);
    
    // Accumulate lighting from all sources
    for (int i = 0; i < numLights; i++) {
        vec3 L = normalize(lightPositions[i] - fragWorldPos);
        float distance = length(lightPositions[i] - fragWorldPos);
        float attenuation = 1.0 / (distance * distance); // Inverse-square law
        vec3 radiance = lightColors[i] * attenuation;
        
        Lo += calculatePBR(N, V, L, albedo, metallic, roughness, radiance);
    }
    
    // Ambient (temporary — replaced by IBL or GI in production)
    vec3 ambient = vec3(0.03) * albedo;
    vec3 color = ambient + Lo;
    
    // Tone mapping and gamma correction (see Module 8)
    color = color / (color + vec3(1.0));  // Reinhard tone mapping
    color = pow(color, vec3(1.0 / 2.2));  // Gamma correction
    
    fragColor = vec4(color, 1.0);
}
```

---

## 7. Performance Impact & GPU Internal Behavior

### 7.1 GPU Cost Analysis

| Operation | Cost per Pixel | Bottleneck Type |
|-----------|---------------|-----------------|
| 4× texture samples (albedo, normal, metallic, roughness) | ~4 texture units, ~4KB bandwidth | **Bandwidth** |
| `DistributionGGX()` | ~6 ALU ops | Compute |
| `GeometrySmith()` | ~8 ALU ops | Compute |
| `fresnelSchlick()` | ~4 ALU ops + 1 `pow()` | Compute |
| Denominator + final multiply | ~6 ALU ops | Compute |
| **Total per light** | **~24 ALU + 4 tex** | |
| **10 lights at 4K** | **24 × 10 × 8.3M pixels ≈ 2 billion ops** | **ALU-heavy** |

### 7.2 Optimization Strategies

| Strategy | Savings | Implementation |
|----------|---------|----------------|
| ORM Texture Packing | 66% fewer texture samples | Pack AO (R), Roughness (G), Metallic (B) into one texture |
| Split-Sum Approximation | Avoid per-pixel integration for IBL | Pre-compute BRDF LUT (2D texture indexed by NdotV × roughness) |
| Half-resolution Specular | 75% fewer specular calculations | Compute specular at half-res, bilateral upsample |
| Deferred Rendering | Eliminate overdraw | Calculate PBR once per visible pixel (see Module 5) |
| `mediump` for intermediate values | 2× register savings | Use 16-bit where full precision isn't needed |

---

## 8. Common Mistakes & Debugging

| Mistake | Visual Symptom | Root Cause | Fix |
|---------|---------------|------------|-----|
| Albedo not in linear space | Washed out, over-bright | sRGB texture sampled without conversion | Use `sRGB` texture format or `pow(tex.rgb, vec3(2.2))` |
| Missing `/ PI` in diffuse | Scene is ~3.14× too bright | Lambertian not normalized | Always divide diffuse by PI |
| Pure black or white albedo | Unrealistic lighting response | Physical impossibility | Clamp albedo to [0.04, 0.96] |
| `NdotV` or `NdotL` goes negative | NaN propagation, black/white flicker | Unclamped dot product in denominator | Use `max(dot(N,V), 0.001)` everywhere |
| Roughness = 0.0 exactly | Infinitely bright specular spike | GGX D term → infinity | Clamp minimum roughness to 0.04 |
| Fresnel not applied to energy split | Double-bright edges | kD not derived from kS | `kD = (1.0 - F) * (1.0 - metallic)` |

### Debug Visualization Modes

```glsl
// Add to your fragment shader for per-channel debugging
uniform int debugMode;

vec3 applyDebug(vec3 color, vec3 N, vec3 albedo, float metallic,
                float roughness, float ao, vec3 F0) {
    switch (debugMode) {
        case 0: return color;                       // Normal render
        case 1: return N * 0.5 + 0.5;               // World normals
        case 2: return albedo;                       // Base color (linear)
        case 3: return vec3(metallic);               // Metallic mask
        case 4: return vec3(roughness);              // Roughness map
        case 5: return vec3(ao);                     // AO map
        case 6: return F0;                           // Fresnel F0
        case 7: return vec3(max(dot(N, V), 0.0));    // NdotV (check normals face camera)
    }
    return color;
}
```
