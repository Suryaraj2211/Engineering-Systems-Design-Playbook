# Chapter 19 — Scene Management & Resource Management

---

## 1. Scene Graph (Hierarchical Transforms)

```javascript
class SceneNode {
    constructor(name) {
        this.name = name;
        this.children = [];
        this.parent = null;
        
        // Local transform (relative to parent)
        this.position = [0, 0, 0];
        this.rotation = [0, 0, 0]; // Euler angles
        this.scale = [1, 1, 1];
        
        // Computed matrices
        this.localMatrix = mat4.identity();
        this.worldMatrix = mat4.identity();
        
        // Optional: renderable data
        this.mesh = null;
        this.material = null;
    }
    
    addChild(child) {
        child.parent = this;
        this.children.push(child);
        return this;
    }
    
    updateWorldMatrix(parentWorldMatrix = null) {
        // Build local matrix: T × R × S
        this.localMatrix = mat4.multiply(
            mat4.translation(this.position),
            mat4.multiply(
                mat4.rotationY(this.rotation[1]),
                mat4.scaling(this.scale)
            )
        );
        
        // World = parent's world × my local
        if (parentWorldMatrix) {
            this.worldMatrix = mat4.multiply(parentWorldMatrix, this.localMatrix);
        } else {
            this.worldMatrix = this.localMatrix;
        }
        
        // Recursively update children
        for (const child of this.children) {
            child.updateWorldMatrix(this.worldMatrix);
        }
    }
}
```

### Example: Robot Arm
```
SCENE GRAPH:
════════════
  root (world)
    └── body (pos: 0, 1, 0)
         ├── head (pos: 0, 1.5, 0)
         ├── upperArm (pos: 0.4, 1.2, 0, rotZ: -30°)
         │    └── lowerArm (pos: 0.3, 0, 0, rotZ: -45°)
         │         └── hand (pos: 0.25, 0, 0)
         └── upperArmL (mirror)

  When the body rotates 90°:
    Everything (head, arms, hands) rotates with it automatically.
  When the upperArm rotates:
    Only the lowerArm and hand rotate — the body and head don't.
```

---

## 2. Resource Manager (Asset Cache)

```javascript
class ResourceManager {
    constructor(device) {
        this.device = device;
        this.textures = new Map();   // path → GPUTexture
        this.meshes = new Map();     // path → { vb, ib, count }
        this.shaders = new Map();    // name → GPUShaderModule
        this.loading = new Map();    // path → Promise (prevent duplicate loads)
    }
    
    async loadTexture(path) {
        // Return cached texture if already loaded
        if (this.textures.has(path)) return this.textures.get(path);
        
        // Prevent duplicate loads of the same texture
        if (this.loading.has(path)) return this.loading.get(path);
        
        const promise = this._loadTextureInternal(path);
        this.loading.set(path, promise);
        
        const texture = await promise;
        this.textures.set(path, texture);
        this.loading.delete(path);
        return texture;
    }
    
    async _loadTextureInternal(path) {
        const response = await fetch(path);
        const blob = await response.blob();
        const bitmap = await createImageBitmap(blob, { colorSpaceConversion: 'none' });
        
        const texture = this.device.createTexture({
            size: [bitmap.width, bitmap.height],
            format: 'rgba8unorm',
            usage: GPUTextureUsage.TEXTURE_BINDING |
                   GPUTextureUsage.COPY_DST |
                   GPUTextureUsage.RENDER_ATTACHMENT
        });
        
        this.device.queue.copyExternalImageToTexture(
            { source: bitmap },
            { texture: texture },
            [bitmap.width, bitmap.height]
        );
        
        return texture;
    }
    
    destroy() {
        for (const tex of this.textures.values()) tex.destroy();
        this.textures.clear();
        this.meshes.clear();
    }
}
```

---

## 3. Shader Manager

```javascript
class ShaderManager {
    constructor(device) {
        this.device = device;
        this.modules = new Map();
        this.pipelines = new Map();
    }
    
    getModule(name, code) {
        if (!this.modules.has(name)) {
            this.modules.set(name, this.device.createShaderModule({ code }));
        }
        return this.modules.get(name);
    }
    
    getPipeline(key, descriptor) {
        if (!this.pipelines.has(key)) {
            this.pipelines.set(key, this.device.createRenderPipeline(descriptor));
        }
        return this.pipelines.get(key);
    }
}
```

---

## 4. Asset Pipeline Overview

```
ASSET PIPELINE:
═══════════════
  OFFLINE (build time):
    .blend → export → .gltf/.glb     (3D models)
    .psd → export → .png/.ktx2       (textures, compressed)
    .hlsl → compile → SPIR-V/.wgsl   (shaders)
    
  RUNTIME (load time):
    1. Fetch .gltf file
    2. Parse JSON metadata (materials, meshes, textures)
    3. Load binary buffers (vertex/index data)
    4. Upload to GPU buffers
    5. Load textures → GPU textures with mipmaps
    6. Create materials → bind groups
    7. Build scene graph from gltf node hierarchy
```
