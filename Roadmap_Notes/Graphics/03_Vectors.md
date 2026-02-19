# Chapter 03 — Vectors (2D & 3D)

## Why Vectors?
Vectors describe directions and magnitudes. Every surface normal, light direction, camera orientation, and velocity in a 3D engine is a vector. You cannot build a renderer without mastering them.

---

## 1. What Is a Vector?

A vector is an arrow with **direction** and **length (magnitude)**. Unlike a point (which is a location), a vector represents a displacement.

```
VECTOR vs POINT:
════════════════
  Point A = (3, 2)  → "I am HERE at coordinates 3, 2"
  Vector v = (3, 2) → "I am moving 3 right and 2 up FROM wherever I am"
  
  A vector doesn't have a fixed position. It can be placed anywhere.
  A point is a fixed location in space.
```

### Vector Notation
```
2D: v = (vx, vy) = (3, 4)
3D: v = (vx, vy, vz) = (1, 2, -3)

In GLSL/WGSL:
  vec2 v2 = vec2(3.0, 4.0);
  vec3 v3 = vec3(1.0, 2.0, -3.0);
  vec4 v4 = vec4(1.0, 2.0, -3.0, 1.0); // w=1 for points, w=0 for directions
```

---

## 2. Vector Operations

### Addition
```
MANUAL CALCULATION:
  a = (2, 3), b = (4, -1)
  a + b = (2+4, 3+(-1)) = (6, 2)
  
  VISUAL:
     b starts    a+b ends
     where a     here
     ends        ↗
       ↗       ↗
  a starts
  here
  
  "Walk vector a, then walk vector b. You arrive at a+b."
```

### Subtraction
```
  a = (5, 3), b = (2, 1)
  a - b = (5-2, 3-1) = (3, 2)
  
  GRAPHICS USE: Direction from point A to point B:
  Direction = B - A
  
  Camera at (0, 0, 5), Object at (3, 2, 0):
  Direction = Object - Camera = (3, 2, 0) - (0, 0, 5) = (3, 2, -5)
  "From camera, go right 3, up 2, forward 5 to reach the object."
```

### Scalar Multiplication
```
  v = (3, 4), scalar = 2
  2v = (6, 8) — same direction, twice as long
  
  -v = (-3, -4) — same length, opposite direction
  0.5v = (1.5, 2) — same direction, half as long
  
  GRAPHICS USE: Scaling velocity, adjusting light intensity.
```

---

## 3. Vector Length (Magnitude)

$$|v| = \sqrt{v_x^2 + v_y^2 + v_z^2}$$

### Manual Calculations
```
2D: v = (3, 4)
  |v| = √(3² + 4²) = √(9 + 16) = √25 = 5

3D: v = (1, 2, 2)
  |v| = √(1² + 2² + 2²) = √(1 + 4 + 4) = √9 = 3

3D: v = (2, -3, 6)
  |v| = √(4 + 9 + 36) = √49 = 7

GRAPHICS USE:
  Distance between two points = length of their difference vector:
  A = (1, 0, 0), B = (4, 0, -4)
  dist = |B - A| = |(3, 0, -4)| = √(9 + 0 + 16) = √25 = 5
```

---

## 4. Normalization — Making Unit Vectors

A **unit vector** has length 1. It represents pure direction with no magnitude.

$$\hat{v} = \frac{v}{|v|}$$

### Manual Calculation
```
v = (3, 4), |v| = 5
  v̂ = (3/5, 4/5) = (0.6, 0.8)
  
  VERIFY: |(0.6, 0.8)| = √(0.36 + 0.64) = √1.0 = 1 ✓

v = (1, 2, 2), |v| = 3
  v̂ = (1/3, 2/3, 2/3) = (0.333, 0.667, 0.667)

CRITICAL IN GRAPHICS:
  Surface normals MUST be normalized before lighting calculations.
  Light directions MUST be normalized.
  If you forget, dot products give wrong values → broken lighting!
  
  // In GLSL:
  vec3 N = normalize(vNormal);  // ALWAYS do this in the fragment shader
  vec3 L = normalize(lightPos - fragPos);
```

---

## 5. Dot Product — The Most Important Operation

$$a \cdot b = a_x b_x + a_y b_y + a_z b_z$$

Also equals: $a \cdot b = |a| \cdot |b| \cdot \cos\theta$

### Manual Calculations
```
EXAMPLE 1:
  a = (1, 0, 0), b = (0, 1, 0)
  a · b = 1×0 + 0×1 + 0×0 = 0
  
  These vectors are PERPENDICULAR! Dot product = 0 means 90°.

EXAMPLE 2:
  a = (1, 0, 0), b = (1, 0, 0)
  a · b = 1×1 + 0×0 + 0×0 = 1
  
  Same direction! For unit vectors, max dot product = 1 (0°).

EXAMPLE 3:
  a = (1, 0, 0), b = (-1, 0, 0)
  a · b = 1×(-1) + 0×0 + 0×0 = -1
  
  Opposite direction! For unit vectors, dot product = -1 (180°).

EXAMPLE 4:
  a = (2, 3), b = (4, -1)
  a · b = 2×4 + 3×(-1) = 8 - 3 = 5
  
  Finding the angle:
  |a| = √(4+9) = √13 ≈ 3.606
  |b| = √(16+1) = √17 ≈ 4.123
  cosθ = 5 / (3.606 × 4.123) = 5 / 14.87 = 0.336
  θ = arccos(0.336) ≈ 70.3°
```

### Dot Product Meaning Table
```
DOT PRODUCT VALUES (for unit vectors):
══════════════════════════════════════
  Value    Angle    Meaning
  ─────    ─────    ───────
  1.0      0°       Same direction (light hitting surface head-on)
  0.707    45°      Moderate angle
  0.0      90°      Perpendicular (light grazing the surface)
  -0.5     120°     Facing away (light behind surface)
  -1.0     180°     Completely opposite directions

GRAPHICS RULE:
  NdotL = dot(Normal, LightDir)
  If NdotL > 0: surface faces the light → illuminate it
  If NdotL ≤ 0: surface faces AWAY from light → in shadow
  
  // In GLSL:
  float NdotL = max(dot(N, L), 0.0); // Clamp to prevent negative light!
```

---

## 6. Cross Product — Finding Perpendicular Directions (3D Only)

$$a \times b = (a_y b_z - a_z b_y, \quad a_z b_x - a_x b_z, \quad a_x b_y - a_y b_x)$$

The result is a vector **perpendicular** to both inputs.

### Manual Calculation
```
a = (1, 0, 0) [right], b = (0, 1, 0) [up]
  
  x = a_y×b_z - a_z×b_y = 0×0 - 0×1 = 0
  y = a_z×b_x - a_x×b_z = 0×0 - 1×0 = 0
  z = a_x×b_y - a_y×b_x = 1×1 - 0×0 = 1
  
  a × b = (0, 0, 1) [forward]
  
  RIGHT × UP = FORWARD ✓ (right-hand rule!)

ANOTHER EXAMPLE:
  a = (2, 3, 4), b = (5, 6, 7)
  
  x = 3×7 - 4×6 = 21 - 24 = -3
  y = 4×5 - 2×7 = 20 - 14 = 6
  z = 2×6 - 3×5 = 12 - 15 = -3
  
  a × b = (-3, 6, -3)
  
  VERIFY perpendicularity:
  (-3, 6, -3) · (2, 3, 4) = -6 + 18 - 12 = 0 ✓
  (-3, 6, -3) · (5, 6, 7) = -15 + 36 - 21 = 0 ✓
```

### Graphics Applications
```
SURFACE NORMAL FROM TRIANGLE:
  Triangle vertices: P0 = (0,0,0), P1 = (1,0,0), P2 = (0,1,0)
  Edge1 = P1 - P0 = (1, 0, 0)
  Edge2 = P2 - P0 = (0, 1, 0)
  Normal = Edge1 × Edge2 = (0, 0, 1) → points upward (out of the XY plane)
  
  // In GLSL:
  vec3 normal = normalize(cross(edge1, edge2));

IMPORTANT: Cross product order matters!
  a × b = -(b × a)  (opposite direction!)
  Reversing the order FLIPS the normal → surface appears inside-out!
```

---

## 7. Vector Reflection

When light hits a surface and bounces off:

$$R = I - 2(I \cdot N)N$$

where $I$ is the incident direction, $N$ is the surface normal, $R$ is the reflected direction.

```
MANUAL CALCULATION:
  Light coming from above-left: I = (1, -1, 0), normalized → (0.707, -0.707, 0)
  Surface normal pointing up: N = (0, 1, 0)
  
  I · N = 0.707×0 + (-0.707)×1 + 0×0 = -0.707
  
  R = I - 2(-0.707)N = (0.707, -0.707, 0) - (-1.414)(0, 1, 0)
    = (0.707, -0.707, 0) + (0, 1.414, 0)
    = (0.707, 0.707, 0)
    
  CHECK: Light came from upper-left, bounced to upper-right ✓
  
  // In GLSL:
  vec3 R = reflect(I, N); // Built-in function!
```

---

## 8. Practice Problems

### Problem 1
Normalize vector v = (6, 0, 8).

**Solution:**
```
|v| = √(36 + 0 + 64) = √100 = 10
v̂ = (6/10, 0/10, 8/10) = (0.6, 0.0, 0.8)
Verify: √(0.36 + 0 + 0.64) = √1.0 = 1 ✓
```

### Problem 2
Find the angle between a = (1, 1, 0) and b = (0, 1, 0).

**Solution:**
```
a · b = 0 + 1 + 0 = 1
|a| = √2 ≈ 1.414, |b| = 1
cosθ = 1 / (1.414 × 1) = 0.707
θ = arccos(0.707) = 45°
```

### Problem 3
Compute the cross product of a = (0, 0, 1) and b = (1, 0, 0).

**Solution:**
```
x = 0×0 - 1×0 = 0
y = 1×1 - 0×0 = 1
z = 0×0 - 0×1 = 0

a × b = (0, 1, 0) → the up vector!
```

### Problem 4
A surface normal is N = (0, 1, 0). Light direction (toward surface) is L = (0.5, -0.866, 0). Calculate NdotL.

**Solution:**
```
NdotL = 0×0.5 + 1×(-0.866) + 0×0 = -0.866

This is NEGATIVE → the light is hitting the BACKFACE of the surface.
In the shader: max(NdotL, 0.0) = 0.0 → no illumination ✓
```

### Problem 5
Distance from camera at (0, 2, 10) to object at (3, 6, 2)?

**Solution:**
```
diff = (3-0, 6-2, 2-10) = (3, 4, -8)
dist = √(9 + 16 + 64) = √89 ≈ 9.43 units
```
