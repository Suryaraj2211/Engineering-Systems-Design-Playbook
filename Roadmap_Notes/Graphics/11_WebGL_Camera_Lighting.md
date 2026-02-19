# Chapter 11 — WebGL Camera & Lighting

---

## 1. Camera System Implementation

```javascript
class Camera {
    constructor(fov, aspect, near, far) {
        this.position = [0, 2, 10];
        this.target = [0, 0, 0];
        this.up = [0, 1, 0];
        this.fov = fov * Math.PI / 180; // Convert to radians
        this.aspect = aspect;
        this.near = near;
        this.far = far;
        this.yaw = -90;   // Horizontal rotation (degrees)
        this.pitch = 0;    // Vertical rotation (degrees)
    }
    
    getViewMatrix() {
        return lookAt(this.position, this.target, this.up);
    }
    
    getProjectionMatrix() {
        return perspectiveMatrix(this.fov, this.aspect, this.near, this.far);
    }
    
    // FPS-style camera: WASD to move, mouse to look
    processMouseMovement(dx, dy, sensitivity = 0.1) {
        this.yaw   += dx * sensitivity;
        this.pitch -= dy * sensitivity;
        this.pitch = Math.max(-89, Math.min(89, this.pitch)); // Prevent flipping
        
        // Update target direction from yaw/pitch
        const yawRad = this.yaw * Math.PI / 180;
        const pitchRad = this.pitch * Math.PI / 180;
        this.target = [
            this.position[0] + Math.cos(pitchRad) * Math.cos(yawRad),
            this.position[1] + Math.sin(pitchRad),
            this.position[2] + Math.cos(pitchRad) * Math.sin(yawRad)
        ];
    }
    
    moveForward(speed) {
        const dir = normalize(subtract(this.target, this.position));
        this.position[0] += dir[0] * speed;
        this.position[1] += dir[1] * speed;
        this.position[2] += dir[2] * speed;
        this.target[0] += dir[0] * speed;
        this.target[1] += dir[1] * speed;
        this.target[2] += dir[2] * speed;
    }
}
```

---

## 2. Phong Lighting Model

### 2.1 The Three Components

```
PHONG LIGHTING:
═══════════════
  Final Color = Ambient + Diffuse + Specular
  
  AMBIENT: Constant base illumination (simulates indirect light).
    ambient = ambientStrength × lightColor × objectColor
    = 0.1 × (1,1,1) × (1, 0, 0) = (0.1, 0, 0)  (dim red)
    
  DIFFUSE: Light hitting the surface (Lambert's cosine law).
    NdotL = max(dot(Normal, LightDir), 0.0)
    diffuse = NdotL × lightColor × objectColor
    
    Example: N = (0,1,0), L = (0.707, 0.707, 0) (light from upper-right)
    NdotL = 0×0.707 + 1×0.707 + 0×0 = 0.707
    diffuse = 0.707 × (1,1,1) × (1,0,0) = (0.707, 0, 0)
    
  SPECULAR: Shiny highlight (view-dependent).
    R = reflect(-L, N)      // Reflection of light direction
    spec = pow(max(dot(R, V), 0.0), shininess) × lightColor
    
    Higher shininess → smaller, tighter highlight (metal).
    Lower shininess → large, soft highlight (plastic).
```

### 2.2 Complete Phong Fragment Shader

```glsl
#version 300 es
precision highp float;

in vec3 v_normal;
in vec3 v_fragPos;

uniform vec3 u_lightPos;
uniform vec3 u_viewPos;
uniform vec3 u_lightColor;
uniform vec3 u_objectColor;

out vec4 fragColor;

void main() {
    // Normalize interpolated normal
    vec3 N = normalize(v_normal);
    vec3 L = normalize(u_lightPos - v_fragPos);
    vec3 V = normalize(u_viewPos - v_fragPos);
    vec3 R = reflect(-L, N);
    
    // Ambient
    float ambientStrength = 0.1;
    vec3 ambient = ambientStrength * u_lightColor;
    
    // Diffuse
    float diff = max(dot(N, L), 0.0);
    vec3 diffuse = diff * u_lightColor;
    
    // Specular (Blinn-Phong uses halfway vector — more efficient)
    vec3 H = normalize(L + V);
    float spec = pow(max(dot(N, H), 0.0), 32.0);
    vec3 specular = spec * u_lightColor * 0.5;
    
    // Combine
    vec3 result = (ambient + diffuse + specular) * u_objectColor;
    fragColor = vec4(result, 1.0);
}
```

---

## 3. Point Light Attenuation

```glsl
// Real lights get dimmer with distance (inverse-square law)
float distance = length(u_lightPos - v_fragPos);
float attenuation = 1.0 / (1.0 + 0.09 * distance + 0.032 * distance * distance);

// Apply attenuation to diffuse and specular (NOT ambient)
vec3 result = ambient + (diffuse + specular) * attenuation;
```

```
ATTENUATION VALUES:
═══════════════════
  Distance    Attenuation (quadratic)    Light Remaining
  ────────    ───────────────────────    ───────────────
  1m          1.0 / 1.122 = 0.89        89%
  5m          1.0 / 2.25  = 0.44        44%
  10m         1.0 / 5.1   = 0.20        20%
  20m         1.0 / 14.6  = 0.07        7%
  50m         1.0 / 85.5  = 0.01        1%
```

---

## 4. Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Normal not normalized in fragment | Lighting varies unpredictably | `normalize(v_normal)` in fragment shader |
| Light direction reversed | Lit side appears shadowed | Use `lightPos - fragPos`, not reverse |
| Missing view-space/world-space consistency | Specular highlights in wrong position | Transform ALL vectors to same space |
| Ambient too high | Scene looks flat, no contrast | Keep ambient 0.05-0.15 |
| Negative NdotL not clamped | Dark side of objects becomes negative (subtracts light) | `max(dot(N,L), 0.0)` |
