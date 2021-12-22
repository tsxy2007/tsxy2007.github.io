---
title: DirectX12-功能支持检测
date: 2021-12-22 09:40:05
tags: DirectX12
---

##功能支持检测
```
// 调用方法：
 virtual HRESULT STDMETHODCALLTYPE CheckFeatureSupport( 
            D3D12_FEATURE Feature,
            _Inout_updates_bytes_(FeatureSupportDataSize)  void *pFeatureSupportData,
            UINT FeatureSupportDataSize
            );
```
#### 一：Feature: 枚举类型D3D12_FEATURE中的成员之一，用于指定我们希望检测的功能支持类型：
1. D3D12_FEATURE_D3D12_OPTIONIS: 检测当前驱动对Direct3D12各种功能的支持情况。
2. D3D12_FEATURE_ARCHITECTURE:检测图形适配器中GPU的硬件体系架构特性。
3. D3D12_FEATURE_FEATURE_LEVELS: 检测对功能级别的支持情况。
4. D3D12_FEATURE_FORMAT_SUPPORT: 检测对给定纹理格式的支持情况（例如，指定的格式能否用于渲染目标？或，指定的格式能否用于混合技术？）。
5. D3D12_FEATURE_MULTISAMPLE_QUALITY_LEVELS: 检测多重采样功能的支持情况。
#### 二： pFeatureSupportData:指向某种数据结构的指针，该结构中存有检索到特定功能支持的信息。此结构体的具体类型取决于Feature参数。
1. 如果Feature参数指定为D3D12_FEATURE_D3D12_OPTIONIS，则传回的是一个D3D12_FEATURE_DATA_D3D12_OPTIONIS
2. 如果Feature参数指定为D3D12_FEATURE_ARCHITECTURE，则传回的是一个D3D12_FEATURE_DATA_ARCHITECTURE.
3. 如果Feature参数指定为D3D12_FEATURE_FEATURE_LEVELS，则传回的是一个D3D12_FEATURE_DATA_FEATURE_LEVELS.
4. 如果Feature参数指定为D3D12_FEATURE_FORMAT_SUPPORT，则传回的是一个D3D12_FEATURE_DATA_FORMAT_SUPPORT.
5. 如果Feature参数指定为D3D12_FEATURE_MULTISAMPLE_QUALITY_LEVELS，则传回的是一个D3D12_FEATURE_DATA_MULTISAMPLE_QUALITY_LEVELS.
#### 三： FeatureSupportDataSize:传回pFeatureSupportData参数中的数据结构的大小。