---
title: DirectX12-创建渲染目标视图
date: 2020-05-27 06:02:35
tags: DirectX12
---

资源不能与渲染流水线中的阶段直接绑定，所以我们必须先为资源创建视图（描述符），并将其绑定到流水线阶段。
例如：为了将后台缓冲区绑定到流水线的输出合并阶段，需要为该后台缓冲区创建一个渲染目标视图。
1.获取交换链中的缓冲区资源：
~~~
HRESULT IDXGISwapChain::GetBuffer
{
    UINT Buffer, // 希望获得的特定后台缓冲区的索引
    REFIID riid,
    void ** ppSurface// 返回一个指向ID3D12Resource接口的指针，这便是希望获得的后台的缓冲区。
}
~~~
2.使用ID3D12Device::CreateRenderTargetView方法来为获取的后台缓冲区创建渲染目标视图
~~~
void ID3D12Device::CreateRenderTargetView(
    ID3D12Resource* pResource, // 指定用作渲染目标的资源
    const D3D12_RENDER_TARGET_VIEW_DESC* pDesc,//指向D3D12_RENDER_TARGET_VIEW_DESC数据结构的实例的指针。
    D3D12_CPU_DESCRIPTOR_HANDLE DestDescriptro//引用所创建渲染目标视图的描述符句柄
);


CD3DX12_CPU_DESCRIPTOR_HANDLE rtvHeapHandle( mRtvHeap->GetCPUDescriptorHandleForHeapStart( ) );
	for (UINT i = 0; i < SwapChainBufferCount; i++)
	{
		ThrowIfFailed( mSwapChain->GetBuffer( i, IID_PPV_ARGS( &mSwapChainBuffer[i] ) ) );
		md3dDevice->CreateRenderTargetView( mSwapChainBuffer[i].Get( ), nullptr, rtvHeapHandle );
		rtvHeapHandle.Offset( 1, mRtvDescriptorSize );
	}

~~~