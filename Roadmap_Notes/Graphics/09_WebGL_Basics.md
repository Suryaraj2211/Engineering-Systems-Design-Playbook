# Chapter 09 — WebGL Basics

## What is WebGL?
WebGL is a JavaScript API for rendering 2D/3D graphics in the browser using the GPU. WebGL 1 is based on OpenGL ES 2.0; WebGL 2 on OpenGL ES 3.0.

---

## 1. Getting the Context

```html
<canvas id="canvas" width="800" height="600"></canvas>
<script>
const canvas = document.getElementById('canvas');

// Try WebGL 2 first (better features), fallback to WebGL 1
const gl = canvas.getContext('webgl2') || canvas.getContext('webgl');
if (!gl) { alert('WebGL not supported!'); }

// CRITICAL: Match canvas resolution to CSS size to avoid blurriness
canvas.width = canvas.clientWidth * window.devicePixelRatio;
canvas.height = canvas.clientHeight * window.devicePixelRatio;
gl.viewport(0, 0, canvas.width, canvas.height);

// Set clear color and clear
gl.clearColor(0.1, 0.1, 0.15, 1.0); // Dark background
gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
</script>
```

---

## 2. Shader Programs — Complete Workflow

```javascript
// STEP 1: Write shaders as strings
const vertexShaderSource = `#version 300 es
    layout(location = 0) in vec2 a_position;
    layout(location = 1) in vec3 a_color;
    
    out vec3 v_color; // Pass to fragment shader
    
    void main() {
        v_color = a_color;
        gl_Position = vec4(a_position, 0.0, 1.0);
    }
`;

const fragmentShaderSource = `#version 300 es
    precision mediump float;
    
    in vec3 v_color;
    out vec4 fragColor;
    
    void main() {
        fragColor = vec4(v_color, 1.0);
    }
`;

// STEP 2: Compile a single shader
function createShader(gl, type, source) {
    const shader = gl.createShader(type);
    gl.shaderSource(shader, source);
    gl.compileShader(shader);
    
    // ALWAYS check for errors!
    if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
        console.error('Shader compile error:', gl.getShaderInfoLog(shader));
        gl.deleteShader(shader);
        return null;
    }
    return shader;
}

// STEP 3: Link vertex + fragment into a program
function createProgram(gl, vsSource, fsSource) {
    const vs = createShader(gl, gl.VERTEX_SHADER, vsSource);
    const fs = createShader(gl, gl.FRAGMENT_SHADER, fsSource);
    const program = gl.createProgram();
    gl.attachShader(program, vs);
    gl.attachShader(program, fs);
    gl.linkProgram(program);
    
    if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
        console.error('Program link error:', gl.getProgramInfoLog(program));
        return null;
    }
    return program;
}

const program = createProgram(gl, vertexShaderSource, fragmentShaderSource);
```

### What Happens Internally
```
SHADER COMPILATION INSIDE THE GPU:
══════════════════════════════════
  1. gl.shaderSource → Stores GLSL source text on GPU driver.
  2. gl.compileShader → GPU driver COMPILES text to GPU machine code.
     This is CPU-expensive (~1-10ms per shader).
     DO NOT COMPILE SHADERS EVERY FRAME.
  3. gl.linkProgram → Links vertex + fragment shaders together.
     Validates that outputs of vertex match inputs of fragment.
  4. gl.useProgram → GPU activates this compiled program for subsequent draws.
```

---

## 3. Drawing a Colored Triangle

```javascript
// Interleaved vertex data: position (2D) + color (RGB)
const vertices = new Float32Array([
//  x      y      r    g    b
    0.0,   0.5,   1.0, 0.0, 0.0,  // Top vertex (red)
   -0.5,  -0.5,   0.0, 1.0, 0.0,  // Bottom-left (green)
    0.5,  -0.5,   0.0, 0.0, 1.0   // Bottom-right (blue)
]);

// Create VAO (Vertex Array Object) — stores all attribute state
const vao = gl.createVertexArray();
gl.bindVertexArray(vao);

// Create buffer and upload data
const buffer = gl.createBuffer();
gl.bindBuffer(gl.ARRAY_BUFFER, buffer);
gl.bufferData(gl.ARRAY_BUFFER, vertices, gl.STATIC_DRAW);

// Configure attribute 0: position (2 floats, stride=20 bytes, offset=0)
gl.enableVertexAttribArray(0);
gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 5 * 4, 0);
//                     loc size type  norm stride  offset

// Configure attribute 1: color (3 floats, stride=20 bytes, offset=8)
gl.enableVertexAttribArray(1);
gl.vertexAttribPointer(1, 3, gl.FLOAT, false, 5 * 4, 2 * 4);

// Draw!
gl.useProgram(program);
gl.bindVertexArray(vao);
gl.drawArrays(gl.TRIANGLES, 0, 3); // 3 vertices = 1 triangle
```

### Understanding Stride and Offset
```
MEMORY LAYOUT (interleaved buffer):
═══════════════════════════════════
  Byte:  0    4    8    12   16   20   24   28   32   36   40   44   48   52   56
         x₀   y₀   r₀   g₀   b₀   x₁   y₁   r₁   g₁   b₁   x₂   y₂   r₂   g₂   b₂
         └─── pos ──┘└── color ──┘  └── pos ──┘└── color ──┘
              │                         │
              ├─── stride = 20 bytes ──►┤
              
  Position attribute: size=2, stride=20, offset=0  (starts at byte 0)
  Color attribute:    size=3, stride=20, offset=8  (starts at byte 8)
```

---

## 4. Attributes vs. Uniforms

| | Attribute | Uniform |
|--|----------|---------|
| Data | Different per vertex | Same for ALL vertices in a draw call |
| Set by | Buffer + `vertexAttribPointer` | `gl.uniform*()` calls |
| Example | position, color, UV, normal | MVP matrix, time, lightPos |
| GLSL keyword | `in` (WebGL2) or `attribute` (WebGL1) | `uniform` |

```javascript
// Setting uniforms
gl.useProgram(program); // MUST be called BEFORE setting uniforms!

const timeLoc = gl.getUniformLocation(program, 'u_time');
gl.uniform1f(timeLoc, performance.now() / 1000.0);

const colorLoc = gl.getUniformLocation(program, 'u_color');
gl.uniform3f(colorLoc, 1.0, 0.5, 0.0); // orange

const matLoc = gl.getUniformLocation(program, 'u_mvp');
gl.uniformMatrix4fv(matLoc, false, mvpMatrix); // 4×4 matrix
```

---

## 5. The Render Loop

```javascript
function render(time) {
    time *= 0.001; // Convert ms to seconds
    
    // Resize canvas to match CSS dimensions
    if (canvas.width !== canvas.clientWidth || canvas.height !== canvas.clientHeight) {
        canvas.width = canvas.clientWidth * devicePixelRatio;
        canvas.height = canvas.clientHeight * devicePixelRatio;
        gl.viewport(0, 0, canvas.width, canvas.height);
    }
    
    // Clear
    gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
    
    // Update uniforms
    gl.useProgram(program);
    gl.uniform1f(timeLoc, time);
    
    // Draw
    gl.bindVertexArray(vao);
    gl.drawArrays(gl.TRIANGLES, 0, 3);
    
    requestAnimationFrame(render); // Loop at ~60fps
}
requestAnimationFrame(render);
```

---

## 6. Common Mistakes & Debugging

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Wrong attribute size in `vertexAttribPointer` | Garbled geometry, vertices in wrong places | Match size to data (2 for vec2, 3 for vec3) |
| Missing `gl.useProgram()` before uniforms | Uniforms silently fail, nothing draws | Always call `useProgram` before setting uniforms |
| `gl.enableVertexAttribArray()` not called | Black screen (attributes read as 0) | Enable every attribute you use |
| Not checking compile errors | Silent failure, blank screen | Always check `getShaderInfoLog` |
| Canvas CSS size ≠ buffer size | Blurry rendering | Multiply by `devicePixelRatio` |
| Missing depth test enable | Objects overlap randomly | `gl.enable(gl.DEPTH_TEST)` |

```javascript
// Debug: Check for WebGL errors
function checkGLError(gl, label) {
    const err = gl.getError();
    if (err !== gl.NO_ERROR) {
        const errors = {
            [gl.INVALID_ENUM]: 'INVALID_ENUM',
            [gl.INVALID_VALUE]: 'INVALID_VALUE',
            [gl.INVALID_OPERATION]: 'INVALID_OPERATION',
            [gl.OUT_OF_MEMORY]: 'OUT_OF_MEMORY',
        };
        console.error(`WebGL Error at ${label}: ${errors[err] || err}`);
    }
}
```
