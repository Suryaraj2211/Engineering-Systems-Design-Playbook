# Chapter 23 — Complete Roadmap & Revision

---

## 1. Learning Path Summary

```
YOUR JOURNEY:
═════════════
  PHASE 1: MATHEMATICS (Chapters 1-6)                          ~2 weeks
    Arithmetic → Trig → Vectors → Matrices → Transforms → Projections
    
  PHASE 2: GPU FUNDAMENTALS (Chapters 7-8)                     ~1 week
    GPU Architecture → Complete Graphics Pipeline
    
  PHASE 3: WEBGL (Chapters 9-12)                               ~3 weeks
    Context & Shaders → Buffers & Textures → Camera & Lighting → Advanced
    
  PHASE 4: SHADERS (Chapters 13-14)                             ~2 weeks
    GLSL Fundamentals → Advanced Shaders (Procedural, Post-Process, PBR)
    
  PHASE 5: WEBGPU (Chapters 15-17)                             ~3 weeks
    Basics & Init → Pipelines & Bind Groups → Compute Shaders
    
  PHASE 6: ENGINE ARCHITECTURE (Chapters 18-19)                ~2 weeks
    Game Loop, ECS, Render Architecture → Scene Graph, Resource Manager
    
  PHASE 7: ADVANCED RENDERING (Chapters 20-21 + Advanced/)     ~4 weeks
    Shadows, Deferred, SSR, TAA, HDR, PBR → Performance Tuning
    
  PHASE 8: PROJECTS & CAPSTONE (Chapter 22)                    ~6 weeks
    9 Progressive Projects → Capstone Engine
    
  TOTAL: ~23 weeks (6 months) for a dedicated learner.
```

---

## 2. Revision Checklist

### Math (Can You Do These From Memory?)
- [ ] Convert 60° to radians without a calculator
- [ ] Compute dot product of two 3D vectors manually
- [ ] Compute cross product of two 3D vectors manually
- [ ] Normalize a 3D vector manually
- [ ] Multiply a 4×4 matrix by a vec4 manually
- [ ] Explain why TRS order matters (with an example of what goes wrong)
- [ ] Derive the perspective divide's effect on a point

### GPU & Pipeline
- [ ] Draw the complete graphics pipeline from memory (9 stages)
- [ ] Explain barycentric interpolation in 2 sentences
- [ ] Explain why GPU branching is expensive (warp divergence)
- [ ] Name the 4 GPU memory levels and their relative speeds

### WebGL
- [ ] Write the minimum code to draw a triangle from scratch
- [ ] Explain stride and offset in `vertexAttribPointer`
- [ ] Set up a framebuffer with color and depth attachments
- [ ] Implement a texture load with correct filtering and flip

### Shaders
- [ ] Write a Phong/Blinn-Phong fragment shader from memory
- [ ] Explain why `normalize()` is needed in the fragment shader
- [ ] Write a basic shadow comparison formula
- [ ] Explain the difference between `precision mediump` and `highp`

### WebGPU
- [ ] Explain the 3-step initialization (adapter → device → configure)
- [ ] Create a render pipeline with vertex/fragment stages
- [ ] Set up a bind group with a uniform buffer
- [ ] Write a basic compute shader with dispatch sizing

### Engine Architecture
- [ ] Implement a fixed-timestep game loop
- [ ] Explain ECS (Entity-Component-System) vs OOP inheritance
- [ ] Draw a render pass order diagram (shadow → depth → gbuf → light → post)
- [ ] Implement a basic scene graph with recursive world matrix

---

## 3. Where to Go Next

After completing this curriculum:

1. **Advanced Track** (`Advanced_Rendering/` — Modules 01-09):
   PBR Theory → BRDF Deep Dive → Global Illumination → Shadow Mapping →
   Deferred Rendering → Screen-Space Techniques → TAA → HDR → Architecture

2. **Expert/Research Track** (`Expert_Research/` — Modules 01-07):
   Rendering Innovation → GPU Deep Dive → Research Rendering →
   Engine Core Design → AI-Augmented Rendering → Research Development →
   Innovator Blueprint

3. **Recommended External Resources:**
   - [learnopengl.com](https://learnopengl.com) — Best free WebGL/OpenGL tutorial
   - [Real-Time Rendering (4th Ed.)](https://www.realtimerendering.com/) — The reference book
   - [PBR Book (pbr-book.org)](https://pbr-book.org/) — Physically Based Rendering bible
   - [WebGPU Spec](https://www.w3.org/TR/webgpu/) — Official specification
   - [Shadertoy](https://shadertoy.com) — Learn by experimenting with shaders
