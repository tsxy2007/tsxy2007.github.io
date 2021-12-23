---
title: DirectX12-命令队列和命令列表
date: 2021-12-22 16:55:11
tags: DirectX12
---

GPU维护至少一个命令队列（CommandQueue）。借助Directx3D api，Cpu可以利用(CommandList)将命令提交到这个队列中。当一系列命令被提交到命令队列之时，它们并不会被GPU立即执行。

### 命令队列（CommandQUeue）
通过填写D3D12_COMMAND_QUEUE_DESC结构体描述队列，再调用ID3D12Device::CreateCommandQueue方法创建队列。代码如下
```
Microsoft::WRL::ComPtr<ID3D12CommandQueue> mCommandQueue;
D3D12_COMMAND_QUEUE_DESC queueDesc = {};
queueDesc.Type = D3D12_COMMAND_LIST_TYPE_DIRECT;
queueDesc.Flags = D3D12_COMMAND_QUEUE_FLAG_NONE;
md3dDevice->CreateCommandQueue(
    &queueDesc,
    IID_PPV_ARGS(&mCommandQueue)
);
// 利用 ID3D12CommandQueue::ExecuteCommandLists可以将命令列表里的命令添加到命令队列里：
void ID3D12CommandQueue::ExecuteCommandLists(
    UINT Count, // 第二个参数里命令列表数组中命令列表的数量;
    ID3D12CommandList* const* ppCommandLists // 待执行的命令列表数量，指向命令列表数组中第一个元素的指针。
    );
```
## 命令分配器（CommandAllocator）

是一种和命令列表有关的内存管理类接口。记录命令列表内的命令，实际是存储在与之关联的命令分配器（CommandAllocator）上。当执行ID3D12CommandQueue::ExecuteCommandLists执行命令列表的时候，命令队列就用引用分配器里的命令。
```
ID3D12Device::CreateCommandAllocator(
    D3D12_COMMAND_LIST_TYPE type,
    REFIID riid,
    void** ppCommandAllocator
);
```
1. type:指定与此命令分配器相关联的命令列表类型。
    a) D3D12_COMMAND_LIST_TYPE_DIRECT.存储的是一系列可供GPU直接执行的命令
    b) D3D12_COMMAND_LIST_TYPE_BUNDLE.将命令列表打包，构建命令列表时会产生一定的CPU开销，一种优化方法。

2. riid： 带创建ID3D12CommandAllocator接口的COMID。
3. ppCommandAllocator： 输出指向所建命令分配器的指针。
## 命令列表（CommandList）

ID3D12GraphicsCommandList接口封装了一系列图形渲染命令，继承ID3D12CommandList接口。
一下代码是向命令列表添加命令：
```
// 创建命令
ID3D12Device::CreateCommandList(
    UINT nodeMask, // 对于仅有一个GPU的系统而言，要将此值设为0，对于多GPU的系统而言值为所间命令列表相关联的物理GPU。
    D3D12_COMMAND_LIST_TYPE type, // 命令列表的类型
    ID3D12CommandAllocator* pCommandAllocator,// 所建命令列表相关联的命令分配器。它的类型必须与所创命令列表的类型匹配。
    ID3d12PipelineState* pInitialState,// 指定命令列表的渲染流水线初始状态。
    REFIID riid,// 带创建ID3D12CommandList接口的COMID。
    void **ppCommandList//输出指向所建命令列表的指针。
);
// 提交命令
mCommandList->RSSetViewports(1,&mScreenViewport);//设置视口
mCommandList->ClearRenderTargetView(mBackBufferView,Colors::LightSteelBlue,0,nullptr);//清除渲染目标视图
mCommandList->DrawIndexedInstanced(36,1,0,0,0);//发起绘制调用命令
mCommandList->Close();//结束记录命令 调用ID3D12CommandQueue::ExecuteCommandLists之前需要先调用close

// 重置命令
ID3D12GraphicsCommandList::Reset(
    ID3D12CommandAllocator* pAllocator,
    ID3D12PipelineState* pInitialState
);
```
    我们可以创建出多个关联于同一命令分配器的命令列表，但是不能同时用它们来记录命令。因此，当其中一个命令列表在记录命令时，必须关闭同一命令分配器的其他命令列表。
### 没有确定GPU执行完命令分配器中所有命令之前，千万不要重置命令分配器；

## 命令与多线程
Directx12是多线程。命令列表也是一种发挥Direct3D多线程优势的途径。我们可以创建N个线程，每个线程分别构建一个命令列表来绘制1/n的场景物体。以下是多线程环境使用命令列表注意的问题：
1. 命令列表并非自由线程对象。也就是说，多线程既不能同时共享相同的命令列表，也不能同时调用同一命令列表的方法。
2. 命令分配器也不是线程自由对象。也就是说，多线程既不能同时共享相同的命令分配器，也不能同时调用同一命令分配器的方法。
3. 命令队列是线程自由对象，所以多线程可以同时访问同一命令队列，也能够同时调用它的方法。特别是每个线程都能同时向命令队列提交他们自己所生成的命令列表。
4. 出于性能的原因，应用程序必须在初始化期间，指出用于并行记录命令的命令列表最大数量
