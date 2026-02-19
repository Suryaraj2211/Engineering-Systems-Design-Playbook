# Chapter 21 — PBR & Performance Tuning

---

## 1. PBR Material Workflow Summary

```
METALLIC-ROUGHNESS WORKFLOW (industry standard):
════════════════════════════════════════════════
  TEXTURE MAPS:
    Albedo (Base Color):  RGB color without lighting information
    Normal Map:           Tangent-space surface detail
    ORM (packed):         R = Ambient Occlusion, G = Roughness, B = Metallic
    Emissive:             Self-illuminating surfaces
    
  MATERIAL PARAMETERS:
    roughness = 0.0 → mirror smooth (puddles, chrome)
    roughness = 1.0 → completely matte (chalk, concrete)
    metallic  = 0.0 → dielectric (wood, plastic, skin)
    metallic  = 1.0 → conductor (gold, iron, copper)
    
  REFERENCE F0 VALUES:
    Dielectrics: F0 ≈ 0.04 (water, glass, plastic — all similar!)
    Gold:   F0 = (1.0, 0.71, 0.29)
    Silver: F0 = (0.95, 0.93, 0.88)
    Copper: F0 = (0.95, 0.64, 0.54)
    Iron:   F0 = (0.56, 0.57, 0.58)
```

---

## 2. Performance Profiling Strategy

### 2.1 GPU vs CPU Bound Detection
```
IS YOUR APP GPU-BOUND OR CPU-BOUND?
════════════════════════════════════
  Test 1: Halve the screen resolution (e.g., 1080p → 540p).
    FPS doubles → GPU BOUND (fill-rate or bandwidth).
    FPS same   → CPU BOUND (JavaScript or draw call overhead).
    
  Test 2: Reduce scene complexity (fewer objects, fewer lights).
    FPS improves → Could be CPU (draw calls) or GPU (vertex processing).
    
  Test 3: Simplify fragment shader to output flat color.
    FPS improves → Fragment shader is the bottleneck (too many ALU/texture reads).
    FPS same     → Vertex processing or CPU is the bottleneck.
```

### 2.2 Performance Budget (60fps = 16.67ms)
```
FRAME BUDGET AT 60FPS:
══════════════════════
  JavaScript (CPU):           ≤ 4ms
    - Transform updates:        1ms
    - Frustum culling:          0.5ms
    - Draw call submission:     2ms
    - Input/physics:            0.5ms
    
  GPU rendering:              ≤ 12ms
    - Shadow maps:              2ms
    - G-Buffer:                 2ms
    - SSAO:                     1ms
    - Lighting:                 3ms
    - Post-processing:          2ms
    - Headroom:                 2ms
    
  TOTAL:                      ≤ 16ms
```

---

## 3. Common Performance Optimizations

| Technique | Savings | Complexity |
|-----------|---------|-----------|
| Frustum culling (CPU) | Skip 50-80% of draw calls | Low |
| Back-face culling (GPU) | Skip 50% of triangles | Trivial (`gl.enable(gl.CULL_FACE)`) |
| Instancing | 100× fewer draw calls for repeated objects | Low |
| LOD (level of detail) | 10× fewer triangles for distant objects | Medium |
| Texture compression (BC7/ASTC) | 4x less VRAM, 4x less bandwidth | Low |
| Mipmap generation | Fewer cache misses for distant textures | Trivial |
| Depth pre-pass | Eliminates overdraw in main pass | Low |
| Half-res effects | SSAO/SSR at 1/4 pixel count | Low |
| Shader simplification | Remove unnecessary ALU | Medium |
| Occlusion culling | Skip objects hidden behind walls | High |

---

## 4. Debugging Rendering Issues

```
SYSTEMATIC DEBUGGING FLOWCHART:
═══════════════════════════════
  PROBLEM: Something looks wrong on screen.
  
  → Is the GEOMETRY correct?
    Render with flat white color, no lighting.
    Geometry missing → check vertex buffer, index buffer, draw count.
    Geometry deformed → check model matrix, TRS order.
    
  → Are the NORMALS correct?
    Visualize normals as colors: fragColor = vec4(N * 0.5 + 0.5, 1.0)
    All blue → normals are all (0,0,1), not transformed to world space.
    Faceted → normals are flat per-face (need smooth normals).
    
  → Is LIGHTING correct?
    Test with single directional light only.
    Too dark → ambient too low, or normals inverted.
    Only lit on one side → light direction reversed.
    
  → Are TEXTURES correct?
    Check UVs: fragColor = vec4(v_texCoord, 0.0, 1.0)
    All one color → UVs not set or attribute not bound.
    Stretched/distorted → wrong UV unwrapping.
    Upside down → forgot gl.UNPACK_FLIP_Y_WEBGL.
```
