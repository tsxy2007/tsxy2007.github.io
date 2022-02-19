---
title: DirectX12-深度模板测试
date: 2022-02-19 10:54:57
tags: DirectX12
---
## 简介

    模板缓冲区是一种离屏缓冲区,我们可以利用它实现一些特殊效果.模板缓冲区,后台缓冲区以及深度缓冲区都有着相同的分辨率,这样一来这三者相同位置上的像素就能一一对应起来.模板缓冲区起到的作用所起到的作用就如同印刷过程中所使用的模板一样,我们可以用它来阻止特定像素片段渲染至后台缓冲区中.举个例子,当渲染一面镜子时,我们需要将物体反映到镜子所在的平面上.当然,只应绘制出镜子中的镜像部分.这时,我们就能通过模板缓冲区来阻止镜子范围以外镜像部分的绘制操作.
    设置模板缓冲区(以及深度缓冲区)状态,就需填写D3D12_DEPTH_STENCIL_DESC结构体实例,并将其赋予流水线状态对象的D3D12_GRAPHICS_PIPELINE_STATE_DESC::DepthStencilState字段.

## 学习目标

1. 探究如何通过填写流水线状态对象中D3D12_GRAPHICS_PIPELINE_STATE_DESC::DepthStencilState字段来控制深度缓冲区以及模板缓冲区.
2. 学习通过模板缓冲区来防止镜像被绘制至镜子以外的区域,以此来实现正确的镜像效果.
3. 了解双重混合的机制,从而利用模板缓冲区来有效的杜绝这一情况发生.
4. 知晓深度复杂性的概念,并介绍两种方法来度量场景的深度复杂性.

## 深度/模板缓冲区的格式及其资源数据的清理

深度/模板缓冲区也是一种纹理,因而必须用下列特定的数据格式来创建它.格式如下:
1. DXGI_FORMAT_D32_FLOAT_S8X24_UINT:此格式用一个32位浮点数来指定深度缓冲区,并以另一个32位无符号整数来指定模板缓冲区.其中无符号整数里的8位用于将模板缓冲区映射到[0,255],另外24位不可用,仅作填充占位
2. DXGI_FORMAT_D24_UNORM_S8_UINT:指定一个无符号的24位深度缓冲区,并将其映射到[0,1]内,另外8位用于令模板缓冲区映射到范围[0,255]

```
// 指定深度缓冲区的格式;
DXGI_FORMAT mDepthStencilFormat = DXGI_FORMAT_D24_UROM_S8_UINT;
depthStencilDesc.Format = mDepthStencilFormat;
```
我们在绘制每一帧画面之初,用以下方法重置模板缓冲区的局部数据
```
void STDMETHODCALLTYPE ClearDepthStencilView( 
D3D12_CPU_DESCRIPTOR_HANDLE DepthStencilView,
D3D12_CLEAR_FLAGS ClearFlags,
FLOAT Depth,
UINT8 Stencil,
UINT NumRects,
const D3D12_RECT *pRects) = 0;
```
1. DepthStencilView: 待清理的深度/模板缓冲区视图的描述符。
2. ClearFlags： 指定为D3D12_CLEAR_FLAG_DEPTH仅清理深度缓冲区，指定D3D12_CLEAR_FLAG_STENCIL 只清理模板缓冲区，D3D12_CLEAR_FLAG_DEPTH|D3D12_CLEAR_FLAG_STENCIL 同时清理两种缓冲区。
3. Depth: 将此浮点值设置到深度缓冲区中的每个像素。此浮点值x必须满足$0 \leq x \leq 1$
4. Stencil : 将设置整数值到模板缓冲区的每个橡树。此整数值n必须满足 $0 \leq n \leq 255$
5. NumRects : 数组pRects中所指引的矩形数量
6. pRects : 一个D3D12_RECT类型数组，标定了一系列深度/模板缓冲区内要清理的区域。

## 模板测试
处理过程如下：

if (StencilRef & StencilReadMask $\Delta$ Value & StencilReadMask)  
    accept pixel   
else   
    reject pixel

1. 左运算数由程序中定义的模板参考值StencilRef 与程序内定义的掩码值StencilReadMask 通过AND（与）运算加以确定
2. 右运算数由正在接受模板测试的特定像素位于模板缓冲区中对应值Value与程序中定义的掩码值StencilReadMask经过AND计算加以确定
   
运算符 $\Delta$ 是D3D12_COMPARISON_FUNC枚举类型定义的比较函数之一：
```
enum D3D12_COMPARISON_FUNC
    {
        D3D12_COMPARISON_FUNC_NEVER	= 1,
        D3D12_COMPARISON_FUNC_LESS	= 2,
        D3D12_COMPARISON_FUNC_EQUAL	= 3,
        D3D12_COMPARISON_FUNC_LESS_EQUAL	= 4,
        D3D12_COMPARISON_FUNC_GREATER	= 5,
        D3D12_COMPARISON_FUNC_NOT_EQUAL	= 6,
        D3D12_COMPARISON_FUNC_GREATER_EQUAL	= 7,
        D3D12_COMPARISON_FUNC_ALWAYS	= 8
    } 	D3D12_COMPARISON_FUNC;
```
1. D3D12_COMPARISON_FUNC_NEVER: 该函数总是返回false
2. D3D12_COMPARISON_FUNC_LESS： 用运算符<替换$\Delta$
3. D3D12_COMPARISON_FUNC_EQUAL: 用运算符==替换$\Delta$
4. D3D12_COMPARISON_FUNC_LESS_EQUAL: 用运算符$\leq$替换$\Delta$
5. D3D12_COMPARISON_FUNC_GREATER：用运算符>替换$\Delta$
6. D3D12_COMPARISON_FUNC_NOT_EQUAL:用运算符$\not =$替换$\Delta$
7. D3D12_COMPARISON_FUNC_GREATER_EQUAL:用运算符$\geq$替换$\Delta$
8. D3D12_COMPARISON_FUNC_ALWAYS: 此函数总是返回true
   
## 描述深度/模板状态

```
struct D3D12_DEPTH_STENCIL_DESC
{
    BOOL DepthEnable;
    D3D12_DEPTH_WRITE_MASK DepthWriteMask;
    D3D12_COMPARISON_FUNC DepthFunc;
    BOOL StencilEnable;
    UINT8 StencilReadMask;
    UINT8 StencilWriteMask;
    D3D12_DEPTH_STENCILOP_DESC FrontFace;
    D3D12_DEPTH_STENCILOP_DESC BackFace;
};
```
### 深度信息的相关设置
1. DepthEnable: 是否开启深度缓冲。禁用时，物体的绘制顺序就变得极为重要，否则位于遮挡物之后的像素片段也将绘制出来。
2. DepthWriteMask： 可将此参数设置为D3D12_DEPTH_WRITE_MASK_ZERO或者D3D12_DEPTH_WRITE_MASK_ALL。但是不能共存。假设DepthEnable为true，若把此参数设置为D3D12_DEPTH_WRITE_MASK_ZERO便会禁止对深度缓冲区的写操作，但仍可执行深度测试。D3D12_DEPTH_WRITE_MASK_ALL，则通过深度测试与模板测试的深度数据将被写入深度缓冲区。
3. DepthFunc： 将该参数指定为枚举类型D3D12_COMPARISON_FUNC的成员之一，以此来定义深度测试所用的比较函数。此项一般设置设为D3D12_COMPARISON_FUNC_LESS

### 模板信息的相关设置
1. StencilEnable：是否开启模板测试
2. StencilReadMask：该项用于模板测试如下：StencilRef & StencilReadMask $\Delta$ Value & StencilReadMask
3. StencilWriteMask：当模板缓冲区被更新时，我们可以通过写掩码来屏蔽特定为的写入操作。如果我们希望防止前4位数据被改写，便可以将写掩码设置位0x0f。 默认一般不屏蔽任何一位模板值 0xff。
4. FrontFace：填写D3D12_DEPTH_STENCILOP_DESC结构体实例，以指示根据模板测试与深度测试结果，应对正面朝向的三角形要进行何种模板运算。
5. BackFace：填写D3D12_DEPTH_STENCILOP_DESC结构体实例，以指示根据模板测试与深度测试结果，应对背面朝向的三角形要进行何种模板运算。

```
struct D3D12_DEPTH_STENCILOP_DESC
    {
    D3D12_STENCIL_OP StencilFailOp;
    D3D12_STENCIL_OP StencilDepthFailOp;
    D3D12_STENCIL_OP StencilPassOp;
    D3D12_COMPARISON_FUNC StencilFunc;
    } 
```
1. StencilFailOp: 枚举类型D3D12_STENCIL_OP中的成员之一，描述了当前像素片段在模板测试失败时，应该怎样更新模板缓冲区。
2. StencilDepthFailOp： 枚举类型D3D12_STENCIL_OP中的成员之一，描述了当前像素片段在通过模板测试，却在深度测试失败时，应该怎样更新模板缓冲区。
3. StencilPassOp： 枚举类型D3D12_STENCIL_OP中的成员之一，描述了当前像素片段在通过模板测试和深度测试时，应该怎样更新模板缓冲区。
4. StencilFunc：D3D12_COMPARISON_FUNC之一，定义模板测试所用的比较函数。

```
enum D3D12_STENCIL_OP
{
    D3D12_STENCIL_OP_KEEP	= 1,
    D3D12_STENCIL_OP_ZERO	= 2,
    D3D12_STENCIL_OP_REPLACE	= 3,
    D3D12_STENCIL_OP_INCR_SAT	= 4,
    D3D12_STENCIL_OP_DECR_SAT	= 5,
    D3D12_STENCIL_OP_INVERT	= 6,
    D3D12_STENCIL_OP_INCR	= 7,
    D3D12_STENCIL_OP_DECR	= 8
} 
```
1. D3D12_STENCIL_OP_KEEP    : 不修改模板缓冲区，即保持当前的数据
2. D3D12_STENCIL_OP_ZERO	: 将模板缓冲区中的元素设置位0
3. D3D12_STENCIL_OP_REPLACE : 将模板缓冲区中的元素替换为用于模板测试的模板参考值只有我们将深度/模板缓冲区状态块绑定到渲染流水线时，才能设置StencilRef值。
4. D3D12_STENCIL_OP_INCR_SAT: 对模板缓冲区中的元素进行递增操作。如果递增超出最大值，则将此模板缓冲区元素限定为最大值
5. D3D12_STENCIL_OP_DECR_SAT: 对模板缓冲区中的元素进行递减操作。如果递减值小于0，则将改模板缓冲区元素限定为0.
6. D3D12_STENCIL_OP_INVERT	: 将模板缓冲区中的元素数据按二进制位进行反转。
7. D3D12_STENCIL_OP_INCR	: 对模板缓冲区中的元素进行递增操作。如果超过最大值，则环回至0.
8. D3D12_STENCIL_OP_DECR	: 对模板缓冲区中的元素进行递减操作。如果小于0，则环回至最大值
   
## 创建和绑定深度/模板状态
```
//
D3D12_GRAPHICS_PIPELINE_STATE_DESC::DepthStencilState设置此字段；
//设置模板参考值。
mD3DCommandList->OMSetStencilRef(0);
```