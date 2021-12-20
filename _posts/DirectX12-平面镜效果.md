---
title: DirectX12-平面镜效果
date: 2021-11-01 22:09:59
tags: DirectX12
---
## 镜像概述
### 1.将地板，墙壁以及骷髅头实物照常渲染到后台缓冲区（不包括镜子）。注意，此步骤不修改模板缓冲区
### 2.清理模板缓冲区，将其整体置零。
### 3.仅将镜面渲染到模板缓冲区中。
```
D3D12_RENDER_TARGET_BLEND_DESC::RenderTargetWriteMask = 0; // 禁止其他颜色写入后台缓冲区
D3D12_DEPTH_STENCIL_DESC::DeptWriteMask = D3D12_DEPTH_WRITE_MASK_ZERO; // 禁止向深度缓冲区的写操作;
```
### 4.我们将骷髅头的镜像渲染至后台缓冲区及模板缓冲区。
### 5.将镜面渲染到后台缓冲区中。