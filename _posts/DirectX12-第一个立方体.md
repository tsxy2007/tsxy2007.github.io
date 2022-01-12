---
title: DirectX12-第一个立方体
date: 2022-01-10 09:46:44
tags:
---

经过前9篇文章的学习我们好像学了很多东西,但是都是看不到实际成果,这让我们很受伤.这篇文章我们将看到我们第一个成果立方体,一个很初级的东西.好了废话不多说我们来看看绘制一个立方体,我们还要做哪些工作.
## 顶点与输入布局
定义了顶点结构体之后,我们需要向Direct3D提供该顶点结构的描述,使它了解怎样处理每个结构体的每个成员.用户提供给Direct3D的这种描述被称为**输入布局描述**。
```
struct Vertex
{
    XMFLOAT3 Pos;
    XMFLOAT4 Color;
};

typedef struct D3D12_INPUT_LAYOUT_DESC
{
    const D3D12_INPUT_ELEMENT_DESC* pInputElementDescs;
    UINT NumElements;
} D3D12_INPUT_LAYOUT_DESC;

typedef struct D3D12_INPUT_ELEMENT_DESC
{
  LPCSTR SemanticName;                         // 与元素相关联的特定字符串（语义）
  UINT SemanticIndex;                          // 附加到语义上的索引。
  DXGI_FORMAT Format;                          // 指定定点元素的格式
  UINT InputSlot;                              // 指定传递元素所用的输入槽索引。
  UINT AlignedByteOffset;                      // 在特定输入槽中，从c++顶点结构体的首地址到其中某个元素起始地址的偏移量。
  D3D12_INPUT_CLASSIFICATION InputSlotClass;   // 暂时指定为 D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA ;D3D12_INPUT_CLASSIFICATION_PER_INSTANCE_DATA (实例化)
  UINT InstanceDataStepRate;                   // 目前仅将指定为0.如果使用实例化开启则设置为1;
} D3D12_INPUT_ELEMENT_DESC;

{
{"POSITION",0,DXGI_FORMAT_R32G32B32_FLOAT,0,0,D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA,0},
{"COLOR",0,DXGI_FORMAT_R32G32B32A32_FLOAT,0,12,D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA,0}
}
```
## 顶点缓冲区
ID3D12Resource: 缓冲区;存储顶点的叫做**顶点缓冲区**。既不是多维资源，也不支持mipmap,过滤器以及多重采样。
1. 先填写D3D12_RESOURCE_DESC描述资源
2. 调用ID3D12Device::CreateCommittedResource方法创建ID3D12Resource对象。

静态几何体我会将其放置于**默认堆**（D3D12_HEAP_TYPE_DEFAULT）,我们需要将资源上传到默认堆里，于是我们就用到了**上传堆**将资源从CPU上传到GPU的上传缓冲区，然后将上传缓冲区里的资源复制到默认缓冲区里。
```

Microsoft::WRL::ComPtr<ID3D12Resource> d3dUtil::CreateDefaultBuffer(ID3D12Device* device, ID3D12GraphicsCommandList* cmdList, const void* initData, UINT64 byteSize, Microsoft::WRL::ComPtr<ID3D12Resource>& UploadBuffer)
{
	// 创建默认缓冲区
	ComPtr<ID3D12Resource> defaultBuffer;
	//创建默认缓冲区资源堆
	CD3DX12_HEAP_PROPERTIES DefaultHeap(D3D12_HEAP_TYPE_DEFAULT);
	CD3DX12_RESOURCE_DESC defaultResourceDesc = CD3DX12_RESOURCE_DESC::Buffer(byteSize);
	device->CreateCommittedResource(
		&DefaultHeap,
		D3D12_HEAP_FLAG_NONE,
		&defaultResourceDesc,
		D3D12_RESOURCE_STATE_COMMON,
		nullptr,
		IID_PPV_ARGS(defaultBuffer.GetAddressOf())
	);
	//创建上传缓冲区资源堆
	CD3DX12_HEAP_PROPERTIES UploadHeap(D3D12_HEAP_TYPE_UPLOAD);
	CD3DX12_RESOURCE_DESC UploadResourceDesc = CD3DX12_RESOURCE_DESC::Buffer(byteSize);
	device->CreateCommittedResource(
		&UploadHeap,
		D3D12_HEAP_FLAG_NONE,
		&UploadResourceDesc,
		D3D12_RESOURCE_STATE_GENERIC_READ,
		nullptr,
		IID_PPV_ARGS(UploadBuffer.GetAddressOf())
	);
	// 映射资源
	D3D12_SUBRESOURCE_DATA subResourceData = {};
	subResourceData.pData = initData; // 资源
	subResourceData.RowPitch = byteSize; //大小
	subResourceData.SlicePitch = subResourceData.RowPitch;// 大小
	// 资源同步
	CD3DX12_RESOURCE_BARRIER WriteBarrier = CD3DX12_RESOURCE_BARRIER::Transition(defaultBuffer.Get(),
		D3D12_RESOURCE_STATE_COMMON, D3D12_RESOURCE_STATE_COPY_DEST
	);
	cmdList->ResourceBarrier(1, &WriteBarrier);
	UpdateSubresources<1>(cmdList, defaultBuffer.Get(), UploadBuffer.Get(), 0, 0, 1, &subResourceData);
	CD3DX12_RESOURCE_BARRIER ReadBarrier = CD3DX12_RESOURCE_BARRIER::Transition(
		defaultBuffer.Get(),
		D3D12_RESOURCE_STATE_COPY_DEST,
		D3D12_RESOURCE_STATE_GENERIC_READ
	);
	cmdList->ResourceBarrier(1, &ReadBarrier);
	
	return defaultBuffer;
}
```

```
void D3DApplication::BuildBoxGeometry()
{
	//创建顶点
	std::array<Vertex, 8> vertices =
	{
		Vertex({ XMFLOAT3(-1.0f, -1.0f, -1.0f), XMFLOAT4(Colors::White) }),
		Vertex({ XMFLOAT3(-1.0f, +1.0f, -1.0f), XMFLOAT4(Colors::Black) }),
		Vertex({ XMFLOAT3(+1.0f, +1.0f, -1.0f), XMFLOAT4(Colors::Red) }),
		Vertex({ XMFLOAT3(+1.0f, -1.0f, -1.0f), XMFLOAT4(Colors::Green) }),
		Vertex({ XMFLOAT3(-1.0f, -1.0f, +1.0f), XMFLOAT4(Colors::Blue) }),
		Vertex({ XMFLOAT3(-1.0f, +1.0f, +1.0f), XMFLOAT4(Colors::Yellow) }),
		Vertex({ XMFLOAT3(+1.0f, +1.0f, +1.0f), XMFLOAT4(Colors::Cyan) }),
		Vertex({ XMFLOAT3(+1.0f, -1.0f, +1.0f), XMFLOAT4(Colors::Magenta) })
	};
	// 构造索引
	std::array<std::uint16_t, 36> indices =
	{
		// front face
		0, 1, 2,
		0, 2, 3,

		// back face
		4, 6, 5,
		4, 7, 6,

		// left face
		4, 5, 1,
		4, 1, 0,

		// right face
		3, 2, 6,
		3, 6, 7,

		// top face
		1, 5, 6,
		1, 6, 2,

		// bottom face
		4, 0, 3,
		4, 3, 7
	};

	IndexCount = indices.size();
	vbByteSize = (UINT)vertices.size() * sizeof(Vertex);
	ibByteSize = (UINT)indices.size() * sizeof(std::uint16_t);
	
	// 顶点默认缓冲区
	VertexBufferGPU = d3dUtil::CreateDefaultBuffer(mD3DDevice.Get(),mD3DCommandList.Get(),vertices.data(),
		vbByteSize, VertexUploadBuffer);

	// 索引默认缓冲区
	IndexBufferGPU = d3dUtil::CreateDefaultBuffer(mD3DDevice.Get(), mD3DCommandList.Get(), indices.data(),
		ibByteSize, IndexUploadBuffer);
	
}

```

## 顶点着色器和像素着色器
```
// constant buffer view (cbv)
cbuffer cbPerObject : register(b0)
{
	float4x4 gWorldViewProj;
};
// 顶点着色器 输入
struct VertexIn
{
	float3 PosL : POSITION;
	float4 Color : COLOR;
};
// 顶点着色器 输出
struct VertexOut
{
	float4 PosH : SV_POSITION;
	float4 Color : COLOR;
};

// 顶点着色器
VertexOut VS(VertexIn vin)
{
	VertexOut vout;

	vout.PosH = mul(float4(vin.PosL,1.0f),gWorldViewProj);

	vout.Color = vin.Color;

	return vout;
}

//像素着色器
float4 PS(VertexOut pin) : SV_Target
{
    return pin.Color;
}
```

## 常量缓冲区

### 创建常量缓冲区
   在上一节中提到如下：
   1. **GPU着色器端**
	```
	cbuffer cbPerObject : register(b0)
	{
		float4x4 gWorldViewProj;
	};
	```
   2. **CPU端**

	```
	struct ObjectConstants
	{
		DirectX::XMFLOAT4X4 WorldViewProj = MathHelper::Identity4x4();
	};
	```
   3. **绑定上传资源缓冲区**
	```
	// 创建上传缓冲区
	template<typename T>
	class UploadBuffer
	{
	public:
		UploadBuffer(ID3D12Device* device, UINT elementCount, bool isConstantBuffer)
			:mIsConstantBuffer(isConstantBuffer)
		{
			// 计算资源大小
			mElementByteSize = sizeof(T);
			if (isConstantBuffer)
			{
				mElementByteSize = d3dUtil::CalcConstantBufferByteSize(mElementByteSize)	;
			}
			// 创建上传缓冲区;
			CD3DX12_HEAP_PROPERTIES hProperties(D3D12_HEAP_TYPE_UPLOAD);
			CD3DX12_RESOURCE_DESC ResourceDesc = CD3DX12_RESOURCE_DESC::Buffer	(mElementByteSize * elementCount);
			device->CreateCommittedResource(
				&hProperties,
				D3D12_HEAP_FLAG_NONE,
				&ResourceDesc,
				D3D12_RESOURCE_STATE_GENERIC_READ,
				nullptr,
				IID_PPV_ARGS(&mUploadBuffer)
			);

			mUploadBuffer->Map(0, nullptr, reinterpret_cast<void**>(&mMappedData));
		}
		UploadBuffer(const UploadBuffer& rhs) = delete;
		UploadBuffer& operator=(const UploadBuffer& rhs) = delete;
		~UploadBuffer()
		{
			if (mUploadBuffer != nullptr)
				mUploadBuffer->Unmap(0, nullptr);

			mMappedData = nullptr;
		}

		ID3D12Resource* Resource()const
		{
			return mUploadBuffer.Get();
		}

		void CopyData(int elementIndex, const T& data)
		{
			memcpy(&mMappedData[elementIndex * mElementByteSize], &data, sizeof(T));
		}

	private:
		Microsoft::WRL::ComPtr<ID3D12Resource> mUploadBuffer;
		BYTE* mMappedData = nullptr;
		UINT mElementByteSize = 0;
		bool mIsConstantBuffer = false;
	};

	```
### 常量缓冲区描述符
