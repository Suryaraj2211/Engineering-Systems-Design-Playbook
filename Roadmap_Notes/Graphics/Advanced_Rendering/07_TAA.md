# Module 7 — Temporal Anti-Aliasing (TAA)

> TAA is the dominant anti-aliasing technique in modern engines.
> It uses temporal data (previous frames) to reconstruct sub-pixel detail.

---

## 1. Why Aliasing Happens — The Root Cause

### 1.1 The Nyquist Problem

A pixel is a single point sample. If geometry features are smaller than 1 pixel (e.g., wire fences, foliage edges, distant railing), the rasterizer either samples the geometry or misses it entirely.

```
ALIASING EXAMPLE (1D, horizontal edge):
═══════════════════════════════════════
  Actual geometry edge position (continuous):
  ───────────────────────╱─────────────
                        ╱  (diagonal edge)
  
  Pixel grid (discrete samples):
  ██░░██░░██░░██░░██░░██░░██
  
  Pixel centers land ON the edge (██ = solid) or MISS it (░░ = empty).
  The staircase pattern = "jaggies" = spatial aliasing.
  
  At higher resolution (4× more pixels):
  ████████░░░░████████░░░░████████░░░░
  Same problem, just smaller stairs.
```

### 1.2 Anti-Aliasing Approaches Overview

| Technique | How | Quality | Cost | Modern Usage |
|-----------|-----|---------|------|-------------|
| SSAA (4×) | Render at 4× resolution, downsample | Perfect | 4× everything | Never (too expensive) |
| MSAA (4×) | 4 samples per pixel at edges | Good | 2-4× geometry, bandwidth | Forward rendering only |
| FXAA | Post-process blur on detected edges | OK | ~0.5ms | Fallback / mobile |
| TAA | Temporal accumulation of jittered frames | Excellent | ~0.8ms | **Industry standard** |

---

## 2. TAA Architecture — Step by Step

### 2.1 The Complete Pipeline

```
TAA PIPELINE (per frame):
═════════════════════════
  STEP 1: JITTER the projection matrix by a sub-pixel offset.
    The camera "vibrates" by less than 1 pixel each frame.
    Frame 0: shift right 0.25px, up 0.125px
    Frame 1: shift left 0.375px, down 0.25px
    Frame 2: shift right 0.125px, up 0.375px
    ... (Halton sequence, cycles through 8-16 offsets)
    
  STEP 2: RENDER the scene normally (with jitter applied).
    Each frame sees a SLIGHTLY different view of the sub-pixel world.
    
  STEP 3: REPROJECT current pixels to previous frame's position.
    Using motion vectors, find where each pixel WAS in the last frame.
    
  STEP 4: BLEND current frame with history (previous accumulation).
    If a pixel hasn't moved: blend 95% history + 5% current (very stable).
    If a pixel moved fast: blend 20% history + 80% current (avoid ghosting).
    
  STEP 5: Store the blended result as the new "history" for next frame.
```

### 2.2 Sub-Pixel Jitter (Halton Sequence)

```glsl
// Halton sequence generates well-distributed 2D offsets in [0, 1]
float halton(int index, int base) {
    float result = 0.0;
    float f = 1.0 / float(base);
    int i = index;
    while (i > 0) {
        result += f * float(i % base);
        i = i / base;
        f /= float(base);
    }
    return result;
}

// Apply jitter to projection matrix
mat4 applyJitter(mat4 projection, int frameIndex, vec2 screenSize) {
    vec2 jitter;
    jitter.x = halton(frameIndex % 16, 2) - 0.5; // Range [-0.5, 0.5]
    jitter.y = halton(frameIndex % 16, 3) - 0.5;
    
    // Convert pixel offset to NDC offset
    jitter.x *= 2.0 / screenSize.x; // 1 pixel = 2/width in NDC
    jitter.y *= 2.0 / screenSize.y;
    
    // Add jitter to projection matrix translation
    mat4 jitteredProj = projection;
    jitteredProj[2][0] += jitter.x; // Shift in X
    jitteredProj[2][1] += jitter.y; // Shift in Y
    
    return jitteredProj;
}
```

### 2.3 Motion Vectors — Tracking Where Pixels Moved

```glsl
// In the G-Buffer vertex shader: Calculate per-vertex motion
out vec4 vClipPos;     // Current frame clip position
out vec4 vPrevClipPos; // Previous frame clip position

void main() {
    vClipPos     = uViewProj      * uModel     * vec4(aPosition, 1.0);
    vPrevClipPos = uPrevViewProj  * uPrevModel * vec4(aPosition, 1.0);
    gl_Position  = vClipPos;
}

// In the G-Buffer fragment shader: Output motion vectors
layout (location = 3) out vec2 gMotionVector;

void main() {
    // Current and previous screen positions
    vec2 currentPos = (vClipPos.xy / vClipPos.w) * 0.5 + 0.5;
    vec2 prevPos    = (vPrevClipPos.xy / vPrevClipPos.w) * 0.5 + 0.5;
    
    gMotionVector = currentPos - prevPos; // Screen-space velocity
}
```

### 2.4 The TAA Resolve Shader

```glsl
uniform sampler2D currentFrame;
uniform sampler2D historyFrame;
uniform sampler2D motionVectors;
uniform sampler2D depthBuffer;

out vec4 fragColor;

void main() {
    vec2 uv = gl_FragCoord.xy / screenSize;
    
    // Read motion vector (where was this pixel last frame?)
    vec2 motion = texture(motionVectors, uv).rg;
    vec2 historyUV = uv - motion; // Reproject to previous position
    
    // Sample current frame and reprojected history
    vec3 current = texture(currentFrame, uv).rgb;
    vec3 history = texture(historyFrame, historyUV).rgb;
    
    // ─── NEIGHBORHOOD CLAMPING (prevents ghosting) ───
    // Sample the 3×3 neighborhood of the CURRENT frame
    vec3 neighborMin = vec3(999.0);
    vec3 neighborMax = vec3(-999.0);
    for (int y = -1; y <= 1; y++) {
        for (int x = -1; x <= 1; x++) {
            vec3 neighbor = texture(currentFrame, uv + vec2(x, y) / screenSize).rgb;
            neighborMin = min(neighborMin, neighbor);
            neighborMax = max(neighborMax, neighbor);
        }
    }
    
    // CLAMP history to the current frame's color range
    // If history contains a color that doesn't exist in the current neighborhood,
    // it's stale data (ghosting). Force it into a valid range.
    history = clamp(history, neighborMin, neighborMax);
    
    // ─── ADAPTIVE BLEND FACTOR ───
    // Static pixels: heavy history usage (95%) → very stable, no jitter noise
    // Moving pixels: reduced history (50%) → responsive, less blur
    float speed = length(motion) * screenSize.x;
    float blendFactor = mix(0.95, 0.5, clamp(speed * 10.0, 0.0, 1.0));
    
    // ─── FINAL BLEND ───
    vec3 result = mix(current, history, blendFactor);
    
    fragColor = vec4(result, 1.0);
}
```

---

## 3. TAA Artifacts — Root Causes and Solutions

| Artifact | Visual | Root Cause | Solution |
|----------|--------|-----------|---------|
| **Ghosting** | Trails behind moving objects | History pixel is stale (shows old position) | Neighborhood clamping/clipping of history |
| **Blurring** | Image looks soft compared to no-AA | Over-reliance on history, insufficient sharpening | Add a sharpening pass after TAA (CAS) |
| **Jitter shimmer** | Fine details vibrate/crawl | Halton jitter visible when history is rejected | Use Catmull-Rom filtering for history sample |
| **Disocclusion** | Bright/dark flashing at object edges | Pixel was occluded last frame (no valid history) | Detect via depth comparison; use more current weight |
| **Thin geometry disappears** | Wire fences, antennas vanish | Geometry thinner than 1 pixel missed entirely | This is actually FIXED by TAA jitter over time! |

---

## 4. GPU Cost Analysis

| TAA Component | Texture Reads | Cost (1080p) |
|--------------|---------------|--------------|
| Motion vector read | 1 | ~0.05ms |
| Current frame read | 1 | ~0.05ms |
| History read (bilinear) | 1 | ~0.05ms |
| 3×3 neighborhood (clamping) | 9 | ~0.15ms |
| Depth comparison (disocclusion) | 2 | ~0.05ms |
| Write output + update history | 2 | ~0.1ms |
| **TOTAL** | **16** | **~0.5ms** |

**TAA vs MSAA bandwidth comparison at 4K:**
```
MSAA 4×: 4 × 32MB (color) + 4 × 32MB (depth) = 256MB extra bandwidth
TAA:     1 history texture (32MB) + motion vectors (16MB) = 48MB
TAA savings: 80% less memory bandwidth than MSAA 4×
```

---

## 5. Debugging TAA

```glsl
// Debug: Visualize motion vectors as color
vec3 debugMotionVectors(vec2 motion) {
    return vec3(abs(motion.x) * 100.0,  // Red = horizontal motion
                abs(motion.y) * 100.0,  // Green = vertical motion
                0.0);
}

// Debug: Show where history is being rejected (clamped)
vec3 debugHistoryRejection(vec3 history, vec3 clampedHistory) {
    float rejection = length(history - clampedHistory);
    return vec3(rejection * 10.0, 0.0, 0.0); // Bright red = heavy rejection
}
```
