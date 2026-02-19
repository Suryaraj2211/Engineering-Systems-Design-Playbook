# Chapter 18 — Engine Core Architecture

---

## 1. The Game Loop

```javascript
class Engine {
    constructor(canvas) {
        this.lastTime = 0;
        this.running = false;
    }
    
    start() {
        this.running = true;
        requestAnimationFrame((t) => this.loop(t));
    }
    
    loop(currentTime) {
        if (!this.running) return;
        
        const dt = (currentTime - this.lastTime) / 1000; // Seconds
        this.lastTime = currentTime;
        
        // Cap delta to prevent spiral-of-death after tab switching
        const cappedDt = Math.min(dt, 0.1); // Max 100ms (10fps minimum)
        
        this.update(cappedDt);  // Physics, input, AI (fixed or variable)
        this.render();          // Draw the scene
        
        requestAnimationFrame((t) => this.loop(t));
    }
    
    update(dt) {
        // Update game logic, transform objects, physics
    }
    
    render() {
        // Issue draw calls to GPU
    }
}
```

### Fixed vs Variable Timestep
```
VARIABLE TIMESTEP (simple, used above):
  dt varies per frame (16ms at 60fps, 33ms at 30fps).
  Physics calculations use dt directly.
  Problem: Physics becomes non-deterministic (different FPS = different results).

FIXED TIMESTEP (production-quality):
  Physics runs at fixed rate (e.g., 60Hz = 16.67ms).
  Rendering runs as fast as possible.
  Accumulate real time; run physics in fixed increments.

  class Engine {
      fixedDt = 1/60; // Physics at 60Hz
      accumulator = 0;
      
      loop(currentTime) {
          const dt = (currentTime - this.lastTime) / 1000;
          this.lastTime = currentTime;
          this.accumulator += Math.min(dt, 0.1);
          
          while (this.accumulator >= this.fixedDt) {
              this.physicsUpdate(this.fixedDt);
              this.accumulator -= this.fixedDt;
          }
          
          // Optional: interpolate between physics states for smooth rendering
          const alpha = this.accumulator / this.fixedDt;
          this.render(alpha);
      }
  }
```

---

## 2. Entity Component System (ECS)

```
ECS ARCHITECTURE:
═════════════════
  ENTITY: Just an ID number (e.g., Entity #47).
  
  COMPONENT: Pure data, no logic.
    TransformComponent { position, rotation, scale }
    MeshComponent { vertexBuffer, indexBuffer, materialId }
    LightComponent { color, intensity, range }
    
  SYSTEM: Logic that operates on entities with specific components.
    RenderSystem: For every entity with Transform + Mesh → draw it.
    PhysicsSystem: For every entity with Transform + RigidBody → simulate.
    LightSystem: For every entity with Transform + Light → update light data.

  WHY ECS?
  Traditional OOP:  class Player extends Entity { render(), update(), ... }
    Problem: What if you want a player that's also a light source and a physics object?
    Deep inheritance = "Diamond Problem" = unmaintainable spaghetti.
    
  ECS: Just add components!
    Entity #1 = [Transform, Mesh, Physics, Player]        → player
    Entity #2 = [Transform, Light]                         → lamp
    Entity #3 = [Transform, Mesh, Physics, Player, Light]  → glowing player!
```

```javascript
class World {
    constructor() {
        this.nextId = 0;
        this.components = {}; // { 'Transform': Map<entityId, data>, ... }
        this.systems = [];
    }
    
    createEntity() {
        return this.nextId++;
    }
    
    addComponent(entityId, componentType, data) {
        if (!this.components[componentType]) {
            this.components[componentType] = new Map();
        }
        this.components[componentType].set(entityId, data);
    }
    
    query(...requiredComponents) {
        // Returns all entity IDs that have ALL required components
        const results = [];
        const firstMap = this.components[requiredComponents[0]];
        if (!firstMap) return results;
        
        for (const entityId of firstMap.keys()) {
            if (requiredComponents.every(c => this.components[c]?.has(entityId))) {
                results.push(entityId);
            }
        }
        return results;
    }
    
    update(dt) {
        for (const system of this.systems) {
            system.execute(this, dt);
        }
    }
}
```

---

## 3. Render Loop Architecture

```
RENDER LOOP (per frame):
════════════════════════
  1. UPDATE TRANSFORMS:
     Walk the scene graph. Compute world matrices for all objects.
     worldMatrix = parent.worldMatrix × localTRS
     
  2. FRUSTUM CULLING (CPU):
     Test each object's bounding sphere against camera frustum.
     If outside → skip this object entirely (don't even send to GPU).
     
  3. SORT DRAW CALLS:
     a. Opaque objects: sort FRONT-TO-BACK (minimizes overdraw via early-Z).
     b. Transparent objects: sort BACK-TO-FRONT (required for alpha blending).
     c. Group by material (minimize shader/texture switches).
     
  4. RENDER PASSES:
     Pass 1: Shadow maps (depth from lights)
     Pass 2: Depth pre-pass (optional, zero-overdraw)
     Pass 3: Opaque geometry (G-Buffer or forward)
     Pass 4: Transparent geometry (always forward, back-to-front)
     Pass 5: Post-processing (bloom, TAA, tone mapping)
     
  5. PRESENT:
     Swap the back buffer to the screen.
```

---

## 4. Frame Timing

```javascript
// GPU Timestamp queries (WebGPU)
const querySet = device.createQuerySet({ type: 'timestamp', count: 2 });

// In your render code
renderPass.writeTimestamp(querySet, 0); // Start
// ... draw commands ...
renderPass.writeTimestamp(querySet, 1); // End

// Read results
const elapsed = (timestamp[1] - timestamp[0]) / 1e6; // nanoseconds → ms
```
