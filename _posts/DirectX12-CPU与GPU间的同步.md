---
title: DirectX12-CPU与GPU间的同步
date: 2020-05-24 13:35:35
tags: DirectX12
---
# CPU与GPU同步
## 刷新命令队列
GPU和CPU并行工作的时候，自然会产生一系列的同步问题。
解决此问题其中的一种办法是： 强制CPU等待，直到GPU完成所有命令的处理，达到某个指定的Fence点为止。我们称之为：刷新命令队列，可以通过fence实现这一点。

**原因**
1. 在GPU未结束命令分配器(commandallocator)中所有命令的执行之前，不能将它重置。
2. 在GPU未完成与常量缓冲区相关的绘制命令之前，CPU不可能更新这些常量缓冲区。所以我们在每帧绘制的结尾都会调用D3DApp::FlushCommandQueue函数，以确保GPU在每一帧都能正确完成所有命令的执行。

**创建fence**：
~~~
HRESULT ID3D12Device::CreateFence(UINT64 InitialValue,D3D12_FENCE_FLAGS Flags,REFIID riid,void** pFence);
// 每个fence 维护一个UINT64类型的值，此为用来标记fence point的整数。
~~~

一下是用一个Fence刷新CommandQueue；
~~~
UINT64 mCurrentFence = 0;
void D3DApplication::FlushCommandQueue()
{
 // 增加Fence值，接下来将命令标记到此Fence point
    mCurrentFence++; 
	
    // 向命令队列中添加一条用来设置新Fence point的命令
    // 由于这条命令交由GPU处理（即由GPU端来修改Fence的值），所以在GPU处理完CommandQueue此Signal();
    // 在所有命令完成之前，不会设置新的Fence point；
    ThrowIfFailed( mCommandQueue->Signal( mFence.Get( ), mCurrentFence ) );

// CPU端等待GPU，直到后者执行完这个Fence point之前所有的命令
	if (mFence->GetCompletedValue() < mCurrentFence)
	{
		HANDLE eventHandle = CreateEventEx( nullptr, false, false, EVENT_ALL_ACCESS );
        //若GPU命中当前的Fence point，激发预定事件
		ThrowIfFailed( mFence->SetEventOnCompletion( mCurrentFence, eventHandle ) );
// 等待GPU命中Fence 激发事件；
		WaitForSingleObject( eventHandle, INFINITE );
		CloseHandle( eventHandle );
	}
}
~~~
**效果**

这种解决方案虽然奏效却效率低下,原因如下：

    1. 在每帧的起始阶段,GPU不会执行任何命令，因为等待它处理的命令队列空空如也。这种情况将持续到CPU构建并提交一些GPU执行的命令为止。

    2. 在每帧的收尾阶段，CPU会等待GPU完成命令处理。

## CPU与GPU异步（渲染帧）
这种方案并不完美，因为这意味着在等待GPU处理命令的时候，CPU会处于空闲状态.
解决此问题的一种方案是：以CPU每帧都需要更新的资源作为基本元素，创建一个环形数组（帧资源）。这个环形数组一般由3个帧资源组成。该方案思路是：在处理第n帧的时候，CPU将周而复始地从帧资源数组中获取下一个可用的帧资源。趁着GPU还在处理此前帧之时，CPU将为第n帧更新资源，并构建和提交对应的命令列表。随后，CPU将会继续针对第n+1帧执行同样的工作流程，并不断重复下去。如果帧资源数组共有3个元素，则令CPU比GPU提前处理两帧，以确保GPU可以持续工作。
    这种方案也无法杜绝此类情况，对于我们有什么用处呢？用处就是它使我们可以持续向GPU提供数据。也就是说，当GPU在处理第N帧的命令时，CPU可以继续构建和提交绘制第n+1帧和第n+2帧所用的命令。这将令命令队列保持非空状态，从而使GPU总有任务去执行。

### 帧资源代码如下：
```
class FrameResource
{
public:
	FrameResource(ID3D12Device* device, UINT passCount, UINT objectCount);
	FrameResource(const FrameResource& rhs) = delete;
	FrameResource& operator=(const FrameResource& rhs) = delete;
	~FrameResource();
	
	//  在GPU处理完与此命令分配器相关的命令之前，我们不能对它进行重置。
    //  每帧都要有自己的命令分配器。
	Microsoft::WRL::ComPtr<ID3D12CommandAllocator> CmdListAlloc;
	//  在GPU执行完此常量缓冲区的命令之前，我们不能对他进行更新。
    //  因此每帧都要有自己的常量缓冲区。
	std::unique_ptr<UploadBuffer<PassConstants>> PassCB = nullptr;
	std::unique_ptr<UploadBuffer<ObjectConstants>> ObjectCB = nullptr;
	//通过围栏值将命令标记到此围栏点，这使我们可以检测到GPU是否还在使用这些帧资源。
	UINT64 Fence = 0;
};
```

### 初始化循环帧资源数组

```
void D3DApplication::BuildFrameResource()
{
	for (int i = 0 ;i<gNumFrameResource;i++)
	{
		mFrameResource.push_back(std::make_unique<FrameResource>(mD3DDevice.Get(), 1, (UINT)mAllRitems.size()));
	}
}
```

### CPU端处理第N帧的算法：

```
void D3DApplication::Update()
{
	UpdateCamera();
	// 循环往复的获取帧资源数组中的元素;
	mCurrentFrameResourceIndex = (mCurrentFrameResourceIndex + 1) % gNumFrameResource;
	mCurrentFrameResource = mFrameResource[mCurrentFrameResourceIndex].get();

	// GPU端是否已执行完处理当前帧资源的所有命令
	// 如果没有就令CPU等待,直到GPU完成命令的执行并抵达这个围栏点;
	if (mCurrentFrameResource->Fence != 0 ;mFence->GetCompletedValue()<mCurrentFrameResource->Fence)
	{
		HANDLE eventHandle = CreateEventEx(nullptr, L"false", false, EVENT_ALL_ACCESS);
		mFence->SetEventOnCompletion(mCurrentFrameResource->Fence, eventHandle);
		WaitForSingleObject(eventHandle, INFINITE);
		CloseHandle(eventHandle);
	}
	UpdateObjectCBs();
	UpdateMainPassCB();
}
```
draw:

```
void D3DApplication::Draw()
{
	auto mCurrentCommandAlloc = mCurrentFrameResource->CmdListAlloc;
	mCurrentCommandAlloc->Reset();
	mD3DCommandList->Reset(mCurrentCommandAlloc.Get(), mPSO.Get());


	mD3DCommandList->RSSetViewports(1, &mScreenViewport);
	mD3DCommandList->RSSetScissorRects(1, &mScissorRect);


	auto TmpCurrentBackBuffer = CurrentBackBuffer();
	auto TmpCurrentBackBufferView = CurrentBackBufferView();
	auto TmpDepthStencilView = DepthStencilView(); 
	mD3DCommandList->ResourceBarrier(1, get_rvalue_ptr(CD3DX12_RESOURCE_BARRIER::Transition(TmpCurrentBackBuffer,
		D3D12_RESOURCE_STATE_PRESENT, D3D12_RESOURCE_STATE_RENDER_TARGET)));

	mD3DCommandList->ClearRenderTargetView(TmpCurrentBackBufferView, Colors::LightSteelBlue, 0, nullptr);
	mD3DCommandList->ClearDepthStencilView(TmpDepthStencilView, D3D12_CLEAR_FLAG_DEPTH | D3D12_CLEAR_FLAG_STENCIL, 1.0f, 0, 0, nullptr);
	mD3DCommandList->OMSetRenderTargets(1, &TmpCurrentBackBufferView, true, &TmpDepthStencilView);

	// 绑定描述符堆
	ID3D12DescriptorHeap* descriptorHeaps[] = { mCbvHeap.Get() };
	mD3DCommandList->SetDescriptorHeaps(_countof(descriptorHeaps), descriptorHeaps);
	mD3DCommandList->SetGraphicsRootSignature(mRootSignature.Get());


	//绑定常量缓冲堆
	int passCbvIndex = mPassCbvOffset + mCurrentFrameResourceIndex;
	auto passCbvHandle = CD3DX12_GPU_DESCRIPTOR_HANDLE(mCbvHeap->GetGPUDescriptorHandleForHeapStart());
	passCbvHandle.Offset(passCbvIndex, mCbvSrvUavDescriptorSize);
	mD3DCommandList->SetGraphicsRootDescriptorTable(0, passCbvHandle);

	//绘制命令

	DrawRenderItems(mD3DCommandList.Get(), mOpaqueRitems);

	mD3DCommandList->ResourceBarrier(1, get_rvalue_ptr(CD3DX12_RESOURCE_BARRIER::Transition(TmpCurrentBackBuffer,
		D3D12_RESOURCE_STATE_RENDER_TARGET, D3D12_RESOURCE_STATE_PRESENT)));
	mD3DCommandList->Close();

	ID3D12CommandList* cmdLists[] = { mD3DCommandList.Get() };
	mD3DCommandQueue->ExecuteCommandLists(_countof(cmdLists), cmdLists);

	mSpwapChain->Present(0, 0);
	mCurrBackBuffer = (mCurrBackBuffer + 1) % SwapChainBufferCount;
	
	// 添加围栏值,将命令标记到此围栏点;
	mCurrentFrameResource->Fence = ++mCurrentFence;
	// 向命令队列添加一条指令来设置一个新的围栏点
	// 由于当前的GPU正在执行绘制命令,所以在GPU处理完Signal()函数之前的所有命令以前，并不会设置此新的围栏点。
	mD3DCommandQueue->Signal(mFence.Get(), mCurrentFence);
}
```
**本文章会持续更新**
    
<center> ![项目地址](https://github.com/tsxy2007/MyDirectx12) </center>