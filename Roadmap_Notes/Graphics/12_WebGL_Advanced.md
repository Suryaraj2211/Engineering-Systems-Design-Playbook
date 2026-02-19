# Chapter 12 — WebGL Advanced Techniques

---

## 1. Vertex Array Objects (VAOs) — Eliminate Redundant State Setup

Without VAOs, you must re-specify every vertex attribute (position, normal, UV) before each draw call. A VAO records all that configuration once and replays it with a single `bindVertexArray` call.

```javascript
// CREATE and configure VAO (do this ONCE during initialization)
const vao = gl.createVertexArray();
gl.bindVertexArray(vao);

// Configure all vertex attributes while VAO is bound:
gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
gl.enableVertexAttribArray(0);
gl.vertexAttribPointer(0, 3, gl.FLOAT, false, 0, 0); // Position: 3 floats

gl.bindBuffer(gl.ARRAY_BUFFER, normalBuffer);
gl.enableVertexAttribArray(1);
gl.vertexAttribPointer(1, 3, gl.FLOAT, false, 0, 0); // Normal: 3 floats

gl.bindBuffer(gl.ARRAY_BUFFER, uvBuffer);
gl.enableVertexAttribArray(2);
gl.vertexAttribPointer(2, 2, gl.FLOAT, false, 0, 0); // UV: 2 floats

gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, indexBuffer); // Element buffer is also stored in VAO

gl.bindVertexArray(null); // Unbind

// DRAW (do this EVERY FRAME — just bind and draw, no re-configuration)
gl.bindVertexArray(vao);
gl.drawElements(gl.TRIANGLES, indexCount, gl.UNSIGNED_SHORT, 0);
gl.bindVertexArray(null);
```

**Performance Impact:** Without VAO = 6-8 API calls per object per frame. With VAO = 2 calls (bind + draw). For 500 objects, that is 3000-4000 eliminated calls per frame.

---

## 2. Instanced Drawing — 10,000 Objects in One Draw Call

Instancing draws the same mesh multiple times with per-instance data (position, color, scale) packed into a buffer. The GPU reads per-vertex data normally but advances per-instance data once per copy.

```javascript
// Per-instance model matrices (4x4 = 16 floats each)
const instanceData = new Float32Array(instanceCount * 16);
for (let i = 0; i < instanceCount; i++) {
    // Fill instanceData[i*16 ... i*16+15] with each object's model matrix
    mat4.fromTranslation(tempMat, positions[i]);
    instanceData.set(tempMat, i * 16);
}

const instanceBuffer = gl.createBuffer();
gl.bindBuffer(gl.ARRAY_BUFFER, instanceBuffer);
gl.bufferData(gl.ARRAY_BUFFER, instanceData, gl.DYNAMIC_DRAW);

// A mat4 uses 4 attribute slots (each slot = vec4 = 4 floats)
for (let col = 0; col < 4; col++) {
    const loc = gl.getAttribLocation(program, `a_instanceMatrix`) + col;
    gl.enableVertexAttribArray(loc);
    gl.vertexAttribPointer(loc, 4, gl.FLOAT, false, 64, col * 16); // stride=64, offset=col*16
    gl.vertexAttribDivisor(loc, 1); // KEY: Advance once per INSTANCE, not per vertex
}

// One call draws ALL instances
gl.drawElementsInstanced(gl.TRIANGLES, indexCount, gl.UNSIGNED_SHORT, 0, instanceCount);
```

**When to use:** Particles, foliage, crowd rendering, building arrays, stars — any scenario with many copies of the same geometry.

---

## 3. The WebGL State Machine — Complete Reference

WebGL is a global state machine. Every `gl.enable`, `gl.bind`, and `gl.uniform` call modifies a global state. Understanding the order of state changes is critical for avoiding rendering bugs.

### Depth Testing
```javascript
gl.enable(gl.DEPTH_TEST);         // Enable Z-buffer comparison
gl.depthFunc(gl.LEQUAL);          // Pass if fragment depth <= buffer depth
gl.depthMask(true);               // Write to depth buffer (set false for transparent objects)
gl.clearDepth(1.0);               // Clear depth buffer to maximum distance
```

### Face Culling
```javascript
gl.enable(gl.CULL_FACE);          // Skip triangles facing away from camera
gl.cullFace(gl.BACK);             // Cull back faces (default)
gl.frontFace(gl.CCW);             // Counter-clockwise vertex order = front face
// Saves ~50% of rasterization work for closed geometry
```

### Blending (Transparency)
```javascript
gl.enable(gl.BLEND);
gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA); // Standard alpha blending
// Result = src.rgb * src.a + dst.rgb * (1 - src.a)

// CRITICAL: Transparent objects must be drawn BACK-TO-FRONT (painter's algorithm)
// Opaque objects drawn first with depth write ON, then transparent with depth write OFF
gl.depthMask(false); // Disable depth writing for transparent pass
```

### Stencil Buffer
```javascript
gl.enable(gl.STENCIL_TEST);
gl.clearStencil(0);
gl.clear(gl.STENCIL_BUFFER_BIT);

// Step 1: Draw mask shape, write 1 to stencil buffer
gl.stencilFunc(gl.ALWAYS, 1, 0xFF);     // Always pass stencil test
gl.stencilOp(gl.KEEP, gl.KEEP, gl.REPLACE); // Write ref (1) on depth+stencil pass
gl.colorMask(false, false, false, false); // Don't draw to color buffer
drawMaskShape();

// Step 2: Draw scene, only where stencil == 1
gl.stencilFunc(gl.EQUAL, 1, 0xFF);      // Only pass where stencil == 1
gl.stencilOp(gl.KEEP, gl.KEEP, gl.KEEP); // Don't modify stencil
gl.colorMask(true, true, true, true);    // Draw to color buffer
drawScene();
```

**Use cases:** Portals, mirrors, outline effects, masking UI elements.

### Scissor Test
```javascript
gl.enable(gl.SCISSOR_TEST);
gl.scissor(x, y, width, height); // Only render within this rectangle (in pixels)
// Use for: split-screen rendering, UI clipping, debug viewport
```

---

## 4. Multi-Pass Rendering

Complex visual effects require drawing the scene multiple times with different configurations:

```
MULTI-PASS PIPELINE:
════════════════════
  Pass 1: SHADOW MAP
    - Bind shadow FBO (depth-only attachment)
    - Render scene from LIGHT's perspective
    - Output: depth texture (shadow map)
    
  Pass 2: GEOMETRY (G-Buffer for Deferred Rendering)
    - Bind G-Buffer FBO (multiple color attachments)
    - Render scene from CAMERA's perspective
    - Output: albedo texture, normal texture, depth texture
    
  Pass 3: LIGHTING
    - Bind screen FBO
    - For each light: read G-Buffer textures, compute lighting in screen space
    - Output: lit scene
    
  Pass 4: POST-PROCESSING
    - Bind post-process FBO
    - Apply bloom, tone mapping, FXAA as full-screen quad shaders
    - Output: final image to screen
```

---

## 5. Multiple Render Targets (MRT) — WebGL 2

MRT allows a single fragment shader to write to multiple textures simultaneously. This is the foundation of Deferred Rendering.

```javascript
// Create framebuffer with multiple color outputs
const fbo = gl.createFramebuffer();
gl.bindFramebuffer(gl.FRAMEBUFFER, fbo);

// Attach textures to different color attachment points
gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, albedoTex, 0);
gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT1, gl.TEXTURE_2D, normalTex, 0);
gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT2, gl.TEXTURE_2D, positionTex, 0);
gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.DEPTH_ATTACHMENT, gl.TEXTURE_2D, depthTex, 0);

// Tell WebGL to draw to all three color attachments simultaneously
gl.drawBuffers([
    gl.COLOR_ATTACHMENT0,  // layout(location = 0) → Albedo
    gl.COLOR_ATTACHMENT1,  // layout(location = 1) → Normal
    gl.COLOR_ATTACHMENT2   // layout(location = 2) → Position
]);

// Verify framebuffer completeness
if (gl.checkFramebufferStatus(gl.FRAMEBUFFER) !== gl.FRAMEBUFFER_COMPLETE) {
    console.error("Framebuffer incomplete!");
}
```

**Fragment shader writing to MRT:**
```glsl
#version 300 es
precision highp float;

layout(location = 0) out vec4 gAlbedo;
layout(location = 1) out vec4 gNormal;
layout(location = 2) out vec4 gPosition;

in vec3 v_position;
in vec3 v_normal;
in vec2 v_uv;

uniform sampler2D u_albedoMap;

void main() {
    gAlbedo = texture(u_albedoMap, v_uv);
    gNormal = vec4(normalize(v_normal) * 0.5 + 0.5, 1.0); // Pack [-1,1] → [0,1]
    gPosition = vec4(v_position, 1.0);
}
```

---

## 6. Performance Optimization Rules

| Rule | Why | Cost of Violation |
|------|-----|-------------------|
| Minimize draw calls (target < 500/frame) | Each draw triggers GPU state validation | CPU-bound frame drops |
| Sort objects by material before drawing | Reduces shader/texture switches | Unnecessary state changes |
| Use VAOs | Eliminates redundant attribute setup | 4-8 extra API calls per object per frame |
| Use instancing for repeated geometry | 1 draw call vs. N draw calls | N-1 wasted draw call overhead |
| Frustum cull on CPU | Don't submit invisible objects to GPU | Wasted vertex processing |
| Use compressed textures (ETC2/ASTC) | 4-8x less VRAM and bandwidth | Texture bandwidth bottleneck |
| Use MIP maps | Reduces texture aliasing and bandwidth at distance | Moiré artifacts, wasted sampling |
| Batch transparent objects separately | Must draw back-to-front with depth write off | Sorting artifacts |
