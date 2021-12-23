---
title: DirectX12-CPU与GPU间的同步
date: 2020-05-24 13:35:35
tags: DirectX12
---
## CPU与GPU同步
GPU和CPU并行工作的时候，自然会产生一系列的同步问题。
解决此问题其中的一种办法是： 强制CPU等待，直到GPU完成所有命令的处理，达到某个指定的Fence点为止。我们称之为：刷新命令队列，可以通过fence实现这一点。
创建fence：
~~~
HRESULT ID3D12Device::CreateFence(UINT64 InitialValue,D3D12_FENCE_FLAGS Flags,REFIID riid,void** pFence);
// 每个fence 维护一个UINT64类型的值，此为用来标记fence point的整数。
~~~

一下是用一个Fence刷新CommandQueue；
~~~
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
~~~
## CPU与GPU异步（渲染帧）
这种方案并不完美，因为这意味着在等待GPU处理命令的时候，CPU会处于空闲状态.
解决此问题的一种方案是：以CPU每帧都需要更新的资源作为基本元素，创建一个环形数组（帧资源）。这个环形数组一般由3个帧资源组成。该方案思路是：在处理第n帧的时候，CPU将周而复始地从帧资源数组中获取下一个可用的帧资源。趁着GPU还在处理此前帧之时，CPU将为第n帧更新资源，并构建和提交对应的命令列表。随后，CPU将会继续针对第n+1帧执行同样的工作流程，并不断重复下去。如果帧资源数组共有3个元素，则令CPU比GPU提前处理两帧，以确保GPU可以持续工作。
    这种方案也无法杜绝此类情况，对于我们有什么用处呢？用处就是它使我们可以持续向GPU提供数据。也就是说，当GPU在处理第N帧的命令时，CPU可以继续构建和提交绘制第n+1帧和第n+2帧所用的命令。这将令命令队列保持非空状态，从而使GPU总有任务去执行。