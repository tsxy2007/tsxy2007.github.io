---
title: DirectX12-第三节
date: 2019-04-23 09:00:55
tags:
category:
---
## 命令队列
命令有6种:
1.ID3D12CommandQueue::ExecuteCommandLists:
2.ID3D12CommandQueue::Signal:
3.ID3D12CommandQueue::Wait:
4.ID3D12CommandQueue::UpdateTileMapping:
5.ID3D12CommandQueue::CopyTileMapping:
6.ISwapChain::Present:
命令队列作用：将并发串行化。
硬件层面上GPU有3个并行执行的引擎：复制引擎，计算引擎和图形引擎；
（1） 复制命令队列中的命令只能在复制引擎中执行；
（2） 计算命令队列中的命令只能可以在计算引擎或复制引擎上执行；
（3） 直接命令队列中的命令可以在任意引擎上执行；
只有计算或直接类型命令才允许调用ID3D12CommandQueue接口的UpdateTileMapping和CopyTileMapping方法；
## Fence
Fence可用于控制命令队列中不同命令之间的同步，但是命令队列中同一命令内部的并发性对fence而言是不可见的。
## 命令分配器和命令列表
命令列表：
1.记录状态：在此状态下允许向其添加添加命令；
2.关闭状态：在此状态下允许作为ID3D12CommandQueue::ExecuteCommandLists的实参；
不允许被CPU并发访问；
命令列表并不维护其在命令分配器中所分配的内存的相关信息；需要手动释放内存；方法是ID3D12CommandAllocator::Reset;
## 捆绑包
命令列表不被CPU线程和GPU线程之间并发访问，也不允许被CPU线程并发访问；但是允许被GPU线程并发访问；
## 资源屏障
D3D12_RESOURCE_BARRIER结构体描述：
***分三种类型***：
1. D3D12_RESOURCE_BARRIER_TYPE_TRANSTION 转换资源屏障
2. D3D12_RESOURCE_BARRIER_TYPE_ALIASING  别名资源屏障
3. D3D12_RESOURCE_BARRIER_TYPE_UAV       无序访问视图资源屏障

### 转换资源屏障：
命令队列有且仅有图形/计算类和复制类：
***图形/计算类***： 所有的直接命令队列和计算命令队列
***复制类***： 所有的复制命令队列
#### 子资源权限：
D3D12_RESOURCE_BARRIER中的Transition成员的StateBefore和StateAfter成员表明子资源对命令队列的权限用D3D12_RESOURCE_STATES表示。
（1） 子资源对复制类的权限只能是一下三种：
* D3D12_RESOURCE_STATE_COMMON 公共（可写）
* D3D12_RESOURCE_STATE_COPY_DEST 可作为复制宿（可写）
* D3D12_RESOURCE_STATE_COPY_SOURCE 可作为复制源（只读）
（2） 子资源对图形/计算类的权限可以是任意一种权限，除以上3种还有
* D3D12_RESOURCE_STATE_RESOLVE_DEST 可作为解析宿（可写）
* D3D12_RESOURCE_STATE_RESOLVE_SOURCE 可作为解析源（只读）
* D3D12_RESOURCE_STATE_STREAM_OUT 可作为流输出（可写）
* D3D12_RESOURCE_STATE_DEPTH_WRITE 可作为深度模板（可写）
* D3D12_RESOURCE_STATE_RENDER_TARGET 可作为渲染目标(可写)
* D3D12_RESOURCE_STATE_INDIRECT_ARGUMENT 可作为间接参数（只读）
* D3D12_RESOURCE_STATE_VERTEX_AND_CONSTANT_BUFFER 可作为顶点或常量缓冲（只读）
* D3D12_RESOURCE_STATE_INDEX_BUFFER 可作为索引缓冲（只读）
* D3D12_RESOURCE_STATE_STATE_NON_SHADER_RESOURCE 可作为非像素着色器资源（只读）
* D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOUCE 可作为像素着色器资源（只读）
* D3D12_RESOURCE_STATE_UNORDERED_ACCESS 可作为只读深度模板（只读）
**通用读写权限**：D3D12_RESOURCE_STATE_GENERIC_READ
