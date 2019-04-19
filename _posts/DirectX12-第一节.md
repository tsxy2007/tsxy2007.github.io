---
title: DirectX12 第一节
date: 2019-04-13 00:02:00
tags:
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