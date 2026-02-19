# Module 8 — HDR Rendering & Post-Processing

> This module covers the physics of high dynamic range light,
> the complete post-processing pipeline, and the gamma correction misconception.

---

## 1. What is HDR? — The Dynamic Range Problem

### 1.1 The Numbers

```
REAL-WORLD LUMINANCE VALUES:
════════════════════════════
  Scene                    Luminance (cd/m²)    Relative to 1.0
  ──────                   ──────────────────    ───────────────
  Starlight                0.001                 0.00001
  Moonlight                0.01                  0.0001
  Indoor room              100                   1.0
  Overcast sky              1,000                10
  Clear sky                10,000               100
  Direct sunlight          100,000             1,000
  Sun surface          1,000,000,000       10,000,000
  
  DYNAMIC RANGE of human eye:    ~14 stops (1:16,000)
  DYNAMIC RANGE of 8-bit screen: 8 stops (1:256)
  DYNAMIC RANGE of HDR10 screen: ~10 stops (1:1024)
  
  PROBLEM: PBR shaders output radiance values like 50,000.0 for a sunlit wall.
           A monitor can only display values 0-255 (8-bit per channel).
           We must COMPRESS the dynamic range to fit.
```

### 1.2 LDR vs HDR Rendering Pipeline

```
LDR PIPELINE (old/wrong):
═════════════════════════
  Render → 8-bit RGBA framebuffer (values clamped to 0-1)
  Problem: All values > 1.0 are DESTROYED (clamped to 1.0).
           Sun = white. Sky = white. Clouds = white.
           Everything bright is the same flat white. No detail.

HDR PIPELINE (correct):
═══════════════════════
  Render → 16-bit RGBA16F framebuffer (values from 0.0 to 65,504.0)
  All post-processing happens in HDR (bloom, DoF, motion blur)
  LAST STEP: Tone mapping compresses HDR → [0, 1] → sRGB 8-bit display
```

---

## 2. Linear Color Space — The Foundation

### 2.1 Why Gamma Exists

CRT monitors had a nonlinear voltage-to-brightness response: $brightness = voltage^{2.2}$. To compensate, images were stored "gamma encoded": pixel values are raised to $1/2.2$ before saving. This is the **sRGB** standard.

### 2.2 The Critical Pipeline Rule

```
THE LINEAR WORKFLOW:
════════════════════
  1. READ texture:    sRGB (gamma-encoded) from disk
  2. CONVERT:         to Linear space → pow(color, 2.2)
  3. ALL MATH:        lighting, blending, post-processing in LINEAR
  4. CONVERT:         to sRGB for display → pow(color, 1/2.2)
  5. DISPLAY:         monitor applies its own gamma curve → correct!
  
  If you skip step 2 (do math in sRGB):
    - Blending is wrong (50% blend of two colors gives wrong result)
    - Lighting is wrong (double-gamma: too dark, banding in shadows)
    - PBR math breaks (roughness/Fresnel curves are physically incorrect)
```

```glsl
// Using sRGB texture format (hardware does the conversion automatically):
// In WebGPU: format: 'rgba8unorm-srgb' → hardware linearizes on read
// In WebGL: gl.SRGB8_ALPHA8 → same

// Manual conversion (if not using sRGB texture format):
vec3 sRGBtoLinear(vec3 srgb) {
    return pow(srgb, vec3(2.2));
}

vec3 linearToSRGB(vec3 linear) {
    return pow(linear, vec3(1.0 / 2.2));
}
```

---

## 3. Tone Mapping — Compressing HDR for Display

### 3.1 The Problem

Fragment shader outputs: `vec3(45000.0, 22000.0, 8000.0)` (bright sunny scene).
Monitor needs values in `[0.0, 1.0]`.
Simple clamping (`min(color, 1.0)`) destroys all detail above 1.0.

### 3.2 Tone Mapping Operators (Comparison)

```glsl
// REINHARD (Simple)
// Maps [0, ∞) to [0, 1)
// Problem: Never actually reaches 1.0 (bright areas slightly dim)
vec3 tonemapReinhard(vec3 color) {
    return color / (color + vec3(1.0));
}

// REINHARD (Extended) — Adds a white point
vec3 tonemapReinhardExtended(vec3 color, float whitePoint) {
    vec3 numerator = color * (1.0 + color / (whitePoint * whitePoint));
    return numerator / (1.0 + color);
}

// ACES (Academy Color Encoding System) — Industry standard
// Used by Unreal Engine, Unity, and Hollywood
vec3 tonemapACES(vec3 color) {
    float a = 2.51;
    float b = 0.03;
    float c = 2.43;
    float d = 0.59;
    float e = 0.14;
    return clamp((color * (a * color + b)) / (color * (c * color + d) + e), 0.0, 1.0);
}

// UNCHARTED 2 (Filmic)
vec3 uncharted2Tonemap(vec3 x) {
    float A = 0.15, B = 0.50, C = 0.10, D = 0.20, E = 0.02, F = 0.30;
    return ((x * (A * x + C * B) + D * E) / (x * (A * x + B) + D * F)) - E / F;
}
```

### 3.3 Visual Comparison

```
TONE MAPPING COMPARISON (input value → output):
═══════════════════════════════════════════════
  Input       Clamp      Reinhard    ACES
  ─────       ─────      ────────    ────
  0.00        0.00       0.00        0.00
  0.18        0.18       0.15        0.05    ← mid-grey (key value)
  0.50        0.50       0.33        0.16
  1.00        1.00       0.50        0.30
  2.00        1.00       0.67        0.48    ← Clamp loses EVERYTHING above 1.0
  5.00        1.00       0.83        0.71
  10.00       1.00       0.91        0.82
  100.00      1.00       0.99        0.96
  50000.00    1.00       ~1.00       ~1.00
  
  ACES produces the most film-like S-curve with rich contrast.
  Reinhard is simpler but "washes out" bright scenes.
  Clamp is NEVER acceptable for PBR (destroys highlight detail).
```

---

## 4. Exposure Control — Simulating the Human Eye

### 4.1 Automatic Exposure (Eye Adaptation)

The human eye adapts to brightness over time. Walking from a bright outdoor scene into a dark cave should cause temporary blindness, then gradual adaptation.

```glsl
// Step 1: Calculate average scene luminance (compute shader)
// Downsample the HDR image to 1×1 pixel to get average brightness
float calculateAverageLuminance(sampler2D hdrImage) {
    // Use a chain of downsamples (mipmap-like)
    // Start from full resolution → 1/2 → 1/4 → ... → 1×1
    // At each level, average the luminance of 4 texels
    float avgLum = textureLod(hdrImage, vec2(0.5), maxMipLevel).r;
    return exp(avgLum); // If using log-average luminance
}

// Step 2: Adapt exposure over time (smooth transition)
float adaptExposure(float currentLuminance, float previousExposure, float deltaTime) {
    float targetExposure = 0.18 / currentLuminance; // 0.18 = middle grey
    
    // Smooth adaptation (eyes adapt slower going dark→bright than bright→dark)
    float adaptSpeed = currentLuminance > previousExposure ? 2.0 : 1.0;
    return mix(previousExposure, targetExposure, 1.0 - exp(-deltaTime * adaptSpeed));
}

// Step 3: Apply exposure before tone mapping
vec3 exposedColor = hdrColor * exposure;
vec3 tonemapped = tonemapACES(exposedColor);
vec3 finalColor = linearToSRGB(tonemapped);
```

---

## 5. Bloom Pipeline — The Complete Implementation

### 5.1 Why Bloom Exists

In real cameras and human eyes, extremely bright light sources "bleed" into surrounding pixels due to lens diffraction and scattering in the vitreous humor. Without bloom, bright lights look flat.

### 5.2 The Pipeline

```
BLOOM PIPELINE:
═══════════════
  STEP 1: BRIGHT PASS (Threshold Extract)
    For each HDR pixel, if luminance > threshold (e.g., 1.0):
      output = pixel (contribute to bloom)
    else:
      output = black (below threshold, no bloom)

  STEP 2: PROGRESSIVE DOWNSAMPLE (Create mip chain)
    Full-res (1920×1080) → 1/2 (960×540) → 1/4 (480×270) →
    1/8 (240×135) → 1/16 (120×68) → 1/32 (60×34)
    
    Each level uses a 2×2 box filter (average 4 texels).
    This creates increasingly blurry versions of the bright pixels.

  STEP 3: PROGRESSIVE UPSAMPLE + BLEND (Reverse the chain)
    1/32 → upsample + add to 1/16 → upsample + add to 1/8 →
    → upsample + add to 1/4 → → → add to Full-res
    
    The UPSAMPLE uses a 3×3 tent filter for smooth blending.

  STEP 4: COMPOSITE
    finalColor = hdrScene + bloom * bloomIntensity
```

### 5.3 Bloom Shader Code

```glsl
// BRIGHT PASS (Extract bright pixels)
vec3 brightPass(vec3 hdrColor, float threshold) {
    float brightness = dot(hdrColor, vec3(0.2126, 0.7152, 0.0722));
    float softKnee = 0.5; // Smooth transition around threshold
    float knee = threshold * softKnee;
    float soft = brightness - threshold + knee;
    soft = clamp(soft, 0.0, 2.0 * knee);
    soft = soft * soft / (4.0 * knee + 0.00001);
    float contribution = max(soft, brightness - threshold) / max(brightness, 0.00001);
    return hdrColor * max(contribution, 0.0);
}

// DOWNSAMPLE (13-tap filter for quality)
vec3 downsample(sampler2D src, vec2 uv, vec2 texelSize) {
    // Karis average (for first downsample only) prevents fireflies
    vec3 a = texture(src, uv + texelSize * vec2(-1.0, -1.0)).rgb;
    vec3 b = texture(src, uv + texelSize * vec2( 0.0, -1.0)).rgb;
    vec3 c = texture(src, uv + texelSize * vec2( 1.0, -1.0)).rgb;
    vec3 d = texture(src, uv + texelSize * vec2(-1.0,  0.0)).rgb;
    vec3 e = texture(src, uv).rgb; // Center
    vec3 f = texture(src, uv + texelSize * vec2( 1.0,  0.0)).rgb;
    vec3 g = texture(src, uv + texelSize * vec2(-1.0,  1.0)).rgb;
    vec3 h = texture(src, uv + texelSize * vec2( 0.0,  1.0)).rgb;
    vec3 i = texture(src, uv + texelSize * vec2( 1.0,  1.0)).rgb;
    
    // Weighted 13-tap filter
    return e * 0.25 + (b + d + f + h) * 0.125 + (a + c + g + i) * 0.0625;
}

// UPSAMPLE (3×3 tent filter)
vec3 upsample(sampler2D src, vec2 uv, vec2 texelSize) {
    vec3 result = vec3(0.0);
    result += texture(src, uv + texelSize * vec2(-1.0, -1.0)).rgb * 1.0;
    result += texture(src, uv + texelSize * vec2( 0.0, -1.0)).rgb * 2.0;
    result += texture(src, uv + texelSize * vec2( 1.0, -1.0)).rgb * 1.0;
    result += texture(src, uv + texelSize * vec2(-1.0,  0.0)).rgb * 2.0;
    result += texture(src, uv).rgb * 4.0;
    result += texture(src, uv + texelSize * vec2( 1.0,  0.0)).rgb * 2.0;
    result += texture(src, uv + texelSize * vec2(-1.0,  1.0)).rgb * 1.0;
    result += texture(src, uv + texelSize * vec2( 0.0,  1.0)).rgb * 2.0;
    result += texture(src, uv + texelSize * vec2( 1.0,  1.0)).rgb * 1.0;
    return result / 16.0;
}
```

---

## 6. GPU Cost & Performance

| Post-Process Pass | Texture Reads/Pixel | Cost (1080p) | Bottleneck |
|-------------------|-------------------|--------------|------------|
| Bright Pass Extract | 1 | ~0.1ms | Bandwidth |
| Bloom Downsample (6 levels) | 13 × 6 = 78 total | ~0.5ms | Bandwidth |
| Bloom Upsample (6 levels) | 9 × 6 = 54 total | ~0.4ms | Bandwidth |
| Tone Mapping (ACES) | 1 | ~0.1ms | ALU |
| Gamma Correction | 1 | ~0.05ms | ALU |
| Total Bloom + Tone | | **~1.15ms** | **Bandwidth** |

---

## 7. Common Mistakes & Debugging

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Tone mapping before bloom | Bloom has no effect (values already <1.0) | Always do bloom in HDR, tone map LAST |
| Double gamma correction | Image extremely dark with crushed shadows | Only apply gamma ONCE at the very end |
| Bloom threshold too low | Entire scene glows (everything blooms) | Set threshold ≥ 1.0 for physically-based |
| Bloom fireflies | Hot white pixels cause localized bright splotches | Use Karis average on first downsample |
| LDR framebuffer (RGBA8) | Bright areas all clamp to (1,1,1) white | Use RGBA16F framebuffer for HDR |
| Mixing sRGB and Linear textures | Colors too bright or too dark (double/no gamma) | Use consistent format; sRGB textures auto-convert |

### Debug Visualization

```glsl
// Visualize HDR values (false-color heatmap)
vec3 debugHDR(vec3 hdrColor) {
    float luminance = dot(hdrColor, vec3(0.2126, 0.7152, 0.0722));
    if (luminance < 0.0)  return vec3(1.0, 0.0, 1.0);  // MAGENTA: negative (error!)
    if (luminance < 0.01) return vec3(0.0, 0.0, 0.5);   // Dark blue: very dark
    if (luminance < 0.18) return vec3(0.0, 0.0, 1.0);   // Blue: shadow
    if (luminance < 1.0)  return vec3(0.0, 1.0, 0.0);   // Green: standard range
    if (luminance < 10.0) return vec3(1.0, 1.0, 0.0);   // Yellow: bright
    if (luminance < 100.0)return vec3(1.0, 0.5, 0.0);   // Orange: very bright
    return vec3(1.0, 0.0, 0.0);                          // Red: extremely bright (sun, fire)
}
```
