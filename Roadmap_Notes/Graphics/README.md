# Graphics Engineering Track

This is a professional-grade, 39-module curriculum designed for software engineers seeking deep expertise in real-time rendering, GPU architecture, and low-level graphics APIs. 

This track inherently rejects high-level engine abstractions (like Unity or Unreal blueprints) in favor of building rendering systems from absolute scratch using WebGL, WebGPU, and GLSL/WGSL. It is heavily focused on the mathematical and hardware realities of writing code that executes concurrently across millions of GPU threads.

## Track 1: Beginner to Engine Architect (Chapters 01 - 23)

### Mathematics (01-06)
The foundation of computer graphics is pure mathematics. These modules require manual, step-by-step calculations.
- **Algebra and Trigonometry:** Floating-point precision limitations (IEEE 754), piecewise functions for gamma correction, the unit circle, and manually deriving field of view (FOV) ratios.
- **Vectors and Matrices:** Dot and cross product geometric proofs, reflection vector derivations, and 4x4 homogenous coordinate matrix multiplications.
- **Transformations and Projections:** The TRS (Translate, Rotate, Scale) multiplication order dependency, constructing LookAt matrices, manually calculating Orthographic and Perspective projection frustums, and mitigating Z-fighting via depth precision analysis.

### The Graphics Pipeline and Hardware (07-08)
- **GPU Architecture:** CPU vs. GPU hardware design, SIMT (Single Instruction, Multiple Threads) execution models, warp/wavefront divergence, the 4-level memory hierarchy (Registers, Shared, L2, VRAM), and texture sampling latency.
- **The Pipeline:** The rigid 9-stage sequence from Vertex Assembly to Rasterization (barycentric coordinates interpolating across triangles), Depth Testing, and the Blending Equation.

### WebGL and Shaders (09-14)
- **API Mastery:** WebGL State Machine management, Buffer Object creation (VBO/EBO), interleaved memory layouts (stride vs. offset), and Framebuffer Objects (FBO) for render-to-texture capabilities.
- **GLSL Fundamentals:** Strict typing in GLSL, swizzling, the `attribute/uniform/varying` communication pipeline, and procedural SDF (Signed Distance Field) geometry generation.
- **Lighting:** Implementing the Phong and Blinn-Phong reflection models from scratch strictly using dot products and reflection vectors, normal mapping tangent space conversions, and basic Shadow Mapping theory (depth map comparisons).

### WebGPU and Compute (15-17)
- **Modern Architecture:** The paradigm shift from the global synchronous state machine of WebGL to the asynchronous, explicit Pipeline Object and Command Encoder structure of WebGPU.
- **Resource Binding:** Uniform Buffers versus Storage Buffers, Bind Group layouts, and the transition from GLSL to WGSL.
- **Compute Shaders:** Arbitrary General Purpose GPU (GPGPU) programming. Defines workgroups, invocations, and utilizes shared memory and barrier synchronization to simulate particle physics directly on the GPU, entirely bypassing the CPU.

### Engine Architecture (18-23)
- **Systems Design:** Implementing a deterministic fixed-timestep game loop, an Entity Component System (ECS) for data-oriented memory access, and a hierarchical Scene Graph (parent-child transform multiplication).
- **Optimization:** CPU-bound versus GPU-bound bottleneck profiling, frame budgeting analysis, and immediate mode versus deferred resource management.

## Track 2: Advanced Rendering Systems (9 Modules)

This track focuses on the industry-standard techniques used in modern AAA game engines.
- **Physically Based Rendering (PBR):** The microfacet bidirectional reflectance distribution function (BRDF). Detailed mathematical implementations of the Cook-Torrance model, specifically the GGX normal distribution, Smith Geometry function, and the Fresnel-Schlick approximation with F0 values.
- **Deferred Rendering:** Shifting from Forward Rendering (O(Lights * Geometry)) to Deferred Rendering (O(Lights + Geometry)) by packing normal, albedo, and material data into a multi-target G-Buffer, followed by a precise 2D lighting pass utilizing Light Volumes.
- **Advanced Post-Processing:** High Dynamic Range (HDR) exposure tone mapping (ACES), multi-pass Gaussian Blur for Bloom, Screen Space Ambient Occlusion (SSAO), and the sub-pixel jitter mathematics required for Temporal Anti-Aliasing (TAA) history reprojection.

## Track 3: Expert Research and Innovation (7 Modules)

This track is geared toward individuals preparing for graphics research or primary engine programming roles.
- **GPU Deep Dive:** An exhaustive look at occupancy, register allocation failure, occupancy-bound versus bandwidth-bound operations, and building a software rasterizer purely via Compute Shaders to understand fixed-function hardware bypasses.
- **Modern Rendering Research:** Analysis of Restir (Reservoir Spatio-Temporal Importance Resampling) for millions of dynamic lights, SVGF filtering for real-time path tracing, and the architectural design of Frame Graphs and Multi-threaded Job Systems for Vulkan/WebGPU command submission.
- **AI Rendering:** The future pipeline of Graphics. Analyzes the mathematical foundations of Neural Radiance Fields (NeRF), 3D Gaussian Splatting, and Deep Learning Super Sampling (DLSS) upscaling architectures.
