---
title: DirectX12-混合
date: 2021-10-02 21:31:05
tags: DirectX12
---
## 混合方程：
```
//控制颜色RGB分量
C = C(源) X F(源) + C(目标) X F(目标)
//控制alpha分量
A = A(源) X A(源) + A(目标) X A(目标)
```
## 混合运算

```
// 枚举成员将用作混合方程中的二元运算符：
// alpha 同样适用于alpha项
typedef enum D3D12_BLEND_OP
{
    D3D12_BLEND_OP_ADD = 1, 加法
    D3D12_BLEND_OP_SUBTRACT = 2, 源-目标
    D3D12_BLEND_OP_REV_SUBTRACT = 3, 目标-源
    D3D12_BLEND_OP_MIN=4,min（源，目标）
    D3D12_BLEND_OP_MAX =5 max（源，目标）
} D3D12_BLEND_OP
//通过逻辑运算符对源颜色和目标颜色进行混合，用以取代上述传统的混合方程，
// 但是两种方式只能取其一；
typedef enum D3D12_LOGIC_OP
{
    D3D12_LOGIC_OP_CLEAR =0 ,
    D3D12_LOGIC_OP_SET = (D3D12_LOGIC_OP_CLEAR + 1),
    D3D12_LOGIC_OP_COPY = (D3D12_LOGIC_OP_SET+1),
    D3D12_LOGIC_OP_COPY_INVERTED = (D3D12_LOGIC_OP_COPY+1),
    D3D12_LOGIC_OP_NOOP = (D3D12_LOGIC_OP_COPY_INVERTED+1),
    D3D12_LOGIC_OP_INVERT = (D3D12_LOGIC_OP_NOOP+1),
    D3D12_LOGIC_OP_AND = (D3D12_LOGIC_OP_INVERT+1),
    D3D12_LOGIC_OP_NAND = (D3D12_LOGIC_OP_AND+1),
    D3D12_LOGIC_OP_OR =(D3D12_LOGIC_OP_NAND+1),
    D3D12_LOGIC_OP_NOR = (D3D12_LOGIC_OP_OR+1),
    D3D12_LOGIC_OP_XOR = (D3D12_LOGIC_OP_NOR+1),
    D3D12_LOGIC_OP_EQUIV = (D3D12_LOGIC_OP_XOR+1),
    D3D12_LOGIC_OP_AND_REVERSE = (D3D12_LOGIC_OP_EQUIV+1),
    D3D12_LOGIC_OP_AND_INVERTED = (D3D12_LOGIC_OP_AND_REVERSE+1),
    D3D12_LOGIC_OP_OR_REVERSE = (D3D12_LOGIC_OP_AND_INVERTED+1),
    D3D12_LOGIC_OP_INVERTED = (D3D12_LOGIC_OP_OR_REVERSE+1),
} D3D12_LOGIC_OP
```
## 混合因子
D3D12_BLEND_ZERO: F=(0,0,0)且 F = 0
D3D12_BLEND_ONE: F=(1,1,1)且 F= 1
D3D12_BLEND_SRC_COLOR: F=(rs,gs,bs) 
D3D12_BLEND_INV_SRC_COLOR:
D3D12_BLEND_SRC_ALPHA :
D3D12_BLEND_INV_SRC_ALPHA:
D3D12_BLEND_DEST_ALPHA:
D3D12_BLEND_INV_DEST_ALPHA:
D3D12_BLEND_DEST_COLOR:
D3D12_BLEND_INV_DEST_COLOR:
D3D12_BLEND_SRC_ALPHA_SAT
D3D12_BLEND_BLEND_FACTOR:
```
// 设置混合因子
void ID3D12CraphicsCommandList::OMSetBlendFactor(const FLOAT BlendFactor[4]);
```
## 混合状态
配置非默认混合状态，填写D3D12_BLEND_DESC结构体。
```
typedef struct D3D12_BLEND_DESC{
    BOOL AlphaToCoverageEnable; // true 启用alpha-to-coverage功能，这是一种在渲染叶片或门等纹理时及其有用的一种多重采样技术。
    BOOL IndependentBlendEnable;最多可以支持8个渲染目标。True表明可以向每个渲染目标执行不同的混合操作（不同的混合因子，不同的混合运算以及设置不同的混合禁用或开启状态）。如果将此标记设为false，则意味着所有渲染目标均使用D3D12_BLEND_DESC::RenderTarget数组中第一个元素所描述的方式进行混合。
    D3D12_Render_TARGET_BLEND_DESC RenderTarget[8];总共8个描述混合处理方式；
} D3D12_BLEND_DESC


typedef struct D3D12_RENDER_TARGET_BLEND_DESC
{
    BOOL BlendEnable;//启用常规混合功能；注意不能和LogicOpEnable同时使用
    BOOL LogicOpEnable;//启用逻辑混合运算；
    D3D12_BLEND SrcBlend;//用于指定RGB混合中的源混合因子Fsrc。
    D3D12_BLEND DestBlend;//用于指定RGB混合中的目标混合因子Fdst。
    D3D12_BLEND SrcBlendAlpha;//指定了alpha混合中的源混合因子Fsrc。
    D3D12_BLEND DestBlendAlpha; //指定了alpha混合中目标混合因子Fdst。
    D3D12_BLEND BlendOpAlpha;//指定了alpha混合运算符。
    D3D12_LOGIC_OP LogicOp;//指定了源颜色与目标颜色在混合时所用的逻辑运算符。
    UINT8 RenderTargetWriteMask;//D3D12_COLOR_WRITE_ENABLE标记中一种或多种组合。
}D3D12_RENDER_TARGET_BLEND_DESC


typedef enum D3D12_COLOR_WRITE_ENABLE {
    D3D12_COLOR_WRITE_ENABLE_RED =1,
    D3D12_COLOR_WRITE_ENABLE_GREEN = 2,
    D3D12_COLOR_WRITE_ENABLE_BLUE = 4,
    D3D12_COLOR_WRITE_ENABLE_ALPHA = 8, //禁止向RGB通道的写操作，而仅写入alpha通道的有关数据。
    D3D12_COLOR_WRITE_ENABLE_ALL = (D3D12_COLOR_WRITE_ENABLE_RED |D3D12_COLOR_WRITE_ENABLE_GREEN
    |D3D12_COLOR_WRITE_ENABLE_BLUE|D3D12_COLOR_WRITE_ENABLE_ALPHA)
}D3D12_COLOR_WRITE_ENABLE;

```
## 混合示例
### 禁止颜色的写操作
```
    C = Csrc X Fsrc + Cdst X Fdst
    C = Csrc X (0,0,0) + Cdst X (1,1,1)
    C = Cdst
```
