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
#### 3.创建着色器资源视图描述符
通过创建填写D3D12_SHADER_RESOURCE_VIEW_DESC对象来描述SRV描述符
```
// 参数含义
typedef struct D3D12_SHADER_RESOURCE_VIEW_DESC
    {
    DXGI_FORMAT Format;                 // 视图格式
    D3D12_SRV_DIMENSION ViewDimension;  // 资源维数
    UINT Shader4ComponentMapping;       // 在着色器中堆纹理进行采样时，它将返回纹理坐标处的纹理数据向量
    union 
        {
        D3D12_BUFFER_SRV Buffer;
        D3D12_TEX1D_SRV Texture1D;
        D3D12_TEX1D_ARRAY_SRV Texture1DArray;
        D3D12_TEX2D_SRV Texture2D;
        D3D12_TEX2D_ARRAY_SRV Texture2DArray;
        D3D12_TEX2DMS_SRV Texture2DMS;
        D3D12_TEX2DMS_ARRAY_SRV Texture2DMSArray;
        D3D12_TEX3D_SRV Texture3D;
        D3D12_TEXCUBE_SRV TextureCube;
        D3D12_TEXCUBE_ARRAY_SRV TextureCubeArray;
        D3D12_RAYTRACING_ACCELERATION_STRUCTURE_SRV RaytracingAccelerationStructure;
        } 	;
    } 	D3D12_SHADER_RESOURCE_VIEW_DESC;

typedef struct D3D12_TEX2D_SRV
{
    UINT MostDetailedMip;   // 指定此视图中图像细节最详尽的Mipmap层级的索引
    UINT MipLevels;         // 此视图的mipmap层级数量;
    UINT PlaneSlice;        // 平面切片的索引
    FLOAT ResourceMinLODClamp; // 指定可以访问的最小mipmap层级
} D3D12_TEX2D_SRV;
```
```
//创建逻辑
	CD3DX12_CPU_DESCRIPTOR_HANDLE hDescriptor(mSrvDescriptorHeap->GetCPUDescriptorHandleForHeapStart());
	hDescriptor.Offset(mTextureCbvOffset, mCbvSrvUavDescriptorSize);
	auto woodCrateTex = mTextures["woodCrateTex"]->Resource;

	D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
	srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING; // 在着色器中对纹理进行采样时，它将返回特定纹理坐标处的纹理数据向量。
	srvDesc.Format = woodCrateTex->GetDesc().Format; //视图格式
	srvDesc.ViewDimension = D3D12_SRV_DIMENSION_TEXTURE2D; // 资源的维数
	srvDesc.Texture2D.MostDetailedMip = 0; //指定此视图中图像细节最详尽的mipmap层级的索引
	srvDesc.Texture2D.MipLevels = woodCrateTex->GetDesc().MipLevels; // 此视图的mipmap层级数量，以MostDetailedMip作为起始值。
	srvDesc.Texture2D.ResourceMinLODClamp = 0.f;//指定可以访问的最小mipmap层级。

	mD3DDevice->CreateShaderResourceView(woodCrateTex.Get(), &srvDesc, hDescriptor);
```


#### 4.将纹理绑定到流水线

```
{
			//纹理
			int TexIndex = ri->Mat->DiffuseSrvHeapIndex + mTextureCbvOffset;
			CD3DX12_GPU_DESCRIPTOR_HANDLE tex(mCbvHeap->GetGPUDescriptorHandleForHeapStart());
			tex.Offset(TexIndex, mCbvSrvUavDescriptorSize);
			cmdList->SetGraphicsRootDescriptorTable(3, tex);
}
```

### 过滤器
#### 纹理放大
1. 最近领点采样
2. 双线性插值
#### 纹理缩小（mipmap）
1. 选择与待投影屏幕上的几何分辨率最为匹配的mipmap层级，并根据具体需求选用常数插值或者线性插值
2. 选择与带投影屏幕上的几何分辨率最为匹配的临近的两个mipmap层级，对这两个层级分别应用常量过滤或者线性过滤。
#### 各项异性过滤
略

### 寻址模式
1. 重复寻址模式
2. 边框颜色寻址模式
3. 钳位寻址模式
4. 镜像寻址模式

```
typedef 
enum D3D12_TEXTURE_ADDRESS_MODE
    {
        D3D12_TEXTURE_ADDRESS_MODE_WRAP	= 1,    //重复寻址模式
        D3D12_TEXTURE_ADDRESS_MODE_MIRROR	= 2, //镜像寻址模式
        D3D12_TEXTURE_ADDRESS_MODE_CLAMP	= 3, // 钳位寻址模式
        D3D12_TEXTURE_ADDRESS_MODE_BORDER	= 4, // 边框颜色寻址模式
        D3D12_TEXTURE_ADDRESS_MODE_MIRROR_ONCE	= 5 //镜像寻址模式
    } 	D3D12_TEXTURE_ADDRESS_MODE;

```
### 采样器对象
#### 创建采样器
```
// 采样器描述符
typedef struct D3D12_SAMPLER_DESC
    {
    D3D12_FILTER Filter;    //D3D12_FILTER 枚举类型成员之一，用以指定采样纹理时所用的过滤方式
    D3D12_TEXTURE_ADDRESS_MODE AddressU; // 纹理在u轴方向上所用的寻址模式
    D3D12_TEXTURE_ADDRESS_MODE AddressV; // 纹理在v轴方向上所用的寻址模式
    D3D12_TEXTURE_ADDRESS_MODE AddressW;// 纹理在w轴方向上所用的寻址模式
    FLOAT MipLODBias;   // 设置mipmap层级的偏置值
    UINT MaxAnisotropy; // 最大各向异性值，取值范围[1,16].只有将Filter设置为D3D12_FILTER_ANISOTROPIC或D3D12_FILTER_COMPARISON_ANISOTROPIC之后该项才能生效
    D3D12_COMPARISON_FUNC ComparisonFunc; // 用于实现阴影贴图这样一类特殊应用的高级选项
    FLOAT BorderColor[ 4 ]; // 用于指定在D3D12_TEXTURE_ADDRESS_MODE_BORDER寻址模式下的边框颜色
    FLOAT MinLOD; // 可供选择的最小mipmap层级
    FLOAT MaxLOD; // 可供选择的最大mipmap层级
    } 	D3D12_SAMPLER_DESC;
```
**创建过程**
```
// 只能描述符表
CD3DX12_DESCRIPTOR_RANGE cbvTable;
	cbvTable.Init(
		D3D12_DESCRIPTOR_RANGE_TYPE_SAMPLER,
		1,
		0//绑定的槽号
	);

//描述符堆

	D3D12_DESCRIPTOR_HEAP_DESC srvHeapDesc = {};
	srvHeapDesc.NumDescriptors = 1;
	srvHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_SAMPLER;
	srvHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE;
	mD3DDevice->CreateDescriptorHeap(&srvHeapDesc, IID_PPV_ARGS(&mSrvDescriptorHeap));




// 采样器描述符
D3D12_SAMPLER_DESC sampleDesc = {};
	sampleDesc.Filter = D3D12_FILTER_MIN_MAG_MIP_LINEAR;
	sampleDesc.AddressU = D3D12_TEXTURE_ADDRESS_MODE_WRAP;
	sampleDesc.AddressV = D3D12_TEXTURE_ADDRESS_MODE_WRAP;
	sampleDesc.AddressW = D3D12_TEXTURE_ADDRESS_MODE_WRAP;
	sampleDesc.MinLOD = 0;
	sampleDesc.MaxLOD = D3D12_FLOAT32_MAX;
	sampleDesc.MipLODBias = 0.f;
	sampleDesc.MaxAnisotropy = 1;
	sampleDesc.ComparisonFunc = D3D12_COMPARISON_FUNC_ALWAYS;
	mD3DDevice->CreateSampler(&sampleDesc, mSrvDescriptorHeap->GetCPUDescriptorHandleForHeapStart());
// 绑定流水线
//动态采样器
			cmdList->SetGraphicsRootDescriptorTable(1,

				mSrvDescriptorHeap->GetGPUDescriptorHandleForHeapStart());
```

#### 静态采样器

```
// 创建静态采样器
CD3DX12_STATIC_SAMPLER_DESC pointWrap(
		0, // shaderRegiste着色器寄存器
		D3D12_FILTER_MIN_MAG_MIP_POINT, // 过滤器类型
		D3D12_TEXTURE_ADDRESS_MODE_WRAP,  // addressU u轴寻址模式
		D3D12_TEXTURE_ADDRESS_MODE_WRAP,  // addressV v轴寻址模式
		D3D12_TEXTURE_ADDRESS_MODE_WRAP); // addressW w轴寻址模式

// 绑定静态采样器
auto staticSamplers = GetStaticSamplers();

	CD3DX12_ROOT_SIGNATURE_DESC rootSigdesc(
		_countof(rootParameter), 
		rootParameter, 
		(UINT)staticSamplers.size(), 
		staticSamplers.data(), 
		D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT);
```
 ## <center> [项目地址 分支 纹理映射(第九章)](https://github.com/tsxy2007/MyDirectx12)