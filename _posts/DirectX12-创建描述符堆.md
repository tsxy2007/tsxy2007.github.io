---
title: DirectX12-创建描述符堆
date: 2020-05-27 05:55:42
tags: DirectX12
---


我们需要通过创建描述符堆来存储程序中要用到的描述符/视图。用
ID3D12DescriptorHeap接口表示描述符堆，并用ID3D12Device::CreateDescriptorHeap方法来创建它。

~~~
D3D12_DESCRIPTOR_HEAP_DESC rtvHeapDesc;
	rtvHeapDesc.NumDescriptors = SwapChainBufferCount;
	rtvHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_RTV;
	rtvHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;
	rtvHeapDesc.NodeMask = 0;

	ThrowIfFailed( md3dDevice->CreateDescriptorHeap( &rtvHeapDesc, IID_PPV_ARGS( mRtvHeap.GetAddressOf( ) ) ) );


	D3D12_DESCRIPTOR_HEAP_DESC dsvHeapDesc;
	dsvHeapDesc.NumDescriptors = 1;
	dsvHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_DSV;
	dsvHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;
	dsvHeapDesc.NodeMask = 0;

	ThrowIfFailed( md3dDevice->CreateDescriptorHeap(&dsvHeapDesc, IID_PPV_ARGS( mDsvHeap.GetAddressOf( ) ) ) );

~~~

我们通过句柄来引用描述符的，并以ID3D12DescriptorHeap::GetCPUDescriptorHandleForHeapStart方法来获得描述符堆中第一个描述符的句柄。
~~~
D3D12_CPU_DESCRIPTOR_HANDLE D3DApp::CurrentBackBufferView( ) const
{
	return CD3DX12_CPU_DESCRIPTOR_HANDLE(
		mRtvHeap->GetCPUDescriptorHandleForHeapStart( ),
		mCurrBackBuffer,
		mRtvDescriptorSize );
}

D3D12_CPU_DESCRIPTOR_HANDLE D3DApp::DepthStencilView( ) const
{
	return mDsvHeap->GetCPUDescriptorHandleForHeapStart( );
}

~~~