# Chapter 16 — WebGPU Pipelines & Bind Groups

---

## 1. Uniform Buffers (Passing Data to Shaders)

```javascript
// Create a uniform buffer for MVP matrix (64 bytes = 16 floats × 4 bytes)
const uniformBuffer = device.createBuffer({
    size: 64,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST
});

// Update every frame
device.queue.writeBuffer(uniformBuffer, 0, mvpMatrix);
```

---

## 2. Bind Groups (Resource Binding System)

WebGPU bundles shader resources into **Bind Groups**. Each group contains textures, samplers, uniform buffers, or storage buffers.

```javascript
const bindGroup = device.createBindGroup({
    layout: pipeline.getBindGroupLayout(0), // Group 0
    entries: [
        {
            binding: 0,
            resource: { buffer: uniformBuffer } // MVP matrix
        },
        {
            binding: 1,
            resource: texture.createView()      // Diffuse texture
        },
        {
            binding: 2,
            resource: sampler                    // Texture sampler
        }
    ]
});

// During rendering
renderPass.setBindGroup(0, bindGroup);
```

### Organization Strategy
```
BIND GROUP ARCHITECTURE (production):
═════════════════════════════════════
  Group 0: Per-FRAME data (changes once per frame)
    - Camera matrices (View, Projection)
    - Time, resolution
    
  Group 1: Per-MATERIAL data (changes per material switch)
    - Albedo texture, Normal map, PBR parameters
    - Material-specific uniforms
    
  Group 2: Per-OBJECT data (changes per draw call)
    - Model matrix
    - Object-specific uniforms

  WHY THIS MATTERS:
  Changing a bind group is MUCH cheaper than recreating one.
  Group 0 changes 1×/frame, Group 1 changes ~20/frame, Group 2 changes ~500×/frame.
  By separating them, the GPU only rebinds what actually changed.
```

---

## 3. Storage Buffers (Read/Write from Shaders)

| | Uniform Buffer | Storage Buffer |
|--|---------------|----------------|
| Max size | 64KB (16K floats) | Gigabytes |
| Access | Read-only in shader | Read AND Write |
| Speed | Fastest (cached) | Slightly slower |
| Use case | MVP matrix, light data | Particle positions, compute results |

```wgsl
// Storage buffer for 100,000 particles
struct Particle {
    position: vec3f,
    velocity: vec3f,
}

@group(0) @binding(0)
var<storage, read_write> particles: array<Particle>;
```

---

## 4. Texture Binding

```javascript
// Create a texture
const texture = device.createTexture({
    size: [256, 256],
    format: 'rgba8unorm',
    usage: GPUTextureUsage.TEXTURE_BINDING | GPUTextureUsage.COPY_DST
});

// Create a sampler
const sampler = device.createSampler({
    magFilter: 'linear',
    minFilter: 'linear',
    mipmapFilter: 'linear',
    addressModeU: 'repeat',
    addressModeV: 'repeat'
});
```

```wgsl
// In WGSL shader
@group(1) @binding(0) var diffuseTexture: texture_2d<f32>;
@group(1) @binding(1) var diffuseSampler: sampler;

@fragment
fn main(@location(0) uv: vec2f) -> @location(0) vec4f {
    return textureSample(diffuseTexture, diffuseSampler, uv);
}
```

---

## 5. Performance: WebGPU vs WebGL Overhead

```
DRAW CALL OVERHEAD COMPARISON (CPU-side cost per draw):
═══════════════════════════════════════════════════════
  WebGL:  ~0.1ms per draw call (driver validates everything)
  WebGPU: ~0.01ms per draw call (validation done at pipeline creation)
  
  At 1000 draw calls:
    WebGL:  100ms CPU overhead → only 60fps possible
    WebGPU: 10ms CPU overhead  → plenty of headroom!
    
  WHY: WebGPU does validation ONCE when creating the pipeline.
       WebGL re-validates every `gl.draw*()` call.
```
