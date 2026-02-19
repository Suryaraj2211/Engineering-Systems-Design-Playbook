# Chapter 02 — Trigonometry for Graphics

## Why Trigonometry?
Every rotation, orbit, wave, and circular motion in graphics is built on trigonometry. If you can't convert between degrees and radians, or use sine/cosine to place objects on circles, your 3D engine will fail.

---

## 1. Degrees vs. Radians

### Definition
A **radian** measures angle by arc length on a unit circle (radius = 1).
One full rotation = circumference of unit circle = 2π ≈ 6.2832 radians = 360°.

### Conversion Formulas
```
Degrees → Radians: radians = degrees × (π / 180)
Radians → Degrees: degrees = radians × (180 / π)

MANUAL CALCULATIONS:
  90° → 90 × (π/180) = 90 × 0.01745 = 1.5708 rad  (= π/2)
  45° → 45 × (π/180) = 0.7854 rad  (= π/4)
  180° → 180 × (π/180) = 3.1416 rad  (= π)
  360° → 360 × (π/180) = 6.2832 rad  (= 2π)
  
  1 radian → 1 × (180/π) = 57.296°
```

### Common Angle Reference Table
```
ANGLE REFERENCE:
════════════════
  Degrees   Radians     sin        cos        tan
  ───────   ───────     ───        ───        ───
  0°        0           0.000      1.000      0.000
  30°       π/6         0.500      0.866      0.577
  45°       π/4         0.707      0.707      1.000
  60°       π/3         0.866      0.500      1.732
  90°       π/2         1.000      0.000      undefined
  180°      π           0.000     -1.000      0.000
  270°      3π/2       -1.000      0.000      undefined
  360°      2π          0.000      1.000      0.000
```

---

## 2. The Unit Circle

The unit circle is a circle of radius 1 centered at the origin. Every point on it can be expressed as:

$$x = \cos(\theta), \quad y = \sin(\theta)$$

```
THE UNIT CIRCLE (ASCII):
════════════════════════
                 (0, 1)
                   |   90°
                   |
     (-1, 0) ─────┼───── (1, 0)
      180°         |        0°
                   |
                 (0, -1)
                  270°

  At θ = 0°:    point = (cos0, sin0) = (1, 0)     → right
  At θ = 90°:   point = (cos90, sin90) = (0, 1)   → top
  At θ = 180°:  point = (cos180, sin180) = (-1, 0) → left
  At θ = 270°:  point = (cos270, sin270) = (0, -1) → bottom
  At θ = 45°:   point = (cos45, sin45) = (0.707, 0.707) → diagonal
```

---

## 3. Sine, Cosine, and Tangent — Manual Calculations

### From the Right Triangle

```
RIGHT TRIANGLE:
═══════════════
         ╱|
    h   ╱ |
  (hyp)╱  | o (opposite)
      ╱   |
     ╱ θ  |
    ╱─────┘
       a (adjacent)

  sin(θ) = opposite / hypotenuse = o/h
  cos(θ) = adjacent / hypotenuse = a/h
  tan(θ) = opposite / adjacent   = o/a

MANUAL CALCULATION:
  Triangle with sides 3, 4, 5:
    sin(θ) = 3/5 = 0.6     → θ ≈ 36.87°
    cos(θ) = 4/5 = 0.8
    tan(θ) = 3/4 = 0.75
    
  Verify: sin² + cos² = 0.6² + 0.8² = 0.36 + 0.64 = 1.00 ✓
  (Pythagorean identity: sin²θ + cos²θ = 1, ALWAYS)
```

---

## 4. Rotating a Point — Manual Computation

To rotate point (x, y) by angle θ around the origin:

$$x' = x \cos\theta - y \sin\theta$$
$$y' = x \sin\theta + y \cos\theta$$

### Worked Example: Rotate (3, 0) by 90°
```
STEP BY STEP:
  x = 3, y = 0, θ = 90°
  cos(90°) = 0, sin(90°) = 1
  
  x' = 3 × 0 - 0 × 1 = 0 - 0 = 0
  y' = 3 × 1 + 0 × 0 = 3 + 0 = 3
  
  Result: (0, 3)
  
  CHECK: (3, 0) is on the positive x-axis.
         Rotating 90° counter-clockwise puts it on the positive y-axis: (0, 3) ✓
```

### Worked Example: Rotate (1, 1) by 45°
```
  x = 1, y = 1, θ = 45°
  cos(45°) = 0.707, sin(45°) = 0.707
  
  x' = 1 × 0.707 - 1 × 0.707 = 0.707 - 0.707 = 0
  y' = 1 × 0.707 + 1 × 0.707 = 0.707 + 0.707 = 1.414
  
  Result: (0, 1.414)
  
  CHECK: (1, 1) is at 45° from x-axis, distance √2 ≈ 1.414 from origin.
         Rotating by another 45° puts it at 90° → straight up on y-axis.
         (0, 1.414) ✓ — length preserved!
```

---

## 5. Trigonometry in Graphics — Key Applications

### 5.1 Orbiting an Object
```javascript
// Make a light orbit around a point at radius 5
function updateLightPosition(time) {
    const angle = time * 0.5; // 0.5 radians per second
    const radius = 5.0;
    
    const x = Math.cos(angle) * radius;
    const z = Math.sin(angle) * radius;
    const y = 2.0; // Fixed height
    
    return [x, y, z]; // Circular orbit in the XZ plane
}
```

### 5.2 Wave Motion
```javascript
// Animate a wave (e.g., water surface)
function waveHeight(x, time) {
    return Math.sin(x * 2.0 + time) * 0.5; // Amplitude 0.5, frequency 2
}

// Multiple waves for realistic water:
function oceanHeight(x, z, time) {
    let h = 0;
    h += Math.sin(x * 1.0 + time * 0.8) * 0.3;  // Large slow wave
    h += Math.sin(z * 2.0 + time * 1.2) * 0.15;  // Medium wave
    h += Math.sin(x * 5.0 + z * 3.0 + time * 2.0) * 0.05; // Small ripple
    return h;
}
```

### 5.3 Field of View (FOV) to Focal Length
```
FOV and the projection matrix use tangent:

  fovY = 60° (vertical field of view)
  aspect = 16/9
  
  Focal length factor = 1 / tan(fovY / 2)
                      = 1 / tan(30°)
                      = 1 / 0.577
                      = 1.732

  This value becomes element [1][1] of the Perspective Projection matrix.
  Wider FOV → smaller focal length → more "fisheye" distortion.
```

---

## 6. Practice Problems (with Detailed Solutions)

### Problem 1
Convert 120° to radians.

**Solution:**
```
radians = 120 × (π/180) = 120 × 0.01745 = 2.094 rad
Exact: 120° = 2π/3 radians
```

### Problem 2
A satellite orbits at radius 10. Where is it at angle 150°?

**Solution:**
```
x = cos(150°) × 10 = cos(180° - 30°) × 10 = -cos(30°) × 10 = -0.866 × 10 = -8.66
y = sin(150°) × 10 = sin(180° - 30°) × 10 =  sin(30°) × 10 =  0.500 × 10 =  5.00

Position: (-8.66, 5.00)
```

### Problem 3
Rotate point (0, 5) by -90° (clockwise).

**Solution:**
```
cos(-90°) = 0, sin(-90°) = -1
x' = 0 × 0 - 5 × (-1) = 0 + 5 = 5
y' = 0 × (-1) + 5 × 0 = 0 + 0 = 0

Result: (5, 0) — rotated from top to right ✓
```

### Problem 4
What is sin(225°)?

**Solution:**
```
225° = 180° + 45° (third quadrant: both sin and cos are negative)
sin(225°) = -sin(45°) = -0.707
```

### Problem 5
An object bobs up and down with height = 2 × sin(time). At time = π/6 seconds, what is its height?

**Solution:**
```
height = 2 × sin(π/6) = 2 × 0.5 = 1.0
The object is 1 unit above its center position.
```
