---
layout: post
title:  "From HLSL To CUDA"
subtitle: A graphics engineer approach to general purpose computing on the GPU.
author: "Riccardo Loggini"
tocmaxlevel: 2
tags: [hlsl,cuda]
---
// TODO intro, motivations (market shift, reasearch trends)/

The following content can be seen as a continuation read from [my previous post about compute shaders](/graphics/2021/10/31/D3D12ComputeShaders.html).
# Why CUDA In Graphics
[CUDA is an NVIDIA API][nvidia-cuda] for GPGPU: General-Purpose computing on Graphics Processing Units.\
It refers to using a GPU's massively parallel architecture for computation beyond the traditional rasterization pipeline.

If this sounds familiar, yes, it's the same **Single Instruction, Multiple Threads (SIMT)** execution model behind every compute shader you've written in D3D12.\
CUDA is NVIDIA's own C++-based platform for this, predating and running in parallel to standardized compute shader APIs. \
In the mid-2000s, before compute shaders existed, people did GPGPU by disguising computation as a rendering pass, writing results to a render target via a full-screen pixel shader, then reading it back (e.g. [Fast matrix multiplies using graphics hardware][larsen-mcallister]).\
CUDA (2007) and later compute shaders in D3D11/Vulkan gave GPGPU workloads a proper first-class entry point, with no rendering-pipeline disguise required.

A few concrete places GPGPU work, often CUDA specifically, shows up in modern real-time and offline graphics pipelines:
- **Ray tracing.** NVIDIA's [OptiX framework][optix] (CUDA-based) powers GPU-accelerated ray tracing and, notably, the denoisers used in production renderers like [Arnold][arnold]. Denoising a noisy path-traced image is exactly the kind of general-purpose parallel workload CUDA was built for.
- **Physics and simulation.** Cloth, fluid, and particle simulations ([NVIDIA Flex][flex] and [GPU PhysX][physx]) use GPGPU compute to parallelize what would otherwise be sequential per-particle or per-constraint updates.
- **Sorting and stream compaction.** GPU-driven pipelines lean heavily on primitives like **radix sort** and **stream compaction** — for visibility culling, transparency sorting, indirect draw generation. [CUDA's Thrust][trust] library provides mature, heavily-optimized implementations of these that are harder to hand-roll efficiently in a compute shader.
- **Offline tooling.** Lightmap baking, texture synthesis, and other asset-pipeline tasks that run outside the real-time frame budget are common candidates for CUDA, since portability across GPU vendors isn't a constraint for a build-time tool the way it is for a shipping game. The [Unity Progressive Lightmapper][unity-lightmap], for example, uses [OpenCL][opencl] for its core path-traced bake, and it offers optional [NVIDIA Optix][optix] denoiser.
- **AI-driven image reconstruction.** [Deep-Learning Super Sampling (DLSS)][dlss] and similar upscaling/frame-generation techniques run neural network inference on Tensor Cores accessed through NVIDIA's proprietary NGX SDK. This is arguably the most consequential use of GPGPU in graphics right now, as it changed the performance budget for an entire generation of renderers. (How this squares with the interop cost described below is worth a closer look. See the note under "Tight integration with the frame").

## CUDA Or Compute Shader?
As listed before, a number of graphics applications use CUDA in their computation workflow, then when and why should I still use a compute shader?

### Where compute shaders win

**Tight integration with the frame.** A compute shader dispatch lives on the same command list, uses the same resource-binding and barrier infrastructure, and slots into a [render graph](/graphics/2021/05/31/RenderGraphs.html) exactly like any other pass. CUDA/D3D12 interop is possible, but it means bridging two separate runtimes with explicit shared-resource handles and fence-based cross-API synchronization (an overhead you don't want to pay for something like a per-frame post-process pass). *Example: GPU-driven occlusion culling generating indirect draw arguments that feed directly into the next draw call on the same queue. A CUDA kernel here would need to hand results back across an API boundary before the GPU could even issue the draw.*

> **Isn't DLSS a counterexample?** It runs every frame and it's CUDA/Tensor-Core work, so it looks like it should pay the interop cost just described, but it doesn't, and that's not an accident. DLSS isn't invoked as a raw CUDA kernel with a context switch; it goes through NVIDIA's [NGX SDK][ngx-docs], whose `NVSDK_NGX_D3D12_EvaluateFeature` call takes *your application's own D3D12 command list* as an argument and records the Tensor Core inference inline, the same way any other pass would be. There's no separate CUDA context, no shared-resource handle, no fence-based sync visible to the engine. That inline path is a privilege of NVIDIA's own driver stack, not something available to third-party CUDA code. NVIDIA built NGX this way precisely because the interop cost is real, and DLSS wouldn't be shippable at real-time rates without sidestepping it.

**No cross-context latency.** Anything that has to happen every frame, in lockstep with the rest of the pipeline, is cheaper and simpler as a compute shader than as a CUDA kernel with an interop round-trip. *Examples include tile-based light culling, particle simulation updates and TAA resolve.*

**Cross-vendor portability.** CUDA is for NVIDIA-only hardware. Anything shipping on AMD, Intel, or console hardware (PS5 and Xbox are both AMD GPUs) simply cannot depend on it (more on the alternatives in the next chapter).
### Where CUDA wins

**Library ecosystem.** CUDA nowadays offers very mature and hand-tuded libraries which leverage the underlying hardware at maximum capacity, covering linear algebra, FFTs, deep learning primitives, and multi-GPU communication. To name a few: cuBLAS, cuDNN, cuFFT, Thrust, CUTLASS, TensorRT, NCCL. *Example: training or running inference for a DLSS-style upscaling network needs cuDNN/TensorRT-class primitives — there's no compute-shader equivalent with comparable performance.*

**Profiling depth.** Nsight Compute gives per-kernel occupancy, register pressure, warp execution efficiency, and memory-throughput breakdowns — a level of introspection RenderDoc or PIX don't match for compute shader dispatches. When you actually need to know *why* a kernel is slow, this matters.

**A richer language.** CUDA is C++ which brings templates, classes, recursion, dynamic parallelism (kernels launching kernels), unified memory. HLSL is a much more constrained with C-like shading language up to ShaderModel 5.1 and some C++ from HLSL2021.

**Genuinely real-time, without NGX-style special-casing.** DLSS avoids the interop cost by never crossing an API boundary at all (see above) — but there's a separate category of application that wins with CUDA at real-time or interactive rates for the opposite reason: there's no D3D12/Vulkan frame graph to bridge into in the first place, because the whole render loop is CUDA-native from the start. *Examples: interactive viewport renderers in DCC tools (Blender Cycles X, [NVIDIA Omniverse][omniverse]'s RTX Renderer, Redshift, Chaos Vantage), and video/stream AI effects that sit outside a game's frame graph entirely, like NVIDIA Broadcast or RTX Video Super Resolution.*

## Limitations And Alternatives

### CUDA's limitations
The core limitation, as anticipated before, is that CUDA only runs on NVIDIA GPUs. Code written against it cannot run on AMD, Intel, or mobile GPUs without a translation layer or a rewrite.\
On top of that, there are factors including cost, architecture fragmentation and sub-optimal Api architecture decisions.
- **Architecture fragmentation.** Compute capability differs across GPU generations (`sm_75`, `sm_86`, `sm_89`, `sm_90`, ...), so shipping code either targets specific architectures at compile time or falls back to [PTX JIT-compilation][jit-vitalitylearning] at runtime (a versioning concern a fixed-ISA target like a CPU doesn't have).
- **Low-level responsibility.** Manual memory management, explicit shared-memory tiling, and hand-tuning occupancy/register pressure are all on you: this is the same trade-off as HLSL vs. a higher-level engine abstraction, just one level lower. (More on how Triton addresses this below.)
- **Cost and availability.** part of the price tag on NVIDIA's data-center GPUs isn't there because the silicon itself is worth that much more than a competitor's chip. It's there because **CUDA's ecosystem** lock-in lets NVIDIA charge more than pure hardware competition would allow, and people still pay it rather than deal with the cost of switching to something cheaper. Cloud GPU capacity on NVIDIA hardware is also routinely supply-constrained.

### OpenCL

[OpenCL][opencl] is the original cross-vendor answer to CUDA: running across GPUs, CPUs, and even [Field-Programmable Gate Arrays (FPGAs)](https://www.softserveinc.com/en-us/blog/fpga-development-using-opencl) and [Digital Signal Processors (DSPs)](https://www.thegoodpenguin.co.uk/blog/introduction-to-opencl-on-embedded-systems/). OpenCL 3.1 shipped in 2026 with mandatory SPIR-V ingestion, and layered implementations now run OpenCL over Vulkan and D3D12, widening availability even to platforms without a native driver.

- **Pros:** genuinely vendor-neutral: open standard with public conformance testing, can target the widest range of hardware of anything in this list.
- **Cons:** NVIDIA treats OpenCL as a compliance checkbox rather than a first-class target. It ships a conformant driver but doesn't invest in it the way it does CUDA, so performance on NVIDIA hardware specifically tends to lag. OpenCL's actual center of gravity today has shifted toward mobile and embedded (Arm, Imagination, Qualcomm), not data-center GPU compute. There's no equivalent to cuBLAS/cuDNN-level library maturity, and the API itself is considerably more verbose and lower-level than CUDA's runtime API. Explicit platform/device/context/queue setup for even a simple kernel launch. In practice, most people today reach OpenCL indirectly, through a higher-level layer like [SYCL][sycl], rather than writing it by hand.

### HIP / ROCm (AMD)

[HIP (Heterogeneous-Compute Interface for Portability)][hip] and [ROCm (Radeon Open Compute)][rocm]. AMD's answer, deliberately designed to look like CUDA: the `HIPify` tool auto-converts large amounts of existing CUDA source, and `hipcc` can even compile HIP to run on NVIDIA hardware by wrapping CUDA underneath, giving a degree of two-way portability.

- **Pros:** the lowest-friction migration path if you already have CUDA code and want AMD hardware support; open source.
- **Cons:** ROCm's tooling and library ecosystem is younger and less battle-tested than CUDA's, with a history of rougher Windows support (improving, but real). And because HIP mirrors CUDA so closely, it doesn't raise the abstraction level at all. You're still responsible for the same manual tiling and memory management, just retargeted.

### SYCL

[SYCL][sycl] is Khronos' modern single-source C++ model for heterogeneous compute (implemented via Intel's oneAPI/DPC++, AdaptiveCpp, and others), able to target CPUs and GPUs from multiple vendors by compiling down to CUDA, HIP, Level Zero, or OpenCL depending on backend.

- **Pros:** standard, modern C++ — templates, lambdas, no special language extensions the way CUDA/HLSL need. Genuinely portable across hardware vendors from a single source tree.
- **Cons:** the ecosystem and tooling are still maturing relative to CUDA's. Performance portability is alwats tricky to achieve. Getting genuinely competitive performance across every backend from one source remains a real, generally unsolved problem, not just a marketing claim.

### Triton

[Triton][triton] is where GPU-compute hiring in the job market is actually heading right now. Triton is a Python-embedded language (`@triton.jit`-decorated functions) where you reason about block-level tiles rather than individual threads. The compiler automatically handles memory coalescing, shared-memory allocation, and thread scheduling underneath. It compiles through an [MLIR-based pipeline][mlir] down to PTX for NVIDIA or AMDGCN for AMD.

- **Pros:** dramatically less code than hand-written CUDA for common kernel shapes (matmul, fused elementwise ops, attention), while still getting PTX that competes with hand-tuned CUDA. [FlashAttention](https://flashattention.org/), for example, ships an official Triton reference implementation alongside its hand-tuned CUDA kernel. It's the default kernel-generation backend for PyTorch 2.x's `torch.compile`, and vLLM's attention backends (PagedAttention, RoPE, RMSNorm) are written in it. AMD support has matured quickly too (ROCm 6.2+, covering both CDNA data-center and increasingly RDNA consumer GPUs). Even NVIDIA is investing in it directly — a CUDA Tile IR backend for Triton, targeting Blackwell's tensor cores, was announced in early 2026.
- **Cons:** minimum NVIDIA compute capability is 8.0 (Ampere and newer — Volta/Turing, e.g. V100/T4, aren't supported). You'll need to give up the fine-grained control needed to hand-optimize the last few percent of performance, and it's tuned primarily for the access patterns common in ML kernels (block-structured matmul, reductions, fused elementwise ops) rather than arbitrary general-purpose kernels a graphics engineer might reach for. It's also conceptually still Python/ML-ecosystem-shaped, not a general systems language the way CUDA C++ or HIP are.

### What's best then

If you're specifically aiming at the ML/AI-infra job direction discussed earlier in this series, Triton is arguably more relevant to learn *alongside* CUDA than as a replacement for it. It's rapidly becoming the productivity layer the industry writes kernels in day to day, with raw CUDA reserved for the cases where Triton's abstraction doesn't fit.\
If cross-vendor portability is the hard requirement, HIP is the least-friction path if you're starting from CUDA, otherwise SYCL if you want to start clean in modern C++.

# Terminology
// TODO
| HLSL / D3D12 compute | CUDA equivalent |
|---|---|
| `Dispatch(x, y, z)` | `kernel<<<gridDim, blockDim>>>(...)` |
| `numthreads(x,y,z)` attribute | `blockDim` (set at launch, not in the kernel) |
| `SV_GroupID` | `blockIdx` |
| `SV_GroupThreadID` | `threadIdx` |
| `SV_DispatchThreadID` | computed manually: `blockIdx * blockDim + threadIdx` |
| `groupshared` | `__shared__` |
| `GroupMemoryBarrierWithGroupSync()` | `__syncthreads()` |
| RWStructuredBuffer / UAV via root signature/descriptors | plain device pointer, passed as a kernel argument (CUDA's binding model is much simpler — no descriptor tables) |
| `dxc` (HLSL compiler) | `nvcc` |
| PIX / RenderDoc | Nsight Systems (timeline) / Nsight Compute (per-kernel profiling) |
| Wave intrinsics (`WaveActiveSum`, 32-lane on NVIDIA hardware) | warp-level primitives (`__shfl_sync`, cooperative groups) — same 32-thread warp under the hood, since it's the same silicon |

# Hardware
A modern GPU packs thousands of simple cores designed for high-throughput, data-parallel work, backed by memory bandwidth far exceeding a CPU's.\
TODO
# Debug With Nsight
## Nsight Systems (timeline)
//TODO
## Nsight Compute
// TODO
# Sources
- https://en.wikipedia.org/wiki/CUDA
- [Fast matrix multiplies using graphics hardware][larsen-mcallister]
- [Working out CUDA Just-In-Time (JIT) compilation - vitalitylearning][jit-vitalitylearning]


[larsen-mcallister]: https://dl.acm.org/doi/10.1145/582034.582089
[nvidia-cuda]: https://developer.nvidia.com/cuda
[dlss]: https://developer.nvidia.com/rtx/dlss
[optix]: https://developer.nvidia.com/rtx/ray-tracing/optix
[arnold]: https://www.autodesk.com/products/arnold/overview
[flex]: https://developer.nvidia.com/flex
[physx]: https://developer.nvidia.com/physx-sdk
[trust]: https://developer.nvidia.com/thrust
[unity-lightmap]: https://docs.unity3d.com/2023.2/Documentation/Manual/GPUProgressiveLightmapper.html
[opencl]: https://www.khronos.org/opencl/
[jit-vitalitylearning]: https://vitalitylearning.medium.com/working-out-cuda-just-in-time-jit-compilation-d93eeaeb0118
[ngx-docs]: https://docs.nvidia.com/ngx/ngx-archived/ngx-100/programming-guide/index.html
[omniverse]: https://developer.nvidia.com/omniverse
[sycl]: https://sycl.tech/
[hip]: https://rocm.docs.amd.com/projects/HIP/en/latest/what_is_hip.html
[rocm]: https://github.com/ROCm
[triton]: https://triton-lang.org/main/index.html
[mlir]: https://mlir.llvm.org/
