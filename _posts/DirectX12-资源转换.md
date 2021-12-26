---
title: DirectX12-资源转换
date: 2020-05-25 07:01:27
tags: DirectX12
---

为了实现常见的渲染效果，我们会经常通过GPU对某个资源按顺序进行先写后读两种操作。如果GPU的写操作还没有完成甚至还没开始，
却开始读取资源，便会导致resource hazard（资源冒险）。Direct3D专门针对资源设计了一组相关状态。资源在创建一开始处于默
认状态，该状态将一直持续到应用程序通过Direct3D将其转换为另一种状态为止。如执行写操作，转换为渲染目标状态；读操作，状
态设置为着色器资源状态。

通过命令列表设置转换资源屏障数组；
D3D12_RESOURCE_BARRIER（资源屏障）；
~~~
	cmdList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(defaultBuffer.Get(), 
		D3D12_RESOURCE_STATE_COMMON, D3D12_RESOURCE_STATE_COPY_DEST));
~~~
**(待完善)**