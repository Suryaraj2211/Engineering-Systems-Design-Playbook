# Chapter 08 — The Graphics Pipeline

## What Is the Graphics Pipeline?
The pipeline is the fixed sequence of stages that transforms 3D data (vertices) into 2D colored pixels on your screen. Every WebGL/WebGPU draw call flows through this pipeline.

---

## 1. The Complete Pipeline (Stage by Stage)

```
THE GRAPHICS PIPELINE:
══════════════════════
  INPUT: Vertex buffer + Index buffer
    ↓
  ┌─────────────────────────┐
  │ 1. INPUT ASSEMBLY         │ Reads vertex/index data from GPU buffers.
  │    (Hardware, automatic)  │ Groups vertices into triangles.
  └───────────┬───────────────┘
              ↓
  ┌─────────────────────────┐
  │ 2. VERTEX SHADER          │ ★ PROGRAMMABLE
  │    Runs ONCE per vertex.  │ Transforms vertices from model → clip space.
  │    Input: attributes      │ Output: gl_Position (clip coordinates)
  │    Output: varyings       │ Also passes data to fragment shader.
  └───────────┬───────────────┘
              ↓
  ┌─────────────────────────┐
  │ 3. CLIPPING               │ Removes triangles outside the view frustum.
  │    (Hardware, automatic)  │ Partially-visible triangles are cut and
  │                           │ new vertices are generated.
  └───────────┬───────────────┘
              ↓
  ┌─────────────────────────┐
  │ 4. PERSPECTIVE DIVIDE     │ Divides x, y, z by w component.
  │    (Hardware, automatic)  │ Converts clip space → NDC [-1, 1]
  └───────────┬───────────────┘
              ↓
  ┌─────────────────────────┐
  │ 5. VIEWPORT TRANSFORM     │ Maps NDC → screen pixel coordinates.
  │    (Hardware, automatic)  │ (-1,-1) → (0, 0), (1,1) → (width, height)
  └───────────┬───────────────┘
              ↓
  ┌─────────────────────────┐
  │ 6. RASTERIZATION          │ Converts triangles → fragments (pixels).
  │    (Hardware, automatic)  │ Determines which pixels each triangle covers.
  │                           │ INTERPOLATES varyings across the triangle.
  └───────────┬───────────────┘
              ↓
  ┌─────────────────────────┐
  │ 7. FRAGMENT SHADER        │ ★ PROGRAMMABLE
  │    Runs ONCE per fragment.│ Calculates the final COLOR of each pixel.
  │    Input: interpolated    │ Samples textures, computes lighting.
  │    varyings               │ Output: fragColor (RGBA)
  └───────────┬───────────────┘
              ↓
  ┌─────────────────────────┐
  │ 8. DEPTH TEST             │ Compares fragment depth vs depth buffer.
  │    (Hardware, automatic)  │ If new fragment is CLOSER → keep it.
  │                           │ If farther → discard it. (No overdraw cost!)
  └───────────┬───────────────┘
              ↓
  ┌─────────────────────────┐
  │ 9. BLENDING               │ Combines fragment color with framebuffer.
  │    (Hardware, automatic)  │ Used for transparency: alpha blending.
  │                           │ result = src × alpha + dst × (1 - alpha)
  └───────────┬───────────────┘
              ↓
  OUTPUT: Framebuffer (the final image on screen)
```

---

## 2. Rasterization — How Triangles Become Pixels

### 2.1 The Algorithm

```
RASTERIZATION (per triangle):
═════════════════════════════
  Triangle has 3 screen-space vertices: v0, v1, v2.
  
  For each pixel (x, y) in the triangle's bounding box:
    1. Compute BARYCENTRIC COORDINATES (λ0, λ1, λ2).
       These tell us: "Is this pixel INSIDE the triangle?"
       λ0 + λ1 + λ2 = 1     (if exactly on the triangle)
       All λ ≥ 0              (if inside)
       Any λ < 0              (if outside → skip this pixel!)
       
    2. INTERPOLATE vertex attributes using barycentric weights:
       pixelColor = λ0 × v0.color + λ1 × v1.color + λ2 × v2.color
       pixelUV    = λ0 × v0.uv    + λ1 × v1.uv    + λ2 × v2.uv
       pixelNormal = λ0 × v0.normal + λ1 × v1.normal + λ2 × v2.normal
```

### 2.2 Barycentric Coordinates - Manual Calculation

```
EXAMPLE:
  Triangle vertices (screen pixels):
    V0 = (100, 100)
    V1 = (200, 100)
    V2 = (150, 200)
  
  Test pixel P = (150, 150):
  
  Edge function method:
    E01(P) = (V1.x-V0.x)×(P.y-V0.y) - (V1.y-V0.y)×(P.x-V0.x)
           = (200-100)×(150-100) - (100-100)×(150-100)
           = 100×50 - 0×50 = 5000
           
    E12(P) = (V2.x-V1.x)×(P.y-V1.y) - (V2.y-V1.y)×(P.x-V1.x)
           = (150-200)×(150-100) - (200-100)×(150-200)
           = (-50)×50 - 100×(-50) = -2500 + 5000 = 2500
    
    E20(P) = (V0.x-V2.x)×(P.y-V2.y) - (V0.y-V2.y)×(P.x-V2.x)
           = (100-150)×(150-200) - (100-200)×(150-150)
           = (-50)×(-50) - (-100)×0 = 2500

  All values ≥ 0 → pixel is INSIDE the triangle ✓
  
  Normalized:
    total = 5000 + 2500 + 2500 = 10000
    λ0 = 2500/10000 = 0.25 (closest to V0)
    λ1 = 2500/10000 = 0.25 (closest to V1)
    λ2 = 5000/10000 = 0.50 (closest to V2)
```

---

## 3. Depth Testing — Z-Buffer Algorithm

```
DEPTH BUFFER ALGORITHM:
═══════════════════════
  The depth buffer is a 2D array matching screen resolution.
  Each pixel stores the CLOSEST depth seen so far.
  
  Initialize: every depth = 1.0 (farthest possible).
  
  For each fragment produced by rasterization:
    if (fragment.depth < depthBuffer[x][y]):
        // This fragment is CLOSER to camera than what's already there
        colorBuffer[x][y] = fragment.color   // Update color
        depthBuffer[x][y] = fragment.depth   // Update depth
        // The old farther fragment is DISCARDED
    else:
        // This fragment is BEHIND an existing surface. DISCARD IT.
        // The fragment shader's output is thrown away!
      
  RESULT: Only the closest surface at each pixel is visible.
          No need to sort triangles by distance!
```

---

## 4. Blending — Alpha Transparency
```
ALPHA BLENDING FORMULA:
═══════════════════════
  result = source.rgb × source.a + destination.rgb × (1 - source.a)
  
  EXAMPLE:
    Background (destination) = red (1, 0, 0)
    Glass overlay (source) = blue (0, 0, 1) with alpha = 0.3
    
    result.r = 0 × 0.3 + 1 × 0.7 = 0.7
    result.g = 0 × 0.3 + 0 × 0.7 = 0.0
    result.b = 1 × 0.3 + 0 × 0.7 = 0.3
    
    Final: (0.7, 0.0, 0.3) — reddish with blue tint ✓

  CRITICAL: Transparent objects MUST be drawn BACK-TO-FRONT (sorted by distance).
  If drawn front-to-back, the back objects fail depth test and are discarded!
```

---

## 5. Vertex vs. Fragment Performance

| Stage | Runs | Count (typical scene) | Cost Each | Total |
|-------|------|----------------------|-----------|-------|
| Vertex Shader | Per vertex | 100,000 vertices | ~20 ALU | 2M ops |
| Fragment Shader | Per pixel | 2,000,000 pixels × 3 overdraw | ~50 ALU | 300M ops |

**The Fragment Shader dominates cost by 150×.** This is why shader optimization focuses on the fragment stage.

---

## 6. Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Drawing transparent objects without sorting | Back objects vanish behind transparent front | Sort transparent meshes by distance, draw last |
| Forgetting depth test | Objects draw on top of each other randomly | `gl.enable(gl.DEPTH_TEST)` |
| Wrong winding order | Triangles face inside-out (invisible) | Use consistent CCW winding; enable backface culling |
