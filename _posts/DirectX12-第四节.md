---
title: DirectX12-第四节
date: 2019-05-09 09:41:47
tags: Directx12
category:
---
## 显卡架构和存储管理
![avatar](https://img-blog.csdnimg.cn/20181027170417659.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQwMzgxNDM=,size_27,color_FFFFFF,t_70)

以上图片来自MSDN，很形象的说明了一个GPU至少有三大类复制引擎（Copy Engine）,计算引擎（Compute Engine），3D引擎（3d Engine）。同时这图片进一步说明了Cpu和Gpu之间如何交互命令，以及并行原理：核心还是使用命令列表记录命令，再将命令列表在命令队列中排队，然后各引擎从命令队列中不断获取命令并执行的基本模式。
SMP架构分为三种：
1. **均匀存储器存取（UMA)**
   ![avatar](https://img-blog.csdnimg.cn/20181027170417615.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQwMzgxNDM=,size_27,color_FFFFFF,t_70)
2. **非均匀存储器存取（Nonuniform-Memory-Access，简称NUMA）**
   ![avatar](https://img-blog.csdnimg.cn/20181027170417620.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQwMzgxNDM=,size_27,color_FFFFFF,t_70)
3. **高速缓存相关的存储器结构（cache-coherent Memory Architecture，简称CC-UMA）模型**
   ![avatar](https://img-blog.csdnimg.cn/20181027170417625.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQwMzgxNDM=,size_27,color_FFFFFF,t_70)
