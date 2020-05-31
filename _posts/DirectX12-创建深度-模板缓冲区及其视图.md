---
title: DirectX12-创建深度/模板缓冲区及其视图
date: 2020-05-30 10:49:26
tags: DirectX12
---

## 深度/模板缓冲区
深度缓冲区其实就是一种2D纹理，存储着离观察者最近的可视对象的深度信息（如果使用了模板，还会附有模板信息）。纹理是一种GPU资源，因此我们要通过填写D3D12_RESOURCE_DESC结构体来描述纹理资源，再用ID3D12Device::CreateCommittedResource 方法来创建它。
~~~
typedef struct D3D12_RESOURCE_DESC
    {
    D3D12_RESOURCE_DIMENSION Dimension; // 资源的维度
    UINT64 Alignment; 
    UINT64 Width; // 以纹理为单位来表示的纹理宽度。对于缓冲区资源来说，此项是缓冲区占用的字节数
    UINT Height; // 以纹理为单位来表示的纹理高度
    UINT16 DepthOrArraySize; // 以文素为单位来表示的纹理深度，纹理数组的大小
    UINT16 MipLevels; // mipmap 层级的数量。
    DXGI_FORMAT Format; // DXGI_FORMAT枚举类型中的成员之一，用于指定纹素的格式。
    DXGI_SAMPLE_DESC SampleDesc; // 多重采样的质量级别以及对每个像素的采样次数，
    D3D12_TEXTURE_LAYOUT Layout; // 用于指定纹理的布局。
    D3D12_RESOURCE_FLAGS Flags; // 资源有关的杂项标记。指定为D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL.
    } 	D3D12_RESOURCE_DESC;


    typedef enum D3D12_RESOURCE_DIMENSION
    {
        D3D12_RESOURCE_DIMENSION_UNKNOWN	= 0,
        D3D12_RESOURCE_DIMENSION_BUFFER	= 1,
        D3D12_RESOURCE_DIMENSION_TEXTURE1D	= 2,
        D3D12_RESOURCE_DIMENSION_TEXTURE2D	= 3,
        D3D12_RESOURCE_DIMENSION_TEXTURE3D	= 4
    } 	D3D12_RESOURCE_DIMENSION;
~~~
    GPU资源都存于堆（heap）中，其本质是具有特定属性的GPU显存块。ID3D12Device::CreateCommittedResource方法将根据我们提供的属性创建一个资源与一个堆，并把该资源提交到这个堆中。
~~~
HRESULT STDMETHODCALLTYPE CreateCommittedResource( 
            _In_  const D3D12_HEAP_PROPERTIES *pHeapProperties, //（资源欲提交至的）堆所具有的属性。有一些属性是针对高级用法而设。
            D3D12_HEAP_FLAGS HeapFlags, // 与堆有关的额外选项标记。通常设置为D3D12_HEAP_FLAG_NONE.
            _In_  const D3D12_RESOURCE_DESC *pDesc, // 指向一个D3D12_RESOURCE_DESC实例的指针，用它描述待创建的资源。
            D3D12_RESOURCE_STATES InitialResourceState, // 不管何时，每个资源都会处于一种特定的使用状态。在资源创建时，需要用此参数来设置它的初始状态。对于深度/模板缓冲区来说，通常设置为D3D12_RESOURCE_USAGE_INITTAL,我们随后就会将它的状态转换为D3D12_RESOURCE_STATE_COMMON,再利用ResourceBarrier方法辅以D3D12_RESOURCE_STATE_DEPTH_WRITE状态，将其转换为可以绑定在渲染流水线上的深度/模板缓冲区。
            _In_opt_  const D3D12_CLEAR_VALUE *pOptimizedClearValue, // 指向一个D3D12_CLEAR_VALUE对象的指针。描述一个用于清除资源的优化值。
            REFIID riidResource,
            _COM_Outptr_opt_  void **ppvResource);


typedef struct D3D12_HEAP_PROPERTIES
    {
    D3D12_HEAP_TYPE Type;
    D3D12_CPU_PAGE_PROPERTY CPUPageProperty;
    D3D12_MEMORY_POOL MemoryPoolPreference;
    UINT CreationNodeMask;
    UINT VisibleNodeMask;
    } 	D3D12_HEAP_PROPERTIES;
~~~

实例代码如下：
~~~
// Create the depth/stencil buffer and view.
	D3D12_RESOURCE_DESC depthStencilDesc;
	depthStencilDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
	depthStencilDesc.Alignment = 0;
	depthStencilDesc.Width = mClientWidth;
	depthStencilDesc.Height = mClientHeight;
	depthStencilDesc.DepthOrArraySize = 1;
	depthStencilDesc.MipLevels = 1;

	// Correction 11/12/2016: SSAO chapter requires an SRV to the depth buffer to read from 
	// the depth buffer.  Therefore, because we need to create two views to the same resource:
	//   1. SRV format: DXGI_FORMAT_R24_UNORM_X8_TYPELESS
	//   2. DSV Format: DXGI_FORMAT_D24_UNORM_S8_UINT
	// we need to create the depth buffer resource with a typeless format.  
	depthStencilDesc.Format = DXGI_FORMAT_R24G8_TYPELESS;

	depthStencilDesc.SampleDesc.Count = m4xMsaaState ? 4 : 1;
	depthStencilDesc.SampleDesc.Quality = m4xMsaaState ? (m4xMsaaQuality - 1) : 0;
	depthStencilDesc.Layout = D3D12_TEXTURE_LAYOUT_UNKNOWN;
	depthStencilDesc.Flags = D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL;

    // 创建上传堆;
	D3D12_CLEAR_VALUE optClear;
	optClear.Format = mDepthStencilFormat;
	optClear.DepthStencil.Depth = 1.0f;
	optClear.DepthStencil.Stencil = 0;
	ThrowIfFailed( md3dDevice->CreateCommittedResource(
		&CD3DX12_HEAP_PROPERTIES( D3D12_HEAP_TYPE_DEFAULT ),
		D3D12_HEAP_FLAG_NONE,
		&depthStencilDesc,
		D3D12_RESOURCE_STATE_COMMON,
		&optClear,
		IID_PPV_ARGS( mDepthStencilBuffer.GetAddressOf( ) ) ) );

    // 创建资源描述符
	// Create descriptor to mip level 0 of entire resource using the format of the resource.
	D3D12_DEPTH_STENCIL_VIEW_DESC dsvDesc;
	dsvDesc.Flags = D3D12_DSV_FLAG_NONE;
	dsvDesc.ViewDimension = D3D12_DSV_DIMENSION_TEXTURE2D;
	dsvDesc.Format = mDepthStencilFormat;
	dsvDesc.Texture2D.MipSlice = 0;
	md3dDevice->CreateDepthStencilView( mDepthStencilBuffer.Get( ), &dsvDesc, DepthStencilView( ) );

    // 转换;
	// Transition the resource from its initial state to be used as a depth buffer.
	mCommandList->ResourceBarrier( 1, &CD3DX12_RESOURCE_BARRIER::Transition( mDepthStencilBuffer.Get( ),
		D3D12_RESOURCE_STATE_COMMON, D3D12_RESOURCE_STATE_DEPTH_WRITE ) );
~~~