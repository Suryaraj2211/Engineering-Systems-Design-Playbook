# Chapter 01 — Arithmetic & Algebra for Graphics

## Why Math for Graphics?
Every pixel on your screen is calculated using math. Moving a 3D object = matrix multiplication. Lighting = dot products. Perspective = division. **No shortcuts — understanding the math means understanding graphics.**

---

## 1. Number Systems

### Integers
Whole numbers: ..., -2, -1, 0, 1, 2, 3, ...
Used for: pixel coordinates, loop counters, buffer indices.

### Floating Point (Decimals)
Numbers with fractional parts: 3.14, -0.5, 1.0
Used for: **everything** in graphics — positions, colors, normals, UVs.

```
GPU stores colors as floats: Red = 1.0, Green = 0.5, Blue = 0.0
Screen pixel (255, 128, 0) → GPU sees (1.0, 0.502, 0.0)

Conversion:
  float = integer / 255
  128 / 255 = 0.50196...  (GPU stores this as 0.502 in float16)
  
  integer = float × 255
  0.502 × 255 = 128.01 → rounds to 128
```

### Floating-Point Precision — Why It Matters

```
FLOAT PRECISION TABLE:
══════════════════════
  Type      Bits    Decimal Digits    Range              GPU Usage
  ──────    ────    ──────────────    ─────              ─────────
  float16   16      ~3 digits         ±65,504            Colors, UVs, normals
  float32   32      ~7 digits         ±3.4×10³⁸          Positions, matrices
  float64   64      ~15 digits        ±1.8×10³⁰⁸         Not used on GPU

PRECISION DISASTER EXAMPLE:
  You place an object at position x = 100000.0 (100km from origin).
  float32 has ~7 digits of precision.
  Smallest representable step near 100000: ~0.01
  Your object can't be positioned more precisely than 1cm!
  
  At x = 1000000.0 (1000km): precision drops to ~0.1m
  → Objects visibly jitter when camera is far from origin.
  
  SOLUTION: Keep objects near the origin. Move the WORLD, not the camera.
```

---

## 2. Fractions & Decimals

### Fraction → Decimal
```
Manual Computation (long division):
  3/4 = 3 ÷ 4:
    Step 1: 4 goes into 3 zero times. Write 0.
    Step 2: Add decimal. 4 goes into 30 seven times (28). Remainder 2.
    Step 3: 4 goes into 20 five times (20). Remainder 0. Done.
    Answer: 0.75

  1/3 = 1 ÷ 3:
    3 goes into 10 three times (9). Remainder 1.
    3 goes into 10 three times (9). Remainder 1. (repeating!)
    Answer: 0.333...

  7/8 = 7 ÷ 8 = 0.875
  2/5 = 2 ÷ 5 = 0.4
```

### Graphics Connection: UV Coordinates
UV coordinates map pixels to fractions of texture size:
```
Texture is 512×512 pixels.
Pixel (256, 128) → UV = (256/512, 128/512) = (0.5, 0.25)

UV = (0, 0) → top-left corner of texture
UV = (1, 1) → bottom-right corner of texture
UV = (0.5, 0.5) → exact center of texture

What happens at UV = (1.5, 0.5)?
  Depends on WRAP MODE:
  - REPEAT: wraps → same as (0.5, 0.5) — tiling!
  - CLAMP:  clamps to (1.0, 0.5) — stretches the edge
  - MIRROR: reflects → same as (0.5, 0.5) — mirrored tiling
```

---

## 3. Powers & Roots

### Powers (Exponents)
```
MANUAL CALCULATION:
  2³ = 2 × 2 × 2 = 4 × 2 = 8
  5² = 5 × 5 = 25
  10⁰ = 1  (ANY number to power 0 = 1)
  2⁻¹ = 1/(2¹) = 1/2 = 0.5
  2⁻² = 1/(2²) = 1/4 = 0.25
  
  WHY NEGATIVE EXPONENTS MATTER IN GRAPHICS:
  Light attenuation = 1/distance² = distance⁻²
  A light 3 meters away: intensity = 1/3² = 1/9 = 0.111
  A light 10 meters away: intensity = 1/100 = 0.01 (100× dimmer!)
```

### Square Root
```
MANUAL ESTIMATION OF √50:
  7² = 49  (too low by 1)
  8² = 64  (too high by 14)
  7.07² = 49.98  (very close!)
  √50 ≈ 7.07

GRAPHICS-CRITICAL SQUARE ROOTS:
  √2 ≈ 1.414  (diagonal of a unit square)
  √3 ≈ 1.732  (diagonal of a unit cube)
  √(x² + y²) = distance from origin to point (x, y)
```

### Graphics Connection: Gamma Correction
```
GAMMA CORRECTION:
  Your monitor does NOT display light linearly.
  Input value 0.5 does NOT produce 50% brightness.
  
  Monitor applies: brightness = input^2.2 (gamma curve)
  So 0.5^2.2 = 0.217 → only 22% brightness!
  
  To fix: pre-correct with inverse gamma:
  corrected = linear^(1/2.2) = linear^0.4545
  
  MANUAL CALCULATION:
  linear = 0.5
  corrected = 0.5^0.4545
  
  Step 1: ln(0.5) = -0.693
  Step 2: -0.693 × 0.4545 = -0.315
  Step 3: e^(-0.315) = 0.730
  
  So 0.5 linear → 0.730 on screen → monitor shows 0.730^2.2 ≈ 0.5 ✓
```

---

## 4. Algebra Basics

### Variables & Expressions
```
A variable holds a value: x = 5, position = 3.2

EVALUATING EXPRESSIONS:
  3x + 2    (when x=4) → 3×4 + 2 = 12 + 2 = 14
  x² - 1    (when x=3) → 3² - 1 = 9 - 1 = 8
  2a + 3b   (when a=2, b=5) → 4 + 15 = 19
```

### Solving Linear Equations
```
STEP-BY-STEP:
  Problem: 2x + 3 = 11
  Step 1: Subtract 3 from both sides: 2x = 8
  Step 2: Divide both sides by 2: x = 4
  Check: 2(4) + 3 = 8 + 3 = 11 ✓

  Problem: 5x - 7 = 3x + 9
  Step 1: Subtract 3x from both sides: 2x - 7 = 9
  Step 2: Add 7 to both sides: 2x = 16
  Step 3: Divide by 2: x = 8
  Check: 5(8) - 7 = 33, 3(8) + 9 = 33 ✓
```

### Graphics Connection: Linear Interpolation (LERP)

The most important formula in all of graphics:

$$\text{lerp}(a, b, t) = a + t \times (b - a) = a(1-t) + bt$$

where $t$ ranges from 0.0 to 1.0.

```
MANUAL LERP CALCULATION:
════════════════════════
  Blend between red (1,0,0) and blue (0,0,1) at t = 0.3:
  
  R = 1.0 + 0.3 × (0.0 - 1.0) = 1.0 + 0.3 × (-1.0) = 1.0 - 0.3 = 0.7
  G = 0.0 + 0.3 × (0.0 - 0.0) = 0.0 + 0.0 = 0.0
  B = 0.0 + 0.3 × (1.0 - 0.0) = 0.0 + 0.3 = 0.3
  
  Result: (0.7, 0.0, 0.3) — a reddish-purple

  USES IN GRAPHICS:
  - Color blending between two colors
  - Position interpolation for animation
  - Texture coordinate interpolation across a triangle
  - Camera path smoothing
  - LOD (Level of Detail) transitions
```

---

## 5. Functions

A function takes input and produces output: f(x) = 2x + 1

```
EVALUATION:
  f(0) = 2(0) + 1 = 1
  f(3) = 2(3) + 1 = 7
  f(-1) = 2(-1) + 1 = -1
```

### Essential Graphics Functions

```javascript
// CLAMP: Restrict value to a range
function clamp(value, min, max) {
    return Math.max(min, Math.min(max, value));
}
// clamp(1.5, 0, 1) = 1.0
// clamp(-0.3, 0, 1) = 0.0
// Used to prevent colors going negative or above 1.0

// SMOOTHSTEP: Smooth S-curve interpolation
function smoothstep(edge0, edge1, x) {
    let t = clamp((x - edge0) / (edge1 - edge0), 0, 1);
    return t * t * (3 - 2 * t);
}
// smoothstep(0, 1, 0.0) = 0.0
// smoothstep(0, 1, 0.5) = 0.5  (but with smooth acceleration/deceleration)
// smoothstep(0, 1, 1.0) = 1.0

// STEP: Hard cutoff (like a switch)
function step(edge, x) {
    return x < edge ? 0.0 : 1.0;
}
// step(0.5, 0.3) = 0.0
// step(0.5, 0.7) = 1.0
```

---

## 6. Coordinate Systems

### 2D Coordinates (x, y)
```
      +y (up)
       ↑
       |
       |    • (3, 2)
       |
───────┼──────→ +x (right)
       |
  •(-2,-1)
       |
      -y (down)
```

### Screen vs. Math Coordinates
```
MATH COORDINATES:         SCREEN COORDINATES:
  y-axis goes UP            y-axis goes DOWN
  (0,0) at center           (0,0) at top-left

  Math:                     Screen (HTML Canvas):
    +y ↑                      (0,0)──→ +x
       |                        |
       |                        ↓
    ───┼──→ +x                 +y

CRITICAL: WebGL uses MATH coordinates (y-up, origin at center).
          HTML Canvas uses SCREEN coordinates (y-down, origin at top-left).
          This causes FLIPPED TEXTURES if not handled!
          
FIX: Flip the y-coordinate when loading textures:
     gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true);
```

### 3D Coordinates (x, y, z)
```
      +y (up)
       ↑
       |   / +z (forward — WebGL RIGHT-HAND RULE)
       |  /
       | /
       ┼──────→ +x (right)
       
RIGHT-HAND RULE:
  Point your right-hand fingers along +x.
  Curl them toward +y.
  Your thumb points in the +z direction.
  
  In WebGL: +z points TOWARD the camera (out of screen).
  In DirectX: +z points INTO the screen (left-hand system).
```

---

## 7. Practice Problems (with Detailed Solutions)

### Problem 1
Convert pixel (200, 100) to UV coordinates for a 400×200 texture.

**Solution:**
```
u = x / width  = 200 / 400 = 0.5
v = y / height = 100 / 200 = 0.5
UV = (0.5, 0.5) — the exact center of the texture
```

### Problem 2
Compute lerp(10, 30, 0.75).

**Solution:**
```
lerp(a, b, t) = a + t × (b - a)
= 10 + 0.75 × (30 - 10)
= 10 + 0.75 × 20
= 10 + 15
= 25
```

### Problem 3
Distance between points (1, 2) and (4, 6)?

**Solution:**
```
d = √((x₂-x₁)² + (y₂-y₁)²)
  = √((4-1)² + (6-2)²)
  = √(3² + 4²)
  = √(9 + 16)
  = √25
  = 5

This is a 3-4-5 right triangle! A common pattern in graphics.
```

### Problem 4
A point light has intensity 100 at distance 1m. What is its intensity at 5m?

**Solution:**
```
Intensity follows inverse-square law: I = base / d²
At d = 1:  I = 100 / 1² = 100
At d = 5:  I = 100 / 5² = 100 / 25 = 4

The light is 25× dimmer at 5 meters!
```

### Problem 5
What is 0.8^(1/2.2) (gamma correction for 80% brightness)?

**Solution:**
```
0.8^(1/2.2) = 0.8^0.4545
Using logarithms:
  ln(0.8) = -0.2231
  -0.2231 × 0.4545 = -0.1015
  e^(-0.1015) = 0.9035

Answer: ≈ 0.904
So 80% linear brightness appears as ~90% on a gamma-corrected screen.
```
