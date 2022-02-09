---
title: DirectX12-Phong光照(二)
date: 2022-01-27 19:32:10
tags:
---

前篇文章我们从理论的角度分析了BlinnPhong光照的数学原理，这篇笔记，我们将实践以下，设计以下材质，材质缓冲区，以及光照shader；
## 材质
我们需要把每个物体的光照属性封装起来代码如下：
#### 定义材质属性
```
// 材质
struct Material
{
	// 便于查找材质的唯一对应名称
	std::string Name;

	// 本材质的常量缓冲区索引
	int MatCBIndex = -1;

	// 更新标记。
	int NumFramesDirty = gNumFrameResource;

	//用于着色的材质常量缓冲区数据
	DirectX::XMFLOAT4 DiffuseAlbedo = { 1.f,1.f,1.f,1.f }; // 漫反射照率
	DirectX::XMFLOAT3 FresnelR0 = { 0.01f,0.01f,0.01f }; // 材质属性菲涅尔
	float Roughness = 0.25f; //粗糙度

	DirectX::XMFLOAT4X4 MatTransform = MathHelper::Identity4x4();
};

// 材质缓冲区
struct MaterialConstants
{
	DirectX::XMFLOAT4 DiffuseAlbedo = { 1.f,1.f,1.f,1.f };
	DirectX::XMFLOAT3 FresnelR0 = { 0.01f,0.01f,0.01f };
	float Roughness = 0.25f;
};

```
#### 创建材质
```
std::unordered_map<std::string, std::unique_ptr<Material>> mMaterials;
void D3DApplication::BuildMaterials()
{
	auto grass = std::make_unique<Material>();
	grass->Name = "grass";
	grass->MatCBIndex = 0;
	grass->DiffuseAlbedo = XMFLOAT4(0.2f, 0.6f, 0.2f, 1.f);
	grass->FresnelR0 = XMFLOAT3(0.5f, 0.5f, 0.05f);
	grass->Roughness = 0.1f;
	mMaterials["grass"] = std::move(grass);
}

```
#### 创建材质缓冲区

```
class FrameResource
{
public:
	FrameResource(ID3D12Device* device, UINT passCount, UINT objectCount,UINT materialCount);
	FrameResource(const FrameResource& rhs) = delete;
	FrameResource& operator=(const FrameResource& rhs) = delete;
	~FrameResource();
	
	//
	Microsoft::WRL::ComPtr<ID3D12CommandAllocator> CmdListAlloc;
	//
	std::unique_ptr<UploadBuffer<PassConstants>> PassCB = nullptr;
	//
	std::unique_ptr<UploadBuffer<ObjectConstants>> ObjectCB = nullptr;
	// 材质缓冲区 每帧都有自己的
	std::unique_ptr<UploadBuffer<MaterialConstants>> MaterialCB = nullptr;
	//
	UINT64 Fence = 0;
};

```

#### 更新材质缓冲区
```
void D3DApplication::UpdateMaterialCBs()
{
	auto currMaterialCB = mCurrentFrameResource->MaterialCB.get();

	for (auto& m : mMaterials)
	{
		Material* mat = m.second.get();
		if (mat->NumFramesDirty > 0)
		{
			MaterialConstants matConstants;
			matConstants.DiffuseAlbedo = mat->DiffuseAlbedo;
			matConstants.FresnelR0 = mat->FresnelR0;
			matConstants.Roughness = mat->Roughness;

			currMaterialCB->CopyData(mat->MatCBIndex, matConstants);
			mat->NumFramesDirty--;
		}
	}
}
```

#### 绘制
```
void D3DApplication::DrawRenderItems(ID3D12GraphicsCommandList* cmdList, const std::vector<RenderItem*>& ritem)
{
	UINT matCBByteSize = d3dUtil::CalcConstantBufferByteSize(sizeof(MaterialConstants));

	auto matCB = mCurrentFrameResource->MaterialCB->Resource();
	for (int i = 0; i < ritem.size(); i++)
	{
		auto ri = ritem[i];

		cmdList->IASetVertexBuffers(0, 1, get_rvalue_ptr(ri->Geo->VertexBufferView()));
		cmdList->IASetIndexBuffer(get_rvalue_ptr(ri->Geo->IndexBufferView()));
		cmdList->IASetPrimitiveTopology(ri->PrimitiveType);

		// 通过根描述符添加cbv
		D3D12_GPU_VIRTUAL_ADDRESS matCBAddress = matCB->GetGPUVirtualAddress() + ri->Mat->MatCBIndex * matCBByteSize;
		cmdList->SetGraphicsRootConstantBufferView(2, matCBAddress);

		// 添加绘制命令;
		cmdList->DrawIndexedInstanced(ri->IndexCount, 1, ri->StartIndexLocation, ri->BaseVertexLocation, 0);
	}
}

```

## 三种光源

#### 平行光源
也称作方向光源，是一种距离目标物体极远的光源。因此我们就可以把这种光源发出的所有入射光看作是彼此平行的光纤。（太阳光）
#### 点光源
一个与点光源比较贴切的现实示例是灯泡，它能以球面向各个方向发出光线。特别是对于任意点P，由位置Q处点光源发出的光线，总有一束传播至此点。光源方向如下：

$$L = \frac{Q-P}{||Q-P||} \\$$

**衰减**：

在物理学中，光强会根据平方反比定律而随着距离函数发生衰减，距离光源d处的光强为：

$$I(d) = \frac{I_0}{d^2}$$

$I_0$为距离光源d=1处的光强。

线性衰减函数：
$$att(d)=saturate(\frac{falloffEnd - d}{falloffEnd-falloffStart})$$



#### 聚光灯光源

聚光源典型的例子就是手电筒。

1. 先指定光源方向：
$$L = \frac{Q-P}{||Q-P||} \\$$

2. 光锥的大小
$$k_{spot}(\phi) = max(cos(\phi),0)^s = max(-L*d,0)^s$$


 ## <center> [项目地址](https://github.com/tsxy2007/MyDirectx12)
