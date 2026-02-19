# Module 5 — AI-Augmented Rendering Systems

> This module bridges classical rendering with machine learning,
> covering the architecture of systems like DLSS, NeRFs, and learned denoisers.

---

## 1. AI Upscaling — Why Native Resolution Is Dead

### 1.1 The Cost/Quality Matrix

```
RENDERING COSTS AT DIFFERENT RESOLUTIONS (RTX 4070, Deferred PBR):
══════════════════════════════════════════════════════════════════
  Resolution    Pixels       Render Time    Upscaled To    Final Quality
  ──────────    ──────       ───────────    ───────────    ─────────────
  2160p (4K)    8,294,400    16.2ms         N/A            Reference
  1440p         3,686,400     7.8ms         4K (1.5×)      ~95% of native
  1080p         2,073,600     4.5ms         4K (2.0×)      ~90% of native
   720p           921,600     2.1ms         4K (3.0×)      ~80% of native
   540p           518,400     1.3ms         4K (4.0×)      ~70% of native

  At 1080p→4K upscaling, you save 11.7ms per frame.
  That's enough to add full RT shadows + RT reflections!
```

### 1.2 DLSS Architecture (Deep Learning Super Sampling)

```
DLSS PIPELINE:
══════════════
  INPUTS (from the engine):
  ┌──────────────────────────────────┐
  │ 1. Low-res Color (1080p)         │ ← Current frame, jittered
  │ 2. Motion Vectors (1080p)        │ ← Per-pixel screen-space velocity
  │ 3. Depth Buffer (1080p)          │ ← Linearized depth
  │ 4. Previous 4K Output (History)  │ ← Last frame's reconstructed result
  └──────────────┬───────────────────┘
                 │
                 ▼
  ┌──────────────────────────────────┐
  │ NVIDIA Tensor Cores              │
  │ (Dedicated matrix multiply HW)   │
  │                                  │
  │ Convolutional Neural Network:    │
  │   Encoder: Extract features      │
  │   Temporal Module: Align history │
  │     via motion vectors            │
  │   Decoder: Generate 4K output    │
  │                                  │
  │ Trained on: Pairs of             │
  │   (low-res renders, 16K ground   │
  │    truth images) on NVIDIA        │
  │   supercomputers for weeks        │
  └──────────────┬───────────────────┘
                 │
                 ▼
  ┌──────────────────────────────────┐
  │ OUTPUT: 4K Reconstructed Frame   │
  │ Contains detail that NEVER       │
  │ existed in the 1080p input!      │
  │ (AI "hallucinated" sub-pixel     │
  │  geometry like thin fences,      │
  │  wire meshes, text, and foliage) │
  └──────────────────────────────────┘
```

### 1.3 Critical Engine Requirements for DLSS/FSR Integration

```
CHECKLIST FOR ML UPSCALER INTEGRATION:
═══════════════════════════════════════
  ✅ Sub-pixel jitter MUST be applied to the projection matrix
     (exactly like TAA — Halton sequence, 8-32 phases).
     If jitter is missing, the AI has no temporal sub-pixel data
     and the output looks like a cheap bilinear upscale.

  ✅ Motion Vectors MUST be written for EVERY moving object.
     Particles, UI elements, vertex-animated meshes — ALL of them.
     Missing motion vectors → ghosting trails on moving objects.

  ✅ A "Reactive Mask" texture must be provided.
     Marks pixels that should NOT use temporal history
     (e.g., particle effects, transparent objects, screen-space effects).
     Without this mask, explosions ghost for 10+ frames.

  ✅ Exposure value must be pre-applied BEFORE passing to the upscaler.
     DLSS expects perceptually linear input, not raw HDR radiance.
     If you pass raw HDR, the network overexposes bright areas.

  ❌ Do NOT apply any post-processing (AA, bloom, chromatic aberration)
     before the upscaler. These destroy the clean signal the AI needs.
     Post-processing runs AFTER upscaling, at native 4K resolution.
```

### 1.4 FSR 2.0 (AMD FidelityFX) — The Open Alternative

FSR does not use neural networks. It uses a purely algorithmic approach:
1. **Lanczos Upscale:** High-quality mathematical filter (not ML).
2. **Temporal Accumulation:** Blends jittered low-res frames over time (like TAA).
3. **Sharpening Pass:** Contrast-adaptive sharpening to restore perceived detail.

**GPU Cost:** ~1.0ms at 4K output. (DLSS: ~0.8ms but requires Tensor Cores).

---

## 2. Neural Radiance Fields (NeRFs) — Deep Architecture

### 2.1 The MLP (Multi-Layer Perceptron) Architecture

```
NERF NETWORK ARCHITECTURE:
══════════════════════════
  Input Layer:
    Position (x, y, z) → Positional Encoding → 60 floats
    Direction (θ, φ)   → Positional Encoding → 24 floats
  
  POSITIONAL ENCODING (critical for sharp details):
    γ(p) = [sin(2⁰πp), cos(2⁰πp), sin(2¹πp), cos(2¹πp), ..., sin(2⁹πp), cos(2⁹πp)]
    
    Without positional encoding, the MLP can only learn smooth,
    blurry functions. The sin/cos frequencies allow it to represent
    sharp edges and high-frequency texture details.
  
  Hidden Layers:
    Layer 1-4:  Linear(60 → 256) + ReLU
    Layer 5:    Skip connection (concatenate original input)
    Layer 5-8:  Linear(256+60 → 256) + ReLU
    
  Output Heads:
    Density Head:    Linear(256 → 1) + Softplus  → σ (opacity per point)
    Color Head:      Linear(256+24 → 128) + ReLU → Linear(128 → 3) + Sigmoid → RGB
    
    Note: Density depends ONLY on position (geometry is view-independent).
          Color depends on position AND direction (view-dependent reflections).
```

### 2.2 Volume Rendering (Ray Marching Through the NeRF)

```glsl
// Pseudocode: NeRF Volume Rendering
vec3 renderNeRFRay(vec3 rayOrigin, vec3 rayDir, float tNear, float tFar) {
    int numSamples = 64;
    float stepSize = (tFar - tNear) / float(numSamples);
    
    vec3 accumulatedColor = vec3(0.0);
    float accumulatedTransmittance = 1.0; // Starts fully transparent
    
    for (int i = 0; i < numSamples; i++) {
        float t = tNear + (float(i) + random()) * stepSize; // Stratified sampling
        vec3 samplePos = rayOrigin + t * rayDir;
        
        // Query the neural network at this 3D point
        NerfOutput output = queryMLP(samplePos, rayDir);
        float sigma = output.density;
        vec3 color = output.rgb;
        
        // Beer-Lambert law: how much light is blocked by this sample
        float alpha = 1.0 - exp(-sigma * stepSize);
        
        // Front-to-back alpha compositing
        accumulatedColor += accumulatedTransmittance * alpha * color;
        accumulatedTransmittance *= (1.0 - alpha);
        
        // Early termination: if nearly opaque, stop
        if (accumulatedTransmittance < 0.001) break;
    }
    
    return accumulatedColor;
}
```

### 2.3 Instant-NGP — Making NeRFs Real-Time

The original NeRF queries an MLP 64 times per ray (64 × 8M pixels = 512M network evaluations!). Instant-NGP (NVIDIA, 2022) replaces most of the MLP with a **Multi-Resolution Hash Grid**:

```
INSTANT-NGP ARCHITECTURE:
═════════════════════════
  Instead of 8 layers of 256-wide neurons, use:
  
  1. Hash Grid: 16 levels of resolution (coarse to fine).
     Each level is a 3D grid of trainable feature vectors.
     The 3D position is hashed to look up features at each level.
     
  2. Tiny MLP: Only 2 layers × 64 neurons!
     Input: Concatenated features from all 16 hash levels.
     Output: Density + Color.
  
  LOOKUP: O(1) hash table query (no matrix multiply!)
  TINY MLP: 2×64 = 128 neurons (vs 8×256 = 2048 in original NeRF)
  
  RESULT: Training in 5 seconds. Rendering at 30fps.
```

---

## 3. 3D Gaussian Splatting — Full Rendering Pipeline

### 3.1 Gaussian Definition

Each 3D Gaussian is defined by:
- **Mean (μ):** Center position in 3D world space (3 floats).
- **Covariance (Σ):** A 3×3 symmetric positive-definite matrix defining the ellipsoid shape (6 unique floats).
- **Opacity (α):** How opaque this Gaussian is (1 float).
- **Color (SH):** Spherical Harmonic coefficients for view-dependent color (48 floats for degree-3 SH).

### 3.2 The Complete Splatting Algorithm

```
3D GAUSSIAN SPLATTING RENDERING PIPELINE:
═════════════════════════════════════════
  STEP 1: FRUSTUM CULLING (Compute Shader)
    For each of the ~3 million Gaussians:
      Project μ to screen space using the View-Projection matrix.
      If outside the screen frustum → discard.
    Surviving: ~500,000 Gaussians.

  STEP 2: SORT BY DEPTH (GPU Radix Sort)
    Sort all surviving Gaussians by their screen-space depth (Z).
    This is CRITICAL for correct alpha blending (back-to-front).
    
    GPU Radix Sort of 500K elements: ~0.5ms
    (Uses prefix scan + scatter, highly parallelizable)

  STEP 3: PROJECT COVARIANCE TO 2D (Compute Shader)
    Transform the 3D covariance matrix Σ to a 2D covariance matrix Σ':
    
    J = Jacobian of the projection at point μ (2×3 matrix)
    W = upper-left 3×3 of the View matrix
    Σ' = J × W × Σ × Wᵀ × Jᵀ   (result: 2×2 matrix = screen-space ellipse)
    
    Eigenvalue decomposition of Σ' gives the ellipse major/minor axes.

  STEP 4: RASTERIZE 2D ELLIPSES (Fragment Shader)
    For each sorted Gaussian, draw a screen-aligned quad covering
    the ellipse's bounding box.
    
    In the fragment shader, evaluate the 2D Gaussian function:
    
    float G = exp(-0.5 * dot(offset, inverse(Sigma2D) * offset));
    float alpha = opacity * G;
    
    // Standard front-to-back alpha blending
    fragColor.rgb += accumulatedTransmittance * alpha * color;
    accumulatedTransmittance *= (1.0 - alpha);

  TOTAL COST: ~3ms at 1080p for 3 million Gaussians (RTX 3070).
```

---

## 4. Learned Denoisers — Replacing Handcrafted Filters

### 4.1 Architecture: The U-Net Auto-Encoder

```
U-NET DENOISER ARCHITECTURE:
═════════════════════════════
  INPUT: Noisy 1spp image (1920×1080×3) + Normals + Depth
         Concatenated: 1920×1080×8 channels
  
  ENCODER (Downsample):
    Conv 3×3, 64 filters, stride 2 → 960×540×64
    Conv 3×3, 128 filters, stride 2 → 480×270×128
    Conv 3×3, 256 filters, stride 2 → 240×135×256
    Conv 3×3, 512 filters, stride 2 → 120×68×512   ← Bottleneck
    
  DECODER (Upsample with Skip Connections):
    TransConv 3×3, 256 filters, stride 2 → 240×135×256
    + SKIP CONNECTION from encoder level 3 → 240×135×512
    TransConv 3×3, 128 filters → 480×270×128
    + SKIP CONNECTION from encoder level 2 → 480×270×256
    TransConv 3×3, 64 filters → 960×540×64
    + SKIP CONNECTION from encoder level 1 → 960×540×128
    TransConv 3×3, 3 filters → 1920×1080×3   ← Clean output!
    
  PARAMETER COUNT: ~8 million weights
  INFERENCE COST: ~2ms on Tensor Cores (RTX 3070)
  
  VS. SVGF (handcrafted): ~1.5ms but with boiling/ghosting artifacts
  VS. U-Net (learned):    ~2.0ms but nearly artifact-free
```

### 4.2 Running Inference in WebGPU (Experimental)

```javascript
// Extremely simplified: Running a Conv2D layer in a WebGPU Compute Shader
// Real implementations use libraries like ONNX Runtime Web (WebGPU backend)

const computeShaderCode = `
@group(0) @binding(0) var<storage, read> input: array<f32>;
@group(0) @binding(1) var<storage, read> weights: array<f32>;
@group(0) @binding(2) var<storage, read_write> output: array<f32>;

@compute @workgroup_size(8, 8)
fn conv2d(@builtin(global_invocation_id) gid: vec3u) {
    let x = gid.x;
    let y = gid.y;
    let outChannel = gid.z;
    
    var sum: f32 = 0.0;
    
    // 3×3 convolution kernel
    for (var ky: i32 = -1; ky <= 1; ky++) {
        for (var kx: i32 = -1; kx <= 1; kx++) {
            for (var inC: u32 = 0; inC < inputChannels; inC++) {
                let px = i32(x) + kx;
                let py = i32(y) + ky;
                
                let inputVal = input[py * width * inputChannels + px * inputChannels + inC];
                let weightIdx = outChannel * 9 * inputChannels + 
                                (ky+1) * 3 * inputChannels + 
                                (kx+1) * inputChannels + inC;
                let weightVal = weights[weightIdx];
                
                sum += inputVal * weightVal;
            }
        }
    }
    
    // ReLU activation
    output[y * width * outputChannels + x * outputChannels + outChannel] = max(sum, 0.0);
}
`;
```

---

## 5. Hybrid Neural + Raster Pipelines — The Future

The cutting-edge approach is not "ML replaces rendering." It is **"ML fills the gaps that rasterization cannot."**

```
HYBRID PIPELINE ARCHITECTURE:
═════════════════════════════
  ┌─────────────────────┐
  │ RASTERIZATION        │  Primary visibility, G-Buffer, opaque geometry
  │ (Fast, deterministic)│  Cost: ~4ms
  └──────────┬──────────┘
             │
  ┌──────────▼──────────┐
  │ RAY TRACING          │  Shadows, reflections (stochastic, 1spp)
  │ (Physically correct) │  Cost: ~4ms
  └──────────┬──────────┘
             │
  ┌──────────▼──────────┐
  │ NEURAL DENOISER      │  Clean the 1spp noise
  │ (Learned, adaptive)  │  Cost: ~2ms
  └──────────┬──────────┘
             │
  ┌──────────▼──────────┐
  │ NEURAL UPSCALER      │  Reconstruct 4K from 1080p
  │ (DLSS / FSR)         │  Cost: ~1ms
  └──────────┬──────────┘
             │
  ┌──────────▼──────────┐
  │ POST-PROCESSING      │  Bloom, Color Grading, Gamma
  │ (Classical math)     │  Cost: ~1ms
  └──────────┬──────────┘
             │
             ▼
        FINAL 4K OUTPUT
        Total: ~12ms = 83fps at 4K with full RT!
```
