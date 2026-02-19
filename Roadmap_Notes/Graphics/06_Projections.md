# Chapter 06 — Projection Mathematics

## Why Projection?
Projection transforms 3D coordinates into 2D screen coordinates. Without it, objects at different distances appear the same size — not how human vision works.

---

## 1. Orthographic vs. Perspective Projection

```
ORTHOGRAPHIC (Parallel):          PERSPECTIVE (Realistic):
  Objects appear same size            Objects appear SMALLER
  regardless of distance.            when they are farther away.
  
  Used for: 2D games, CAD,           Used for: 3D games, film,
  technical drawing, UI               everything that looks "real"
  
  Parallel lines stay parallel.       Parallel lines converge to
                                      a vanishing point (like
                                      railroad tracks).
```

---

## 2. Orthographic Projection Matrix

```
ORTHO MATRIX:
═════════════
  Given: left, right, bottom, top, near, far

  | 2/(r-l)   0         0          -(r+l)/(r-l) |
  | 0         2/(t-b)   0          -(t+b)/(t-b) |
  | 0         0        -2/(f-n)    -(f+n)/(f-n) |
  | 0         0         0           1            |

MANUAL EXAMPLE:
  ortho(left=-10, right=10, bottom=-10, top=10, near=0.1, far=100)
  
  M[0][0] = 2 / (10-(-10)) = 2/20 = 0.1
  M[1][1] = 2 / (10-(-10)) = 0.1
  M[2][2] = -2 / (100-0.1) = -0.02
  M[0][3] = -(10+(-10))/(10-(-10)) = 0
  M[1][3] = 0
  M[2][3] = -(100+0.1)/(100-0.1) ≈ -1.002
  
  A point at (5, 5, -50) transforms to:
  x_ndc = 5 × 0.1 + 0 = 0.5
  y_ndc = 5 × 0.1 + 0 = 0.5
  z_ndc = -50 × (-0.02) + (-1.002) = 1.0 - 1.002 = -0.002
  
  Result: (0.5, 0.5, -0.002) → visible in NDC [-1, 1] ✓
```

---

## 3. Perspective Projection Matrix — Full Derivation

### 3.1 The Conceptual Model

```
PERSPECTIVE PROJECTION VISUALIZATION:
═════════════════════════════════════
  Camera          Near Plane        Far Plane
    ●─────────────┤            ├────────────┤
     \            │            │            │
      \           │  Visible   │  Visible   │
       \    FOV   │  Region    │  Region    │
        \  ╱      │            │            │
         \/       │            │            │
        / \       │            │            │
       /   \      │            │            │
      /     \     │            │            │
     /       \    │            │            │
               ├────────────┤
  
  Objects between near and far planes are visible.
  Objects closer than near or farther than far are clipped.
  
  The RATIO (near plane size / object distance) creates the
  perspective foreshortening effect.
```

### 3.2 The Math

For a vertical FOV of `fovy` and aspect ratio `a`:

$$f = \frac{1}{\tan(\text{fovy}/2)}$$

```
PERSPECTIVE MATRIX:
═══════════════════
  | f/a    0      0                 0                |
  |  0     f      0                 0                |
  |  0     0    (f+n)/(n-f)    (2×f×n)/(n-f)        |
  |  0     0     -1                 0                |

  The bottom row [0, 0, -1, 0] is what creates perspective:
  After multiplication, w_clip = -z_eye
  
  The PERSPECTIVE DIVIDE (x/w, y/w, z/w) makes distant objects smaller!
```

### 3.3 Manual Calculation: Full Vertex Projection

```
COMPLETE PROJECTION EXAMPLE:
════════════════════════════
  fovy = 60°, aspect = 16/9, near = 0.1, far = 100
  f = 1/tan(30°) = 1/0.577 = 1.732

  P = | 0.974  0      0       0      |   (f/aspect = 1.732/(16/9) = 0.974)
      | 0      1.732  0       0      |
      | 0      0     -1.002  -0.200  |   ((f+n)/(n-f) ≈ -1.002)
      | 0      0     -1       0      |
  
  Vertex in VIEW SPACE: v = (2, 3, -10, 1)
  (Note: in view space, objects in front of camera have NEGATIVE z!)
  
  Step 1: Matrix × Vector:
    x_clip = 0.974 × 2  = 1.948
    y_clip = 1.732 × 3  = 5.196
    z_clip = (-1.002)×(-10) + (-0.200)×1 = 10.02 - 0.2 = 9.82
    w_clip = (-1)×(-10) = 10
  
  Step 2: PERSPECTIVE DIVIDE (÷ w):
    x_ndc = 1.948 / 10 = 0.1948
    y_ndc = 5.196 / 10 = 0.5196
    z_ndc = 9.82 / 10  = 0.982
  
  Step 3: NDC → Screen (for 1920×1080):
    screen_x = (0.1948 + 1) / 2 × 1920 = 1.1948/2 × 1920 = 1147
    screen_y = (0.5196 + 1) / 2 × 1080 = 1.5196/2 × 1080 = 821
  
  The vertex appears at pixel (1147, 821) on a 1080p display!
```

---

## 4. Depth Buffer Precision

```
DEPTH PRECISION PROBLEM:
════════════════════════
  The z_ndc value is NOT linearly distributed between near and far!
  
  With near = 0.1, far = 100:
    Object at z = -1m   → z_ndc = 0.98    (uses 98% of depth range!)
    Object at z = -10m  → z_ndc = 0.998
    Object at z = -50m  → z_ndc = 0.9996
    Object at z = -100m → z_ndc = 1.0
    
  Almost ALL precision is packed into the first few meters!
  Objects 50-100m away share only 0.04% of the depth range.
  
  RESULT: Distant objects Z-fight (flicker between surfaces).
  
  FIX:
  1. Set near plane as FAR from camera as possible (0.5, not 0.01).
  2. Use "Reversed-Z" projection (near=1.0, far=0.0) for better distribution.
  3. Use logarithmic depth in the vertex shader.
```

---

## 5. JavaScript Implementation

```javascript
function perspectiveMatrix(fovy, aspect, near, far) {
    const f = 1.0 / Math.tan(fovy / 2);
    const rangeInv = 1.0 / (near - far);
    
    return new Float32Array([
        f / aspect,  0,    0,                          0,
        0,           f,    0,                          0,
        0,           0,    (far + near) * rangeInv,   -1,
        0,           0,    2 * far * near * rangeInv,  0
    ]);
}

function orthoMatrix(left, right, bottom, top, near, far) {
    return new Float32Array([
        2/(right-left),  0,              0,             0,
        0,               2/(top-bottom), 0,             0,
        0,               0,              -2/(far-near), 0,
        -(right+left)/(right-left),
        -(top+bottom)/(top-bottom),
        -(far+near)/(far-near),
        1
    ]);
}
```

---

## 6. Practice Problems

### Problem 1
An object is at z = -5 in view space. With near=1, far=100, what is its z_ndc approximately?

**Solution:**
```
z_ndc = (far + near)/(near - far) × z + (2 × far × near)/(near - far) / w
      = (101)/(-99) × (-5) + (200)/(-99) / 5
      = 5.101/1 + (-200)/(99×5)
Simplified: approximately z_ndc ≈ 0.79
This confirms most precision is in the near range.
```

### Problem 2
What happens if near = 0.001? Why is this bad?

**Solution:**
```
With near = 0.001 and far = 100:
  Depth ratio = far/near = 100/0.001 = 100,000
  This means 99.99% of the 24-bit depth buffer precision
  is wasted on the first 1 meter of distance.
  Objects beyond 10m will Z-fight terribly.
  
FIX: Use near = 0.1 at minimum. near = 0.5 for outdoor scenes.
```
