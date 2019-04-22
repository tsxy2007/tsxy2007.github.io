---
title: DirectX12-第二节
date: 2019-04-19 19:16:29
tags: DirectX12
category:
---

## 一： 创建RootSignature
``` c++
CD3DX12_ROOT_SIGNATURE_DESC rootSignatureDesc;
rootSignatureDesc.Init(0, nullptr, 0, nullptr, D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT);
ComPtr<ID3DBlob> signature;
ComPtr<ID3DBlob> error;
D3D12SerializeRootSignature(&rootSignatureDesc, D3D_ROOT_SIGNATURE_VERSION_1, &signature, &error);
m_device->CreateRootSignature(0, signature->GetBufferPointer(), signature->GetBufferSize(), IID_PPV_ARGS(&m_rootSignature));
```
## 二： 创建PSO
``` c++
UINT compileFlags = D3DCOMPILE_DEBUG | D3DCOMPILE_SKIP_OPTIMIZATION;
ComPtr<ID3DBlob> vertexShader;
ComPtr<ID3DBlob> pixelShader;
std::wstring shaderFile = L"shaders.hlsl";
D3DCompileFromFile(shaderFile,nullptr,nullptr,"VSMain","vs_5_0",compileFlags,0,&vertexShader,nullptr);
D3DCompileFromFile(shaderFile,nullptr,nullptr,"PSMain","ps_5_0",compileFlags,0,&pixelShader,nullptr);

D3D12_INPUT_ELEMENT_DESC inputElementDesc[] = 
{
    {"POSITION",0,DXGI_FORMAT_R32G32B32_FLOAT,0,0,D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA,0},

    {"COLOR",0,DXGI_FORMAT_R32G32B32A32_FLOAT,0,12,D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA,0},
};
// 创建pso
D3D12_GRAPHICS_PIPELINE_STATE_DESC psoDesc = {};
psoDesc.InputLayout ={inputElementDesc,_countof(inputElementDesc)};
psoDesc.pRootSignature = m_rootSignture.Get();
psoDesc.VS = CD3DX12_SHADER_BYTECODE(vertexShader.Get());
psoDesc.PS = CD3DX12_SHADER_BYTECODE(pixelShader.Get());
psoDesc.RaterizerState = CD3DX12_RASTERIZER_DESC(D3D12_DEFAULT);
psoDesc.DepthStencilState.DepthEnable = FALSE;
psoDesc.SampleMask = UINT_MAX;
psoDesc.PrimitiveTopologyType = D3D12_PRIMITIVE_TOPOLOGY_TYPE_TRIANGLE;
psoDesc.NumRenderTargets = 1;
psoDesc.RTVFormats[0] = DXGI_FORMAT_R8G8B8A8_UNORM;
psoDesc.SampleDesc.Count = 1;
m_device->CreateGraphicsPipelineState(&psoDesc, IID_PPV_ARGS(&m_pipelineState));
```
## 三： 创建命令队列
```
m_device->CreateCommandList(0,D3D12_COMMAND_LIST_TYPE_DIRECT,m_commandAllocator.Get(),m_pipelineState.Get(),IID_PPV_ARGS(&m_commandList));
m_commandList->Close();
```
## 四： 创建Vertex Buffer
``` c++
Vertex triangleVertices[] = 
{
	{ { 0.0f, 0.25f * m_aspectRatio, 0.0f }, { 1.0f, 0.0f, 0.0f, 1.0f } },
	{ { 0.25f, -0.25f * m_aspectRatio, 0.0f }, { 0.0f, 1.0f, 0.0f, 1.0f } },
	{ { -0.25f, -0.25f * m_aspectRatio, 0.0f }, { 0.0f, 0.0f, 1.0f, 1.0f } }
};

const UINT vertexBufferSize = sizeof(triangleVertices);
m_device->CreateCommittedResource(
    &CD3DX12_HEAP_PROERTIES(D3D12_HEAP_TYPE_UIPLOAD),
    D3D12_HEAP_FLAG_NONE,
    &CD3DX12_RESOURCE_DESC::buffer(vertexBufferSize),
    nullptr,
    IID_PPV_ARGS(&m_vertexBuffer)
);

// 复制这个三角形数据到vertex buffer
UINT8* pVertexDataBegin;
CD3DX12_RANGE readRange(0,0);
m_vertexBuffer->Map(0,&readRange,reinterpret_cast<void**>(&pVertexDataBegin));
memcpy(pVertexDataBegin,triangleVertices,sizeof(triangleVertices));
m_vertexBuffer->Unmap(0,nullptr);
//初始化vertexbuffer view
m_vertexBufferView.BufferLocation = m_vertexBuffer->GetGPUVirtualAddress();
m_vertexBufferView.StrideInBytes= sizeof(Vertex);
m_vertexBufferView.SizeInBytes = vertexBufferSize;
```
## 五：创建Fence 异步并且等待assets被GPU加载
```
ThrowIfFailed(m_device->CreateFence(0, D3D12_FENCE_FLAG_NONE, IID_PPV_ARGS(&m_fence)));
m_fenceValue = 1;
m_fenceEvent = CreateEvent(nullptr, FALSE, FALSE, nullptr);
if (m_fenceEvent == nullptr)
{
	ThrowIfFailed(HRESULT_FROM_WIN32(GetLastError()));
}
WaitForPreviousFrame();
```
## 六： 执行命令
``` c++
// 记录所有命令
PopulateCommandList();
//执行命令
ID3D12CcommandList* ppCommandList[] = {m_commandList.Get()};
m_commandQueue->ExecuteCommandLists(_countof(ppCommandLists),ppCommandLists);
// 推送
m_swapChain->Present(1,0);
WaitForPreviousFrame();


PopulateCommandList()
{
    ThrowIfFailed(m_commandAllocator->Reset());
    ThrowIfFailed(m_commandList->Reset(m_commandAllocator.Get(), m_pipelineState.Get()));

	//Set necessary state
	m_commandList->SetGraphicsRootSignature(m_rootSignature.Get());
	m_commandList->RSSetViewports(1, &m_viewport);
	m_commandList->RSSetScissorRects(1, &m_scissorRect);

	// 投射viewtarget
	m_commandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(m_renderTargets[m_frameIndex].Get(), D3D12_RESOURCE_STATE_PRESENT, D3D12_RESOURCE_STATE_RENDER_TARGET));

	CD3DX12_CPU_DESCRIPTOR_HANDLE rtvHandle(m_rtvHeap->GetCPUDescriptorHandleForHeapStart(), m_frameIndex, m_rtvDescriptorSize);
	m_commandList->OMSetRenderTargets(1, &rtvHandle, FALSE, nullptr);

	// Record commands
	const float clearColor[] = { 0.f,0.2f,.4f,1.f };
	m_commandList->ClearRenderTargetView(rtvHandle, clearColor, 0, nullptr);
	m_commandList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
	m_commandList->IASetVertexBuffers(0, 1, &m_vertexBufferView);
	m_commandList->DrawInstanced(3, 1, 0, 0);

	// Indicate
	m_commandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(m_renderTargets[m_frameIndex].Get(), D3D12_RESOURCE_STATE_RENDER_TARGET, D3D12_RESOURCE_STATE_PRESENT));

	ThrowIfFailed(m_commandList->Close());  
}

void D3D12HelloTriangle::WaitForPreviousFrame()
{
	const UINT64 fence = m_fenceValue;
	ThrowIfFailed(m_commandQueue->Signal(m_fence.Get(), fence));
	m_fenceValue++;
	if (m_fence->GetCompletedValue()<fence)
	{
		ThrowIfFailed(m_fence->SetEventOnCompletion(fence, m_fenceEvent));
		WaitForSingleObject(m_fenceEvent, INFINITE);
	}
	m_frameIndex = m_swapChain->GetCurrentBackBufferIndex();
}

```