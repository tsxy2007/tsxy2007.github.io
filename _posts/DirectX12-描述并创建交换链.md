---
title: DirectX12-描述并创建交换链
date: 2020-05-27 05:30:07
tags: DirectX12
---

初始化流程的下一步是创建交换链。
填写一份DXGI_SWAP_CHAIN_DESC用来描述欲创建交换链的特性。

~~~
typedef struct DXGI_SWAP_CHAIN_DESC
    {
    DXGI_MODE_DESC BufferDesc; // 描述待创建后台缓冲区的属性。
    DXGI_SAMPLE_DESC SampleDesc;//多重采样的质量级别以及对每个像素的采样次数，对于单次采样来说，指定数量为1，质量级别指定0.
    DXGI_USAGE BufferUsage;//如果将数据渲染至后台缓冲区（用它作为渲染目标），因此将此参数指定为DXGI_USAGE_RENDER_TARGET_OUTPUT.
    UINT BufferCount;//交换链中所用的缓冲区数量。
    HWND OutputWindow;//渲染窗口的句柄
    BOOL Windowed; // true 为窗口模式，false，全屏模式
    DXGI_SWAP_EFFECT SwapEffect; // 指定为DXGI_SWAP_EFFECT_FLIP_DISCARD
    UINT Flags;//可选标记，如果指定为DXGI_SWAP_CHAIN_FLAG_ALLOW_MODE_SWITCH,那么程序切换为全屏模式时，选择最适于当前应用程序窗口尺寸的显示模式
    //如果没有指定，将采用当前桌面的显示模式。
    } 	DXGI_SWAP_CHAIN_DESC;


typedef struct DXGI_MODE_DESC
{
    UINT Width; // 缓冲区分辨率的宽度
    UINT Height;// 缓冲区分辨率的高度
    DXGI_RATIONAL RefreshRate; // 刷新频率
    DXGI_FORMAT Format;// // 缓冲区的显示格式
    DXGI_MODE_SCANLINE_ORDER ScanlineOrdering; // 逐行扫描vs隔行扫描
    DXGI_MODE_SCALING Scaling;//图像如何相对屏幕进行拉伸。
} DXGI_MODE_DESC;

typedef struct DXGI_SAMPLE_DESC
{
    UINT Count; // 多重采样数量
    UINT Quality;// 多重采样质量
} DXGI_SAMPLE_DESC;
~~~

一下是实际代码：
~~~
mSwapChain.Reset( );
	DXGI_SWAP_CHAIN_DESC sd;
	sd.BufferDesc.Width = mClientWidth;
	sd.BufferDesc.Height = mClientHeight;
	sd.BufferDesc.RefreshRate.Numerator = 60;
	sd.BufferDesc.RefreshRate.Denominator = 1;
	sd.BufferDesc.Format = mBackBufferFormat;
	sd.BufferDesc.ScanlineOrdering = DXGI_MODE_SCANLINE_ORDER_UNSPECIFIED;
	sd.BufferDesc.Scaling = DXGI_MODE_SCALING_UNSPECIFIED;
	sd.SampleDesc.Count = m4xMsaaState ? 4 : 1;
	sd.SampleDesc.Quality = m4xMsaaState ? (m4xMsaaQuality - 1) : 0;
	sd.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;
	sd.BufferCount = SwapChainBufferCount;
	sd.OutputWindow = mhMainWnd;
	sd.Windowed = true;	
	sd.SwapEffect = DXGI_SWAP_EFFECT_FLIP_DISCARD;
	sd.Flags = DXGI_SWAP_CHAIN_FLAG_ALLOW_MODE_SWITCH;

	ThrowIfFailed( mdxgiFactory->CreateSwapChain(
		mCommandQueue.Get( ),
		&sd,
		mSwapChain.GetAddressOf( )
	) );
~~~

