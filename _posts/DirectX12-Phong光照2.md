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
```