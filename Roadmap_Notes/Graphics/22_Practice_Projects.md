# Chapter 22 — Practice & Capstone Projects

---

## 10 Projects (Progressive Difficulty)

### BEGINNER (Math & Basics)

#### Project 1: Interactive Unit Circle (1 day)
Draw a unit circle. Click to set an angle θ. Display:
- The point (cosθ, sinθ) on the circle
- The sin and cos values as vertical/horizontal lines
- The angle in both degrees and radians

**Skills:** Canvas 2D drawing, trig functions, coordinate transforms.

#### Project 2: Spinning Color Cube (3 days)
Render a 3D cube with each face a different color. Rotate it continuously around the Y axis. Use keyboard to switch between perspective and orthographic projection.

**Skills:** Vertex/index buffers, model/view/projection matrices, depth testing.

#### Project 3: Free-Look Camera (3 days)
Implement a first-person camera with WASD movement and mouse look. Navigate around a grid floor with a few colored cubes placed as landmarks.

**Skills:** View matrix from yaw/pitch, keyboard/mouse input, matrix construction.

### INTERMEDIATE (Lighting & Textures)

#### Project 4: Textured Crate with Phong Lighting (1 week)
Load a crate texture. Apply diffuse, normal, and specular maps. Add a single point light that orbits the crate. Toggle between Phong and Blinn-Phong.

**Skills:** Texture loading, UV mapping, normal mapping, TBN matrix, lighting math.

#### Project 5: Mirror Room (1 week)
Create a room with a mirror on one wall. Render the scene from the mirror's perspective into a framebuffer. Apply that texture to the mirror quad. Add a moving object to verify the reflection updates.

**Skills:** Framebuffer objects, render-to-texture, reflection matrix, multi-pass rendering.

#### Project 6: Stencil Portal (1 week)
Create a doorway. Use the stencil buffer to render a different "world" visible only through the doorway. Add an object that can partially pass through.

**Skills:** Stencil buffer operations, multi-pass rendering, clip space manipulation.

### ADVANCED (Post-Processing & Shadows)

#### Project 7: Bloom Post-Processing Stack (1 week)
Render a scene with glowing objects (values > 1.0). Implement: bright pass extraction → downsample chain → upsample chain → blend with original.

**Skills:** HDR framebuffers, multi-pass rendering, ping-pong FBOs, Gaussian blur.

#### Project 8: PCF Shadow Mapping (2 weeks)
Add a spotlight. Render the scene from the light into a depth texture. In the main pass, sample the shadow map with 5×5 PCF. Add slope-scale bias.

**Skills:** Depth-only rendering, light-space projection, PCF filtering, bias tuning.

#### Project 9: GPU Particle System — WebGPU Compute (2 weeks)
Use a WGSL compute shader to simulate 100,000 particles with gravity and wind. Store positions in a storage buffer. Render with instancing.

**Skills:** Compute shaders, storage buffers, dispatch sizing, GPU simulation.

---

## Capstone: Build a Mini Rendering Engine (4 weeks)

### Architecture Requirements
1. **Game Loop:** `requestAnimationFrame` with delta-time tracking.
2. **Scene Graph:** `Node` class with `.addChild()`, recursive world matrix computation.
3. **Materials:** Material class holding shader reference and bind group.
4. **Lighting:** 1 directional light (sun) + multiple point lights via uniform buffer.
5. **Mesh Loading:** Fetch a `.gltf` file, extract vertex/index arrays, auto-create materials.
6. **Render Pass Manager:** Separate Shadow Pass, Opaque Pass, Transparent Pass, Post-Processing.

### The Challenge Scene
Render the "Damaged Helmet" or "Sponza" glTF model (available on Khronos GitHub). Orbit camera around it. Apply full PBR lighting.

If you can do this at 60fps — congratulations, you are a Graphics Engineer.

---

## 10 Essential Interview Questions

1. **Math:** "What is the dot product of two normalized vectors facing opposite directions?" → **-1**
2. **Math:** "My object's scale is twisting weirdly. Why?" → **Wrong TRS order. Must be T × R × S, not S × R × T.**
3. **WebGL:** "Why is my texture black?" → **Non-POT texture with mipmapping, OR forgot to set sampler uniform to correct texture unit.**
4. **Performance:** "Vertex bound vs fill-rate bound — how do you tell?" → **Halve resolution. If FPS doubles, fill-rate bound. If unchanged, vertex/CPU bound.**
5. **Shaders:** "Why clamp dot(N,L) to 0?" → **Negative dot = light behind surface. Without clamp, you'd subtract light, creating dark halos.**
6. **Architecture:** "Forward vs Deferred — when which?" → **Deferred for 100+ lights (no overdraw lighting). Forward for transparency, MSAA, simple scenes.**
7. **Math:** "What does dividing by W do?" → **Perspective divide. Creates foreshortening: distant objects become smaller in NDC.**
8. **WebGPU:** "Uniform vs Storage buffer?" → **Uniform: small (64KB), read-only, fast. Storage: huge (GB), read-write, slower.**
9. **Pipelines:** "Why does WebGPU bake state into RenderPipeline?" → **Validates once at creation. Eliminates per-draw validation overhead (10× less CPU cost).**
10. **Compute:** "What is workgroupBarrier()?" → **Guarantees all threads in a workgroup reach this point before any continue. Required for shared memory synchronization.**
