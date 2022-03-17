---
title: DirectX12-计算着色器
date: 2022-03-07 17:37:58
tags:
---

## 语法
```
cbuffer cbSettings
{
    // 计算着色器能访问的常量缓冲区数据
};

Texture2D gInputA;
Texture2D gInputB;
RWTexture2D<float4> gOutput;

[numthreads(16,16,1)]
void CS(int3 dispatchThreadID : SV_DispatchThreadID) // 线程ID
{
    gOutput[dispatchThreadID.xy] = gInputA[dispatchThreadID.xy] + gInput[dispatchThreadID.xy];
}
```

1. 通过常量缓冲区访问的全局变量
2. 输入与输出资源
3. [numthreads(X,Y,Z)]属性，指定3D线程网格中的线程数量
4. 每个线程要执行的着色器指令。
5. 线程ID系统值参数。

## 计算流水线状态对象

开启计算着色器，需要特定的计算流水线状态描述。代码如下：
```
	D3D12_COMPUTE_PIPELINE_STATE_DESC horzBlurPSO = {};
	horzBlurPSO.pRootSignature = mPostProcessRootSignature.Get();
	horzBlurPSO.CS =
	{
		reinterpret_cast<BYTE*>(mShaders["horzBlurCS"]->GetBufferPointer()),
		mShaders["horzBlurCS"]->GetBufferSize()
	};
	horzBlurPSO.Flags = D3D12_PIPELINE_STATE_FLAG_NONE;
	ThrowIfFailed(md3dDevice->CreateComputePipelineState(&horzBlurPSO, IID_PPV_ARGS(&mPSOs["horzBlur"])));

```

## 数据的输入与输出资源

### 纹理输入
Texture2D gInputA;
Texture2D gInputB;
通过给输入纹理gInputA与gInputB分别创建SRV
```

cmdList->SetComputeRootDescriptorTable(1, mBlur0GpuSrv);

cmdList->SetComputeRootDescriptorTable(2, mBlur1GpuUav);
```

## 纹理输出与无需访问视图

HLSL如下：
```
RWTexture2D<float4> gOutput;
```
输出资源与输入资源的绑定方法是不同的。为了绑定计算着色器中要执行写操作的资源，我们需要使用无序访问视图的新型视图关联在一起。代码如下：
```
CD3DX12_CPU_DESCRIPTOR_HANDLE mBlur0CpuUav;

D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
	srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
	srvDesc.Format = mFormat;
	srvDesc.ViewDimension = D3D12_SRV_DIMENSION_TEXTURE2D;
	srvDesc.Texture2D.MostDetailedMip = 0;
	srvDesc.Texture2D.MipLevels = 1;

	D3D12_UNORDERED_ACCESS_VIEW_DESC uavDesc = {};

	uavDesc.Format = mFormat;
	uavDesc.ViewDimension = D3D12_UAV_DIMENSION_TEXTURE2D;
	uavDesc.Texture2D.MipSlice = 0;

	md3dDevice->CreateUnorderedAccessView(mBlurMap0.Get(), nullptr, &uavDesc, mBlur0CpuUav);
```
### 根签名如下代码：
```
CD3DX12_DESCRIPTOR_RANGE srvTable;
	srvTable.Init(D3D12_DESCRIPTOR_RANGE_TYPE_SRV, 1, 0);

	CD3DX12_DESCRIPTOR_RANGE uavTable;
	uavTable.Init(D3D12_DESCRIPTOR_RANGE_TYPE_UAV, 1, 0);

	// Root parameter can be a table, root descriptor or root constants.
	CD3DX12_ROOT_PARAMETER slotRootParameter[3];

	// Perfomance TIP: Order from most frequent to least frequent.
	slotRootParameter[0].InitAsConstants(12, 0);
	slotRootParameter[1].InitAsDescriptorTable(1, &srvTable);
	slotRootParameter[2].InitAsDescriptorTable(1, &uavTable);

	// A root signature is an array of root parameters.
	CD3DX12_ROOT_SIGNATURE_DESC rootSigDesc(_countof(slotRootParameter), slotRootParameter,
		0, nullptr,
		D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT);

	// create a root signature with a single slot which points to a descriptor range consisting of a single constant buffer
	ComPtr<ID3DBlob> serializedRootSig = nullptr;
	ComPtr<ID3DBlob> errorBlob = nullptr;
	HRESULT hr = D3D12SerializeRootSignature(&rootSigDesc, D3D_ROOT_SIGNATURE_VERSION_1,
		serializedRootSig.GetAddressOf(), errorBlob.GetAddressOf());

	if(errorBlob != nullptr)
	{
		::OutputDebugStringA((char*)errorBlob->GetBufferPointer());
	}
	ThrowIfFailed(hr);

	ThrowIfFailed(md3dDevice->CreateRootSignature(
		0,
		serializedRootSig->GetBufferPointer(),
		serializedRootSig->GetBufferSize(),
		IID_PPV_ARGS(mPostProcessRootSignature.GetAddressOf())));
```

### 结构化缓冲区资源

```
struct Data
{
    float3 v1;
    float2 v2;
}
```