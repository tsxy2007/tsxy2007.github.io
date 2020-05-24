---
title: DirectX12-CPU与GPU间的交互
date: 2020-05-22 06:25:15
tags:
---

## 命令队列和命令列表：
GPU维护一个命令队列 command queue;
CPU利用命令列表 command list将命令提交到这个队列中

1. 创建命令队列
ID3D12Device::CreateCommandQueue;
ID3D12CommandQueue desc;
ID3D12CommandQueue::ExecuteCommandLists(UINT count,ID3D12CommandList* const* ppCommandLists);
2. ID3D12GraphicsCommandList 接口封装了一系列的图形渲染命令，继承于ID3D12CommandList接口。
~~~
mCommandList->ResetViewports(1,&mScreenViewport); // 重置视口
mCommandList->ClearRenderTargetView(mBackBufferView,Colors::LightSteelBlue,0,nullptr); //清除渲染目标视口
mCommandList->DrawIndexedInstanced(36,1,0,0);//发起绘制调用的命令；
mCommandList->Close();// 结束命令的记录 在调用ID3D12CommandQueue::ExecuteCommandLists提交命令列表之前，一定要将其关闭
~~~
3. ID3D12CommandAllocator: 内存管理类接口，记录在命令列表的命令，实际上是存储在与之关联的命令分配器。
~~~
ID3D12Device::CreateCommandAllocator(D3D12_COMMAND_LIST_TYPE Type,REFIID riid,void ** ppCommandAllocator);
~~~
Type: 指定与此命令分配器相关联的命令列表类型。

    a) D3D12_COMMAND_LIST_TYPE_DIRECT.存储的是一系列可供GPU直接执行的命令
    b) D3D12_COMMAND_LIST_TYPE_BUNDLE. 将命令打包（集合），构建命令列表时会产生一定的CPU开销，因此directx12 提供一种优化方法，允许我们将一系列的命令打成所谓的包。
riid： 待创建ID3D12CommandAllocator接口的COMID。
ppCommandAllocator:输出指向所创建命令分配器的指针。

4.命令列表同样由ID3D12Device接口创建
~~~
HRESULT ID3D12Device::CreateCommandList(UINT nodeMask,D3D12_COMMAND_LIST_TYPE type,ID3D12CommandAllocator* pCommandAllocator,ID3D12PipelineState* pInitialState,REFIID riid,void ** ppCommandList);
/**
1.nodeMask: 仅有一个GPU的系统，要将此值设置为0，对于多GPU系统而言，此节点掩码指定的是所创建命令列表相关联的物理GPU。
2.type：命令列表的类型，常用的选项为D3D12_COMMAND_LIST_TYPE_DIRECT和D3D12_COMMAND_LIST_TYPE_BUNDLE.
3.pCommandAllocator: 与所建命令列表相关联的命令分配器。
4.pInitialState：指定命令列表的渲染流水线初始状态。
5.riid： 略；
6.ppCommandList： 略；
*/
~~~
可以创建多个关联于同一命令分配器的命令列表，但不能同时使用他们来记录命令。
在调用ID3D12CommandQueue::ExecuteCommandList方法之后，我们可以通过ID3D12GraphicsCommandList::Reset方法，安全的复用命令列表C占用的相关底层内存来记录新的命令集。

~~~
HRESULT ID3D12GraphicsCommandList::Reset(ID3D12CommandAllocator* pAllocator,ID3D12PipelineState* pInitialState);
~~~

整体代码如下：
~~~
// 1.创建命令队列
    D3D12_COMMAND_QUEUE_DESC queueDesc = {};
    queueDesc.Type = D3D12_COMMAND_LIST_TYPE_DIRECT;
    queueDesc.Flags = D3D12_COMMAND_QUEUE_FLAG_NONE;
    ThrowIfFailed(md3dDevice->CreateCommandQueue(&queueDesc, IID_PPV_ARGS(&mCommandQueue)));
//2.创建命令内存管理接口：
    ThrowIfFailed(md3dDevice->CreateCommandAllocator(
    D3D12_COMMAND_LIST_TYPE_DIRECT,
    IID_PPV_ARGS(mDirectCmdListAlloc.GetAddressOf())));
//3. 创建命令列表：
    ThrowIfFailed(md3dDevice->CreateCommandList(
	    	0,
		    D3D12_COMMAND_LIST_TYPE_DIRECT,
		    mDirectCmdListAlloc.Get(), // Associated command allocator
		    nullptr,                   // Initial   PipelineStateObject
		    IID_PPV_ARGS(mCommandList.GetAddressOf())));
//4.设置此命令列表状态为关闭
    mCommandList->Close();
~~~