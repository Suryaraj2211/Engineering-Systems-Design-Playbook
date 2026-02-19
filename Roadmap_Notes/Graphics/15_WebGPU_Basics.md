# Chapter 15 — WebGPU Basics

---

## 1. What Is WebGPU?

WebGPU is the successor to WebGL. It provides a modern, low-level GPU API based on the same design as Vulkan, Metal, and DirectX 12.

```
WEBGL vs WEBGPU:
════════════════
  Feature                WebGL                    WebGPU
  ───────                ─────                    ─────
  API Model              OpenGL (1992 design)     Vulkan/Metal (2016 design)
  State Management       Global state machine     Explicit pipeline objects
  Shader Language         GLSL (OpenGL SL)         WGSL (WebGPU SL)
  Compute Shaders         No (extensions only)     YES (first-class!)
  Command Buffers         Immediate mode           Pre-recorded command buffers
  Validation              Driver validates         Browser validates upfront
  Multi-threaded          No                       Yes (workers)
  Performance             Good                     Better (less overhead)
```

---

## 2. Initialization

```javascript
// STEP 1: Request adapter (physical GPU)
const adapter = await navigator.gpu.requestAdapter();
if (!adapter) { alert('WebGPU not supported'); return; }

// STEP 2: Request device (logical GPU connection)
const device = await adapter.requestDevice();

// STEP 3: Configure canvas
const canvas = document.getElementById('canvas');
const context = canvas.getContext('webgpu');
const format = navigator.gpu.getPreferredCanvasFormat(); // 'bgra8unorm' typically

context.configure({
    device: device,
    format: format,
    alphaMode: 'premultiplied'
});
```

### What's Different from WebGL?
```
WEBGL initialization:
  const gl = canvas.getContext('webgl2');
  // Done! GL is ready to use immediately.

WEBGPU initialization:
  1. requestAdapter() → queries available GPUs (integrated vs discrete)
  2. requestDevice()  → creates a logical connection (like opening a file)
  3. configure()      → sets up the swap chain (presentation surface)
  
  WHY SO MANY STEPS?
  Because WebGPU lets you choose WHICH GPU to use,
  request specific features/limits, and handle errors gracefully.
  WebGL just gives you "whatever the browser picks."
```

---

## 3. The Render Pipeline Object

In WebGL, GPU state is set globally. In WebGPU, ALL state is baked into an immutable Pipeline object at initialization time.

```javascript
const pipeline = device.createRenderPipeline({
    layout: 'auto',
    vertex: {
        module: device.createShaderModule({ code: vertexWGSL }),
        entryPoint: 'main',
        buffers: [{
            arrayStride: 5 * 4, // 5 floats × 4 bytes
            attributes: [
                { shaderLocation: 0, offset: 0,  format: 'float32x2' }, // position
                { shaderLocation: 1, offset: 8,  format: 'float32x3' }  // color
            ]
        }]
    },
    fragment: {
        module: device.createShaderModule({ code: fragmentWGSL }),
        entryPoint: 'main',
        targets: [{ format: format }]
    },
    primitive: {
        topology: 'triangle-list',
        cullMode: 'back'
    },
    depthStencil: {
        format: 'depth24plus',
        depthWriteEnabled: true,
        depthCompare: 'less'
    }
});
```

---

## 4. Drawing a Triangle in WebGPU

```javascript
// Vertex buffer
const vertexBuffer = device.createBuffer({
    size: vertices.byteLength,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST
});
device.queue.writeBuffer(vertexBuffer, 0, vertices);

// Render loop
function frame() {
    const commandEncoder = device.createCommandEncoder();
    
    const renderPass = commandEncoder.beginRenderPass({
        colorAttachments: [{
            view: context.getCurrentTexture().createView(),
            clearValue: { r: 0.1, g: 0.1, b: 0.15, a: 1.0 },
            loadOp: 'clear',
            storeOp: 'store'
        }],
        depthStencilAttachment: {
            view: depthTexture.createView(),
            depthClearValue: 1.0,
            depthLoadOp: 'clear',
            depthStoreOp: 'store'
        }
    });
    
    renderPass.setPipeline(pipeline);
    renderPass.setVertexBuffer(0, vertexBuffer);
    renderPass.draw(3); // 3 vertices
    renderPass.end();
    
    device.queue.submit([commandEncoder.finish()]);
    requestAnimationFrame(frame);
}
requestAnimationFrame(frame);
```

---

## 5. WGSL Shaders

```wgsl
// VERTEX SHADER
struct VertexOutput {
    @builtin(position) position: vec4f,
    @location(0) color: vec3f,
};

@vertex
fn main(
    @location(0) pos: vec2f,
    @location(1) color: vec3f
) -> VertexOutput {
    var out: VertexOutput;
    out.position = vec4f(pos, 0.0, 1.0);
    out.color = color;
    return out;
}

// FRAGMENT SHADER
@fragment
fn main(@location(0) color: vec3f) -> @location(0) vec4f {
    return vec4f(color, 1.0);
}
```

### GLSL vs WGSL Quick Reference
```
GLSL                          WGSL
────                          ────
vec3 v;                       var v: vec3f;
gl_Position = ...;            out.position = ...;
gl_FragColor = ...;           return vec4f(...);
attribute vec3 aPos;          @location(0) pos: vec3f
uniform mat4 uMVP;            @group(0) @binding(0) var<uniform> mvp: mat4x4f;
texture(sampler, uv)          textureSample(tex, samp, uv)
```

---

## 6. Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Missing `GPUBufferUsage.COPY_DST` | `writeBuffer` fails | Add `COPY_DST` to usage flags |
| Wrong `arrayStride` | Garbled vertices | stride = total bytes per vertex |
| `loadOp: 'load'` on first frame | Random garbage on screen | Use `'clear'` for initial pass |
| Not creating depth texture | Objects overlap randomly | Create a depth24plus texture and attach it |
