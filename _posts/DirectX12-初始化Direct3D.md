---
title: DirectX12-初始化Direct3D
date: 2020-05-25 23:22:03
tags: DirectX12
---

### 1. 用D3D12CreateDevice函数创建ID3D12Device接口实例
~~~

ComPtr<IDXGIFactory4> mD3DFactory;
void D3DApplication::InitDirect3D()
{
    //创建D3DFactory
    CreateDXGIFactory(IID_PPV_ARGS(&mD3DFactory)); 
    //创建Device
    D3D12CreateDevice(
        nullptr, //设备上下文，null默认主显示适配器
		D3D_FEATURE_LEVEL_11_0, //应用程序需要硬件所支持的最低功能级别。
		IID_PPV_ARGS(&mD3DDevice)//返回的Direct3D 12 设备
    );
}
~~~
### 2. 创建一个ID3D12Fence对象，并查询描述符的大小。
~~~
//创建Fence
    mD3DDevice->CreateFence(
        0,// Fence的初始值
        D3D12_FENCE_FLAG_NONE, //D3D12-FENCE标志类型值的组合，通过使用按位或操作组合而成。结果值指定fence的选项。
        IID_PPV_ARGA(&mFence)//返回的Fence
    );
	// 计算描述符大小
	mRtvDescriptorSize = mD3DDevice->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_RTV);
	mDsvDescriptroSize = mD3DDevice->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_DSV);
	mCbvSrvUavDescriptorSize = mD3DDevice->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV);
~~~
### 3. 检测用户设备对4x MSAA质量级别的支持情况
~~~
   //检测对MSAA 抗锯齿支持情况
    DXGI_FORMAT mBackBufferFormat = DXGI_FORMAT_R8G8B8A8_UNORM;
	D3D12_FEATURE_DATA_MULTISAMPLE_QUALITY_LEVELS mQualityLevels;
	mQualityLevels.Format = mBackBufferFormat;
	mQualityLevels.SampleCount = 4;
	mQualityLevels.Flags = D3D12_MULTISAMPLE_QUALITY_LEVELS_FLAG_NONE;
	mQualityLevels.NumQualityLevels = 0;

	mD3DDevice->CheckFeatureSupport(
		D3D12_FEATURE_MULTISAMPLE_QUALITY_LEVELS,
		&mQualityLevels,
		sizeof(mQualityLevels));
~~~
### 4. 依次创建命令队列，命令列表分配器和主命令列表。
~~~
void D3DApplication::CreateCommandObjects()
{
	// 创建命令队列
	D3D12_COMMAND_QUEUE_DESC queueDesc = {};
	queueDesc.Type = D3D12_COMMAND_LIST_TYPE_DIRECT;
	queueDesc.Flags = D3D12_COMMAND_QUEUE_FLAG_NONE;
	mD3DDevice->CreateCommandQueue(&queueDesc, IID_PPV_ARGS(&mD3DCommandQueue));

	//创建命令分配器
	
	mD3DDevice->CreateCommandAllocator(
		D3D12_COMMAND_LIST_TYPE_DIRECT,
		IID_PPV_ARGS(&mD3DCommandAllocator)
	);

	//创建命令列表
	mD3DDevice->CreateCommandList(
		0,
		D3D12_COMMAND_LIST_TYPE_DIRECT,
		mD3DCommandAllocator.Get(),
		nullptr,
		IID_PPV_ARGS(mD3DCommandList.GetAddressOf())
	);

	mD3DCommandList->Close();
}
~~~
### 5. 描述并创建交换链；
~~~
   void D3DApplication::CreateSwapChain()
   {
	mSpwapChain.Reset();
	DXGI_SWAP_CHAIN_DESC sd;
	// bufferDesc 这个结构体描述了带创建后台缓冲区的属性。
	sd.BufferDesc.Width = mClientWidth; // 缓冲区分辨率的宽度
	sd.BufferDesc.Height = mClientHeight; //缓冲区分辨率的高度
	sd.BufferDesc.RefreshRate.Numerator = 60; // 刷新频率
	sd.BufferDesc.RefreshRate.Denominator = 1;
	sd.BufferDesc.Format = mBackBufferFormat; // 缓冲区的显示格式
	sd.BufferDesc.ScanlineOrdering = DXGI_MODE_SCANLINE_ORDER_UNSPECIFIED; // 逐行扫描 vs 隔行扫描
	sd.BufferDesc.Scaling = DXGI_MODE_SCALING_UNSPECIFIED; // 图像如何相对于屏幕进行拉伸
	// SampleDesc 多重采样的质量级别以及对每个像素的采样次数。
	sd.SampleDesc.Count = b4xMassState ? 4 : 1; 
	sd.SampleDesc.Quality = b4xMassState ? (m4xMsaaQulity - 1) : 0;
	sd.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT; // 由于我们要将数据渲染至后台缓冲区，因此指定为DXGI_USAGE_RENDER_TARGET_OUTPUT
	sd.BufferCount = SwapChainBufferCount;//交换链中所用缓冲区数量。
	sd.OutputWindow = mHWND;// 渲染窗口的句柄
	sd.Windowed = true;//指定为true，程序将在窗口模式下运行，指定为false 则采用全屏。
	sd.SwapEffect = DXGI_SWAP_EFFECT_FLIP_DISCARD;
	sd.Flags = DXGI_SWAP_CHAIN_FLAG_ALLOW_MODE_SWITCH;//可选标记。如果指定为DXGI_SWAP_CHAIN_FLAG_ALLOW_MODE_SWITCH，那么当程序切换为全屏的模式时，将采用当前桌面的显示模式

	mD3DFactory->CreateSwapChain(
		mD3DCommandQueue.Get(), // 指向ID3D12CommandQueue接口的指针。
		&sd, // 指向描述符交换链的结构体指针。
		mSpwapChain.GetAddressOf());//返回所创建的交换链接口。
}
~~~
### 6. 创建应用程序所需要的描述符；
~~~
static const int SwapChainBufferCount = 2;
int mCurrBackBuffer = 0 ; //记录当前后台缓冲区索引.
   void D3DApplication::CreateRtvAndDsvDescriptorHeaps()
{
    // 为每个RenderTargetView（总共三个）设置描述符
	D3D12_DESCRIPTOR_HEAP_DESC rtvHeapDesc;
	rtvHeapDesc.NumDescriptors = SwapChainBufferCount;
	rtvHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_RTV;
	rtvHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;
	rtvHeapDesc.NodeMask = 0;
	mD3DDevice->CreateDescriptorHeap(
		&rtvHeapDesc,
		IID_PPV_ARGS(mRtvHeap.GetAddressOf())
	);
    //为深度模板缓冲区设置描述符
	D3D12_DESCRIPTOR_HEAP_DESC dsvHeapDesc;
	dsvHeapDesc.NumDescriptors = 1;
	dsvHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_DSV;
	dsvHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;
	dsvHeapDesc.NodeMask = 0;
	mD3DDevice->CreateDescriptorHeap(
		&dsvHeapDesc,
		IID_PPV_ARGS(mDsvHeap.GetAddressOf())
	);
}
// 访问其中所存的描述符:
D3D12_CPU_DESCRIPTOR_HANDLE D3DApplication::CurrentBackBufferView()const
{
    return CD3DX12_CPU_DESCRIPTOR_HANDLE(
		mRtvHeap->GetCPUDescriptorHandleForHeapStart(), //获取描述符堆中第一个描述符的句柄
		mCurrBackBuffer,//偏移至后台缓冲区描述符句柄的索引
		mRtvDescriptorSize //描述符所占字节的大小。
	);
}
//获取深度模板缓冲区描述符
D3D12_CPU_DESCRIPTOR_HANDLE D3DApplication::DepthStencilView() const
{
	return mDsvHeap->GetCPUDescriptorHandleForHeapStart();
}

~~~
### 7. 调整后台缓冲区的大小，并为它创建渲染目标视图；
~~~
CD3DX12_CPU_DESCRIPTOR_HANDLE rtvHeapHandle(mRtvHeap->GetCPUDescriptorHandleForHeapStart());
	for (UINT i = 0; i < SwapChainBufferCount; i++)
	{
		mSpwapChain->GetBuffer(i,IID_PPV_ARGS(&mSwapChainBuffer[i]));  //根据索引获取交换链中的缓冲区资源； 
		mD3DDevice->CreateRenderTargetView(
            mSwapChainBuffer[i].Get(), //指定用作渲染目标的资源
            nullptr,//指向D3D12_RENDER_TARGET_VIEW_DESC数据结构实例的指针，该结构描述了资源中元素的数据类型（格式）。如果创建时指定了具体格式，可以设置为nullptr
            rtvHeapHandle//引用所创建渲染目标视图的描述符句柄。
            );
            //偏移到描述符堆中的下一个缓冲区
		rtvHeapHandle.Offset(1, mRtvDescriptorSize);
	}
~~~
### 8. 创建深度/模板缓冲区及与之关联的深度/模板视图
深度模板缓冲区其实是一种2D纹理，存储着离观察者最近的可视对象的深度信息。纹理是一种GPU资源，因此我们可以同过D3D12_RESOURCE_DESC结构体来描述纹理资源，再用ID3D12Device::CreateCommittedResource方法来创建它。
~~~
typedef struct D3D12_RESOURCE_DESC
{
    D3D12_RESOURCE_DIMENSION Dimension;
    UINT64 Alignment;
    UINT64 Width;
    UINT Height;
    UINT16 DepthOrArraySize;
    UINT16 MipLevels;
    DXGI_FORMAT Format;
    DXGI_SAMPLE_DESC SampleDesc;
    D3D12_TEXTURE_LAYOUT Layout;
    D3D12_RESOURCE_MTS_FLAG Misc Flags;
}
~~~
1. **Dimension**: 资源的维度,即以下枚举类型中的成员之一；
enum D3D12_RESOURCE_DIMENSION
{
    D3D12_RESOURCE_DIMENSION_UNKNOWN = 0 ;
    D3D12_RESOURCE_DIMENSION_BUFFER = 1;
    D3D12_RESOURCE_DIMENSION_TEXTURE1D = 2;
    D3D12_RESOURCE_DIMENSION_TEXTURE2D = 3;
    D3D12_RESOURCE_DIMENSION_TEXTURE3D = 4
} D3D12_RESOURCE_DIMENSION
2. **Width** : 以纹素为单位来表示纹理的宽度。对于缓冲区资源来说，此项时缓冲区占用的字节数。
3. **Height**: 以纹素为单位来表示纹理的高度。
4. **DepthOrArraySize**: 以纹素为单位来表示纹理的深度，或者（对于1D和2D纹理来说）是纹理数组的大小。
5. **MipLevels**: mipmap层级的数量。
6. **Format**： 用于指定纹素的格式。
7. **SampleDesc**: 多重采样的质量级别以及每个像素的采样次数。
8. **Layout**: 用于指定纹理布局。
9. **Flags**: 与资源相关的杂项标记。对于深度/模板缓冲区资源来说，要将此项指定为D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL 
~~~
// 描述一个用于清除资源的优化值。
struct D3D12_CLEAR_VALUE
{
    DXGI_FORMAT Format,
    union
    {
        FLOAT Color[4];
        D3D12_DEPTH_STENCIL_VALUE DepthStencil;
    };
}
~~~
~~~
ID3D12Device::CreateCommittedResource( 
            const D3D12_HEAP_PROPERTIES *pHeapProperties,
            D3D12_HEAP_FLAGS HeapFlags,
            const D3D12_RESOURCE_DESC *pDesc,
            D3D12_RESOURCE_STATES InitialResourceState,
            const D3D12_CLEAR_VALUE *pOptimizedClearValue,
            REFIID riidResource,
            void **ppvResource)
~~~
1. **pHeapProperties**: 堆所具有的属性。有一些属性是针对高级用法而设。目前只需关心D3D12_HEAP_PROPERTIES中的D3D12_HEAP_TYPE枚举类型这一主要属性，其中的成员列举如下：
   a) D3D12_HEAP_TYPE_DEFAULT: 默认堆。向这堆里提交的资源，唯独GPU可以访问。
   b) D3D12_HEAP_TYPE_UPLOAD： 上传堆。向此堆提交的都是需要经CPU上传至GPU的资源
   c) D3D12_HEAP_TYPE_READBACK: 回读堆。向这种堆提交的都是需要由CPU读取的资源。
   d) D3D12_HEAP_TYPE_CUSTOM: 此成员应用于高级场景。
2. **HeapFlags** : 与（资源欲提交至的）堆有关的额外选项标志。通常设置为D3D12_HEAP_FLAG_NONE.
3. **pDesc**: 指向一个D3D12_RESOURCE_DESC实例指针。用它描述待建资源
4. **InitialResourceState**: 不管何时，每个资源都会处于一种特定的使用状态。在资源创建时，需要用此参数来设置它的初始状态。对于深度/模板缓冲区来说，通常将其初始状态设置为D3D12_RESOURCE_USAGE_INITTAL,我们随后将它的状态转换为D3D12_RESOURCE_STATE_COMMON，再利用
5. **pOptimizedClearValue**: mipmap层级的数量。
6. **ppvResource**： 用于指定纹素的格式。
~~~
// 创建深度/模板缓冲区描述符
	D3D12_RESOURCE_DESC depthStencilDesc;
	depthStencilDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D; //资源的维度
	depthStencilDesc.Alignment = 0;
	depthStencilDesc.Width = mClientWidth;
	depthStencilDesc.Height = mClientHeight;
	depthStencilDesc.DepthOrArraySize = 1;
	depthStencilDesc.MipLevels = 1;
	depthStencilDesc.Format = mDepthStencilFormat;
	depthStencilDesc.SampleDesc.Count = b4xMassState ? 4 : 1;
	depthStencilDesc.SampleDesc.Quality = b4xMassState ? (m4xMsaaQulity - 1) : 0;
	depthStencilDesc.Layout = D3D12_TEXTURE_LAYOUT_UNKNOWN;
	depthStencilDesc.Flags = D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL;

	D3D12_CLEAR_VALUE optClear;
	optClear.Format = mDepthStencilFormat;
	optClear.DepthStencil.Depth = 1.0f;
	optClear.DepthStencil.Stencil = 0;
    // 创建深度/模板资源堆
	CD3DX12_HEAP_PROPERTIES DSHeap = CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT);
	mD3DDevice->CreateCommittedResource(
		&DSHeap,
		D3D12_HEAP_FLAG_NONE,
		&depthStencilDesc,
		D3D12_RESOURCE_STATE_COMMON,
		&optClear,
		IID_PPV_ARGS(mDepthStencilBuffer.GetAddressOf()));

	// 利用此资源格式，为整个资源的第0mip层创建描述符
	mD3DDevice->CreateDepthStencilView(mDepthStencilBuffer.Get(), nullptr, DepthStencilView());

	// 将资源从初始状态转换为深度模板区
	CD3DX12_RESOURCE_BARRIER ResBarrier = CD3DX12_RESOURCE_BARRIER::Transition(mDepthStencilBuffer.Get(),
		D3D12_RESOURCE_STATE_COMMON, D3D12_RESOURCE_STATE_DEPTH_WRITE);
	mD3DCommandList->ResourceBarrier(1, &ResBarrier);

	// 提交命令
	mD3DCommandList->Close();
	ID3D12CommandList* cmdsLists[] = { mD3DCommandList.Get() };
	mD3DCommandQueue->ExecuteCommandLists(_countof(cmdsLists), cmdsLists);
~~~
### 9.  设置视口和裁剪矩形；
~~~
    D3D12_VIEWPORT mScreenViewport;
    mScreenViewport.TopLeftX = 0;
    mScreenViewport.TopLeftY = 0;
    mScreenViewport.Width = static_cast<float>(mClientWidth);
    mScreenViewport.Height = static_cast<float>(mClientHeight);
    mScreenViewport.MinDepth = 0.0f;
    mScreenViewport.MaxDepth = 1.0f;    
    D3D12_RECT mScissorRect = { 0, 0, mClientWidth, mClientHeight };
~~~
以上方法是我们在初始化的时候需要做的基本工作；以下方法是我们Draw方法：
~~~
    //重复使用记录命令相关内存
    //只有当与GPU关联的命令列表执行完成时，我们才能重置
    mD3DCommandAllocator->Reset();
    //再通过ExecuteCommandLists方法将某个命令列表加入命令队列后，我们便可以重置该命令列表。以此来服用命令列表及内存
	mD3DCommandList->Reset(mD3DCommandAllocator.Get(), nullptr);

    //对资源的状态进行转换，将资源从呈现状态转换为渲染目标状态
	auto TmpCurrentBackBuffer = CurrentBackBuffer();
	auto TmpCurrentBackBufferView = CurrentBackBufferView();
	auto TmpDepthStencilView = DepthStencilView(); 
	CD3DX12_RESOURCE_BARRIER barrier = CD3DX12_RESOURCE_BARRIER::Transition(TmpCurrentBackBuffer,
		D3D12_RESOURCE_STATE_PRESENT, D3D12_RESOURCE_STATE_RENDER_TARGET);
	mD3DCommandList->ResourceBarrier(1, &barrier);
	mD3DCommandList->RSSetViewports(1, &mScreenViewport); // 设置视口大小
	mD3DCommandList->RSSetScissorRects(1, &mScissorRect); // 设置剪裁矩形
	mD3DCommandList->ClearRenderTargetView(TmpCurrentBackBufferView, Colors::Red, 0, nullptr); // 清除后台缓冲区
	mD3DCommandList->ClearDepthStencilView(TmpDepthStencilView, D3D12_CLEAR_FLAG_DEPTH | D3D12_CLEAR_FLAG_STENCIL, 1.0f, 0, 0, nullptr); // 清除深度缓冲区
	mD3DCommandList->OMSetRenderTargets(1, &TmpCurrentBackBufferView, true, &TmpDepthStencilView); //指定要渲染的缓冲区
	CD3DX12_RESOURCE_BARRIER bakbarrier = CD3DX12_RESOURCE_BARRIER::Transition(TmpCurrentBackBuffer,
		D3D12_RESOURCE_STATE_RENDER_TARGET, D3D12_RESOURCE_STATE_PRESENT);
	mD3DCommandList->ResourceBarrier(1, &bakbarrier); // 再次堆资源状态进行转换，将资源从目标渲染状态转回呈现状态；
	mD3DCommandList->Close(); //完成命令记录

	ID3D12CommandList* cmdLists[] = { mD3DCommandList.Get() };
	mD3DCommandQueue->ExecuteCommandLists(_countof(cmdLists), cmdLists);//将待执行的命令列表加入命令队列

	mSpwapChain->Present(0, 0); // 交换后台缓冲区和前台缓冲区
	mCurrBackBuffer = (mCurrBackBuffer + 1) % SwapChainBufferCount; // 交换后台缓冲区和前台缓冲区
	FlushCommandQueue();//等待此帧的命令执行完成。当前的实现确实没有什么效率，过于简单。
~~~

    以上是基本程序流程，我们还没有绘制出三角形，但是也快了。[项目地址](https://github.com/tsxy2007/MyDirectx12)。