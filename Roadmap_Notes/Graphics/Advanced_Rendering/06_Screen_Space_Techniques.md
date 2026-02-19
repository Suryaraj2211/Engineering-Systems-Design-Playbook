# Module 6 — Screen-Space Techniques

> Screen-space techniques operate on the 2D image the GPU has already rendered.
> They're fast because they reuse existing data but fundamentally limited
> because they can only see what's on-screen.

---

## 1. SSAO (Screen-Space Ambient Occlusion)

*See Module 3 for the complete SSAO algorithm and shader code.*

### 1.1 HBAO+ (Horizon-Based AO) — The Production Upgrade

Standard SSAO samples random points in a hemisphere. HBAO+ instead marches along screen-space rays and tracks the "horizon angle":

```glsl
float computeHBAO(vec2 texCoords) {
    vec3 viewPos = getViewSpacePos(texCoords);
    vec3 viewNormal = getViewSpaceNormal(texCoords);
    
    float ao = 0.0;
    int numDirections = 8;   // 8 ray directions in screen space
    int numSteps = 4;        // 4 steps per ray
    
    for (int d = 0; d < numDirections; d++) {
        float angle = float(d) * (2.0 * PI / float(numDirections));
        vec2 rayDir = vec2(cos(angle), sin(angle));
        
        // Add per-pixel random rotation to break banding
        float rotation = texture(noiseTexture, texCoords * noiseScale).r * PI;
        rayDir = mat2(cos(rotation), -sin(rotation), 
                      sin(rotation),  cos(rotation)) * rayDir;
        
        float maxHorizonAngle = -PI * 0.5; // Start at the horizon (no occlusion)
        
        for (int s = 1; s <= numSteps; s++) {
            vec2 sampleUV = texCoords + rayDir * float(s) * stepSize;
            vec3 samplePos = getViewSpacePos(sampleUV);
            
            vec3 horizonVec = samplePos - viewPos;
            float horizonAngle = atan(horizonVec.z, length(horizonVec.xy));
            
            // If this sample is ABOVE the previous horizon, it occludes more
            maxHorizonAngle = max(maxHorizonAngle, horizonAngle);
        }
        
        // The normal points upward. The horizon is at some angle from it.
        // More occlusion = horizon is closer to the normal direction.
        float normalAngle = atan(viewNormal.z, length(viewNormal.xy));
        ao += max(0.0, sin(maxHorizonAngle) - sin(normalAngle));
    }
    
    return 1.0 - (ao / float(numDirections));
}
```

### 1.2 GTAO (Ground Truth AO) — Current Industry Standard

GTAO further refines horizon-based AO by mathematically integrating the visibility function against the cosine term, producing physically plausible results with fewer samples:

```
GTAO vs SSAO vs HBAO+:
═══════════════════════
  Technique    Samples    Quality              Cost (1080p)
  ──────────   ──────     ──────────────       ────────────
  SSAO         32         Good (some halos)    ~1.5ms
  HBAO+        8×4=32     Very good            ~1.2ms
  GTAO         8×2=16     Excellent (cosine-   ~0.8ms
                          weighted integral)
```

---

## 2. SSR (Screen-Space Reflections)

### 2.1 The Algorithm

SSR creates reflections by tracing the reflected view direction through the depth buffer:

```
SSR ALGORITHM:
══════════════
  For each pixel:
    1. Calculate reflection direction: R = reflect(-V, N)
    2. Starting from the pixel's 3D position, march along R in small steps.
    3. At each step, project the 3D position back to screen space.
    4. Sample the depth buffer at the projected screen position.
    5. If the depth buffer value is CLOSER than our march position:
       → We hit something! Sample the COLOR buffer at this UV.
       → This is our reflection!
    6. If we march off the screen edge or exceed max distance:
       → Fallback to environment cubemap.
```

### 2.2 Hi-Z (Hierarchical-Z) Raymarching — The Fast Version

Brute-force marching (step-by-step) is slow. Hi-Z uses a depth buffer mipmap to skip large empty regions:

```glsl
vec2 hiZRaymarch(vec3 origin, vec3 dir, float maxDistance) {
    float mipLevel = 0.0;
    float t = 0.0;
    
    for (int i = 0; i < 64; i++) {
        vec3 marchPos = origin + dir * t;
        
        // Project to screen
        vec4 clipPos = uProjection * vec4(marchPos, 1.0);
        vec2 uv = (clipPos.xy / clipPos.w) * 0.5 + 0.5;
        
        if (uv.x < 0.0 || uv.x > 1.0 || uv.y < 0.0 || uv.y > 1.0)
            return vec2(-1.0); // Off screen
        
        // Sample depth at THIS mip level
        float sceneDepth = textureLod(hiZBuffer, uv, mipLevel).r;
        float marchDepth = linearize(clipPos.z / clipPos.w);
        
        if (marchDepth > sceneDepth) {
            // HIT! But we might have overshot.
            if (mipLevel <= 0.0) {
                // We're at finest LOD — this is a real hit
                return uv;
            }
            // Step BACK and refine at finer LOD
            t -= step;
            mipLevel -= 1.0;
        } else {
            // No hit — take a BIG step (proportional to mip level)
            t += step * pow(2.0, mipLevel);
            mipLevel = min(mipLevel + 1.0, maxMip);
        }
    }
    return vec2(-1.0); // No hit
}
```

### 2.3 GPU Cost Analysis

| SSR Configuration | Steps | Texture Reads | Cost (1080p) |
|-------------------|-------|---------------|--------------|
| Brute-force, 64 steps | 64 | 64 depth reads | ~3.5ms |
| Hi-Z, 32 steps | 32 | 32 mip reads | ~2.0ms |
| Hi-Z + quarter-res | 32 | 32 mip reads | ~0.6ms |
| + temporal accumulation | 16 | 16 mip reads | ~0.4ms |

### 2.4 SSR Artifacts & Solutions

| Artifact | Cause | Solution |
|----------|-------|---------|
| Reflections disappear at screen edges | Ray exits screen bounds | Fade SSR to 0 in last 10% of screen, blend with cubemap |
| Black holes in reflections | Reflected point is behind another object (occluded) | Increase ray thickness; accept some error |
| Stretched/incorrect reflections | Depth buffer can't represent backfaces | Trace at half-resolution and bilateral upsample |
| Rough surfaces show sharp reflections | SSR traces 1 ray (1 direction) | Blur SSR result proportional to roughness (separate pass) |
| Reflection appears on wrong objects | Step size too large, overshooting | Reduce step size; use Hi-Z for adaptive stepping |

---

## 3. Screen-Space Shadows (Contact Shadows)

Traditional shadow maps lack resolution for small-scale shadowing (fingers on a hand, chain links). Screen-Space Shadows add micro-detail:

```glsl
float screenSpaceShadow(vec3 fragPos, vec3 lightDir) {
    float shadow = 0.0;
    float stepSize = 0.02; // Small steps (2cm)
    int numSteps = 16;
    
    vec3 marchPos = fragPos;
    
    for (int i = 0; i < numSteps; i++) {
        marchPos += lightDir * stepSize;
        
        // Project to screen space
        vec4 clipPos = uViewProj * vec4(marchPos, 1.0);
        vec2 uv = (clipPos.xy / clipPos.w) * 0.5 + 0.5;
        
        float sceneDepth = texture(gDepth, uv).r;
        float marchDepth = clipPos.z / clipPos.w * 0.5 + 0.5;
        
        // If geometry is CLOSER to the camera than our march position
        // along the light direction, that geometry is blocking the light
        if (marchDepth > sceneDepth + 0.001) {
            shadow = 1.0;
            break;
        }
    }
    
    return shadow;
}
```

**GPU Cost:** ~0.3ms at 1080p (very cheap — only 16 steps). Use as a complement to shadow maps, not a replacement.

---

## 4. The Fundamental Limitation of All Screen-Space Techniques

```
THE SCREEN-SPACE CONTRACT:
══════════════════════════
  WHAT YOU HAVE: A 2D depth buffer + 2D color buffer.
  
  WHAT YOU DON'T HAVE:
    ✗ Geometry behind the camera
    ✗ Geometry occluded by closer objects
    ✗ Geometry outside the camera frustum
    ✗ The BACK SIDE of any visible surface
    ✗ The interior of any transparent object
    
  CONSEQUENCE:
    SSR reflects a ball → perfect.
    Camera rotates so the ball is off-screen → reflection VANISHES.
    
    SSAO darkens a corner → perfect.
    The wall creating the corner is off-screen → AO disappears.
    
  MITIGATION:
    Always have a FALLBACK for when screen-space data is missing:
    - SSR → fallback to environment cubemap
    - SSGI → fallback to baked lightmap or ambient probe
    - SS Shadows → fallback to shadow map
    
    Blend between techniques using confidence weights based on
    how much of the screen-space data is available.
```

---

## 5. Performance Summary Table

| Technique | Cost (1080p) | Bottleneck | Quality | Missing Data Handling |
|-----------|-------------|------------|---------|----------------------|
| SSAO | ~1.5ms | Bandwidth (32 depth reads/pixel) | Good | N/A (multiplicative only) |
| HBAO+ | ~1.2ms | Bandwidth (32 horizon reads) | Very Good | N/A |
| GTAO | ~0.8ms | Bandwidth + ALU | Excellent | N/A |
| SSR (Hi-Z) | ~2.0ms | Bandwidth (depth mip marching) | Good | Cubemap fallback |
| SSR (quarter-res) | ~0.6ms | Same, reduced resolution | Acceptable | Same + bilateral upsample |
| Contact Shadows | ~0.3ms | Bandwidth (16 depth reads) | Subtle detail | Shadow map fallback |

---

## 6. Debugging Screen-Space Techniques

```glsl
// Visualize SSR confidence (where reflections are reliable vs fallback)
vec3 debugSSR(vec2 hitUV, float confidence) {
    if (hitUV.x < 0.0) return vec3(1.0, 0.0, 0.0);     // RED: no hit (fallback to cubemap)
    if (confidence < 0.3) return vec3(1.0, 1.0, 0.0);   // YELLOW: low confidence
    return vec3(0.0, 1.0, 0.0);                          // GREEN: reliable SSR
}

// Visualize SSAO kernel coverage
vec3 debugSSAO(float ao) {
    // Heatmap: dark red = heavy occlusion, green = open
    return mix(vec3(0.5, 0.0, 0.0), vec3(0.0, 1.0, 0.0), ao);
}
```
