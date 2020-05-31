---
title: DirectX12-设置视口
date: 2020-05-31 16:38:25
tags: DirectX12
---

# 设置视口

    我们通常会将3D场景绘制到与整个屏幕（在全屏模式下）或整个窗口工作区大小相当的后台缓冲区中。如果只想绘制到后台缓冲区的某个矩形子区域中。我们把后台缓冲区中的矩形子区域叫做视口，一下时结构体描述它：
    ~~~
    typedef struct D3D12_VIEWPORT
    {
        FLOAT TopLeftX;
        FLOAT TopLeftY;
        FLOAT Width;
        FLOAT Height;
        FLOAT MinDepth; // 最小深度值
        FLOAT MAXDepth; // 最大深度值
    } D3D12_VIEWPORT ;

    ~~~

## TIP:
1.不能为同一个渲染目标指定多个视口，而多视口则是一种用于对多个渲染目标同时进行渲染的高级技术。
2.命令列表一旦被重置，视口也需要随之而重置。

# 设置裁剪矩形：
我们可以在相对于后台缓冲区定义一个裁剪矩形，在此矩形外的像素都将被剔除，这个方法用于优化程序性能。
裁剪矩形由类型为RECT的D3D12_RECT结构体定义而成：
~~~
typedef struct tagRECT
{
    LONG left;
    LONG top;
    LONG right;
    LONG bottom;
} RECT;
~~~

在Direct3D中，要用ID3D12GraphicsCommandList::RSSetScissorRects 方法来设置裁剪矩形。下面的示例将创建并设置一个覆盖后台缓冲区左上角1/4 区域的裁剪矩形：
~~~
mScissorRect = {0,0,mClientWidth/2,mClientHeight/2};
mCommandList->RSSetScissorRects(1,&mScissorRect); // 数量/指针
~~~
## TIP：
不能为同一个渲染目标指定多个裁剪矩形。 多裁剪矩形是一种用于同时对多个渲染目标进行渲染的高级技术。