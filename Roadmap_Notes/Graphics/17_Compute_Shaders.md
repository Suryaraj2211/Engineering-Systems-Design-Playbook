# Chapter 17 — Compute Shaders

---

## 1. What Are Compute Shaders?

Compute shaders are general-purpose GPU programs that are NOT tied to the graphics pipeline. They don't draw triangles — they process DATA.

```
RENDER SHADER:    Input: Vertices → Output: Pixels (goes to screen)
COMPUTE SHADER:   Input: ANY data → Output: ANY data (stays in buffers)

USE CASES:
  - Physics simulation (100,000 particles)
  - Image processing (blur, edge detection)
  - Sorting (GPU radix sort)
  - AI inference (matrix multiplication)
  - Frustum culling (determine which objects are visible)
```

---

## 2. Workgroups — How Compute Dispatches Work

```
COMPUTE DISPATCH MODEL:
═══════════════════════
  You dispatch a GRID of workgroups.
  Each workgroup contains a fixed number of threads (invocations).
  
  dispatch(4, 4, 1)  →  4 × 4 × 1 = 16 workgroups
  @workgroup_size(8, 8, 1) → each workgroup has 8 × 8 = 64 threads
  
  TOTAL: 16 workgroups × 64 threads = 1024 compute shader invocations
  
  WORKGROUP GRID:
  ┌────┐┌────┐┌────┐┌────┐
  │WG00││WG10││WG20││WG30│
  └────┘└────┘└────┘└────┘
  ┌────┐┌────┐┌────┐┌────┐
  │WG01││WG11││WG21││WG31│
  └────┘└────┘└────┘└────┘
  ┌────┐┌────┐┌────┐┌────┐
  │WG02││WG12││WG22││WG32│
  └────┘└────┘└────┘└────┘
  ┌────┐┌────┐┌────┐┌────┐
  │WG03││WG13││WG23││WG33│
  └────┘└────┘└────┘└────┘
  
  Inside WG00 (8×8 threads):
  Each thread knows its @builtin(global_invocation_id) = unique (x, y, z)
```

---

## 3. Example: Particle Physics Simulation

```wgsl
struct Particle {
    pos: vec3f,
    vel: vec3f,
}

@group(0) @binding(0) var<storage, read_write> particles: array<Particle>;
@group(0) @binding(1) var<uniform> dt: f32; // Delta time

@compute @workgroup_size(64)
fn main(@builtin(global_invocation_id) id: vec3u) {
    let i = id.x;
    if (i >= arrayLength(&particles)) { return; }
    
    // Apply gravity
    particles[i].vel.y -= 9.81 * dt;
    
    // Update position
    particles[i].pos += particles[i].vel * dt;
    
    // Bounce off ground
    if (particles[i].pos.y < 0.0) {
        particles[i].pos.y = 0.0;
        particles[i].vel.y *= -0.8; // Lose 20% energy on bounce
    }
}
```

```javascript
// Dispatch compute
const commandEncoder = device.createCommandEncoder();
const computePass = commandEncoder.beginComputePass();
computePass.setPipeline(computePipeline);
computePass.setBindGroup(0, computeBindGroup);

// Dispatch enough workgroups to cover all particles
const numWorkgroups = Math.ceil(particleCount / 64);
computePass.dispatchWorkgroups(numWorkgroups);
computePass.end();

device.queue.submit([commandEncoder.finish()]);
```

---

## 4. Workgroup Shared Memory

Threads within a workgroup can share a fast local memory:

```wgsl
var<workgroup> sharedData: array<f32, 64>;

@compute @workgroup_size(64)
fn main(@builtin(local_invocation_id) localId: vec3u) {
    // Each thread loads one element into shared memory
    sharedData[localId.x] = particles[globalId.x].pos.y;
    
    workgroupBarrier(); // WAIT for ALL threads to finish loading
    
    // Now all 64 values are available to every thread
    // Example: find the minimum Y in this workgroup
    // (parallel reduction)
}
```

---

## 5. GPU Synchronization

```
SYNCHRONIZATION RULES:
══════════════════════
  Between threads in SAME workgroup:
    Use workgroupBarrier() — guarantees all threads reach this point.
    
  Between DIFFERENT workgroups:
    NO SYNCHRONIZATION POSSIBLE within a single dispatch!
    Workgroups execute in UNDEFINED order.
    
    Solution: Use multiple dispatches.
    Dispatch 1: Write results to a buffer.
    Dispatch 2: Read that buffer after the first dispatch completes.
    
  Between compute and render:
    Submit compute commands BEFORE render commands in the command queue.
    WebGPU guarantees sequential execution within a queue.
```

---

## 6. Choosing Workgroup Size

| Workgroup Size | Total Threads | Best For |
|---------------|---------------|----------|
| (64, 1, 1) | 64 | 1D arrays (particles, sorting) |
| (8, 8, 1) | 64 | 2D images (post-processing) |
| (4, 4, 4) | 64 | 3D volumes (voxels, fluid sim) |
| (256, 1, 1) | 256 | Maximum occupancy on some GPUs |

**Rule of thumb:** Use a multiple of 32 (NVIDIA warp size) or 64 (AMD wavefront size). 64 is safe for both.
