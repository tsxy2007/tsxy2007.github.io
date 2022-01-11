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
			mElementByteSize = d3dUtil::CalcConstantBufferByteSize(mElementByteSize);
		}
		// 创建上传缓冲区;
		CD3DX12_HEAP_PROPERTIES hProperties(D3D12_HEAP_TYPE_UPLOAD);
		CD3DX12_RESOURCE_DESC ResourceDesc = CD3DX12_RESOURCE_DESC::Buffer(mElementByteSize * elementCount);
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