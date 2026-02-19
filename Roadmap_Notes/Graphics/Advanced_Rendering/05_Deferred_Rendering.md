# Module 5 — Deferred Rendering Architecture

> This module explains why forward rendering fails at scale,
> the complete G-Buffer architecture, and bandwidth optimization strategies.

---

## 1. The Scaling Problem of Forward Rendering

### 1.1 The Overdraw Catastrophe

```
FORWARD RENDERING COMPLEXITY:
═════════════════════════════
  Scene: 10,000 triangles, 100 point lights, 1080p (2M pixels)
  Average overdraw: 3× (each pixel written by 3 overlapping triangles)
  
  Fragment Shader executions:
    2,000,000 × 3 overdraw × 100 lights × ~33 ALU (PBR per light)
    = 19,800,000,000 ALU operations
    = ~19.8 GFLOPS just for lighting!
    
  BUT: Only the FRONT-MOST fragment survives the depth test.
  The other 2 layers of computation are completely WASTED.
  
  Wasted work: 66% of all Fragment Shader execution = thrown away.
```

### 1.2 Forward+ (Clustered Forward) — A Partial Solution

Before jumping to deferred, Forward+ reduces wasted light computation:

```
FORWARD+ PIPELINE:
══════════════════
  Step 1: Depth pre-pass (render depth only, no lighting).
  Step 2: Divide the screen into 16×16 pixel tiles.
  Step 3: For each tile, determine which lights' volumes overlap it.
          Store a per-tile light list in a buffer.
  Step 4: During the main forward pass, each fragment checks ONLY
          the lights assigned to its tile (maybe 3-5 instead of 100).
          
  COST: 2M pixels × 3 overdraw × 5 lights × 33 ALU = 990M ops (5× faster!)
  BUT: Still wastes 66% on overdraw.
```

---

## 2. The Deferred Rendering Solution

### 2.1 The Two-Pass Architecture

```
DEFERRED RENDERING:
═══════════════════
  PASS 1: G-BUFFER GENERATION
    Draw the entire scene. Output raw surface properties to textures.
    The Depth Buffer eliminates overdraw — only the closest pixel's
    data survives in the G-Buffer.
    
    NO LIGHTING MATH HAPPENS. Just write albedo, normal, metallic, etc.
    
  PASS 2: LIGHTING PASS
    Draw a SINGLE full-screen quad.
    For each pixel:
      1. Read surface data from G-Buffer textures.
      2. Calculate PBR for all lights.
      3. Write final lit color.
    
    Each pixel is processed EXACTLY ONCE. Zero overdraw.
    
  TOTAL: 2,000,000 pixels × 100 lights × 33 ALU = 6.6 GFLOPS
  SAVINGS: 3× less work than Forward (no overdraw)
```

### 2.2 G-Buffer Structure (Production Layout)

| Texture | Format | Size (4K) | Contents |
|---------|--------|-----------|----------|
| RT0: Albedo + Metallic | RGBA8 | 32MB | RGB = Base Color, A = Metallic |
| RT1: Normal | RG16F or RGB10A2 | 32-64MB | World-space normal (compressed) |
| RT2: Roughness + AO + Extra | RGBA8 | 32MB | R = Roughness, G = AO, B/A = Extra |
| Depth | D32F | 32MB | Hardware depth buffer |
| **TOTAL** | | **128-160MB** | |

### 2.3 G-Buffer Fragment Shader

```glsl
#version 300 es
precision highp float;

// Inputs from vertex shader
in vec3 vWorldPos;
in vec3 vNormal;
in vec2 vTexCoord;
in mat3 vTBN;

// Material textures
uniform sampler2D uAlbedo;
uniform sampler2D uNormalMap;
uniform sampler2D uOrmMap;  // AO(R), Roughness(G), Metallic(B)

// Multiple Render Targets
layout (location = 0) out vec4 gAlbedoMetallic;   // RT0
layout (location = 1) out vec4 gNormal;             // RT1
layout (location = 2) out vec4 gRoughnessAO;        // RT2

void main() {
    // Sample textures
    vec3 albedo = pow(texture(uAlbedo, vTexCoord).rgb, vec3(2.2)); // sRGB → Linear
    vec3 orm = texture(uOrmMap, vTexCoord).rgb;
    
    // Normal mapping
    vec3 normalMap = texture(uNormalMap, vTexCoord).rgb * 2.0 - 1.0;
    vec3 N = normalize(vTBN * normalMap);
    
    // Write G-Buffer
    gAlbedoMetallic = vec4(albedo, orm.b);          // RGB: albedo, A: metallic
    gNormal         = vec4(N * 0.5 + 0.5, 1.0);    // Encode normal to [0,1]
    gRoughnessAO    = vec4(orm.g, orm.r, 0.0, 1.0); // R: roughness, G: AO
}
```

### 2.4 Lighting Pass Shader

```glsl
#version 300 es
precision highp float;

// G-Buffer textures
uniform sampler2D gAlbedoMetallic;
uniform sampler2D gNormal;
uniform sampler2D gRoughnessAO;
uniform sampler2D gDepth;

// Camera and light data
uniform vec3 uCameraPos;
uniform mat4 uInvViewProj;

in vec2 vTexCoord;
out vec4 fragColor;

// Reconstruct world position from depth
vec3 reconstructWorldPos(vec2 uv, float depth) {
    vec4 clipPos = vec4(uv * 2.0 - 1.0, depth * 2.0 - 1.0, 1.0);
    vec4 worldPos = uInvViewProj * clipPos;
    return worldPos.xyz / worldPos.w;
}

void main() {
    // Read G-Buffer
    vec4 albedoMetallic = texture(gAlbedoMetallic, vTexCoord);
    vec3 albedo = albedoMetallic.rgb;
    float metallic = albedoMetallic.a;
    
    vec3 N = texture(gNormal, vTexCoord).rgb * 2.0 - 1.0;
    N = normalize(N);
    
    vec2 roughnessAO = texture(gRoughnessAO, vTexCoord).rg;
    float roughness = roughnessAO.r;
    float ao = roughnessAO.g;
    
    float depth = texture(gDepth, vTexCoord).r;
    vec3 worldPos = reconstructWorldPos(vTexCoord, depth);
    vec3 V = normalize(uCameraPos - worldPos);
    
    // Accumulate lighting
    vec3 Lo = vec3(0.0);
    for (int i = 0; i < numLights; i++) {
        vec3 L = normalize(lightPositions[i] - worldPos);
        float dist = length(lightPositions[i] - worldPos);
        float attenuation = 1.0 / (dist * dist);
        vec3 radiance = lightColors[i] * attenuation;
        
        Lo += calculatePBR(N, V, L, albedo, metallic, roughness, radiance);
    }
    
    vec3 ambient = vec3(0.03) * albedo * ao;
    fragColor = vec4(ambient + Lo, 1.0);
}
```

---

## 3. Normal Encoding/Decoding — Saving Bandwidth

### 3.1 The Problem

A world-space normal requires 3 floats × 16 bits = 48 bits per pixel.
For a 4K G-Buffer: 48 × 8.3M pixels = 400 Mbit = 50MB just for normals!

### 3.2 Octahedral Normal Encoding (2 Channels Instead of 3)

Since normals are always unit length ($|N| = 1$), we can reconstruct the Z component from X and Y. Octahedral encoding maps the surface of a sphere to a 2D square:

```glsl
// ENCODE: vec3 normal → vec2 (save 33% bandwidth)
vec2 encodeNormalOctahedral(vec3 n) {
    n /= (abs(n.x) + abs(n.y) + abs(n.z));
    if (n.z < 0.0) {
        n.xy = (1.0 - abs(n.yx)) * vec2(n.x >= 0.0 ? 1.0 : -1.0, 
                                          n.y >= 0.0 ? 1.0 : -1.0);
    }
    return n.xy * 0.5 + 0.5; // Map to [0, 1]
}

// DECODE: vec2 → vec3 normal
vec3 decodeNormalOctahedral(vec2 encoded) {
    encoded = encoded * 2.0 - 1.0;
    vec3 n = vec3(encoded.xy, 1.0 - abs(encoded.x) - abs(encoded.y));
    if (n.z < 0.0) {
        n.xy = (1.0 - abs(n.yx)) * vec2(n.x >= 0.0 ? 1.0 : -1.0, 
                                          n.y >= 0.0 ? 1.0 : -1.0);
    }
    return normalize(n);
}
```

### 3.3 Position Reconstruction (Eliminating the Position Buffer)

Storing world-space positions in the G-Buffer wastes 3×16-bit = 48 bits per pixel. Instead, reconstruct from depth:

```
POSITION RECONSTRUCTION FROM DEPTH:
════════════════════════════════════
  G-Buffer stores: depth (from hardware depth buffer) — FREE, already exists.
  Lighting pass: Reads depth, applies inverse View-Projection to recover world position.
  
  SAVINGS: Eliminates an entire 64MB texture from the G-Buffer!
  This is the SINGLE most impactful bandwidth optimization in deferred rendering.
```

---

## 4. Light Volumes — Lighting Only Affected Pixels

### 4.1 The Problem with Full-Screen Lighting

Drawing a full-screen quad and looping 100 lights means 100 lights × 2M pixels = 200M PBR evaluations. But a small lamp 30 meters away only affects maybe 50 pixels!

### 4.2 Light Volume Architecture

```
LIGHT VOLUMES:
══════════════
  For each point light:
    1. Calculate bounding sphere radius from light intensity.
       radius = sqrt(intensity / threshold)  (inverse-square cutoff)
    
    2. Draw a 3D icosphere mesh at the light's position.
    
    3. Front faces: If the sphere's front face passes depth test,
       the light MAY affect this pixel.
    
    4. Back faces: If we're INSIDE the light volume (camera is close
       to the light), draw back faces instead.
    
    5. In the fragment shader: Read G-Buffer, compute PBR for ONLY
       this one light.
    
  RESULT: A tiny desk lamp only runs PBR for the ~200 pixels
          its sphere covers. Not for the entire 2M pixel screen!
```

---

## 5. Bandwidth Analysis — The True Bottleneck

```
DEFERRED RENDERING BANDWIDTH BUDGET (4K, per frame):
═══════════════════════════════════════════════════
  G-BUFFER WRITE (Pass 1):
    RT0 (RGBA8):     32 MB write
    RT1 (RG16F):     32 MB write (octahedral normals)
    RT2 (RGBA8):     32 MB write
    Depth (D32F):    32 MB write
    SUBTOTAL WRITE: 128 MB

  G-BUFFER READ (Pass 2):
    RT0 read:         32 MB
    RT1 read:         32 MB
    RT2 read:         32 MB
    Depth read:       32 MB
    SUBTOTAL READ:   128 MB

  TOTAL BANDWIDTH:   256 MB per frame × 60 fps = 15.4 GB/s

  RTX 4070 bandwidth: 504 GB/s
  G-Buffer alone:     3% of total bandwidth

  BUT ADD: Shadow maps, SSAO, SSR, Bloom, TAA history...
  REALISTIC total:   30-50% of GPU bandwidth
```

---

## 6. Deferred Rendering Limitations & Solutions

| Limitation | Cause | Solution |
|-----------|-------|---------|
| No transparency | G-Buffer stores 1 depth per pixel | Separate forward pass for glass/particles |
| No MSAA | Lighting pass operates on textures, not geometry | Use TAA (Module 7) or FXAA |
| High memory usage | 3-4 render targets at full resolution | Use compact formats, octahedral normals, depth reconstruction |
| Wasted bandwidth on small lights | Full-screen quad touches ALL pixels | Light volumes or tiled/clustered approach |
| Material variety limited | G-Buffer has fixed structure | Encode material IDs, use branching in lighting shader |

---

## 7. WebGPU Implementation: Tile-Based Deferred (Mobile)

On Apple M-series and mobile GPUs, the GPU processes the screen in tiny 32×32 pixel tiles using on-chip Tile Memory (~128KB, as fast as L1 cache).

```
TILE-BASED DEFERRED ON MOBILE:
══════════════════════════════
  Desktop GPU (Immediate Mode):
    G-Buffer Pass → Writes 128MB to VRAM → Lighting Pass reads 128MB from VRAM
    BANDWIDTH: 256 MB round-trip through slow VRAM
    
  Mobile GPU (Tile-Based):
    G-Buffer Pass → Writes G-Buffer to Tile Memory (on-chip, ~0.1ms access)
    Lighting Pass → Reads from SAME Tile Memory (no VRAM trip!)
    Only the FINAL lit color (32MB) is written to VRAM
    BANDWIDTH: 32 MB (8× less than desktop!)
    
  WebGPU achieves this via render pass loadOp/storeOp:
    storeOp: 'discard' for intermediate textures
    (tells the GPU: "Don't write this G-Buffer to VRAM, keep it in tile memory")
```

---

## 8. Debugging Deferred Renderers

```glsl
// G-Buffer visualization shader
uniform int debugView;

vec3 debugGBuffer(vec2 uv) {
    switch (debugView) {
        case 0: return texture(gAlbedoMetallic, uv).rgb;            // Albedo
        case 1: return texture(gNormal, uv).rgb;                    // Normals (encoded)
        case 2: return vec3(texture(gRoughnessAO, uv).r);           // Roughness
        case 3: return vec3(texture(gRoughnessAO, uv).g);           // AO
        case 4: return vec3(texture(gAlbedoMetallic, uv).a);        // Metallic
        case 5: return vec3(texture(gDepth, uv).r);                 // Raw depth
        case 6: return vec3(linearizeDepth(texture(gDepth, uv).r)); // Linear depth
    }
    return vec3(0.0);
}
```
