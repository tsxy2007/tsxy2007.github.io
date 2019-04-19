---
title: DirectX12 第一节
date: 2019-04-13 00:02:00
tags: DirectX12
category:
---
## 一：创建Device

### 1.创建Factory
```
UINT dxgiFactoryFlags = 0;
ComPtr<IDXGIFactory6> factory;
CreateDXGIFactory2(dxgiFactoryFlags,IID_PPV_ARGS(&factory));
```
### 2.创建上下文 Adapter
 
 ```
 factory->EnumWarpAdapter(IID_PPV_ARGS(&warpAdapter));
 ```
 ### 3.创建Device
 ```
 D3D12CreateDevice(warpAdapter.Get(),D3D_FEATURE_LEVEL_11_0,IID_PPV_ARGS(&m_device)); 
 // DirectX 12 需要设置为 D3D_FEATURE_LEVEL_11_0 或者以上
 ```

 ## 二： 创建命令队列
```
 D3D12_COMMAND_QUEUE_DESC queueDesc = {};
 queueDesc.Flags = D3D12_COMMAND_QUEUE_FLAG_NONE;
 queueDesc.Type = D3D12_COMMAND_LIST_TYPE_DIRECT;
 m_device->CreateCommandQueue(&queueDesc,IID_PPV_ARGS(&m_commandQueue));
```
### Type 类型：
D3D12_COMMAND_LIST_TYPE_DIRECT（直接命令队列）;
D3D12_COMMAND_LIST_TYPE_BUNDLE（捆绑包）;
D3D12_COMMAND_LIST_TYPE_COMPUTE（计算命令队列）;
D3D12_COMMAND_LIST_TYPE_COPY（复制命令队列）;
D3D12_COMMAND_LIST_TYPE_VIDEO_DECODE（视频解码命令队列）; 
D3D12_COMMAND_LIST_TYPE_VIDEO_PROCESS（视频处理命令队列）;

## 三：创建交换缓存区
```
DXGI_SWAP_CHAIN_DESC1 swapChainDesc = {};
swapChainDesc.BufferCount = 3;
swapChainDesc.Width = 1280;
swapChainDesc.Height = 800;
swapChainDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
swapChainDesc.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;
swapChainDesc.SwapEffect = DXGI_SWAP_EFFECT_FLIP_DISCARD;
swapChainDesc.SampleDesc.Count = 1;

ComPtr<IDXGISwapChain> swapChain;
factory->CreateSwapChainForHwnd(
    m_commandQueue.Get(),
    Win32Application::GetHwnd();
    &swapChainDesc,
    nullptr,
    nullptr,
    &swapChain
);
```
## 四.创建描述符堆对象
```
D3D12_DESCRIPTOR_HEAP_DESC rtvHeapDesc = {};
rtvHeapDesc.NumDescriptors = 3;
rtvHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_RTV;
rtvHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_TYPE_RTV;
m_device->CreateDescriptorHeap(&rtvHeapDesc,IID_PPV_ARGS(&m_rtvHeap));
m_rtvDescriptorSize = m_device->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_RTV);
```
### Type
**D3D12_DESCRIPTOR_HEAP_TYPE : 指定堆中描述符类型的类型化值**。
  D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV （用于组合常量缓冲区、着色器资源和无序访问视图的描述符堆。）, 
  D3D12_DESCRIPTOR_HEAP_TYPE_SAMPLER（采样器的描述符堆。）,
  D3D12_DESCRIPTOR_HEAP_TYPE_RTV（目标视图的描述符堆。）,
  D3D12_DESCRIPTOR_HEAP_TYPE_DSV（深度模板视图的描述符堆。）,
  D3D12_DESCRIPTOR_HEAP_TYPE_NUM_TYPES（描述符堆的类型数。）
### Flags
**D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE**:可以在描述符堆上选择性地设置D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE标志，以指示它被绑定在命令列表上，以供着色器引用。没有此标志创建的描述符堆允许应用程序在将描述符复制到着色器可见描述符堆之前在CPU内存中阶段描述符，这是一种便利。但是，应用程序也可以直接将描述符创建为着色器可见描述符堆，而不需要在CPU上执行任何操作。此标志仅适用于CBV、SRV、UAV和采样器。它不适用于其他描述符堆类型，因为着色器不直接引用其他类型。
**D3D12_DESCRIPTOR_HEAP_FLAG_NONE**: 默认
## 五.创建RenderTargetView
```
CD3DX12_CPU_DESCRIPTOR_HANDLE rtvHandle(m_rtvHeap->GetCPUDescriptorHandleForHeapStart());
for (UINT i = 0; i < FrameCount; i++)
{
	ThrowIfFailed(m_swapChain->GetBuffer(i, IID_PPV_ARGS(&m_renderTargets[i])));
	m_device->CreateRenderTargetView(m_renderTargets[i].Get(), nullptr, rtvHandle);
	rtvHandle.Offset(1, m_rtvDescriptorSize);
}				
```
## 六.创建命令分配器
```
m_device->CreateCommandAllocator(D3D12_COMMAND_LIST_TYPE_DIRECT,IID_PPV_ARGS(&m_commandAllocator));
```