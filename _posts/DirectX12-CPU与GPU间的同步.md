---
title: DirectX12-CPU与GPU间的同步
date: 2020-05-24 13:35:35
tags:
---

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

这种方案并不完美，因为这意味着在等待GPU处理命令的时候，CPU会处于空闲状态，第七章解决此问题；