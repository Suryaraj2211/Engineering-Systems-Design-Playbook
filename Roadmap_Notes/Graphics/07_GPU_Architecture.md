# Chapter 07 — GPU Architecture

## Why Understand the GPU?
Writing shaders without understanding GPU hardware is like writing SQL without understanding databases. You'll produce code that WORKS but runs 100× slower than necessary.

---

## 1. CPU vs. GPU — Fundamental Architecture Difference

```
CPU (Central Processing Unit):
══════════════════════════════
  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
  │  Core 0  │ │  Core 1  │ │  Core 2  │ │  Core 3  │
  │  (FAST)  │ │  (FAST)  │ │  (FAST)  │ │  (FAST)  │
  │  4 GHz   │ │  4 GHz   │ │  4 GHz   │ │  4 GHz   │
  │ Smart:   │ │ Smart:   │ │ Smart:   │ │ Smart:   │
  │ branches │ │ branches │ │ branches │ │ branches │
  │ predict  │ │ predict  │ │ predict  │ │ predict  │
  └──────────┘ └──────────┘ └──────────┘ └──────────┘
  
  4-16 cores, each extremely fast and versatile.
  Great at: complex logic, branching, serial algorithms.
  
GPU (Graphics Processing Unit):
═══════════════════════════════
  ┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐
  │ C ││ C ││ C ││ C ││ C ││ C ││ C ││ C ││ C ││ C ││ C ││ C │
  └───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘
  ┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐
  │ C ││ C ││ C ││ C ││ C ││ C ││ C ││ C ││ C ││ C ││ C ││ C │
  └───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘└───┘
  ... (hundreds or thousands more)

  1000-10000+ tiny cores, each relatively slow.
  Great at: doing the SAME operation on millions of data points simultaneously.
  
  KEY INSIGHT: A fragment shader runs on EVERY PIXEL SIMULTANEOUSLY.
  At 1080p, that's 2,073,600 copies of your shader running at once!
```

---

## 2. How the GPU Processes Your Shader

### 2.1 SIMT Execution (Single Instruction, Multiple Threads)

```
WARP/WAVEFRONT EXECUTION:
═════════════════════════
  The GPU doesn't run 1 thread at a time like a CPU.
  It runs threads in groups of 32 (NVIDIA: "Warp") or 64 (AMD: "Wavefront").
  
  ALL 32 threads in a Warp execute the SAME instruction at the SAME time.
  
  Warp of 32 pixel shaders:
    Clock 1: ALL 32 threads execute: float d = dot(N, L);
    Clock 2: ALL 32 threads execute: float atten = 1.0 / (d * d);
    Clock 3: ALL 32 threads execute: vec3 color = albedo * atten;
  
  PROBLEM WITH IF-STATEMENTS:
    if (metallic > 0.5) {
        // Path A: 10 instructions
    } else {
        // Path B: 15 instructions
    }
    
    If 20 of 32 threads take Path A and 12 take Path B:
    Clock 1-10: ALL 32 threads run Path A (12 threads sit idle, wasting power)
    Clock 11-25: ALL 32 threads run Path B (20 threads sit idle)
    
    TOTAL: 25 clocks instead of 10 or 15. This is called DIVERGENCE.
    COST: up to 2× slower!
```

---

## 3. GPU Memory Hierarchy

```
GPU MEMORY LAYERS (fastest to slowest):
═══════════════════════════════════════
  ┌─────────────────────────┐
  │ REGISTERS (per thread)   │  ~256KB total per SM
  │ Speed: 0 latency         │  Fastest! Direct access.
  │ Size: ~64 registers      │  Your shader variables live here.
  └──────────┬──────────────┘
             │  (~1 cycle)
  ┌──────────▼──────────────┐
  │ SHARED MEMORY (L1 Cache) │  ~48-128KB per SM
  │ Speed: ~5 cycles          │  Shared between threads in a workgroup.
  │ Used by: compute shaders  │  Explicitly managed in WebGPU.
  └──────────┬──────────────┘
             │  (~30 cycles)
  ┌──────────▼──────────────┐
  │ L2 CACHE                  │  ~4-6MB total
  │ Speed: ~30 cycles         │  Shared across entire GPU.
  │ Automatic caching.        │
  └──────────┬──────────────┘
             │  (~400+ cycles!)
  ┌──────────▼──────────────┐
  │ VRAM (Video Memory)       │  8-24GB (GDDR6/HBM)
  │ Speed: ~400+ cycles       │  Textures, buffers, framebuffers.
  │ Bandwidth: 300-900 GB/s   │  This is where textures are sampled from.
  └─────────────────────────┘

  KEY TAKEAWAY:
  Accessing VRAM is 400× slower than accessing a register.
  Every texture() call in your shader goes to VRAM (cached through L2).
  MINIMIZE TEXTURE READS → fastest possible shader.
```

---

## 4. Texture Sampling — What Really Happens

```
WHEN YOU WRITE: texture(sampler, uv):
═════════════════════════════════════
  1. GPU calculates the EXACT float UV coordinates.
  2. UV is multiplied by texture dimensions to get a texel coordinate.
     uv = (0.5, 0.5) on 512×512 texture → texel (256, 256)
  3. Bilinear filtering: sample the 4 NEAREST texels and interpolate.
     texel (256, 256), (257, 256), (256, 257), (257, 257)
     Each texel read hits the texture cache (or VRAM on cache miss).
  4. If mipmapping is enabled, GPU also interpolates between TWO mip levels
     (trilinear filtering) → 8 texel reads total!
  5. Anisotropic filtering (16×): up to 128 texel reads PER PIXEL!

  COST COMPARISON:
    No texture:           0 VRAM reads       (fastest)
    Nearest filtering:    1 VRAM read         
    Bilinear:             4 VRAM reads        
    Trilinear:            8 VRAM reads        
    Aniso 16×:            up to 128 reads!    (slowest)
```

---

## 5. The Fixed-Function Hardware

Not everything in the pipeline is programmable:

```
WHAT YOU PROGRAM (shaders):          WHAT THE HARDWARE DOES (fixed):
═══════════════════════════          ═══════════════════════════════
  Vertex Shader                       Input Assembly (reading vertices)
  Fragment Shader                     Rasterization (triangle → pixels)
  Compute Shader                      Depth Testing (z-buffer comparison)
                                      Blending (alpha compositing)
                                      MSAA (multi-sample anti-aliasing)

  The fixed-function stages are FREE — they're dedicated silicon.
  Rasterization costs 0ms of shader time (it's hardware).
  Depth testing costs 0ms of shader time (it's hardware).
  
  You PAY for: vertex shader math, fragment shader math, texture reads.
```

---

## 6. Bottleneck Types

| Bottleneck | Meaning | Symptom | Fix |
|-----------|---------|---------|-----|
| **Vertex Bound** | Too many vertices | Low FPS even with simple shaders | Reduce mesh polygon count, use LOD |
| **Fill Rate Bound** | Too many pixels being shaded | Low FPS on high-res, fine on low-res | Reduce resolution, simplify fragment shader |
| **Bandwidth Bound** | Too many texture reads | GPU utilization low, memory bus saturated | Pack textures, reduce resolution, use compression |
| **CPU Bound** | JavaScript too slow | GPU utilization low, CPU at 100% | Reduce draw calls, use instancing, minimize state changes |

---

## 7. Practice Problems

### Problem 1
At 4K resolution (3840×2160), how many fragment shader invocations occur per frame?

**Solution:** 3840 × 2160 = 8,294,400 invocations PER DRAW CALL. With 3× overdraw: ~25 million.

### Problem 2
A shader samples 4 textures per pixel at 1080p. How many texture reads per frame?

**Solution:** 2,073,600 pixels × 4 textures × 4 texels (bilinear) = 33,177,600 texel reads.

### Problem 3
Why is `if (condition) { heavy_computation; }` worse on GPU than CPU?

**Solution:** On CPU, threads skip the branch entirely. On GPU, ALL 32 threads in a warp must execute both paths if even 1 thread takes the other branch. The "skipped" threads sit idle (divergence).
