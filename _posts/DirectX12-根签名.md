---
title: DirectX12-常量缓冲区视图
date: 2020-12-08 20:08:42
tags: DirectX12
---
## 类型:
### 描述符表(descriptor table): 
    描述符表引用的是描述符堆中的一块连续范围,用于确定要绑定的资源
### 根描述符(root descriptor): 
    通过直接设置跟描述符即可指示要绑定的资源,而且无需将他存于描述符堆中.但是,只有常量缓冲区的CBV,以及缓冲区SRV/UAV(着色器资源视图/无序访问视图)才可以跟描述符的身份进行绑定.纹理的SRV不能作为跟描述符来实现资源绑定;
### 根常量:
    借助根常量可直接绑定一系列32位的常量值;

## 大小:
### 放入一个根签名的数据以64DWORLD为限.三种根参数类型占用空间如下:
    1. 描述符表:每个描述符表占用1DWORLD.
    2. 根描述符: 每个根描述符(64位的GPU虚拟地址)占用2DWORLD;  
    3. 根常量: 每个常量32位,占用一个DOWRLD;

## 创建方式:
### 描述符表:
```
    CD3DX12_DESCRIPTOR_RANGE cbvTable0;
    cbvTable0.Init(D3D12_DESCRIPTOR_RANGE_TYPE_CBV, 1, 0);

    CD3DX12_ROOT_PARAMETER slotRootParameter[1];

	// Create root CBVs.
    slotRootParameter[0].InitAsDescriptorTable(1, &cbvTable0);
    
    CD3DX12_ROOT_SIGNATURE_DESC rootSigDesc(_countof(slotRootParameter), slotRootParameter, 0, nullptr, 
        D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT);
```
### 描述符
```
 	CD3DX12_ROOT_PARAMETER slotRootParameter[1];
	slotRootParameter[0].InitAsConstantBufferView(0);// 测试描述符视图

    // draw 
    mCommandList->SetGraphicsRootSignature(mRootSignature.Get());
    //设置描述指向的内容;
    auto passCB = mCurrFrameResource->PassCB->Resource();
	mCommandList->SetGraphicsRootConstantBufferView(1, passCB->GetGPUVirtualAddress());
```
### 根签名
````
	CD3DX12_ROOT_PARAMETER slotRootParameter[1];
	slotRootParameter[0].InitAsConstants(1, 2, 2); // 測試根常量

    // 直接设置参数;
    float Radius = 1.f;
	mCommandList->SetGraphicsRoot32BitConstants(2, 1, &Radius, 0);
````
