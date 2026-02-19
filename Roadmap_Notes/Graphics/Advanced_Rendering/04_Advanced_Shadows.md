# Module 4 — Advanced Shadow Mapping

> This module covers the mathematics of shadow mapping, the artifacts that plague it,
> and the filtering techniques that make shadows production-quality.

---

## 1. Shadow Mapping — The Core Algorithm

### 1.1 The Two-Pass Architecture

```
SHADOW MAPPING PIPELINE:
════════════════════════
  PASS 1: Render the scene FROM THE LIGHT'S PERSPECTIVE.
    Camera = Light position
    Direction = Light direction
    Output = Depth buffer only (no color needed)
    This "shadow map" stores the closest surface depth for every pixel
    the light can see.

  PASS 2: Render the scene FROM THE CAMERA'S PERSPECTIVE.
    For each fragment:
      1. Transform fragment position into the light's clip space.
      2. Sample the shadow map at that position.
      3. Compare: Is the fragment's depth > the shadow map's depth?
         YES → Fragment is BEHIND something the light already hit → IN SHADOW
         NO  → Fragment IS the closest surface to the light → LIT
```

### 1.2 The Math of Light-Space Projection

```glsl
// In the CAMERA's fragment shader:
vec3 fragWorldPos = vWorldPosition;

// Transform fragment into the LIGHT's clip space
vec4 fragLightSpace = lightViewProjection * vec4(fragWorldPos, 1.0);

// Perspective divide (NDC)
vec3 projCoords = fragLightSpace.xyz / fragLightSpace.w;

// Map from [-1, 1] to [0, 1] for texture sampling
projCoords = projCoords * 0.5 + 0.5;

// Sample the shadow map
float closestDepth = texture(shadowMap, projCoords.xy).r;
float currentDepth = projCoords.z;

// Depth comparison
float shadow = currentDepth > closestDepth ? 1.0 : 0.0;
// shadow = 1.0 means IN SHADOW, 0.0 means LIT
```

### 1.3 Light Projection Matrix Selection

| Light Type | Projection | Shadow Map Format |
|-----------|-----------|-------------------|
| Directional (Sun) | Orthographic | 2D texture (covers entire scene) |
| Point (Bulb) | 6x Perspective (Cubemap) | Cube texture (6 faces) |
| Spot | Perspective | 2D texture (cone frustum) |

```glsl
// Directional Light: Orthographic projection
mat4 lightProj = ortho(-20.0, 20.0, -20.0, 20.0, 0.1, 100.0);

// Spot Light: Perspective projection matching cone angle
mat4 lightProj = perspective(spotAngle * 2.0, 1.0, 0.1, farPlane);

// Point Light: Render 6 faces of a cubemap
// Each face uses perspective(90°, 1.0, near, far)
```

---

## 2. Shadow Acne — The Depth Bias Problem

### 2.1 What Causes Shadow Acne

```
SHADOW ACNE VISUALIZATION:
══════════════════════════
  Shadow map texel covers a 10cm × 10cm area of the floor.
  The shadow map stores a SINGLE depth value for that entire area.
  
  The camera renders 100 fragments within that same 10cm area.
  Due to floating-point precision (and the surface not being perfectly
  aligned with the shadow map grid), some fragments calculate a depth
  that is 0.0001 units DEEPER than the shadow map value.
  
  Shadow test: 0.5001 > 0.5000? → YES → IN SHADOW (WRONG!)
  
  The surface appears to shadow ITSELF, creating a moiré pattern
  of alternating lit/shadowed pixels = "Shadow Acne."
  
  TOP VIEW OF FLOOR (with shadow acne):
   ░▓░▓░▓░▓░▓░▓░▓░▓░▓
   ▓░▓░▓░▓░▓░▓░▓░▓░▓░
   ░▓░▓░▓░▓░▓░▓░▓░▓░▓
   ▓░▓░▓░▓░▓░▓░▓░▓░▓░
```

### 2.2 The Depth Bias Fix

Push the shadow comparison slightly toward the light to prevent self-shadowing:

```glsl
float bias = 0.005; // Simple constant bias
float shadow = currentDepth - bias > closestDepth ? 1.0 : 0.0;
```

**Problem with constant bias:** Too little bias → acne remains. Too much bias → shadows "detach" from objects (called "Peter Panning" — the shadow floats away from the base of the caster).

### 2.3 Slope-Scale Bias (The Proper Solution)

Surfaces angled sharply relative to the light need MORE bias. Surfaces perpendicular to the light need almost none.

```glsl
float computeBias(vec3 normal, vec3 lightDir) {
    float cosAngle = max(dot(normal, lightDir), 0.0);
    
    // steeper angle → larger bias needed
    float slopeBias = 0.05 * (1.0 - cosAngle); // 0 at perpendicular, 0.05 at grazing
    float constantBias = 0.005;
    
    return max(slopeBias, constantBias);
}

float bias = computeBias(N, L);
float shadow = currentDepth - bias > closestDepth ? 1.0 : 0.0;
```

---

## 3. PCF (Percentage-Closer Filtering)

### 3.1 The Hard Shadow Problem

A single depth comparison produces binary results: either fully lit (0.0) or fully shadowed (1.0). This creates razor-sharp, jagged shadow edges that alias terribly at lower resolutions.

### 3.2 PCF Algorithm

Instead of 1 comparison, perform NxN comparisons around the sample point and average the results:

```glsl
float computePCF(vec3 projCoords, float currentDepth, float bias) {
    float shadow = 0.0;
    vec2 texelSize = 1.0 / vec2(textureSize(shadowMap, 0));
    
    // 5×5 kernel = 25 comparisons
    int halfKernel = 2;
    for (int x = -halfKernel; x <= halfKernel; x++) {
        for (int y = -halfKernel; y <= halfKernel; y++) {
            float pcfDepth = texture(shadowMap, 
                projCoords.xy + vec2(x, y) * texelSize).r;
            shadow += currentDepth - bias > pcfDepth ? 1.0 : 0.0;
        }
    }
    
    shadow /= float((halfKernel * 2 + 1) * (halfKernel * 2 + 1)); // /25
    return shadow;
}
```

### 3.3 PCF GPU Cost Analysis

| Kernel Size | Comparisons | Texture Reads | Quality | Performance (1080p) |
|-------------|-------------|---------------|---------|-------------------|
| 1×1 | 1 | 1 | Hard edges, jagged | ~0.2ms |
| 3×3 | 9 | 9 | Slightly softened | ~0.4ms |
| 5×5 | 25 | 25 | Soft, good quality | ~0.8ms |
| 7×7 | 49 | 49 | Very soft | ~1.5ms |
| 9×9 | 81 | 81 | Extremely soft | ~2.5ms |

**Optimization: Use `textureGather()`**
On modern GPUs, `textureGather()` returns the 4 nearest texels in a single hardware operation (1 texture read instead of 4). A 5×5 PCF can be done with 9 `textureGather` calls instead of 25 regular reads.

### 3.4 PCSS (Percentage-Closer Soft Shadows)

Real shadows have variable penumbra width: hard near the caster, soft far from it.

```
PCSS PENUMBRA MATH:
═══════════════════
  penumbraWidth = (lightSize × (receiverDepth - avgBlockerDepth)) / avgBlockerDepth
  
  If the shadow caster is CLOSE to the receiver:
    avgBlockerDepth ≈ receiverDepth → penumbra ≈ 0 → sharp shadow
    
  If the shadow caster is FAR from the receiver:
    avgBlockerDepth << receiverDepth → penumbra is large → soft shadow
    
  PCSS algorithm:
    1. Blocker search: Sample NxN area to find average blocker depth.
    2. Calculate penumbra width from the formula above.
    3. Run PCF with kernel size proportional to penumbra width.
```

---

## 4. Cascaded Shadow Maps (CSM) — Large Scenes

### 4.1 The Resolution Problem

A single 2048×2048 shadow map covering a 500m² outdoor level means each texel covers:
$500m / 2048 = 0.24m$ per texel. The shadow of a 2cm-thick sword blade is invisible!

### 4.2 CSM Architecture

Split the camera's view frustum into multiple segments. Each segment gets its own shadow map.

```
CASCADED SHADOW MAPS (3 cascades):
══════════════════════════════════
  Camera ← near                                               far →
  
  ├───── Cascade 0 ─────┤
  │   0m – 10m           │   2048×2048 shadow map
  │   10m / 2048 texels  │   = 0.5cm per texel (razor sharp!)
  │                      │
  ├──────── Cascade 1 ───────┤
  │         10m – 50m        │   2048×2048 shadow map
  │         40m / 2048       │   = 2cm per texel (good quality)
  │                          │
  ├──────────── Cascade 2 ───────────┤
  │             50m – 200m           │   2048×2048 shadow map
  │             150m / 2048          │   = 7.3cm per texel (acceptable)
  │                                  │
```

### 4.3 CSM Shader Implementation

```glsl
float computeCSMShadow(vec3 fragWorldPos, vec3 N, vec3 L) {
    // Determine which cascade this fragment falls into
    float viewDepth = abs((uView * vec4(fragWorldPos, 1.0)).z);
    
    int cascadeIndex = 0;
    if (viewDepth > cascadeSplits[0]) cascadeIndex = 1;
    if (viewDepth > cascadeSplits[1]) cascadeIndex = 2;
    
    // Transform to the selected cascade's light space
    vec4 fragLightSpace = cascadeLightVP[cascadeIndex] * vec4(fragWorldPos, 1.0);
    vec3 projCoords = fragLightSpace.xyz / fragLightSpace.w * 0.5 + 0.5;
    
    float bias = computeBias(N, L);
    float shadow = computePCF(projCoords, projCoords.z, bias);
    
    return shadow;
}
```

### 4.4 Cascade Split Strategies

| Strategy | Split Points (200m far plane) | Pro | Con |
|----------|------------------------------|-----|-----|
| Uniform | 66m, 133m, 200m | Simple | Near objects get poor resolution |
| Logarithmic | 6m, 36m, 200m | Best near-camera quality | Poor at medium range |
| Practical (PSSM) | mix(uniform, log, λ=0.5) | Balanced | Most common in production |

```
PRACTICAL SPLIT FORMULA:
════════════════════════
  C_log(i) = near × (far/near)^(i/N)
  C_uni(i) = near + (far - near) × (i/N)
  C(i)     = λ × C_log(i) + (1 - λ) × C_uni(i)
  
  With λ = 0.5, near = 0.5, far = 200, N = 3:
    Split 0: 10.5m
    Split 1: 48.7m
    Split 2: 200m
```

### 4.5 Cascade Transition Blending

Without blending, there's a visible hard line where cascade 0 ends and cascade 1 begins (shadow resolution suddenly drops). Fix by blending in a transition zone:

```glsl
// Smooth cascade transition
float cascadeFade = smoothstep(
    cascadeSplits[cascadeIndex] - fadeWidth,
    cascadeSplits[cascadeIndex],
    viewDepth
);

// If we're in the fade zone, sample BOTH cascades and blend
if (cascadeFade > 0.0 && cascadeIndex < MAX_CASCADES - 1) {
    float shadowCurrent = sampleCascade(cascadeIndex, fragWorldPos);
    float shadowNext    = sampleCascade(cascadeIndex + 1, fragWorldPos);
    shadow = mix(shadowCurrent, shadowNext, cascadeFade);
}
```

---

## 5. GPU Cost & Performance Trade-offs

| Configuration | Shadow Map Memory | Render Passes | Total Cost (1080p) |
|--------------|-------------------|---------------|-------------------|
| No shadows | 0 | 0 | 0ms |
| 1× 2048 map, no PCF | 16MB | 1 | ~0.5ms |
| 1× 2048 map, 5×5 PCF | 16MB | 1 | ~1.0ms |
| 3× CSM 2048, 3×3 PCF | 48MB | 3 | ~2.0ms |
| 3× CSM 4096, 5×5 PCF | 192MB | 3 | ~4.0ms |
| 3× CSM 4096, PCSS | 192MB | 3 + blocker | ~5.5ms |

---

## 6. Common Mistakes & Debugging

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Shadow map too small | Blocky pixelated shadows | Increase resolution or use CSM |
| Bias too small | Shadow acne (moiré stripes) | Increase bias; use slope-scale bias |
| Bias too large | Peter Panning (shadows detach) | Reduce bias; use front-face culling during shadow pass |
| Missing perspective divide | Shadows in wrong position entirely | `projCoords.xyz /= projCoords.w` |
| Sampling outside shadow map | Random shadow artifacts at edges | Clamp UV to [0,1] or return 0.0 outside bounds |
| Cascade splits poorly balanced | Near shadows blurry, far shadows sharp | Use PSSM (practical split scheme) |

### Debug Visualization

```glsl
// Visualize cascade assignment as colors
vec3 debugCascade(int cascadeIndex) {
    if (cascadeIndex == 0) return vec3(1.0, 0.0, 0.0); // Red = near
    if (cascadeIndex == 1) return vec3(0.0, 1.0, 0.0); // Green = mid
    if (cascadeIndex == 2) return vec3(0.0, 0.0, 1.0); // Blue = far
    return vec3(1.0); // White = outside all cascades
}

// Visualize shadow map depth as greyscale
vec3 debugShadowMap(vec2 projCoords) {
    float depth = texture(shadowMap, projCoords).r;
    return vec3(depth); // Black = close to light, White = far from light
}
```
