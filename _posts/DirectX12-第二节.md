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
## 创建命令队列