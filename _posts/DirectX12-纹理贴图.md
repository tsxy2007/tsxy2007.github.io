---
title: DirectX12-纹理贴图
date: 2021-03-02 09:54:22
tags: DirectX12
---

## 学习目标：
1. 学习如何将局部纹理映射到网格三角形上；
2. 探究如何创建和启用纹理；
3. 学会如何通过纹理过滤来创建更为平滑的图像；
4. 探索如何用寻址模式来进行多次贴图；
5. 探究如何将多个纹理进行组合，从而创建出新的纹理与特效；
6. 学习如何通过纹理动画来创建一些基本效果；

## 纹理坐标：

## 纹理数据源：
### DDS格式概述：
1. mipmap;
2. GPU能自行解压的压缩格式
3. 纹理数组；
4. 立方体图（cube map）
5. 体纹理（volume texture）
低动态范围图像： DXGI_FORMAT_B8G8R8A8_UNORM 或者 DXGI_FORMAT_B8G8R8X8_UNORM
高动态范围图像：DXGI_FORMAT_R16G16B16A16_FLOAT

### 压缩格式：
1. BC1:
2. BC2:
3. BC3:
4. BC4:
5. BC5:
6. BC6:
### 创建DDS文件：
```
texconv -m 10 -f BC3_UNORM tree Array.dds
```

### 创建以及启用纹理：
#### 1.加载DDS文件
```
HRESULT DirectX::CreateDDSTextureFromFile12(
    _In_ ID3D12Device* device, // 指向用于创建纹理资源的D3D设备的指针
	_In_ ID3D12GraphicsCommandList* cmdList, // 提交GPU命令的命令列表
	_In_z_ const wchar_t* szFileName,// 预加载的图像文件名；
	_Out_ ComPtr<ID3D12Resource>& texture, // 返回载有图像数据的纹理资源
	_Out_ ComPtr<ID3D12Resource>& textureUploadHeap, // 返回的纹理资源，在此当作一个上传堆，用于将图像数据copy到默认堆，
	_In_ size_t maxsize,
	_Out_opt_ DDS_ALPHA_MODE* alphaMode)
```
#### 2.着色器资源视图堆
创建纹理资源后，需要创建一个SRV描述符（着色器资源图），
```
D3D12_DESCRIPTOR_HEAP_DESC srvHeapDesc = {};
srvHeapDesc.NumDescriptors = 3;
srvHeapDesc.Type = D3D12_DESCRIPTROR_HEAP_TYIPE_CBV_SRV_UAV;
srvHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE;
ThrowIfFailed(md3dDevice->CreateDescriptorHeap(
    &srvHeapDesc,IID_PPV_ARGS(&mSrvDescriptorHeap)
));
```
#### 创建着色器资源视图描述符
