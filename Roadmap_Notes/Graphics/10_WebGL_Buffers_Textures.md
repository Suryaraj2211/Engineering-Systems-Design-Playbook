# Chapter 10 — WebGL Buffers & Textures

---

## 1. Index Buffers — Drawing with Shared Vertices

A cube has 8 vertices but 36 indices (6 faces × 2 triangles × 3 vertices).
Without indexing, you'd duplicate vertices: 36 × 3 floats = 108 floats.
With indexing: 8 × 3 floats + 36 indices = 24 + 36 = 60 values. **44% savings.**

```javascript
const positions = new Float32Array([
    -1, -1,  1,   // 0: front-bottom-left
     1, -1,  1,   // 1: front-bottom-right
     1,  1,  1,   // 2: front-top-right
    -1,  1,  1,   // 3: front-top-left
    -1, -1, -1,   // 4: back-bottom-left
     1, -1, -1,   // 5: back-bottom-right
     1,  1, -1,   // 6: back-top-right
    -1,  1, -1    // 7: back-top-left
]);

// Each face = 2 triangles. Winding order: Counter-Clockwise.
const indices = new Uint16Array([
    0, 1, 2, 0, 2, 3,  // Front face
    1, 5, 6, 1, 6, 2,  // Right face
    5, 4, 7, 5, 7, 6,  // Back face
    4, 0, 3, 4, 3, 7,  // Left face
    3, 2, 6, 3, 6, 7,  // Top face
    4, 5, 1, 4, 1, 0   // Bottom face
]);

// Upload index buffer
const indexBuffer = gl.createBuffer();
gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, indexBuffer);
gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, indices, gl.STATIC_DRAW);

// Draw with indices
gl.drawElements(gl.TRIANGLES, 36, gl.UNSIGNED_SHORT, 0);
```

---

## 2. Textures — Complete Setup

```javascript
// STEP 1: Create and configure texture
const texture = gl.createTexture();
gl.bindTexture(gl.TEXTURE_2D, texture);

// Set filtering (what happens when texture is scaled)
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR_MIPMAP_LINEAR);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.REPEAT);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.REPEAT);

// STEP 2: Load image
const image = new Image();
image.onload = () => {
    gl.bindTexture(gl.TEXTURE_2D, texture);
    // Flip Y because WebGL origin is bottom-left, images are top-left
    gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true);
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, image);
    gl.generateMipmap(gl.TEXTURE_2D);  // Create smaller versions for distant viewing
};
image.src = 'texture.png';

// STEP 3: Use texture in shader
gl.activeTexture(gl.TEXTURE0);  // Activate texture slot 0
gl.bindTexture(gl.TEXTURE_2D, texture);
gl.uniform1i(gl.getUniformLocation(program, 'u_texture'), 0);  // Sampler uses slot 0
```

### Texture Filtering Explained
```
TEXTURE FILTERING:
══════════════════
  NEAREST (gl.NEAREST):
    Picks the SINGLE closest texel. Looks pixelated up close.
    Used for: pixel art, retro games.
    
  LINEAR (gl.LINEAR / Bilinear):
    Averages 4 nearest texels. Smooth but slightly blurry.
    Used for: most 3D textures.
    
  LINEAR_MIPMAP_LINEAR (Trilinear):
    Bilinear on 2 mip levels, then interpolates between them.
    Smoothest. Used for: production quality.
    
  MIPMAPS:
    Pre-generated smaller copies: 512→256→128→64→32→16→8→4→2→1
    When texture is far away, GPU samples smaller mip → fewer artifacts.
    Cost: 33% extra VRAM (1/4 + 1/16 + 1/64 + ... ≈ 1/3 extra).
```

---

## 3. Framebuffer Objects (FBO) — Rendering to Texture

```javascript
// Create a framebuffer for off-screen rendering
const fb = gl.createFramebuffer();
gl.bindFramebuffer(gl.FRAMEBUFFER, fb);

// Create color attachment (texture to render INTO)
const colorTex = gl.createTexture();
gl.bindTexture(gl.TEXTURE_2D, colorTex);
gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, 512, 512, 0, gl.RGBA, gl.UNSIGNED_BYTE, null);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, colorTex, 0);

// Create depth attachment (required for 3D scenes!)
const depthRB = gl.createRenderbuffer();
gl.bindRenderbuffer(gl.RENDERBUFFER, depthRB);
gl.renderbufferStorage(gl.RENDERBUFFER, gl.DEPTH_COMPONENT24, 512, 512);
gl.framebufferRenderbuffer(gl.FRAMEBUFFER, gl.DEPTH_ATTACHMENT, gl.RENDERBUFFER, depthRB);

// Check completeness
if (gl.checkFramebufferStatus(gl.FRAMEBUFFER) !== gl.FRAMEBUFFER_COMPLETE) {
    console.error('Framebuffer not complete!');
}

// RENDER TO TEXTURE:
gl.bindFramebuffer(gl.FRAMEBUFFER, fb);
gl.viewport(0, 0, 512, 512);
gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
// ... draw scene ...

// RENDER TO SCREEN (use the texture):
gl.bindFramebuffer(gl.FRAMEBUFFER, null); // null = default framebuffer (screen)
gl.viewport(0, 0, canvas.width, canvas.height);
// Now colorTex contains the rendered image — use it as a regular texture!
```

### Use Cases
```
FRAMEBUFFER USES:
═════════════════
  - Shadow mapping: Render depth from light's perspective → shadow map texture
  - Post-processing: Render scene → texture → apply blur/bloom/color grading
  - Reflections: Render scene from mirror's perspective → apply to mirror surface
  - Deferred rendering: Write G-Buffer data to multiple textures simultaneously
```

---

## 4. Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Non-power-of-2 texture + mipmaps | Black texture or WebGL error | Resize to POT or disable mipmaps |
| Forgot `gl.UNPACK_FLIP_Y_WEBGL` | Texture appears upside down | Set before `texImage2D` |
| Wrong texture slot in uniform | Texture shows wrong image | Match `gl.TEXTURE0` with `gl.uniform1i(..., 0)` |
| FBO missing depth attachment | 3D objects overlap randomly | Always attach depth renderbuffer |
| Reading from texture while writing to it | Undefined behavior, glitches | Use separate textures for input and output |
