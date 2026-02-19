# Chapter 13 — GLSL Fundamentals

---

## 1. GLSL Data Types

```glsl
// Scalars
float f = 1.5;     // 32-bit float (ALWAYS use .0 to mark as float: 1.0 not 1)
int i = 42;         // 32-bit integer
bool b = true;      // Boolean

// Vectors
vec2 uv = vec2(0.5, 0.5);
vec3 color = vec3(1.0, 0.0, 0.0);  // Red
vec4 pos = vec4(1.0, 2.0, 3.0, 1.0);

// SWIZZLING: Access components by name (xyzw, rgba, stpq all work!)
float x = pos.x;            // First component
vec3 rgb = pos.xyz;          // First 3 components
vec2 yz = pos.yz;            // Second and third
vec3 bgr = color.bgr;       // Reverse order! Used for color manipulation
vec4 rrrr = color.xxxx;     // Repeat a component 4 times

// Matrices
mat2 m2; // 2×2
mat3 m3; // 3×3 (used for normal matrix)
mat4 m4; // 4×4 (used for MVP)

// Samplers (texture references)
uniform sampler2D diffuseMap;
uniform samplerCube envMap;
```

---

## 2. Built-In Functions (Most Used in Graphics)

```glsl
// MATH
float a = max(x, 0.0);          // Clamp negative to 0 (lighting!)
float b = min(x, 1.0);          // Cap at 1.0
float c = clamp(x, 0.0, 1.0);   // Both: restrict to [0, 1]
float d = mix(a, b, t);         // lerp: a*(1-t) + b*t
float e = smoothstep(0.0, 1.0, x); // Smooth S-curve interpolation
float f = step(0.5, x);         // 0 if x < 0.5, 1 if x >= 0.5
float g = fract(x);             // Fractional part (x - floor(x))

// VECTOR
float len = length(v);          // Vector magnitude
vec3 n = normalize(v);          // Unit vector
float d = dot(a, b);            // Dot product
vec3 c = cross(a, b);           // Cross product
vec3 r = reflect(I, N);         // Reflection vector
float dist = distance(a, b);   // Distance between two points

// TEXTURE
vec4 color = texture(sampler, uv); // Sample a texture at UV coordinates

// DERIVATIVES (fragment shader only)
float dx = dFdx(value);         // Rate of change in screen X
float dy = dFdy(value);         // Rate of change in screen Y
// Used for: computing mip levels, screen-space effects, edge detection
```

---

## 3. Precision Qualifiers

```glsl
precision highp float;    // 32-bit (desktop, position, depth)
precision mediump float;  // 16-bit (mobile-friendly, colors, UVs)
precision lowp float;     // 8-bit (rarely used)

// RULE OF THUMB:
// Use highp for: positions, depth, transformation math
// Use mediump for: colors, UVs, lighting results (where precision < critical)
// On desktop: everything is highp internally regardless
// On MOBILE: mediump saves power and increases performance!
```

---

## 4. Vertex Shader ↔ Fragment Shader Communication

```glsl
// VERTEX SHADER
out vec3 v_normal;       // Interpolated across triangle in rasterizer
out vec2 v_texCoord;
out vec3 v_worldPos;

void main() {
    v_normal = mat3(u_normalMatrix) * a_normal;
    v_texCoord = a_texCoord;
    v_worldPos = (u_model * vec4(a_position, 1.0)).xyz;
    gl_Position = u_mvp * vec4(a_position, 1.0);
}

// FRAGMENT SHADER
in vec3 v_normal;        // Received interpolated value
in vec2 v_texCoord;
in vec3 v_worldPos;

void main() {
    vec3 N = normalize(v_normal); // MUST RE-NORMALIZE (interpolation changes length!)
    vec4 texColor = texture(u_diffuse, v_texCoord);
    // ... lighting math ...
}
```

### Why Re-Normalize?
```
INTERPOLATION CHANGES VECTOR LENGTH:
═════════════════════════════════════
  Vertex A normal: (0, 1, 0)     length = 1.0
  Vertex B normal: (1, 0, 0)     length = 1.0
  
  Midpoint interpolation: (0.5, 0.5, 0) 
  length = √(0.25 + 0.25) = √0.5 = 0.707  ← NOT 1.0!
  
  Using this un-normalized normal → lighting is 29% dimmer at the midpoint!
  ALWAYS normalize in the fragment shader.
```

---

## 5. Normal Mapping

```glsl
// Normal map stores tangent-space normals in RGB texture
// R = tangent (X), G = bitangent (Y), B = normal (Z)
// Values [0-1] in texture → remap to [-1, 1]:
vec3 normalMap = texture(u_normalMap, v_texCoord).rgb * 2.0 - 1.0;

// Transform from tangent space to world space using TBN matrix
vec3 T = normalize(v_tangent);
vec3 B = normalize(v_bitangent);
vec3 N = normalize(v_normal);
mat3 TBN = mat3(T, B, N);

vec3 worldNormal = normalize(TBN * normalMap);
// Use worldNormal instead of v_normal for ALL lighting calculations!
```

---

## 6. Common GLSL Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Integer division: `1/2` instead of `1.0/2.0` | Result = 0 (integer math!) | Always use `.0`: `1.0/2.0 = 0.5` |
| Not normalizing interpolated normals | Lighting brightness varies incorrectly | `normalize(v_normal)` in fragment shader |
| Using `==` to compare floats | Comparison fails (precision!) | Use `abs(a-b) < 0.001` |
| `pow(0.0, 0.0)` | Undefined on some GPUs → NaN | Guard: `pow(max(base, 0.001), exp)` |
| Forgetting `precision` declaration | Shader won't compile on mobile | Always declare `precision mediump float;` |
