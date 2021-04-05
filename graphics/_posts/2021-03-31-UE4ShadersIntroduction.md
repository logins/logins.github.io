---
layout: post
title:  "UE4 Shaders Introduction"
subtitle: What shader objects are in Unreal Engine and how they work.
author: "Riccardo Loggini"
tocmaxlevel: 2
tags: [rendering,shaders,ue4,rdg]
---

# Why Talking About UE4 Shaders

Unreal Engine 4 is becoming more and more used by game companies, even for the ones that before were using Unity, mainly due to the success of Fortnite, that proved Unreal to be a great solution for any platform.  
As a programmer focusing on graphics, it becomes essential for me to understand the architecture behind this commercial tripleA engine renderer, and so I decided to write an introduction to it.  
The engine has been in development for more than 8 years, it went under the hand of many different developers and it is always changing: another reason to keep up to date on the subject.  
This wants to be a humble introduction to a topic that is really huge and that can be extended in a myriad of directions, depending on the area of interest. For this reason a number of mechanics and details will be left over.

> Note: This post is written as of Unreal Engine 4.26 and due to the nature of the engine, some of the following information might be deprecated in the future.

# Terminology

Before starting with the main content of the article, it is important to have in mind specific terminologies that UE4 introduces in its code.

*   **View**: a single “viewport” looking at the FScene. When playing in split screen or rendering left and right eye in VR for example, we are going to have two views.
*   **Vertex Factory**: class encapsulating vertex data and it is linked to the input of a vertex shader. We have different vertex factory types depending on the kind of mesh we are rendering.
*   **Render Proxy**: data that represents an asset (e.g. a material) or an object in the scene. It is conceptually owned by the render thread which should be the only thread to modify its values. It is used to hold transient (e.g. per-frame) data. Proxies get periodically updated with the objects they represent through synchronization mechanisms.
*   **RHI (Render Hardware Interface)**: the graphics abstraction layer to support multiple platforms in the renderer.
*   **RHI Thread**: closest async thread to the GPU. It receives input from the render thread and will asynchronously run render commands.
*   **Render Dependency Graph (RDG)** is the most up-to-date way of implementing graphics code as of UE4.26.
*   **Pooled Resource**: graphics resource created and handled by the RDG. Their availability is guaranteed during RDG passes execution only.
*   **External Resource**: graphics resource created independently from the RDG.

# RDG Dynamics

The main difference of the RDG compared to the previous render system, is that it is a graph-based scheduling system, where each node of the system is a **Render Pass**.  
A Render Pass can be seen as a single set of graphics operations (copies, dispatches or draws).  
With a graph system like the RDG we can potentially have the following advantages

*   Automatically figure out and operate resource transitions among passes only when needed.
*   Optimize pass workflow:
    *   Load balancing: spread computation equally in order to avoid performance hitches.
    *   Parallelization of computing passes if they are independent  from each other (and so if the resources they use do not require the result of the other pass).

*   Memory management optimization, e.g. re-using temporary resources for multiple passes without the need of the programmer to manually write the specific case.

>Note: These are the theoretical possibilities of a render graph, but there is no guarantee that the RDG satisfies all of them in every case. Reason is the magnitude of the Unreal Engine and the fact that the engine was developed for a long time before the RDG was introduced.

The concept of the RDG is the following: instead of executing render passes straight away, we first have a **Setup Stage** that is meant to consider all the render passes to create an optimized workflow, and then only in a later **Running Stage** execute the passes logic in a dependent order.

![](/assets\img\posts\2021-03-31-UE4ShadersIntroduction\RDGDynamics_Scheme.jpg){:.postImg}

For more information you can visit [the official documentation](https://docs.unrealengine.com/en-US/ProgrammingAndScripting/Rendering/RenderDependencyGraph/index.html) and also read the description of functions defined in `RenderGraphBuilder.h`.


## Setup Stage

All begins with the FRenderModule, triggered by the Render Thread main function, to start building passes for the visible Views and all the objects associated with them.
<br/>If we are using the default renderer, the job system will execute
```cpp
FDeferredShadingSceneRenderer::Render(FRHICommandListImmediate& RHICmdList)
```
which is a huge function (thousands of lines) containing all the main logic to set up the entire render flow.
<br/>From that function, all the RDG passes will be created, having this syntax in the generic case:

```cpp
// Instantiate the resources we need for our pass
FShaderParameterStruct* PassParameters = GraphBuilder.AllocParameters&lt;FShaderParameterStruct>();

// Fill in the pass parameters
PassParameters->MyParameter = GraphBuilder.CreateSomeResource(MyResourceDescription, TEXT("MyResourceName"));

// Define pass and add it to the RDG builder 
GraphBuilder.AddPass(
			RDG_EVENT_NAME("MyRDGPassName"),
			PassParameters,
			ERDGPassFlags::Raster,
			[PassParameters, OtherDataToCapture](FRHICommandList& RHICmdList)
		{
			// … pass logic here, render something! ...
		}
```
When this code executes, the GraphBuilder will store information to generate an RDG pass, using the following input

*   **Pass Name**: This will ultimately be represented by an object of type `FRDGEventName` containing the description of the pass. It is used for debugging and profiling tools.
*   **Pass Parameters**: This object is expected to derive from a **Shader Parameter Struct**  which has to be created with GraphBuilder.AllocParameters() and needs to be defined with the macro `BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParameters, )` (more on this subject in the Shader Parameters section of this article) and it can come from either: 
    *   A shader uniform buffer proper to a shader (e.g. in the case we have only one shader like a compute shader) and so in this case the PassParameters will be of type `FMyShader::FParameters`.
    *   A generically defined shader uniform buffer, which will usually be defined in the source (.cpp) file. The usual name of these buffers will include “PassParameters” to specify that they are used for a whole pass instead of a single shader.
*   **Pass Flags**: Set of flags of type `ERDGPassFlags`, they are mainly used to specify the kind of operations we will be doing inside the pass, e.g. raster, copy and compute.
*   **Lambda Function**: This will contain the “body” of our pass, and so the logic to execute at run time. With the lambda we can capture any number of objects we want to later use to set up our rendering operations. 

Among the captured variables, we often find the PassParameters. This happens when we need to independently set parameters to our shaders at the time the pass is executed.  
Despite using the PassParameters inside the lambda function to set the shader parameters in the command list for each shader, we still need to input PassParameters in the `GraphBuilder::AddPass` function.  
This happens for very important reasons: the lifetime of resources we create (e.g. buffers and textures), to use in the RDG, will depend on the references they have in the PassParameters!  
Such resources CPU memory is guaranteed to be valid to be accessed by passes that specify them in the input PassParameters.  
The RDG will also operate automatic resource transitions for buffers and textures listed in PassParameters, except the ones which are also bound for write access. For those we will have to manually call a transition before the render happens.

For more information you can read function descriptions of `RenderGraphBuilder.h` and [FRDGBuilder class](https://docs.unrealengine.com/en-US/API/Runtime/RenderCore/FRDGBuilder/index.html).

## Running Stage

With the term Running Stage we mean the time when the lambda function of an RDG pass gets executed. This will happen asynchronously and the exact moment is completely up to the RDG.  
When the lambda body executes, the available input will be the variables captured by the lambda and a command list (either RHIComputeCommandList& for Compute / AsyncCompute workloads or FRHICommandList& for raster operations).  
What essentially happens inside the lambda body is the following

*   Set Pipeline state object: e.g. setting rasterizer, blend and depth/stencil states.
*   Set Shaders and their Attributes: select what shaders to use, bind them to the current pipeline and set parameters for them, which means binding resources to the shader slots, on the current command list.
*   Send copy/draw/dispatch command: send render commands on the command list.

>Note: RDG lambda bodies can present different syntax across the engine, depending on the area they have been built for, because different programmers have different programming styles.
>If looking for a very simple example of RDG pass, you can visit `AtmosphereRendering.cpp` around line 521.

### Mesh Draw Commands And Pass Processors

>Note: Mesh Draw Commands and Pass Processors seem to be already semi-outdated concepts, and they were presented in 2019 when the RDG was not yet a thing.
>They are listed here because they are often seen inside RDG passes. Using them might not lead to the best performance, but they might still have functionality that the standard RDG syntax does not provide yet.

UE4.22 introduced **FMeshDrawCommand** object type, which stores everything the engine needs to know for a single mesh draw call in a specific pass.  
It is a stateless object and built with data driven patterns in mind: the engine will ultimately see that as something to render without knowing where it comes from (except using debug variables).  
The main two advantages of using mesh draw commands are:

*   More caching capabilities: we can store draw information of visible objects when they are added to the scene.
*   Easier to merge draw calls: merge draw calls for what resource they use instead of for where they come from.

To generate draw commands we are going to need a **Mesh Pass Processor** object.  
These objects take the role of Drawing Policies, which are now deprecated, and will do the following:

*   Decide if the input object should be considered or not.
*   Select shaders to use.
*   Collect pass bindings, vertex factory bindings, material bindings.
*   Build Mesh draw commands, that will be executed straight away for the current frame.

Inside the RDG lambda function, we are gonna find mesh processor usage wrapped inside a call to `DrawDynamicMeshPass` function, which is always needed to provide a context to generate mesh draw commands, since it provides a reference to a **FDynamicPassMeshDrawListContext** command list.

Code comments specifically state that it is used for legacy code or for non-performance effective functions, such as Editor related.

# Shader Objects

The base class for shaders is `FShader`, but we find two main types of shaders that we can use:

*   **FGlobalShader**: all the shaders deriving from it are part of the global shaders group. A global shader produces a single instance across the engine, and it can only use global parameters.
*   **FMaterialShader**: all the derived classes are the ones that use parameters tied to materials. If they also use parameters tied to the vertex factory, then the class to refer to is **FMeshMaterialShader**.

## Shader Parameters

We call Shader Parameters the objects that identify resource types used by a shader.  
These parameters will be used when setting resources for a graphics or compute operation.  
The process of setting a shader parameter will consist in binding a resource to the command list at the index specified by the shader parameter.

![](/assets\img\posts\2021-03-31-UE4ShadersIntroduction\ShaderParamsUsage_Scheme.jpg){:.postImg}

We have the following types of Shader Parameters, as seen in ShaderParametersUtils.h and ShaderParameters.h:

*   **FShaderParameter**: shader parameter's register binding. e.g. float1/2/3/4, can be an array, UAV.
*   **FShaderResourceParameter**: shader resource binding (textures or samplerstates).
*   **FRWShaderParameter**: class that binds either a UAV or SRV of a resource.
*   **TShaderUniformBufferParameter**: shader uniform buffer binding with a specific structure (templated). This parameter references a struct which contains all the resources defined for a specific shader. More info later.

As many cases in the Unreal Engine, shader parameter classes make heavy use of macros to define how they are composed.  
The most important part of a shader parameter is its **Layout**, an internal variable defined at compile time that specifies its structure, composed of **Layout Fields**.

Layout is defined inside the class with macros like:
```cpp
	LAYOUT_FIELD(MyDataType, MyFiledName);
	LAYOUT_FIELD(FShaderResourceParameter, UAVParameter);
```
The way the layout will be used completely depends on the type of shader parameter, and it can contain any data (e.g. a parameter index).  
Its purpose is always to hold information about shader parameters (e.g. CBVs, SRVs, UAVs in D3D12) so that we can use them to bind resources at the moment of executing the shader in the command list.

### Shader Uniform Buffer Parameter

The concept of **Uniform Buffer Parameter** in Unreal Engine is very different from what we are used to in standard computer graphics: here it is essentially defined as a struct of shader parameters.  
Uniform Buffers, as previously mentioned in the RDG chapter, can be defined using a **Shader Parameter Struct** macro, either inside a shader class declaration or in global scope.

```cpp
	BEGIN_SHADER_PARAMETER_STRUCT(FMyShaderParameters, )

		SHADER_PARAMETER_RDG_TEXTURE(Texture2D, InputTexture)

		SHADER_PARAMETER_SAMPLER(SamplerState, InputSampler)

		RENDER_TARGET_BINDING_SLOTS()

	END_SHADER_PARAMETER_STRUCT()
```
This family of macros is very flexible and it can contain:



*   Shader Parameters: textures, samplers, buffers and descriptors. For a full list of macros reference to `ShaderParameterMacros.h`.
*   Nested Structs: we can encapsulate the definition of a shader parameter struct into another. This is achieved using the macro `SHADER_PARAMETER_STRUCT(StructType,MemberName)` and `SHADER_PARAMETER_STRUCT_ARRAY(..)`.
*   Binding Slots: available with the macro `RENDER_TARGET_BINDING_SLOTS()` that adds an assignable RenderTargets to the struct used to bind render targets.

#### Usage

*   If defined outside a shader, the name “FMyShaderParameters” will usually be **F&lt;MyPassName>PassParameters**. These shader parameter structs are used as Pass Parameters for the RDG, as described on the RDG section of this article.
*   If defined inside a shader class, the name “FMyShaderParameters” will usually be **FParameters**. We will also need to use this macro at the top of the shader class `SHADER_USE_PARAMETER_STRUCT(FMyShaderClass, FBaseClassFromWhatMyShaderDerivesFrom);`.

There is usually only one shader parameter struct per shader class, and In this way, we will be able to use the struct type inside an RDG pass body by writing `FMyShaderClass::FParameters`.  
`FParameters` will correspond to a uniform buffer parameter for the shader and it is supposed to hold the series of shader parameters that are specific for the graphics operation we want to operate.  
The way they work is the following:  
First create an object of type `FParameters` and fill it with references to the resources we want to use in the shader, then use such parameters when generating an RDG pass.  
We have then here 2 possible cases:

*   If the FParameters are relative to a compute shader, then `MyComputeShader::FParameters` will be passed as RDG Pass parameters, and later used to set resources on the shader before the dispatch command.
*   If the `FParameters` are relative to a shader in the graphics pipeline, they will be passed in the captured lambda or they will be created inside the lambda body from other captured data, then they will be assigned to the shader by calling `SetShaderParameters(CmdList, MyShader, MyInputParams);`

### Set Shader Parameters

Most of the times when using a shader inside an RDG pass, the function
```cpp
SetShaderParameters(TRHICmdList& RHICmdList, const TShaderRef&lt;TShaderClass>& Shader, TShaderRHI* ShadeRHI, const typename TShaderClass::FParameters& Parameters)
```
from `ShaderParameterStruct.h`  will be called to bind input resources to a specific shader.  
The function will first call ValidateShaderParameters(Shader, Parameters); to check that all the input shader resources cover all the expected shader parameters.  
Then it will start to bind all the resources to the relative parameters: every parameter type listed at the beginning of this section (e.g. `FShaderParameter`, `FShaderResourceParameter`, etc.) will have their own call for getting bound to the command list.  
A scheme of when we set a uniform buffer resource follows:

![](/assets\img\posts\2021-03-31-UE4ShadersIntroduction\SetUBShaderParameter_Scheme.jpg){:.postImg}

**BufferIndex** is used for all the `FParameterStructReference`, but also for the basic `FParameters` elements, since they are stored in buffers as well.  
What happens inside `CmdList::SetUniformBuffer` with such input parameters is completely up to the render platform we are using and it varies a lot from case to case.

## Compilation Process

There are various components and situations that can trigger the shader compiler, and in general this should be triggered by the Editor, either when editing assets or when cooking.  
Shader compilation path will also depend on the type of shader, e.g. if it is a global or a material shader.

When we cook an asset, for example, the materials associated with it will be cooked as well (without considering if materials were already in cache).  
A material will be first fully loaded in memory, then in post load will execute UMaterialInstance::BeginCacheForCookedPlatformData function, that will cache all the shaders and prepare them for the final saving process. Specifically, it will call **FMaterial::CacheShaders** that will do the following:

1. Try loading the material **Shader Map**, a container that stores all the shader references for the material.
2. If not already present, try to load it from the **Derived Data Cache (DDC)**, a permanent memory available only for Editor operations, that stores previously cooked data such as shader maps.
3. Check if the shader map is complete, this is done by comparing the current shader map hash** **with the one present on the cache.
4. If the shader map is incomplete or missing, the system will call BeginCompileShaderMap that will kick an async compile jobs for the current shader map, and so for all the shaders referenced into it.
5. Once all finished, check compilation results by calling FShaderCompilingManager::ProcessCompiledShaderMaps function.

**ShaderCompiler.cpp**, which is responsible to handle the queued shader compilations, will execute the following operations for each material shader:

1. When `FShaderCompileUtilities::ExecuteShaderCompileJob` function is called, it will trigger the actual shader compilation, which immediately enters in platform-dependent code. For this reason, the following steps are D3D12 platform specific, just to give an example.
2. For D3D12 `CompileD3DShader` function will be invoked which, after finalizing shader code with pre-compiler directives, will compile it by using DXC (the external D3D12 shader compiler program).
3. After compilation, there is an important step done when the `GenerateFinalOutput` function gets called: it creates the **Shader Resource Table (SRT)**. This table contains reflection data about the shader parameters, and it is obtained by extracting information from the compiled shader blob. SRT is used when we assign resources to shader slots to draw or dispatch something, so it is at the core of shader mechanics. If you have any bugs in this area they can cause crashes or other weird behaviors, so it is really important to understand how this system works, expecially when the `BuildResourceTableMapping` function is getting called.

### Debug Shader Compilation

In UE4 is possible to debug shader compilation. Credits to [this old UE4 blog post](https://www.unrealengine.com/en-US/tech-blog/debugging-the-shader-compiling-process) that explains how to.

1. Change ConsoleVariables.ini file to contain the following CVars
*   r.ShaderDevelopmentMode=1
*   r.DumpShaderDebugInfo=1
1. That will allow to Dump Shader intermediate Files, that can be found in: `YourGameDirectory\Saved\ShaderDebugInfo\YourPlatformName` these files will be generated each time you Save or Appy Changes to a material in Editor or when you BuildCookRun with UAT.
2. In each of the generated folders, shaders are grouped in VertexFactories and lastly in each folder you will find the following files:
    1. &lt;shader name>.usf : this is the last UE4 shader intermediate file before getting converted to platform specific.
    2. CompileHLSL.bat : this handy generated .bat file can take the .usf file and calling the platform specific compiler to generate the relative platform specific shader.
    3. DirectCompile.txt : this text contains the full command line argument to feed into ShaderCompilerWorker project in order to compile that specific shader!
3. So what we need to do at this point is: Switch to `Debug Editor - Win64` Solution.
4. Put a breakpoint in `D3DhaderCompiler.cpp` (or the one of your selected platform) where you need to.
5. Set `ShaderCompilerWorker` as the Startup Project, add the command line args found in DirectCompile.txt and run it
6. Easy Profit!

It is also important to notice we have the following additional debugging flags in `ConsoleVariables.ini`
```
;r.Shaders.AllowCompilingThroughWorkers=0

; Uncomment when running with a graphical debugger (but not when profiling)
;r.Shaders.Optimize=0

; When this is enabled, shaders will have extra debugging info. This could change patch sizes, uniqueness, etc and will recompile the shaders
;r.Shaders.KeepDebugInfo=1

; Uncomment to skip shader compression. Can save a significant time when using debug shaders.
;r.Shaders.SkipCompression=1
```

# Conclusions

This article made an introduction to what a shader object means in Unreal but there are a number of important topics that have been left over, and they might be the subject for future posts.  
Something that was not described is materials and their connection with shaders and vertex factories, and so how mesh material shaders work. Another missing topic is shader cache, and another one is shader permutations.  
There are very few resources on the web, but you can check the links on the sources sections to see what I have personally found.

# Sources

*	Unreal Engine 4 GitHub repo (available only on request)  
[https://github.com/EpicGames/UnrealEngine](https://github.com/EpicGames/UnrealEngine) 

*   Official UE4 Documentation  
[https://docs.unrealengine.com/en-US/RenderingAndGraphics/index.html](https://docs.unrealengine.com/en-US/RenderingAndGraphics/index.html)

*   Realities.io - Creating a Custom Mesh Component in UE4  
[https://medium.com/realities-io/creating-a-custom-mesh-component-in-ue4-part-1-an-in-depth-explanation-of-vertex-factories-4a6fd9fd58f2](https://medium.com/realities-io/creating-a-custom-mesh-component-in-ue4-part-1-an-in-depth-explanation-of-vertex-factories-4a6fd9fd58f2)

*   Matt Hoffman’s Blog  
[https://medium.com/@lordned](https://medium.com/@lordned)

*   GDC 2019 Marcus Wassmer - Refactoring the Mesh Drawing Pipeline for UE4.22  
[https://www.youtube.com/watch?v=qx1c190aGhs](https://www.youtube.com/watch?v=qx1c190aGhs)