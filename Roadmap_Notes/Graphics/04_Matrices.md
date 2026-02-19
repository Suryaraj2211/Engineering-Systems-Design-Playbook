# Chapter 04 — Matrices

## Why Matrices?
Matrices encode transformations: rotation, scaling, translation, projection. A single 4×4 matrix can describe the ENTIRE journey of a vertex from model space to screen pixels.

---

## 1. What Is a Matrix?

A matrix is a rectangular grid of numbers. In graphics, we use 2×2, 3×3, and 4×4 matrices.

```
2×2 MATRIX:      3×3 MATRIX:          4×4 MATRIX:
| a  b |         | a  b  c |         | a  b  c  d |
| c  d |         | d  e  f |         | e  f  g  h |
                 | g  h  i |         | i  j  k  l |
                                     | m  n  o  p |
```

### Matrix Notation
```
ELEMENT ACCESS:
  M[row][column]
  
  | 1  2  3 |
  | 4  5  6 |    M[0][0] = 1, M[0][2] = 3, M[1][1] = 5
  | 7  8  9 |    M[2][0] = 7
```

---

## 2. Matrix Multiplication — Step by Step

### The Rule
To multiply A (m×n) by B (n×p), each element of the result C is the **dot product** of a ROW from A with a COLUMN from B.

### 2×2 Example (Complete Manual Calculation)
```
A = | 1  2 |    B = | 5  6 |
    | 3  4 |        | 7  8 |

C[0][0] = Row0(A) · Col0(B) = 1×5 + 2×7 = 5 + 14 = 19
C[0][1] = Row0(A) · Col1(B) = 1×6 + 2×8 = 6 + 16 = 22
C[1][0] = Row1(A) · Col0(B) = 3×5 + 4×7 = 15 + 28 = 43
C[1][1] = Row1(A) · Col1(B) = 3×6 + 4×8 = 18 + 32 = 50

C = | 19  22 |
    | 43  50 |
```

### 4×4 Matrix × Vector (The Most Common Graphics Operation)
```
TRANSFORM A VERTEX BY A MATRIX:
═══════════════════════════════
  M = | 1  0  0  5 |      v = | 3 |
      | 0  1  0  2 |          | 1 |
      | 0  0  1 -3 |          | 0 |
      | 0  0  0  1 |          | 1 |   ← w component (always 1 for points)
  
  Result[0] = 1×3 + 0×1 + 0×0 + 5×1 = 3 + 5 = 8
  Result[1] = 0×3 + 1×1 + 0×0 + 2×1 = 1 + 2 = 3
  Result[2] = 0×3 + 0×1 + 1×0 + (-3)×1 = 0 - 3 = -3
  Result[3] = 0×3 + 0×1 + 0×0 + 1×1 = 1
  
  Result = (8, 3, -3, 1)
  
  This matrix TRANSLATED the point (3,1,0) by (5, 2, -3) → (8, 3, -3)!
```

### CRITICAL: Multiplication Order Matters!
```
A × B ≠ B × A  (matrices are NOT commutative!)

GRAPHICS CONVENTION (column-major, OpenGL/WebGL):
  Final = Projection × View × Model × vertex
  
  Read RIGHT to LEFT:
  1. Apply Model transform (move/rotate/scale the object)
  2. Apply View transform (move the camera)
  3. Apply Projection (create perspective)
```

---

## 3. The Identity Matrix

The identity matrix does nothing — like multiplying by 1:

```
I = | 1  0  0  0 |
    | 0  1  0  0 |
    | 0  0  1  0 |
    | 0  0  0  1 |

For any matrix M:  M × I = M  and  I × M = M
```

---

## 4. Common Graphics Matrices

### Translation Matrix (Move an Object)
```
T(tx, ty, tz) = | 1  0  0  tx |
                | 0  1  0  ty |
                | 0  0  1  tz |
                | 0  0  0   1 |

EXAMPLE: Translate (1, 2, 3) by (10, 0, -5):

| 1  0  0  10 |   | 1 |     |1×1 + 0×2 + 0×3 + 10×1|     | 11 |
| 0  1  0   0 | × | 2 |  =  |0×1 + 1×2 + 0×3 +  0×1|  =  |  2 |
| 0  0  1  -5 |   | 3 |     |0×1 + 0×2 + 1×3 + (-5)×1|   | -2 |
| 0  0  0   1 |   | 1 |     |0×1 + 0×2 + 0×3 +  1×1|     |  1 |
```

### Scale Matrix (Resize an Object)
```
S(sx, sy, sz) = | sx  0   0  0 |
                |  0  sy  0  0 |
                |  0  0  sz  0 |
                |  0  0   0  1 |

EXAMPLE: Scale (2, 3, 4) by factor 2 in all axes:

| 2  0  0  0 |   | 2 |   | 4 |
| 0  2  0  0 | × | 3 | = | 6 |
| 0  0  2  0 |   | 4 |   | 8 |
| 0  0  0  1 |   | 1 |   | 1 |
```

### Rotation Matrix (Around Z-axis for 2D)
```
Rz(θ) = | cosθ  -sinθ  0  0 |
        | sinθ   cosθ  0  0 |
        |   0      0   1  0 |
        |   0      0   0  1 |

EXAMPLE: Rotate (1, 0, 0) by 90° around Z:
  cos90 = 0, sin90 = 1

| 0  -1  0  0 |   | 1 |   | 0×1 + (-1)×0 |   | 0 |
| 1   0  0  0 | × | 0 | = | 1×1 +  0×0   | = | 1 |
| 0   0  1  0 |   | 0 |   | 0             |   | 0 |
| 0   0  0  1 |   | 1 |   | 1             |   | 1 |

(1,0,0) rotated 90° → (0,1,0) ✓ (moves from right to up)
```

---

## 5. Combining Transformations

### The TRS Order (Translate × Rotate × Scale)
```
CORRECT ORDER: Final = T × R × S × vertex
  Step 1 (rightmost first): Scale the vertex
  Step 2: Rotate the scaled vertex
  Step 3: Translate the rotated, scaled vertex

WRONG ORDER: Final = S × R × T × vertex
  Step 1: Translate first → vertex moves to (10, 0, 0)
  Step 2: Rotate → rotates around ORIGIN, not around object center!
  Step 3: Scale → scales relative to origin, stretching incorrectly
  
  RESULT: Object orbits the world origin instead of spinning in place!
```

### Manual Combined Example
```
Scale by 2, then translate by (5, 0, 0):

S = | 2  0  0  0 |    T = | 1  0  0  5 |
    | 0  2  0  0 |        | 0  1  0  0 |
    | 0  0  2  0 |        | 0  0  1  0 |
    | 0  0  0  1 |        | 0  0  0  1 |

Combined M = T × S:
  M[0][0] = 1×2 + 0×0 + 0×0 + 0×0 = 2
  M[0][3] = 1×0 + 0×0 + 0×0 + 5×1 = 5
  ... (identity portions carry through)

M = | 2  0  0  5 |
    | 0  2  0  0 |
    | 0  0  2  0 |
    | 0  0  0  1 |

Apply to vertex (1, 1, 0):
  Result = | 2×1 + 5 | = | 7 |
           | 2×1 + 0 |   | 2 |
           | 2×0 + 0 |   | 0 |
           | 1       |   | 1 |

CHECK: (1,1,0) scaled by 2 → (2,2,0), then translated by (5,0,0) → (7,2,0) ✓
```

---

## 6. Practice Problems

### Problem 1
Multiply: A = [2, 3; 1, 4] × B = [1, 0; 0, 1]

**Solution:** B is the identity matrix. A × I = A. Answer: [2, 3; 1, 4]

### Problem 2
Transform point (0, 0, 0) by translation T(3, -2, 7).

**Solution:**
```
| 1  0  0  3 |   | 0 |   | 3 |
| 0  1  0 -2 | × | 0 | = |-2 |
| 0  0  1  7 |   | 0 |   | 7 |
| 0  0  0  1 |   | 1 |   | 1 |
Answer: (3, -2, 7)
```

### Problem 3
Scale point (5, 10, 15) by (0.5, 0.5, 0.5).

**Solution:**
```
Each coordinate × 0.5: (2.5, 5.0, 7.5)
This halves the object's size.
```

### Problem 4
Rotate (1, 0, 0) by 45° around the Z-axis.

**Solution:**
```
cos45 = 0.707, sin45 = 0.707
x' = 1×0.707 - 0×0.707 = 0.707
y' = 1×0.707 + 0×0.707 = 0.707
Result: (0.707, 0.707, 0) — 45° diagonal ✓
```

### Problem 5
Why does S × T ≠ T × S?

**Solution:**
```
S(2) × T(5, 0, 0) applied to origin (0, 0, 0):
  S × T = scale by 2, THEN translate
  But S scales the TRANSLATION! T becomes (10, 0, 0).
  Final: (10, 0, 0) — WRONG (translated too far)
  
T(5, 0, 0) × S(2) applied to origin:
  T × S = translate by (5,0,0), THEN scale object
  Translation is unaffected by scale.
  Final for origin: (5, 0, 0) — correct
  
Matrix multiplication is NOT commutative. Order MATTERS.
```
