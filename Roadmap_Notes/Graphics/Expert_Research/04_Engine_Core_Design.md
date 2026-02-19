# Module 4 — Large-Scale Engine Core Design

> This module goes beyond rendering into the systems engineering
> required to build a professional, production-grade engine.

---

## 1. Frame Graph Architecture (Production-Level)

### 1.1 The Declarative Pass System

In a research-grade engine, the renderer isn't a sequence of function calls. It's a **Directed Acyclic Graph (DAG)** compiled at the start of each frame.

```javascript
// PRODUCTION FRAME GRAPH API DESIGN:
class FrameGraph {
    // Declare a render pass with its inputs and outputs
    addPass(name, setup, execute) {
        const passNode = { name, reads: [], writes: [], executeFn: execute };
        setup(new PassBuilder(passNode)); // User declares dependencies
        this.passes.push(passNode);
    }
    
    // Compile: topological sort, memory aliasing, barrier insertion
    compile() {
        this.sortedPasses = topologicalSort(this.passes);
        this.allocateTransientResources();
        this.insertBarriers();
    }
    
    // Execute all passes in dependency order
    execute(commandEncoder) {
        for (const pass of this.sortedPasses) {
            if (pass.refCount === 0) continue; // Dead pass! Never execute.
            pass.executeFn(commandEncoder);
        }
    }
}

// USAGE BY GAME CODE:
frameGraph.addPass('ShadowMap', 
    (builder) => {
        builder.write('shadowDepth', { format: 'depth32float', size: [2048, 2048] });
    },
    (cmd) => {
        const pass = cmd.beginRenderPass(shadowPassDescriptor);
        // Draw shadow casters...
        pass.end();
    }
);

frameGraph.addPass('GBuffer',
    (builder) => {
        builder.write('gPosition', { format: 'rgba16float' });
        builder.write('gNormal',   { format: 'rgba16float' });
        builder.write('gAlbedo',   { format: 'rgba8unorm'  });
    },
    (cmd) => { /* Draw scene geometry */ }
);

frameGraph.addPass('Lighting',
    (builder) => {
        builder.read('gPosition');
        builder.read('gNormal');
        builder.read('gAlbedo');
        builder.read('shadowDepth');
        builder.write('hdrColor', { format: 'rgba16float' });
    },
    (cmd) => { /* Full-screen lighting compute dispatch */ }
);
```

### 1.2 Automatic Dead Pass Elimination

```
LIVE PASS ANALYSIS:
═══════════════════
  If the developer disables Bloom in the settings:
  
  Pass "BloomExtract" writes [bloomBright] → READ BY "BloomBlur"
  Pass "BloomBlur" writes [bloomResult]    → READ BY "Composite"
  
  When Bloom is disabled:
    "Composite" no longer reads [bloomResult].
    → "BloomBlur" has refCount = 0 → eliminated.
    → "BloomExtract" has refCount = 0 → eliminated.
    
  TWO PASSES AUTOMATICALLY REMOVED. Zero GPU cost. Zero code changes.
```

### 1.3 Transient Resource Allocation — The Linear Allocator

```javascript
class TransientResourceAllocator {
    constructor(heapSize) {
        // Pre-allocate ONE massive GPU heap at engine startup
        this.heap = device.createBuffer({ size: heapSize, usage: ... });
        this.offset = 0;
        this.aliasMap = new Map(); // resource name → heap offset
    }
    
    allocate(name, sizeBytes) {
        // Check if any dead resource's memory can be reused
        for (const [deadName, deadOffset] of this.aliasMap) {
            if (this.isDead(deadName) && this.getSize(deadName) >= sizeBytes) {
                this.aliasMap.set(name, deadOffset); // ALIAS! Same memory!
                return deadOffset;
            }
        }
        // No alias found; bump-allocate from heap
        const offset = this.offset;
        this.offset += align(sizeBytes, 256); // GPU requires 256-byte alignment
        this.aliasMap.set(name, offset);
        return offset;
    }
    
    resetFrame() {
        this.offset = 0; // ZERO allocation cost per frame!
        this.aliasMap.clear();
    }
}
```

**Memory Savings Example:**
```
WITHOUT ALIASING:                    WITH ALIASING:
  Shadow Map:   16 MB                Shadow Map:   16 MB  ← offset 0
  G-Position:   63 MB                G-Position:   63 MB  ← offset 16MB
  G-Normal:     63 MB                G-Normal:     63 MB  ← offset 79MB
  G-Albedo:     32 MB                G-Albedo:     32 MB  ← offset 142MB
  SSAO:         32 MB                SSAO:         32 MB  ← offset 174MB
  HDR Color:    63 MB                HDR Color:    63 MB  ← offset 0 (Shadow is dead!)
  Bloom Bright: 16 MB                Bloom Bright: 16 MB  ← offset 63MB (reusing G-Normal!)  
  Bloom Result: 16 MB                Bloom Result: 16 MB  ← offset 79MB (reusing G-Normal!)
  TOTAL:       301 MB                TOTAL:       ~206 MB (32% savings)
```

---

## 2. Job System Design (Multithreaded CPU Architecture)

### 2.1 The Problem with `setInterval` / Single-Threaded Loops

JavaScript's main thread is fundamentally single-threaded. A game loop that performs frustum culling, physics, animation, and render submission on a single thread wastes 93% of a 16-core CPU.

### 2.2 WebWorker-Based Job System

```javascript
// MAIN THREAD: Job Dispatcher
class JobSystem {
    constructor(workerCount = navigator.hardwareConcurrency - 1) {
        this.workers = [];
        this.jobQueue = [];
        this.pendingJobs = 0;
        this.resolveFrame = null;
        
        for (let i = 0; i < workerCount; i++) {
            const worker = new Worker('jobWorker.js');
            worker.onmessage = (e) => this.onJobComplete(e.data);
            this.workers.push({ worker, busy: false });
        }
    }
    
    // Submit a batch of independent jobs
    dispatch(jobName, dataChunks) {
        this.pendingJobs = dataChunks.length;
        return new Promise((resolve) => {
            this.resolveFrame = resolve;
            for (const chunk of dataChunks) {
                this.jobQueue.push({ jobName, data: chunk });
            }
            this.flushQueue();
        });
    }
    
    flushQueue() {
        for (const w of this.workers) {
            if (!w.busy && this.jobQueue.length > 0) {
                const job = this.jobQueue.shift();
                w.busy = true;
                // Transfer data (zero-copy via Transferable)
                w.worker.postMessage(job, [job.data.buffer]);
            }
        }
    }
    
    onJobComplete(result) {
        this.pendingJobs--;
        // Find the worker and mark it free
        // ... (omitted for brevity)
        this.flushQueue(); // Immediately assign next job
        if (this.pendingJobs === 0) this.resolveFrame();
    }
}

// USAGE:
const jobs = new JobSystem();

async function gameLoop() {
    // Split 100,000 entities into 16 chunks of 6,250
    const chunks = splitEntities(allEntities, 16);
    
    // ALL 16 CHUNKS RUN IN PARALLEL ON SEPARATE CPU CORES
    await jobs.dispatch('frustumCull', chunks);
    
    // After all workers complete, build the draw command buffer
    submitRenderCommands();
    requestAnimationFrame(gameLoop);
}
```

### 2.3 Lock-Free Job Queue (Advanced)

For C++/Rust engines, WebWorker `postMessage` is too slow. Instead, use a **Lock-Free Multi-Producer Multi-Consumer Queue** based on atomic Compare-And-Swap (CAS):

```
LOCK-FREE QUEUE (CAS-based):
═════════════════════════════
  Memory Layout: Ring buffer of 4096 job slots.
  
  Atomic head pointer (producers write here)
  Atomic tail pointer (consumers read from here)
  
  PUSH (Producer):
    1. Load head atomically.
    2. Write job to slots[head].
    3. Attempt atomicCAS(head, head, head+1).
       If another thread beat us, retry from step 1.
    
  POP (Consumer):
    1. Load tail atomically.
    2. If tail == head → queue is empty, go to sleep.
    3. Read job from slots[tail].
    4. Attempt atomicCAS(tail, tail, tail+1).
       If another thread beat us, retry from step 1.
  
  ZERO LOCKS. ZERO MUTEXES. All threads work simultaneously.
```

---

## 3. Asset Streaming & Virtual Texturing

### 3.1 The Mip-Tail Feedback System

```
VIRTUAL TEXTURING PIPELINE:
════════════════════════════
  All textures are pre-processed into 128×128 pixel "Pages" stored on SSD.
  Each page is a specific Mip level of a specific texture.
  
  STEP 1: G-Buffer Pass
    The shader writes a "Feedback Buffer" alongside the G-Buffer.
    For each pixel, record: { TextureID, RequiredMipLevel, PageXY }
    
    // In fragment shader:
    float mipLevel = textureQueryLod(albedoAtlas, uv).x;
    feedbackBuffer[pixelIndex] = encodeFeedback(textureID, mipLevel, pageXY);
  
  STEP 2: Readback (Async)
    A Compute Shader reads the Feedback Buffer at 1/16th resolution.
    It builds a unique Set of all requested Pages.
    
    // Pseudo
    for each pixel in feedbackBuffer (downsampled):
        requestedPages.add(decodeFeedback(pixel));
  
  STEP 3: Streaming
    Compare requestedPages vs loadedPages.
    For each missing page:
        Issue async SSD read: loadPage(textureID, mipLevel, pageXY);
        On completion: Upload page into the Physical Texture Atlas.
        Update the Page Table (indirection texture) to point to the new location.
  
  STEP 4: Next Frame Rendering
    The shader samples a Page Table (indirection texture) to find
    the physical UV of the requested page in the atlas.
    
    vec2 physicalUV = texture(pageTable, virtualUV).xy;
    vec3 color = texture(physicalAtlas, physicalUV).rgb;

  RESULT: Only the VISIBLE pages at the CORRECT resolution are in VRAM.
          A 100GB open world uses ~500MB of VRAM.
```

---

## 4. Live Debugging Infrastructure

### 4.1 GPU Timestamp Queries (Per-Pass Timing)

```javascript
// WebGPU Per-Pass Performance Measurement
const querySet = device.createQuerySet({ type: 'timestamp', count: 20 });
const queryBuffer = device.createBuffer({ 
    size: 20 * 8, // 8 bytes per timestamp (u64)
    usage: GPUBufferUsage.QUERY_RESOLVE | GPUBufferUsage.COPY_SRC 
});
const readbackBuffer = device.createBuffer({
    size: 20 * 8,
    usage: GPUBufferUsage.MAP_READ | GPUBufferUsage.COPY_DST
});

// During rendering:
const encoder = device.createCommandEncoder();

encoder.writeTimestamp(querySet, 0); // START: Shadow Pass
// ... shadow pass ...
encoder.writeTimestamp(querySet, 1); // END: Shadow Pass

encoder.writeTimestamp(querySet, 2); // START: G-Buffer Pass
// ... gbuffer pass ...
encoder.writeTimestamp(querySet, 3); // END: G-Buffer Pass

// Resolve and readback
encoder.resolveQuerySet(querySet, 0, 4, queryBuffer, 0);
encoder.copyBufferToBuffer(queryBuffer, 0, readbackBuffer, 0, 4 * 8);
device.queue.submit([encoder.finish()]);

// Read results on CPU (next frame to avoid stall)
await readbackBuffer.mapAsync(GPUMapMode.READ);
const timestamps = new BigUint64Array(readbackBuffer.getMappedRange());
const shadowPassMs  = Number(timestamps[1] - timestamps[0]) / 1_000_000;
const gbufferPassMs = Number(timestamps[3] - timestamps[2]) / 1_000_000;
readbackBuffer.unmap();

console.log(`Shadow: ${shadowPassMs.toFixed(2)}ms, G-Buffer: ${gbufferPassMs.toFixed(2)}ms`);
```

### 4.2 Hot Shader Reloading

```javascript
// Watch shader files on disk. Recompile pipelines on change.
// (Node.js backend sends WebSocket notifications to the browser)
const ws = new WebSocket('ws://localhost:9001/shader-watcher');
ws.onmessage = async (event) => {
    const { shaderPath, newSource } = JSON.parse(event.data);
    
    // Recompile ONLY the affected pipeline
    const module = device.createShaderModule({ code: newSource });
    const newPipeline = await device.createRenderPipelineAsync({
        ...existingDescriptor,
        fragment: { ...existingDescriptor.fragment, module }
    });
    
    // Atomically swap the pipeline reference
    pipelineRegistry.set(shaderPath, newPipeline);
    console.log(`🔥 Hot-reloaded: ${shaderPath}`);
};
```

### 4.3 Debug Visualization Modes

Every professional engine must support rendering the scene with `FragColor` overridden to visualize internal data:

```glsl
// Universal debug output in the final composite shader
uniform int debugMode; // Set by UI dropdown

vec3 applyDebugView(vec3 finalColor, vec3 albedo, vec3 normal, 
                     float depth, float ao, float roughness, float metallic) {
    switch(debugMode) {
        case 0: return finalColor;                           // Normal rendering
        case 1: return albedo;                               // Base color only
        case 2: return normal * 0.5 + 0.5;                   // World normals (RGB)
        case 3: return vec3(depth);                           // Linear depth (greyscale)
        case 4: return vec3(ao);                              // Ambient occlusion
        case 5: return vec3(roughness);                       // Roughness map
        case 6: return vec3(metallic);                        // Metallic mask
        case 7: return vec3(fract(depth * 100.0));            // Depth precision bands
        case 8: return heatmap(passTimingMs / targetFrameMs); // Performance heatmap
    }
    return finalColor;
}
```
