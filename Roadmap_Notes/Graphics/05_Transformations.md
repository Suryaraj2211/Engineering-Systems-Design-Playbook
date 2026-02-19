# Chapter 05 — Transformation Matrices

## Why Transformations?
Every object in a 3D scene starts at the world origin (0,0,0). Transformations move, rotate, and scale objects into their correct positions. Understanding the transformation pipeline is essential for building any 3D engine.

---

## 1. The Transformation Pipeline

```
VERTEX JOURNEY (Model to Screen):
══════════════════════════════════
  MODEL SPACE → WORLD SPACE → VIEW SPACE → CLIP SPACE → NDC → SCREEN

  Model Space:
    Vertex stored as authored (e.g., by Blender).
    A cube has vertices at (-1,-1,-1) to (1,1,1).
    
  World Space:
    Model Matrix moves the object into the game world.
    M = Translate(10, 0, -5) × Rotate(45°, Y) × Scale(2, 2, 2)
    Now the cube is at position (10, 0, -5), rotated 45°, scaled 2×.
    
  View Space (Eye/Camera Space):
    View Matrix positions everything relative to the camera.
    If camera is at (0, 5, 20) looking at origin:
    V = lookAt(eye=[0,5,20], target=[0,0,0], up=[0,1,0])
    All objects move so the camera is at the origin looking down -Z.
    
  Clip Space:
    Projection Matrix creates perspective or orthographic projection.
    P = perspective(fov=60°, aspect=16/9, near=0.1, far=100)
    Now objects have a W component for perspective divide.
    
  NDC (Normalized Device Coordinates):
    After dividing by W: all visible coordinates are in [-1, 1].
    
  Screen Space:
    Final pixel coordinates on the monitor:
    screenX = (ndcX + 1) / 2 × width
    screenY = (ndcY + 1) / 2 × height
```

---

## 2. Model Matrix — Constructing TRS

```javascript
// Build a Model Matrix from Transform, Rotate, Scale
function createModelMatrix(position, rotationAngle, scale) {
    // Translation
    const T = [
        1, 0, 0, 0,
        0, 1, 0, 0,
        0, 0, 1, 0,
        position[0], position[1], position[2], 1  // Column-major!
    ];
    
    // Rotation around Y axis
    const c = Math.cos(rotationAngle);
    const s = Math.sin(rotationAngle);
    const R = [
        c,  0, s, 0,
        0,  1, 0, 0,
       -s,  0, c, 0,
        0,  0, 0, 1
    ];
    
    // Scale
    const S = [
        scale[0], 0, 0, 0,
        0, scale[1], 0, 0,
        0, 0, scale[2], 0,
        0, 0, 0, 1
    ];
    
    return multiplyMatrices(T, multiplyMatrices(R, S)); // T × R × S
}
```

---

## 3. View Matrix — The LookAt Camera

### 3.1 Derivation from Scratch
```
LOOKAT MATRIX CONSTRUCTION:
═══════════════════════════
  Given:
    eye    = camera position = (0, 5, 10)
    target = what camera looks at = (0, 0, 0)
    up     = world up direction = (0, 1, 0)

  Step 1: Forward direction (camera looks BACKWARD in OpenGL!):
    forward = normalize(eye - target) = normalize(0, 5, 10) = (0, 0.447, 0.894)

  Step 2: Right direction:
    right = normalize(cross(up, forward))
          = normalize(cross((0,1,0), (0, 0.447, 0.894)))
          = normalize((0.894, 0, 0))   ← actually (1, 0, 0) after normalize

  Step 3: Recalculate true up:
    trueUp = cross(forward, right) = (0, 0.894, -0.447) → normalized

  Step 4: Build the matrix:
    View = | right.x    right.y    right.z    -dot(right, eye)   |
           | trueUp.x   trueUp.y   trueUp.z   -dot(trueUp, eye)  |
           | forward.x  forward.y  forward.z  -dot(forward, eye) |
           | 0          0          0           1                  |
```

### 3.2 JavaScript Implementation
```javascript
function lookAt(eye, target, up) {
    const forward = normalize(subtract(eye, target));
    const right = normalize(cross(up, forward));
    const trueUp = cross(forward, right);
    
    return new Float32Array([
        right[0],   trueUp[0],  forward[0], 0,
        right[1],   trueUp[1],  forward[1], 0,
        right[2],   trueUp[2],  forward[2], 0,
        -dot(right, eye), -dot(trueUp, eye), -dot(forward, eye), 1
    ]);
}
```

---

## 4. Homogeneous Coordinates — The W Component

### 4.1 Why We Need 4D Vectors for 3D Graphics

Translation CANNOT be represented by a 3×3 matrix. Proof:
```
3×3 matrix × (0,0,0) = (0,0,0) ALWAYS.
You cannot translate the origin with a 3×3 matrix!

SOLUTION: Add a 4th coordinate (w):
  Point:     (x, y, z, 1) → w=1 enables translation
  Direction: (x, y, z, 0) → w=0 ignores translation

| 1  0  0  tx |   | 0 |   | tx |    (point at origin → translated!) ✓
| 0  1  0  ty | × | 0 | = | ty |
| 0  0  1  tz |   | 0 |   | tz |
| 0  0  0   1 |   | 1 |   |  1 |

| 1  0  0  tx |   | dx |   | dx |   (direction → NOT translated!) ✓
| 0  1  0  ty | × | dy | = | dy |   Directions represent offsets,
| 0  0  1  tz |   | dz |   | dz |   not positions, so they ignore translation.
| 0  0  0   1 |   |  0 |   |  0 |
```

---

## 5. Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Wrong TRS order (S × R × T) | Object orbits world origin instead of spinning in place | Always use T × R × S |
| Row-major vs column-major confusion | Objects in wrong positions | WebGL/OpenGL uses column-major. DirectX uses row-major. |
| Forgetting to normalize after rotation | Accumulated floating-point error → stretched object | Re-normalize rotation basis vectors periodically |
| Not negating dot products in View matrix | Camera movements go backwards | `viewMatrix = -dot(axis, eye)` for the translation column |

---

## 6. Practice Problems

### Problem 1
Apply T(3, 0, 0) × S(2) to vertex (1, 1, 0).

**Solution:**
```
First S: (1×2, 1×2, 0×2) = (2, 2, 0)
Then T: (2+3, 2+0, 0+0) = (5, 2, 0)
```

### Problem 2
Build a LookAt matrix for camera at (0, 0, 5) looking at origin with up = (0, 1, 0).

**Solution:**
```
forward = normalize(eye - target) = normalize(0, 0, 5) = (0, 0, 1)
right = normalize(cross((0,1,0), (0,0,1))) = normalize((1,0,0)) = (1, 0, 0)
trueUp = cross((0,0,1), (1,0,0)) = (0, 1, 0)

View = | 1  0  0   0 |
       | 0  1  0   0 |
       | 0  0  1  -5 |
       | 0  0  0   1 |

This just translates everything by -5 in Z (moves world so camera is at origin).
```
