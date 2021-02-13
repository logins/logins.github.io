---
layout: post
title:  "Resource Handling in D3D12"
subtitle: Managing resources in memory using D3D12.
author: "Riccardo Loggini"
tocmaxlevel: 2
tags: [rendering,dx12,d3d12]
---

# Why Talking about DX12 Resource Handling
Among the many changes that happened in the transition between Direct3D version 11 and 12, the idea of shader resources has been reworked quite a lot and the process to handle resources has become much more granular than before.  
D3D12 introduces a new binding model for resources compared to D3D11.  
Objects called resource views are created to reference buffer resources to the graphics pipeline, but now all the operations that were happening when binding a resource in D3D11 (e.g. make it resident in GPU memory, lifetime in memory checks, state handling, CPU-to-GPU memory mapping checks) are now fragmented in different calls to the SDK so that we can execute just a subset of them depending on the current needs.  
I talked about resource views in my previous [article regarding the Root Signature]({% post_url 2020-06-26-DX12RootSignatureObject %}).  
To use resources we need to know how we can use them, how they are integrated within the pipeline, what are the available types and how they are structured.

{% include githubLink.html 
description="To check a practical usage of resource handling you can refer to my github repo FirstDX12Renderer by clicking here."
link="https://github.com/logins/FirstDX12Renderer" %}

# Resource Types

A resource in D3D12 is created starting from a description object, usually of type [D3D12_RESOURCE_DESC](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_resource_desc) or, if we are using d3dx12.h header library, of type [CD3DX12_RESOURCE_DESC](https://docs.microsoft.com/en-us/windows/win32/direct3d12/cd3dx12-resource-desc).  
These objects will be fed to functions like [ID3D12Device::CreateCommittedResource](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createcommittedresource), depending on how and where we want to allocate such resource.  
More details about allocation types in the Memory Management chapter.  
When allocating a resource we usually get a relative [ID3D12Resource](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nn-d3d12-id3d12resource) interface pointer that represents it. Still, in most cases, we will then need to associate a View object to that resource so that it can be represented in the pipeline state object (e.g. bind it to the root signature).  
Views are also created from a relative description object, depending on the type of view.  
With a resource description we can fundamentally describe 2 resource types: buffers and textures.

**Buffers** can be bound to the graphics pipeline as:
    

-   Constant buffer (using a Constant Buffer View) in root signature
-   Shader resource (using a Shader Resource View) in root signature
-   Unordered access resource (using an Unordered access View) in root signature
-   Index buffer in the PSO
-   Vertex buffer in the PSO

**Textures** can be bound to the graphics pipeline as:
    
-   Shader resource (using a Shader Resource View) in root signature
-   Unordered access resource (using an Unordered Access View) in root signature  
-   Depth-stencil buffer (using a Depth Stencil View) in the PSO 
-   Render target buffer (using a Render Target View) in the PSO

Buffers are usually simpler resources to handle than textures, because textures are made to have more control over how their content is stored in memory and how it is modified.  
Textures will be treated in a standalone blog post in the near future.

# Resource Views

A **Resource View** is an object that fully qualifies and determine the usage type of a resource in the rendering pipeline.  
Resource views were already present since D3D11: their existence comes from the fact that resources are stored with general purpose formats.  
These formats can be identified in different ways depending on the usage we want to make out of each single resource.  
For example, a resource created with [DXGI format](https://docs.microsoft.com/en-us/windows/win32/api/dxgiformat/ne-dxgiformat-dxgi_format) `DXGI_FORMAT_R32G32B32A32_TYPELESS` can later at runtime be used as `DXGI_FORMAT_R32G32B32A32_FLOAT` or as `DXGI_FORMAT_R32G32B32A32_UINT` (or other types) depending on how we identify it with the corresponding view object.\\
One of the constraints we have with texel format is that they need to belong to the same **Format Family** which is usually the part of the format that follows `FORMAT_` and last until the first underscore "_" character.\\
For example `R32G32B32` family, defining types for three-component 96-bit data, includes the following formats `DXGI_FORMAT_R32G32B32_TYPELESS`, `DXGI_FORMAT_R32G32B32_FLOAT`, `DXGI_FORMAT_R32G32B32_UINT`, `DXGI_FORMAT_R32G32B32_SINT`.  
  

A view also defines properties of usage for the resource when set in the pipeline.\\
We have:

- **Constant Buffer View (CBV)** created with [ID3D12Device::CreateConstantBufferView](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createconstantbufferview) method to access shader constant buffers.

-   **Shader Resource View (SRV)** created with [ID3D12Device::CreateShaderResourceView](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createshaderresourceview) method to access a shader resource such as a constant buffer, a texture buffer (buffer with texture data stored in it), a texture or a sampler.
    
-   **Unordered Access View (UAV)** created with [ID3D12Device::CreateUnorderedAccessView](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createunorderedaccessview) method to access a unordered resource (roughly a subset of the shader resource types) with atomic operations (thread-safe) in a pixel or a compute shader.
    
On a separate category we also have:

-   **DepthStencilView (DSV)** created with [ID3D12Device::CreateDepthStencilView](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createdepthstencilview) method to access a texture resource during depth-stencil testing.
    
-   **Render Target View (RTV)** created with [ID3D12Device::CreateRenderTargetView](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createrendertargetview) method to access a texture resource as a render-target.

- **Stream Output View (SOV)**  created with   [ID3D12Device::CreateStreamOutputView](https://microsoft.github.io/DirectX-Specs/d3d/CountersAndQueries.html#stream-output-counters) to reference a stream output buffer.

These views can only be allocated on CPU descriptor heaps (more on this later) and they are transferred to GPU only at the moment of registering commands on the command list (e.g. by using functions like [OMSetRenderTargets](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/nf-d3d12-id3d12graphicscommandlist-omsetrendertargets)).

The last type of views are: 

- **Index Buffer View (IBV)** created with [D3D12_INDEX_BUFFER_VIEW](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_index_buffer_view) structure to reference an index buffer on GPU.

- **Vertex Buffer View (VBV)** created with [D3D12_VERTEX_BUFFER_VIEW](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_vertex_buffer_view) structure to reference a vertex buffer on GPU.

which are also available in CPU only, and used at command list record time, but they do not have specific descriptor heaps to be allocated on.\\  
These last two are also set directly at command list registration by using [ID3D12GraphicsCommandList::IASetIndexBuffer](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-iasetindexbuffer) method and [ID3D12GraphicsCommandList::IASetVertexBuffers](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-iasetvertexbuffers) method.

# Descriptors

A **Descriptor** is a memory storage for a Resource View in CPU and/or GPU.\\
If they are allocated both on CPU and GPU, we will have to use copy operations to transfer changes from one side to the other upon needs.  
The root signature will ultimately use descriptors: such descriptors will contain views that reference the resources (and the type of usage) we want in the pipeline.

>Note: the driver does not track the validity of descriptors: the application has the responsibility over the validity of the descriptors in use, and so the relative stored view objects and the resources referenced by them.

D3D12 wants a descriptor bound for all the slots required by shaders, and so by the root signature.\\
Sometimes we want to use a shader without using all its resource slots.\\
That is why we can create **Null Descriptors** by, when creating a view, specifying a resource desc but not the resource itself.

Some other times we might not need to specify custom settings to describe the usage of a resource and we can leave the API set the default settings for the view object we want to create.\\
These types are called **Default Descriptors**, and they are made by, when creating a view, setting the `VIEW_DESC` input to `nullptr` and just filling the input resource parameter.

## Descriptor Heaps

Descriptors are allocated in **Descriptor Heaps**, a different data structure from where we allocate resources: we will first need to allocate a descriptor heap to then be able to store descriptors in it.

Descriptor heaps are created from a device with [ID3D12Device::CreateDescriptorHeap](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createdescriptorheap) that needs a `D3D12_DESCRIPTOR_HEAP_DESC` object as input.  
When using such method, the flag **Shader Visible**    `D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE` is quite revant, because it will determine if the descriptor heap will be allocated in GPU other than CPU.

If this flag is used, there is no automatic synchronization mechanism between CPU and GPU side, and all the changes we want to operate in them will have to be done manually.  
To bind a descriptor heap to a command list, the heap needs to be shader visible.

Descriptor heaps are mainly of two types

-   **CBV_SRV_UAV** that will be able to store descriptor ranges containing SBVs, SRVs and UAVs. These are the only types of descriptors to be referenced in a descriptor heap since the only ones that can be used as resources by shaders.
    
-   **SAMPLER** that will be able to store sampler objects definitions.
    

>Note: Only a single `CBV_SRV_UAV` descriptor heap and a single `SAMPLER` descriptor heap can be bound to the command list at the same time.  
The command list will have to bind a descriptor heap with [ID3D12GraphicsCommandList::SetDescriptorHeaps](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-setdescriptorheaps) method in order to use any descriptor!

## Descriptor Handles

It is important to note that the methods generating the views all have:

-   Input a resource and/or a `VIEW_DESC` object.  
  
-   Output a **CPU descriptor handle** that represents the start of the descriptor heap that holds the view object.

A **Descriptor Handle** is a wrapper object for a memory address where a descriptor is stored.

The API expects the programmer to work exclusively with the descriptor handle memory address (and relative pointer arithmetics) while dereferencing such address is undefined behavior.

There are two types of descriptor handle:

-   **CPU Descriptor Handle** contained in [D3D12_CPU_DESCRIPTOR_HANDLE](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_cpu_descriptor_handle) structure, holds a pointer to a descriptor allocated on CPU.
    
-   **GPU Descriptor Handle** contained in [D3D12_GPU_DESCRIPTOR_HANDLE](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_gpu_descriptor_handle) structure, holds a pointer to a descriptor handle allocated on GPU.// A valid GPU descriptor handle comes from a descriptor heap allocated on GPU.
    

We can retrieve the descriptor handle of the first descriptor allocated in a descriptor heap with

-   [ID3D12DescriptorHeap::GetCPUDescriptorHandleForHeapStart](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/nf-d3d12-id3d12descriptorheap-getcpudescriptorhandleforheapstart)
    
-   [ID3D12DescriptorHeap::GetGPUDescriptorHandleForHeapStart](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/nf-d3d12-id3d12descriptorheap-getgpudescriptorhandleforheapstart) where this returns a valid address when the descriptor heap has been allocated on GPU as well (shader visible).
    

When handling descriptor handles we will often need to perform pointers arithmetic. For doing that we need to take into consideration the **Descriptor Size** in memory, and that is hardware dependent (usually 32 or 64 bytes).  
It is possible to retrieve the exact current size with the function:

-   [ID3D12Device::GetDescriptorHandleIncrementSize](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/nf-d3d12-id3d12device-getdescriptorhandleincrementsize)
    

so that will be the unit measure to, for example, adding or subtracting from a descriptor handle address to move from one descriptor to another in memory.

## Copying Descriptors

When handling descriptors, in many different situations we might need to copy descriptor ranges from one descriptor heap to another.

One of the most common use cases is as follows

1.  First create ranges of descriptors, and so view objects, on CPU only (CPU visible) with the attributes we need for a specific command list usage. This is also known as **Staging Descriptors**.
    
2.  After all that we can copy all these descriptor ranges to a descriptor heap on GPU (shader visible), currently bound to the command list, so we will be able to use them in the rendering operations.
    
To copy descriptors from one heap to another we can use either

-   [ID3D12Device::CopyDescriptors](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/nf-d3d12-id3d12device-copydescriptors)
    
-   [ID3D12Device::CopyDescriptorsSimple](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/nf-d3d12-id3d12device-copydescriptorssimple)
    

where the input descriptors need to be on CPU and the destination descriptors can be on CPU only or on GPU as well (shader visible).

## Using Descriptors in Root Signature

Once we have updated the descriptors we want to use on the descriptor heap currently bound to the command list, we need to reference them in the root signature to be able to ultimately reference the resources we want to use in shaders.

![](/assets/img/posts/2020-07-31-DX12ResourceHandling/Descriptors_Scheme.jpg){:.postImg}

Resuming the steps for the flow we usually go through with descriptors:

 1. Create a descriptor heap on GPU.
    
 1. Bind such descriptor heap to the command list.
    
 2. Create views to represent resources we want to use in shaders and copy the relative descriptors to the GPU descriptor heap.
    
 3. Create a descriptor range with [D3D12_DESCRIPTOR_RANGE](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_descriptor_range) structure to define the link between our descriptor range in memory and the shader registers we want to use with it. A descriptor range will define 
    * The range type (either CBV, SRV, UAV or SAMPLER)
        
    * Num descriptors in this range
        
    * Base shader register for the range, e.g. if we want to use an SRV descriptor range to be referenced from t0 in shaders, here we are going to set 0.
        
    * DescriptorOffsetFromTableStart descriptor offset from the staring one referenced by the root table that contains this descriptor range.
        
    * Register Space is described in the previous article about the root signature.
    
6.  Create a Root Table object with [D3D12_ROOT_DESCRIPTOR_TABLE](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_root_descriptor_table) structure that references such range.
    
7.  Create a Root Parameter with [D3D12_ROOT_PARAMETER](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_root_parameter) structure referencing the previous descriptor table, or being a bit less verbose, using first [CD3DX12_DESCRIPTOR_RANGE1](https://docs.microsoft.com/en-us/windows/win32/direct3d12/cd3dx12-descriptor-range1) and then [CD3DX12_ROOT_PARAMETER1](https://docs.microsoft.com/en-us/windows/win32/direct3d12/cd3dx12-root-parameter1) structure and call `InitAsDescriptorTable`.  
    The root parameter is essentially just a wrapper for either a root constant, a root descriptor or a root table and it is used to initialize the root signature.
    
8.  Create the root signature with the created root parameter bound into it.
    
9.  To finish we want to set the root table content bound to the root signature at runtime before issuing render commands. We can do so with:
    
    *  [ID3D12GraphicsCommandList::SetGraphicsRootDescriptorTable](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-setgraphicsrootdescriptortable) method for a graphics pipeline state.
        
    *  [ID3D12GraphicsCommandList::SetComputeRootDescriptorTable](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-setcomputerootdescriptortable) method for a compute pipeline state.
    

>Note: This was an example using a root descriptor table, but other cases include root constants and root descriptors. Still using descriptor tables is the most common case.\\
You can read more about root descriptors in my previous [article about the root signature]({% post_url 2020-06-26-DX12RootSignatureObject %}).

# Memory Management

Starting by briefly describing the types of GPU memory we can consider within an application:

-   **Dedicated Video Memory**: reserved to be read and written on by the graphics device, this is the place for a big part of the resources used by the shaders at the moment of graphics commands execution.  
    The amount can be queried from the adapter with [DXGI_ADAPTER_DESC.DedicatedVideoMemory](https://docs.microsoft.com/en-us/windows/win32/api/dxgi/ns-dxgi-dxgi_adapter_desc).  
    To give an example, Nvidia GeForce RTX2080ti has 11 GB of dedicated video memory.
    
-   **Dedicated System Memory**: allocated at boot time, it is a part of dedicated video memory that the GPU reserves for internal processes and therefore not available for use by applications. The amount can be queried from [DXGI_ADAPTER_DESC.DedicatedSystemMemory](https://docs.microsoft.com/en-us/windows/win32/api/dxgi/ns-dxgi-dxgi_adapter_desc).
    
-   **Shared System Memory**: is a region of the GPU memory which is visible from CPU memory and used to transfer resources from the latter to the former.  
    This wide visibility is paid off by this memory being slower than dedicated video memory, and it should be used just for the resource upload phase.  
    The amount can be queried from [DXGI_ADAPTER_DESC.SharedSystemMemory](https://docs.microsoft.com/en-us/windows/win32/api/dxgi/ns-dxgi-dxgi_adapter_desc).
    

That being said, a series of graphics adapters uses an **Uniform Memory Access (UMA)** model, where the memory available for the user is all shared system memory.  
This factor consequently produces lower performance, and this kind of architecture is often found in lower spec, integrated GPUs on laptops.  
The series of GPU architectures identified by the API to have both dedicated and shared system memory available to the programmer are called **Non-UMA (NUMA)**. Working with a GPU NUMA architecture raises the expectation to have the most part of available memory to be dedicated.  
  
When a resource in GPU memory can be accessed, it is called **resident**.  
When most of the d3d12 objects are created, an amount of memory in GPU is made resident, to then being switched back to non-resident upon objects deletion.  
All the allocations need to be manually handled by the programmer.

Unlike previous APIs, D3D12 does not track the lifetime of objects in memory. This means that when we want to deallocate a resource, we need to make sure that all the graphics operations that involve it have finished executing first.

Resources are associated with a **virtual address space** in GPU memory that is mapped to physical memory.

Direct3D graphics system manages resources in memory to a **Subresource** granularity, which means a subresource is the minimal object we can access from memory (at exclusion of tiled resources).  
D3d12 provides methods to access to specific subresources such as [ID3D12Resource::ReadFromSubresource](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/nf-d3d12-id3d12resource-readfromsubresource) or [ID3D12Resource::WriteToSubresource](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/nf-d3d12-id3d12resource-writetosubresource).

-   For buffer resources, each of them can be identified as containers of subresources: we can linearly subdivide the memory of a buffer into multiple smaller areas that will contain a resource each. Still, this is not necessary.
    
-   For texture resources the subresource assignment is more complicated.  
    For example, in Texture2D resources each mip level is a subresource.  
    This and other details will be treated in a future standalone article about textures.
    
## Alignment

When talking about D3D12 memory management, **alignment** for a resource is the unit measure of memory blocks required to allocate such resource.  
The allocation size required for a resource will be equal to the first multiple of alignment greater than the resource size.  
Alignment is always a power of two, and we can query it alongside the resource’s size by using [ID3D12Device::GetResourceAllocationInfo](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-getresourceallocationinfo) function, which takes one or more [D3D12_RESOURCE_DESC](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_resource_desc) as input.

D3D12 heap resource alignment appears in forms of common fixed constants:

-   64KB for every buffer and base textures types
    
-   4MB for MSAA textures
    
> Note: these constants are intended in terms of resource heap allocations.  
  
We can also use `D3D12_DEFAULT_RESOURCE_PLACEMENT_ALIGNMENT` constant instead of specifying buffer and base textures alignment size for the functions that require it and
`D3D12_DEFAULT_MSAA_RESOURCE_PLACEMENT_ALIGNMENT` for MSAA textures, but we can also set the field to 0, and that will be automatically resolved by the SDK depending on the resource. Still those values represent the minimum size for a resource heap memory allocation.

### Alignment In Buffers

In the case of buffers, we can use a single resource allocation in memory to contain multiple subresources: 64KB is quite a big size for a single buffer resource (e.g. a model-view-projection matrix with 4 bytes per component would be just 64 bytes in total size).

![](/assets/img/posts/2020-07-31-DX12ResourceHandling/AlignmentBuffer_Scheme.jpg){:.postImg}

Still, the minimum size requirement for a Constant buffer resource is 256 bytes, still much smaller than the requirement to perform a resource allocation. This number is also represented by the `D3D12_CONSTANT_BUFFER_DATA_PLACEMENT_ALIGNMENT` constant.  
Vertex buffers have a maximum alignment of 4 bytes, so much smaller than constant buffers, and they can still be contained in the same resource heap as the constant buffers.

### Small Resources

In the case a texture resource is small enough that its most detailed mip level can fit inside a single alignment size, that texture will be technically called **small resource** and can use a smaller alignment of 4KB.  
Small resources can be MSAA texture as well, and if their most detailed mip fits an alignment, they can use a smaller alignment of 64KB.  
>Note: Buffers cannot be small resources.  

There are specific rules to use small resources correctly, and needless to say, it is just a memory optimization, but to know more about it you can read the article [Secrets of Direct3D 12](https://asawicki.info/news_1726_secrets_of_direct3d_12_resource_alignment) by Adam Sawicki.

## Heap Types

A heap is the lowest level object to manage physical memory in D3D12.  
Heap memory residency covers the whole object as a unit, so we cannot make resident just a part of it.  
Heaps are objects usually created by using a heap description object [D3D12_HEAP_DESC](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/ns-d3d12-d3d12_heap_desc), which contain a field [D3D12_HEAP_TYPE](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_heap_type) enumeration to specify what type of heap we want to instantiate. Heap types fundamentally split in two categories: abstracted types and custom type.  
**Abstracted heap types** are the ones that abstract the current hardware, so that there is a smooth transition between different platforms and relative adapters (e.g. passing from NUMA to UMA GPUs). There are three types of abstracted heaps: default, upload and readback.

-   **Default Heaps** reside in dedicated GPU video memory, so they will have GPU exclusive access and they will be the fastest to be accessed from.
    
-   **Upload Heaps** reside in shared GPU video memory, they will be slower than default heap and generally less memory available to expand, but they will have CPU memory access and they will be optimized for resource upload.
    
-   **Readback Heaps** also reside in shared GPU video memory and they will be optimized to read resources back from GPU to CPU.
    

>Note: To instantiate a resource in a default heap we are going to need an upload heap to transfer the resource from CPU to GPU, and then copy such resource from the upload heap to the default heap. This is explained in the Resource Mapping chapter.  
  
>Note: texture resources cannot be allocated in Upload and Readback heaps.

**Custom heap type** was created to optimize resource allocation based on hardware: by using it, the application can specify memory pool and CPU cache properties.  
This type of heap comes useful for specific scenarios such as for a multi-adapter application.

## Resource Allocation Types

We can classify resource allocation types between Committed, Reserved or Placed.

### Committed Resources

![](/assets\img\posts\2020-07-31-DX12ResourceHandling\CommittedResource_Scheme.jpg){:.postImg}

Instantiating a committed resource will create a resource heap big enough to fit the whole resource, this is the most standard allocation type inherited from the previous D3D versions.  
We can allocate a committed resource by using a graphics device object and calling [ID3D12Device::CreateCommittedResource](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createcommittedresource).

### Placed Resources

![](/assets\img\posts\2020-07-31-DX12ResourceHandling\PlacedResource_Scheme.jpg){:.postImg}

Placed Resources are the ones that will be instantiated in an already existing heap big enough to contain the resource. For this reason, they are the lightest weight resource types available, and they are the fastest to create and destroy.  
They are created by calling the [ID3D12Device::CreatePlacedResource](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createplacedresource#remarks) function.  
Placed resources provide a lighter way to reuse memory than resource creation and destruction: **resource aliasing**.

### Resource Aliasing

Resource aliasing allows multiple placed resources to overlap on their memory usage in the heap, but only a single one between the overlapped resources can be used at a time.  
Placed resources can use resource aliasing in two main ways: simple and advanced model.

**Simple Model** defines two possible states for the placed resources: active or inactive.  
If a resource is inactive it cannot be read or written by the GPU and all the resources are created by being in an inactive state.  
To transition a resource from inactive to active or vice versa we need to use a transition with [D3D12_RESOURCE_ALIASING_BARRIER](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_resource_aliasing_barrier): the resource passed in the `pResourceAfter` field will be activated and, at the same time, all the other resources sharing memory with the chosen one will be deactivated (and them being overlapping placed and/or reserved resources).  
After activation, if the resource is a render target or a depth-stencil resource, it will require initialization by either:

-   A Clear operation: [ClearRenderTargetView](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-clearrendertargetview) or [ClearDepthStencilView](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-cleardepthstencilview).
    
-   A [DiscardResource](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-discardresource) operation, that will declare the content as being discarded, this is quicker than a full clear (in UnrealEngine4 it is comparable to a FastClear operation) and it is preferred when we know that the next rendering operations will rewrite the whole buffer.
    
-   A Copy operation: [CopyBufferRegion](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-copybufferregion), [CopyTextureRegion](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-copytextureregion), or [CopyResource](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-copyresource).
    
**Advanced Model** is not recommended up until we need to optimize the resource aliasing from simple mode usage.  
With the advanced model we can avoid rules about active/deactive resource state but we still have to:

-   Put an aliasing barrier between any usage of two overlapping resources within the same [ExecuteCommandLists](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12commandqueue-executecommandlists) operation.
    
-   Initialise render target and depth-stencil resources just after the barrier like doing with the simple model.
    
>Note: An aliasing barrier is needed only for usages of overlapping resources within the same call to ExecuteCommandLists, because placing an aliasing barrier is an equivalent operation of how ExecuteCommandLists synchronizes its operations internally.

For more in depth information about resource aliasing you can refer to [ID3D12Device::CreatePlacedResource](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createplacedresource#remarks) method and [Memory Aliasing](https://docs.microsoft.com/en-us/windows/win32/direct3d12/memory-aliasing-and-data-inheritance) official documentation.

### Reserved Resources

![](/assets\img\posts\2020-07-31-DX12ResourceHandling\ReservedResource_Scheme.jpg){:.postImg}

Reserved Resources will allocate a virtual memory range but, initially, without association to any physical GPU memory heap. This is because we can later upload and map parts of the resource, from the CPU to GPU, upon user needs.  
They are the equivalent of [Tiled Resources](https://docs.microsoft.com/en-us/windows/win32/direct3d11/tiled-resources) in D3D11.  
Normally, the GPU can instantiate memory at a subresource granularity, that means that if we just need part of a subresource, for example just a part of a texture mip, we are still gonna need to upload the whole subresource in memory. This can turn very expensive if the resources in play are really large. A tiled resource can overcome this problem.  
Another usage is creating very large textures as a collection of smaller textures that are likely to be used at the same time, so we can load specific “sub textures” upon need.  
>Note: reserved resources can also operate resource aliasing like placed resources.  

To get more in depth about this topic and relative usage, it is advised to consult the official D3D11 [documentation of Tiled Resources](https://docs.microsoft.com/en-us/windows/win32/direct3d11/tiled-resources).

## Resource Mapping

Resource mapping allows a resource on GPU to be updated with contents from CPU memory.  
This is achieved by editing a resource on CPU, buffering all the changes and then reproducing them same in the GPU-allocated resource.  
The process is acted by a technique called **write combining** that cannot guarantee correct read/write ordering because changes to the memory area are buffered up and executed in groups.  
This operation is executed through a PCIe memory aperture ([more specific infos on the internet](https://fgiesen.wordpress.com/2013/01/29/write-combining-is-not-your-friend/)).

In D3D11, to update a buffer, we were already using an explicit mapping operation:  
```cpp
// Open a memory mapping CPU-to-GPU to update the buffer.
D3D11_MAPPED_SUBRESOURCE myMappedResource;
d3d11DeviceContext->Map(
myBuffer.Get(),
0,
D3D11_MAP_WRITE_DISCARD,
0,
&myMappedResource
);

memcpy(myMappedResource.pData, myBufferData, sizeof(myBufferData));

d3d11DeviceContext->Unmap(constantBuffer.Get(), 0);
```

When unmap gets called, all the editing to the resource will get reproduced to GPU memory.  
For a resource to be mappable in D3D11 it needs to have the usage to be `DYNAMIC` from its descriptor. 

Some resources go under the category of **Non-Mappable resources**, which are the ones allocated in Non-Mappable memory, a region included in the dedicated video memory (and so, directly editable only by the graphics device).  
An example are vertex buffers, and in these cases, we are going to need to call [ID3D11DeviceContext::UpdateSubresource](https://docs.microsoft.com/en-us/windows/win32/api/d3d11/nf-d3d11-id3d11devicecontext-updatesubresource) method where the device will take care of executing the operation for us.
```cpp
// Update the vertex buffer.
d3d11DeviceContext->UpdateSubresource(
vertexBuffer.Get(),
0,
NULL,
vertexBufferData,
sizeof(vertexBufferData),
0);
```

For the D3D12 counterpart, the operation involves different objects: we first need to create an **upload buffer**, a resource mapped in GPU shared memory, so that we can edit it from CPU and all the changes will be reflected in the GPU counterpart.  
The first operation is to create the upload buffer and mapping it to a buffer in CPU memory:
```cpp
ComPtr<ID3D12Resource> myUploadBuffer;
// ...
d3d12Device->CreateCommittedResource(
&CD3DX12_HEAP_PROPERTIES( D3D12_HEAP_TYPE_UPLOAD ),
...
D3D12_RESOURCE_STATE_GENERIC_READ,
IID_PPV_ARGS( &myUploadBuffer ) );

// …  
void* myUploadCPUPtr; // Filled by Map method

myUploadBuffer->Map( 0, nullptr, &myUploadCPUPtr );

myUploadCPUEndPtr = myUploadCPUPtr + UploadBufferSize;
```
  

When mapping the resource by [ID3D12Resource::Map](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12resource-map) method, we also declare the range of memory that is open for read from CPU, so that the system can keep coherency with the respective content in GPU.  
We set this to `nullptr` because we do not need to read from CPU, but only write into it.  
The `Map` function will store a pointer to CPU allocated data in its last argument.  
Then when we want to render a frame, we will first update the content of such resource (or of a subresource allocated inside that resource)  
```cpp
float myBufferData[]; //… resource content

// SIZE_T represents the maximum size to which a pointer can point
SIZE_T myBufferSize = sizeof(myBufferData); // in bytes  
// If the data we want to update is larger than the upload buffer max size, operation is invalid

// We first align data pointer
UINT alignUnit = sizeof<float>; // This depends on the data type stored in the buffers  
// alignUnit needs to be a power of 2  
if ( (0 == uAlign) || (uAlign & (uAlign-1)) )
ThrowException("non-pow2 alignment");

UINT dataLocation = reinterpret_cast< SIZE_T >(myUploadCPUPtr);

// Align location to the next multiple of alignUnit
dataLocation = (dataLocation + (alignUnit-1)) & ~(alignUnit-1);

// Pointer to UINT8 because the smallest data we can possibly point to
UINT8* alignedDataPtr = reinterpret_cast< UINT8* >( dataLocation );

if(myUploadCPUPtr + myBufferSize > myUploadCPUEndPtr)
ThrowException("Not enough space in the upload resource!");  
  
// Finally proceeding with filling the content of the sub-resource in the upload heap
memcpy(alignedDataPtr, myBufferData, myBufferSize);

// Then we create a view for the just uploaded buffer
D3D12_CONSTANT_BUFFER_VIEW_DESC constantBufferViewDesc = {
myUploadBuffer->GetGPUVirtualAddress(),
myBufferSize
};

// Allocate the view in a descriptor heap
d3dDevice->CreateConstantBufferView(
&constantBufferViewDesc,
... ));

// Later on binding that view to the root signature
```
  
>Note: It is worth to mention that if we have a single sub-resource, like in this case, we can identify the resource “myBuffer” as the whole upload buffer, so we would not need an initial alignment but the sub-resource process was listed for demonstration purposes.  
  

Note that here the resource was allocated in an Upload Heap and then used straight away.  
This can be non-optimal since the upload heap is allocated in GPU shared memory and we pay the cost to handle a memory that is also editable from the CPU.  
In addition to that, shared memory on GPU cannot guarantee read/write operations ordering, because writing operations are executed in groups.  
A better way handle it is to use a **default heap** and copy the content of the upload heap in it once we finish the editing on the resource.  
The advantage of doing that is a default heap is faster for the GPU to operate from, compared to an upload heap.  
Memory in the default heap will not be directly editable from the CPU, we will always have to pass through an upload heap copy operation to change it, but the GPU operations on the resource will be faster.
```cpp
// Instead of creating a view from the resource in the upload heap, we will copy the content to a default heap first and create a view out from it  
ID3D12Resource* myDestinationResource

// We will first create the (empty) destination resource in a default heap
ThrowIfFailed(device->CreateCommittedResource(
&CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),  
...  
D3D12_RESOURCE_STATE_COPY_DEST,
nullptr,
IID_PPV_ARGS(&myDestinationResource)));

// Then we copy the content of the upload heap into the destination resource  
D3D12_SUBRESOURCE_DATA subresourceData = {};

subresourceData.pData = myBufferData;
subresourceData.RowPitch = myBufferSize;
subresourceData.SlicePitch = subresourceData.RowPitch;

UpdateSubresources(commandList.Get(),
myDestinationResource, myUploadBuffer.Get(),
alignedDataPtr - reinterpret_cast< UINT8* >(myUploadCPUPtr) , // offset in bytes to allocated resource
0, 1, &subresourceData);
```

In this case we can use the [UpdateSubresources](https://docs.microsoft.com/en-us/windows/win32/direct3d12/updatesubresources1) function to update the resource (in this case sub-resource) allocated in the upload buffer.

>Note: UpdateSubresource gives the possibility to update multiple resources, allocated within a single buffer, at the same time. If multiple subresources need updating, subresourceData will become an array with information per each element to update.

When a command list using that resource executes, it expects the resource to be unmapped and all the changes to be already applied, but no checks are done about it, and this is again up to the user.

## Fence Objects For Resource Activity Tracking

**Fences** are objects that can help to keep track of resources activity, so that we can react upon events with custom functions, such as reusing memory of a resource that finished being used and not needed anymore.  
A fence works by **signaling** a command queue first, an operation that can be seen as inserting a command into that command list.  
When the command queue executes and reaches the fence’s signal, the fence object will be notified with an event so we can react to it from CPU side.  
Fences are useful in many cases, but in general, we signal a command queue just after a command from which we want to react upon its termination.

A fence object, usually identified by [ID3D12Fence](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/nn-d3d12-id3d12fence) type, is created from a device with [ID3D12Device::CreateFence](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createfence) method:
```cpp  
ComPtr<ID3D12Fence> myFence;  
d3d12Device->CreateFence(0, D3D12_FENCE_FLAG_NONE, IID_PPV_ARGS(&myFence));  
```
Its usage can be subdivided in two steps:

-   Signaling the command queue with [ID3D12CommandQueue::Signal](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12commandqueue-signal) method with an integer fence value that uniquely identifies the signal happened between the command queue and the fence object. We get to choose the value that we are going to use to signal the fence, so later we know what to expect.  
    `myCmdQueue->Signal(myFence.Get(), myFenceValue);`
    
-   On a tick function, we poll the fence object with [ID3D12Fence::GetCompletedValue](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12fence-getcompletedvalue) to check if the fence value has been reached.  
    `if (InFence->GetCompletedValue() <= myFenceValue)  
    { /* React Upon Signal Reached */ }`
      
    

Optionally, we can bind an event to the fence by using [ID3D12Fence::SetEventOnCompletion](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12fence-seteventoncompletion) that will specify what function to call when the signal is reached.  
The event is represented by a [HANDLE](https://docs.microsoft.com/en-us/windows/win32/WinProg/windows-data-types) object, obtained by calling [::CreateEvent](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-createeventa).  
This mechanic can be used by specific functions such as `::WaitForSingleObject` that will stall the current thread up until the event gets signaled.
```cpp 
if (myFence->GetCompletedValue() < myFenceValue) {
    myFence->SetEventOnCompletion(myFenceValue, myFenceEvent);

    ::WaitForSingleObject(myFenceEvent, INFINITE);
}
```
  
To see a small example of a fence used to control a ring buffer for resource allocations, you can consult [the official documentation](https://docs.microsoft.com/en-us/windows/win32/direct3d12/fence-based-resource-management).

# Resource State Transitions

In D3D12 we manually define the purpose of a resource by transitioning its state, and this action is decoupled from resource binding for optimization purposes.  
Shaders expect each referenced resource with a determined **state** (e.g. reading from a texture, that texture needs to be in read state) and it is up to the programmer to make sure the resources are in the expected state.  
Resources might need to be created with a specific state and/or remain on a specific state, depending on the type of resource and also where is created.  
For example, resources allocated in upload heaps must be created on one of the states from `D3D12_RESOURCE_STATE_GENERIC_READ`, and resources allocated in readback heaps must be created and cannot change from state `D3D12_RESOURCE_STATE_COPY_DEST`.  
For list of specification you can refer to [the official documentation](https://docs.microsoft.com/en-us/windows/win32/direct3d12/using-resource-barriers-to-synchronize-resource-states-in-direct3d-12).

If we need to transition just a set of subresources within a single resource, we can do it as well.

A state transition is acted by objects called transition barriers that are created from a barrier description object (depending on the barrier type) and that will be entered into a command list using ad-hoc functions.

### Resource Barriers

Resource barriers are used to change the state of use for resources, among the available states from [D3D12_RESOURCE_STATES](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/ne-d3d12-d3d12_resource_states) and still respecting the limitations imposed by the system (e.g. a resource in a readback heap cannot change state).

They are created using a A [D3D12_RESOURCE_TRANSITION_BARRIER](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_resource_transition_barrier) object and we are going to need to specify the current state of the resource, other than the new one for the transition.  
The way of always knowing the current state for a resource is up to the programmer, and usually there are containers in place in a graphics application that keep track of it.
The description object will be then used with the command list calling [ID3D12GraphicsCommandList::ResourceBarrier](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-resourcebarrier) function.  
For a practical example you can refer to [Lesson 3 of 3DGEP.com](https://www.3dgep.com/learning-directx-12-3/) .  
D3D12 also provides the **debug layer**, a system that among other things, can also warn us if we are using a resource in a wrong state, for debug purposes.

### UAV Barriers

Unordered access view (UAV) barriers are used to make all the UAV accesses to a resource (both read and write) to complete before starting more UAV operations on it.  
This is mainly useful when we have two or more UAV operations between draw or dispatch calls where we expect some of them to finish (e.g a write) before starting the others.  
They are created from a [D3D12_RESOURCE_UAV_BARRIER](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_resource_uav_barrier) description object, and if we don’t specify the target UAV resource, it will mean that any UAV access could require a barrier.

### Aliasing Barriers

Aliasing barriers are the one used for placed or reserved resources, when in the same command list execution we want to switch the use between overlapping resources, as mentioned in the Placed Resources chapter.  
They are created starting from a [D3D12_RESOURCE_ALIASING_BARRIER](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_resource_aliasing_barrier) description object.


# ~ Sources

-   [Official D3D12 Graphics documentation](https://docs.microsoft.com/en-us/windows/win32/api/_direct3d12/)

-   [3DGEP - Part 3](https://www.3dgep.com/learning-directx-12-3)
    
-   [Diligent Graphics](http://diligentgraphics.com/diligent-engine/architecture/d3d12/#Buffers_and_Buffer_Views)
    
-   [Adam Sawicki - Resource Alignment](https://asawicki.info/news_1726_secrets_of_direct3d_12_resource_alignment)
    
-   [Braynzar Soft - Constant Buffers ](https://www.braynzarsoft.net/viewtutorial/q16390-directx-12-constant-buffers-root-descriptor-tables)