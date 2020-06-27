---
layout: post
title:  "The DX12 Root Signature Object"
subtitle: Composition and usage of the object that holds references to shader resources in Direct3D 12.
author: "Riccardo Loggini"
tocmaxlevel: 3
tags: [rendering,dx12]
---

# Why talking about the Root Signature?

The Root Signature is the object that represents the link between the command list and the resources used by the pipeline.  
It specifies the data types that shaders should expect from the application, and also which pipeline state objects are compatible (those compiled with the same layout) for the next draw/dispatch calls.

{% include githubLink.html 
description="To check a practical usage of the root signature you can refer to my github repo FirstDX12Renderer by clicking here."
link="https://github.com/logins/FirstDX12Renderer" %}

> Note: Reader here is assumed to have a general understanding of D3D12 graphics pipeline and resources. You can read [my article about the Pipeline State Object (PSO)]({% post_url 2020-04-12-DX12PipelineStateObject %}) and an article fully dedicated to shader resources will come later on.

# Composition

![](/assets/img/posts/2020-06-26-DX12RootSignatureObject/RootSignature_Scheme.jpg){:.postImg}

The root signature object is composed by static sampler descriptors and **root parameters**. These last can be either root constant, root descriptors or root tables.
The maximum size of the root signature object is 64 DWORDs, where 1 DWORD is 32 bits/4 bytes. Majority of this space will be taken by the root parameters, which have different size depending on their type.

Root signature object creation starts from defining a [D3D12_ROOT_SIGNATURE_DESC](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_root_signature_desc) that, alongside with flags defined before, it is essentially supposed to contain an array of [D3D12_ROOT_PARAMETER](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_root_parameter) objects.
```cpp
D3D12_ROOT_PARAMETER[] rootParams;
D3D12_STATIC_SAMPLER_DESC[] staticSamplers;
// Set root params here, more details later
D3D12_ROOT_SIGNATURE_DESC myRootSignatureDesc = {
    _countof(rootParams),
    rootParams,
    _countof(staticSamplers),
    staticSamplers,
    rootSignatureFlags
};
```

A graphics command list has both a graphics and a compute root signature. They are different and independent from each other.
This root signature description will then be serialized with [D3D12SerializeRootSignature](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/nf-d3d12-d3d12serializerootsignature) function, which will create a corresponding binary blob.  
The blob can then be used as input of [ID3D12Device::CreateRootSignature](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/nf-d3d12-id3d12device-createrootsignature) to finally create the [ID3D12RootSignature](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nn-d3d12-id3d12rootsignature) referenced object.  

> Note: the binary blob is made to store the root signature in permanent memory, e.g. in shader cache, so once it has been computed, it can be loaded on next applications run without the need of recomputing it again!

As a small optimization on some hardware, we can start by specifying some flags to deny root signature access to some pipeline stages  
```cpp
D3D12_ROOT_SIGNATURE_FLAGS rootSignatureFlags =
D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT |
D3D12_ROOT_SIGNATURE_FLAG_DENY_HULL_SHADER_ROOT_ACCESS |
D3D12_ROOT_SIGNATURE_FLAG_DENY_DOMAIN_SHADER_ROOT_ACCESS |
D3D12_ROOT_SIGNATURE_FLAG_DENY_GEOMETRY_SHADER_ROOT_ACCESS |
D3D12_ROOT_SIGNATURE_FLAG_DENY_PIXEL_SHADER_ROOT_ACCESS;

```    

As an alternative to creating the traditional root signature descriptor, we can also make use of [D3DX12 Header Library](https://github.com/microsoft/DirectX-Graphics-Samples/blob/master/Libraries/D3DX12/d3dx12.h) and instead use [CD3DX12_ROOT_PARAMETER1](http://cd3dx12_root_parameter1) to define the array of root parameters that we want to use, and [CD3DX12_VERSIONED_ROOT_SIGNATURE_DESC](https://docs.microsoft.com/en-us/windows/win32/direct3d12/cd3dx12-versioned-root-signature-desc) to describe the root signature we want to use:
```cpp  
CD3DX12_ROOT_PARAMETER1 rootParameters[n];

// Init root parameters here, more details later

CD3DX12_VERSIONED_ROOT_SIGNATURE_DESC rootSignatureDesc;

rootSignatureDesc.Init_1_1(_countof(rootParameters), rootParameters, 0U, nullptr, rootSignatureFlags);
```

We serialize and create the root signature object by using [D3DX12SerializeVersionedRootSignature](https://docs.microsoft.com/en-us/windows/win32/direct3d12/d3dx12serializeversionedrootsignature) and [ID3D12Device::CreateRootSignature](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createrootsignature):
```cpp
::D3DX12SerializeVersionedRootSignature(rootSignatureDesc, InVersion, rootSignatureBlob.GetAddressOf(), errorBlob.GetAddressOf());

ComPtr<ID3D12RootSignature> rootSignature;

Device->CreateRootSignature(0, rootSignatureBlob->GetBufferPointer(), rootSignatureBlob->GetBufferSize(), IID_PPV_ARGS(&rootSignature));
```
  
From Shader Model 5.1 is also possible to define the entire root signature in shader code. Since this method goes beyond the scope of this article, you can refer to [the official documentation](https://docs.microsoft.com/en-us/windows/win32/direct3d12/specifying-root-signatures-in-hlsl).

## Version 1.1

The root signature version 1.1 introduces optimizations concerning the editability of root descriptors and the data pointed by them.  
By specifying if a descriptor or its resource is meant to be static, so not changing during command list recording (setup) and execution, it will allow the driver to make optimizations on the root signature.  
For example, from the driver, if a resource pointed by a descriptor table is found to be static, then the driver can promote that descriptor from the table to the root signature itself, making it a root descriptor instead.  
The effect will correspond in removing a level of indirection from the descriptor, making it faster to be accessed. More info about level of indirection will be found in the following chapters.

We can first check the supported feature version by the current device we are using with:
```cpp
D3D12_FEATURE_DATA_ROOT_SIGNATURE featureData = {};

featureData.HighestVersion = D3D_ROOT_SIGNATURE_VERSION_1_1;

if (FAILED(m_GraphicsDevice->CheckFeatureSupport(D3D12_FEATURE_ROOT_SIGNATURE, &featureData, sizeof(D3D12_FEATURE_DATA_ROOT_SIGNATURE))))
{
featureData.HighestVersion = D3D_ROOT_SIGNATURE_VERSION_1_0;
}
```
  

Then the version can be used as an input parameter when we serialize the versioned root signature (referring to the previous code examples).

  
The following concepts are introduced in root signature version 1.1:

-   **Static Descriptors**: the driver will take for granted that these descriptors will not change after being binded to the root signature or, in the case of table descriptor ranges, to not change anymore after the descriptor table has been bound to the root signature.  
    This is the default state for CBV and SRV in root signatures v1.1.
    
-   **Volatile Descriptors**: the driver will consider these descriptors to possibly change during the recording(setup) of the command list. They still cannot change when the command list have commands “on-the-fly” in execution.  
    By default the data pointed by CBV and SRV will be considered static while set at execute (see below) and for UAV relative data will be considered volatile.
    
-   **Static Data**: driver assumes the data to not change after the descriptor has been referenced in the root signature or descriptor table.  
    By default the relative descriptor is set to be static.  
    This state will grant the maximum level of optimization for the driver.
    
-   **Static Data While Set At Execute**: driver assumes that the data pointed by the descriptor can change up until the command list starts executing and stays unchanged for the rest of the execution.  
    By default the relative descriptor is considered static.
    
-   **Volatile Data**: the driver will consider the data pointed by the descriptor to be editable both at command list registration and during command list execution.  
    By default the descriptor will also be considered volatile.  
    This is the default state for every descriptor in root signatures version 1.0.
    

These concepts translate into flags that can be set for each root parameter with [D3D12_ROOT_DESCRIPTOR_FLAGS](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/ne-d3d12-d3d12_root_descriptor_flags) and each descriptor range with [D3D12_DESCRIPTOR_RANGE_FLAGS](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/ne-d3d12-d3d12_descriptor_range_flags) for root signature objects of version 1.1.  
These objects will be part of [D3D12_ROOT_SIGNATURE_DESC1](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/ns-d3d12-d3d12_root_signature_desc1) .

If we do not set static/volatility flags for our root descriptors and descriptor ranges, using a root signature version 1.1 will set descriptors to be static and data static while set at execute for CBVs and SRVs, while for UAVs data will be set as volatile. The default choice for UAVs was made because usually data in UAVs is meant to be changed while executing a compute shader.

More details can be found [in the official documentation](https://docs.microsoft.com/en-us/windows/win32/direct3d12/root-signature-version-1-1).

> Note: Root signatures defined in shaders will be compiled to version 1.1 by default if using modern HLSL compilers (the ones aware of version 1.1).

# Root Parameters

A root parameter is one entry in the root signature. It can be either a Constant, a Descriptor or a Table.

Each entry type has a different **cost towards a maximum limit**, and a **level of indirection**: a measure of complexity to access that resource.

  

Each root parameter (either defined by a D3D12_ROOT_PARAMETER or CD3DX12_ROOT_PARAMETER1) will have a **shader visibility** parameter that will state what shader types are allowed to access the resource, defined as [D3D12_SHADER_VISIBILITY](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_shader_visibility) enum. This is a tool for the programmer to differentiate resources needed in various stages, for example we can define two root parameters that bind to the same slot, but one with visibility vertex and one with visibility pixel, so that different resources will be bound to the same shader register depending on what shader is invoking them.  
For example, vertex shader may declare  
`Texture2D myVertexInputTexture : register(t0);`

And at the same time, pixel shader can have  
`Texture 2D MyPixelInputTexture : register(t0);`

And the two still referring to different textures.  
We need to be sure that visibility on two different root parameters referencing the same shader register in the same **namespace** (so in the same set of shader bindings) does not overlap, like one of them specifies visibility `_ALL`, because in that case, layout will be invalid and root signature creation will fail.  
In terms of performance, changing shader visibility to one stage or all stages will cost the same.

## Root Constants

Root constants are constants inlined in the root arguments. They will show up as constant buffers in the shaders.  
Each root constant is measured in chunks of 32-bit values.

Root constants have no level of indirection, meaning they are used without any intermediate object to query in order to access them. This makes them the fastest type of resources to access.

Each root constant cost 1 **DWORD** (32 bits/4 bytes) out of the 64 maximum size of the root signature, since it is a 32-bit constant.  
When we have bigger data to upload, such as a model-view-projection matrix, we are gonna need to upload several (e.g. size of the matrix in bytes divided by 4) constants at the same time.

  

Root constants are defined in the root signature with a [D3D12_ROOT_CONSTANTS](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_root_constants) object and the relative root signature slot type needs to be `D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS` .  
Alternatively, we make use of [D3DX12 Header Library](https://github.com/microsoft/DirectX-Graphics-Samples/blob/master/Libraries/D3DX12/d3dx12.h) and instead use 
```cpp 
struct MyDataType {
    float myData1;
    int myData2;
}  
CD3DX12_ROOT_PARAMETER1 rootParameters[1];

rootParameters[0].InitAsConstants( sizeof(MyDataType) / 4, … );
```
to speed up the insertion process.

Dynamic indexing is not permitted in the root signature space, so we cannot use a data type like  
```cpp
struct MyData{  
    int[2] myIndexes;  
}  
```
as root constants. We are going to need to define them separately like 
```cpp
struct MyData{  
    int myIndex1;  
    int myIndex2;  
}
```
or we can store a constant buffer view descriptor instead (that is going to be slower to access).

On the shader side we need to define our constants as constant buffers.  
```hlsl
struct MyDataInShader{
    float myData1;
    int myData2;
}

ConstantBuffer<MyDataInShader> myDrawConstants : register(b1, space0);
```

What really connects the data from CPU side to the shader is the register name and register space.  

> Note: Shader’s constant buffers are seen as sets of 4x32-bit values by the system.  
> Each set of shader’s constant will be seen as a scalar array of 32-bit values, dynamically indexable and read only.  
  

## Root Descriptors

Root Descriptors are descriptors inlined in the root arguments. They are made to contain descriptors that are accessed most often.  
They can only contain Constant Buffer Views(CBV), and raw or unstructured Shader Resource Views(SRV) or Unordered Access Views(UAVs).  
Supported SRV/UAV formats are only 32-bit FLOAT/UINT/SINT.  
SRVs of ray tracing acceleration structures are also supported.

Root descriptors have 1 level of indirection, meaning that in order to access them we are going to need to query one extra object. This makes them slower to access than root constants.

Each root descriptor costs 2 DWORDs out of 64 maximum available in the root signature, because 2 DWORDs = 64-bit is the size of a **GPU virtual address**.

> Note: Root descriptors do not operate size checks (as opposed to descriptors in descriptor heaps).

A root descriptor inside the root signature object is described with a [D3D12_ROOT_DESCRIPTOR](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_root_descriptor) object and the root signature’s slot type needs to be either `D3D12_ROOT_PARAMETER_TYPE_CBV`, `D3D12_ROOT_PARAMETER_TYPE_SRV` or `D3D12_ROOT_PARAMETER_TYPE_UAV`.  
Alternatively, we make use of [D3DX12 Header Library](https://github.com/microsoft/DirectX-Graphics-Samples/blob/master/Libraries/D3DX12/d3dx12.h) and instead use `InitAsConstantBufferView(..)`, `InitAsShaderResourceView(..)` or `InitAsUnorderedAccessView(..)` from [CD3DX12_ROOT_PARAMETER1](https://docs.microsoft.com/en-us/windows/win32/direct3d12/cd3dx12-root-parameter1) :
```cpp
CD3DX12_ROOT_PARAMETER1 rootParameters[1];

rootParameters[0].InitAsConstantBufferView( … );
// OR
rootParameters[0].InitAsShaderResourceView( … );  
// OR
rootParameters[0].InitAsUnorderedAccessView( … );
```
to speed up the insertion process.

On the shader side, for a constant buffer view, we can have a dynamic indexed field (compared to root constants where this was not possible)
```glsl
struct MyShaderDataType{  
    uint MyParam1;
    float MyParamArray[3];
    int MyParam3;  
}

ConstantBuffer<MyShaderDataType> myConstantBuffer : register(b6);
```

We are still limited to have the constant buffer “myConstantBuffer” to be a single variable and not an array, because that would still map to a root descriptor, and indexing is not supported in the root signature.  
  

## Root Tables

Root Tables are pointers to a range of descriptors in the descriptor heap. With it we can reference CBVs, SRVs, UAVs and Samplers.

A descriptor table is not an allocation of memory: it just stores a set of **offset and length** references to a single descriptor heap.  
A **descriptor heap** is an array of resource descriptors, which each points to a resource stored in a resource heap.

![](/assets/img/posts/2020-06-26-DX12RootSignatureObject/DescriptorTable_Scheme.jpg){:.postImg}

  

> Note: Sampler Descriptors always go in a standalone descriptor heap from the others.  
  

Each descriptor table occupies 1 DWORD (32 bits/4 bytes) out of the 64 maximum available in the root signature.  
Root tables element occupy less space than the root descriptors but have 2 levels of indirection. This makes them slower to access them compared to root descriptors and even more slow compared to the root constants case.

A descriptor table is made out of **Descriptor Ranges** belonging to the same descriptor heap, so that if we have to declare 20 descriptors of the same type, we do not have to individually specify them.  
Types of stores ranges can be one of [D3D12_DESCRIPTOR_RANGE_TYPE](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_descriptor_range_type) which are basically either CBV, UAV, SRV or texture sampler (this last to be placed in standalone tables that point to a sampler descriptor heap).

Each range is defined by a [D3D12_DESCRIPTOR_RANGE](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_descriptor_range) .

> Note: Inside shader code we can dynamically index among a single descriptor range. Be aware though that out of bounds indexing produces undefined behavior and in such situations the device might reset.

We mainly have two ways of using descriptor tables:

-   _Dynamic Indexing_: Relying on a static descriptor table that refers to a static and big descriptor heap, where we have wide descriptor ranges exposed to shaders to be dynamically indexed and pick the resources we need in a certain situation.
    
-   _Table Versioning_: Create or reuse different descriptor tables depending on the pass we are executing and the relative descriptors we need. We must still ensure to not edit descriptor heap portions when “in-flight” rendering commands are using them.
    

  
In a single range, we can have multiple descriptors taking the same table offset: they are called **Aliased Descriptors**. All descriptors with the same table offset will result in the same one, so in shaders using one or the other will result in the same operation.  
To make use of aliased descriptor we need to compile the shaders with `D3D10_SHADER_RESOURCES_MAY_ALIAS` flag or the `/res_may_alias` option in FXC.  
  
Once we define all the ranges we need, we place them in an array to give as input to a [D3D12_ROOT_DESCRIPTOR_TABLE](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_root_descriptor_table) that will represent the table in the root signature object.

> Note: Samplers require to be in a standalone table from CBVs, UAVs, SRVs.  

A descriptor table is set in the command list with

-   [CmdList->SetGraphicsRootDescriptorTable](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-setgraphicsrootdescriptortable) for a graphics pipeline
    
-   [CmdList->SetComputeRootDescriptorTable](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-setcomputerootdescriptortable) for a compute pipeline
    
# Root Arguments

Root arguments is the pool of memory that contains the values of all the root parameters from a root signature.

We can change the root arguments from the command list with [SetGraphicsRoot32BitConstants](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-setgraphicsroot32bitconstants) , [SetGraphicsRootConstantBufferView](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-setgraphicsrootconstantbufferview), [SetGraphicsRootShaderResourceView](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-setgraphicsrootshaderresourceview), [SetGraphicsRootUnorderedAccessView](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-setgraphicsrootunorderedaccessview), [SetGraphicsRootDescriptorTable](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-setgraphicsrootdescriptortable)  
e.g.
```cpp
cmdList->SetGraphicsRoot32BitConstants(0, sizeof(Eigen::Matrix4f) / 4, mvpMatrix.data(), 0);
```
  
and of course the compute counterparts are available as well.  
The full list can be found in the [ID3D12GraphicsCommandList](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nn-d3d12-id3d12graphicscommandlist) interface.

# Static Samplers

Static samplers are samplers where the state is fully defined and immutable.  
They are still part of the root signature, but the difference with dynamic samplers is that they cannot be dynamically assigned and indexed.  
Static samplers are not stored in a descriptor heap (as opposed to the dynamic ones) and there is no additional cost on using them, so if a sampler is meant to be immutable, it should be defined as static.  
They are defined with a [D3D12_STATIC_SAMPLER_DESC](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_static_sampler_desc) object and they do not count toward the maximum size of the root signature.

# Using a Root Signature

A root signature can be used either with bundles or with command lists. This article will not describe bundles and it will focus on command list usage.

When creating the PSO we assign the reference to the (previously) created root signature.  
Assuming we are using the [D3DX12 Header Library](https://github.com/microsoft/DirectX-Graphics-Samples/blob/master/Libraries/D3DX12/d3dx12.h) we have: 
```cpp 
struct PipelineStateStreamType {  
…  
} pipelineStateStream;

pipelineStateStream.pRootSignature = MyRootSignatureComPtr.Get();
```
Then we will need to set the root signature when start working with a command list.
When start working with a command list, the root signature is undefined, we need to explicitly set it with [SetGraphicsRootSignature](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-setgraphicsrootsignature)
```cpp
cmdList->SetGraphicsRootSignature(MyRootSignatureComPtr.Get());
```

If we set a root signature from a command list, all previous root signature bindings become outdated and the application expects all new bindings before the next draw/dispatch command.

  

> Note: we assign the root signature object in the PSO because it is meant to be serialized and store with it. The advantage of doing that is that when the PSO will be used again, we can compare its root signature with other ones at runtime, and quickly determine if the loaded PSO is compatible with them.

Just assigning the root signature in the PSO does not set it in the pipeline! It is compulsory to set it with an API call (e.g. from the command list).

> Note: The root signature set on the command list must match the one set in the PSO when the draw/dispatch command is executed, otherwise the behavior is undefined!

Shaders do not have to know about the root signature’s configuration, while they need to match the root layout specified in the PSO.

As mentioned before, is also a way to specify the root signature directly in shader code: when the shader will be serialized, the root signature will be serialized with it.

# Best Practices

-   Focus on make the root signature as small as possible, trading off root tables space for root constants or vice versa.
    
-   Consider trading off CBVs root descriptors or table entry bindings, that are accessed frequently, for the relative root constant, because they will be much faster to be accessed from.
    
-   Use SRV and UAV root descriptors, so directly in the root signature instead of in a table range, Only in the case where you have a lot of draw/dispatch commands using them.
    
-   Try to use the same root signature for many different Pipeline State Objects(PSO) that share the same (or part of) resource bindings.
    
-   Minimize the number of changes to a specific root signature in use because of the re-initialization costs.
    
-   Cache values addressed in the root signature in CPU and change the value of a root signature parameter only when a true change is detected.
    
-   Limit shader visibility to resources to only the needed pipeline stages.
    
-   Set descriptors and relative data on “static while set at execute” as much as possible, which is also the default behavior in version 1.1, so changing them before the relative command list starts execution.
    
-   Group CBV descriptors in tables when the descriptors are all supposed to change at the same time.
    
-   Moderate the use of big root tables just for the intention of re-using them, keep the number of entries for it and for the root signature in general as low as possible.
    
-   Remember to define all the resource bindings after a change in the root signature, because all the previous ones will be cleared.
    

# References

-   Official D3D12 Documentation  
    [https://docs.microsoft.com/en-us/windows/win32/direct3d12/root-signatures](https://docs.microsoft.com/en-us/windows/win32/direct3d12/root-signatures)
    
-   3DGEP Learding DX12 - Part2  
    [https://www.3dgep.com/learning-directx-12-2/](https://www.3dgep.com/learning-directx-12-2/)
    
-   NVIDIA’s DX12 Do’s and Don’ts  
    [https://developer.nvidia.com/dx12-dos-and-donts](https://developer.nvidia.com/dx12-dos-and-donts)