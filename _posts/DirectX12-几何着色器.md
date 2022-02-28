---
title: DirectX12-几何着色器
date: 2022-02-23 19:09:30
tags:
---

几何着色器处于domain shader 与 pixel shader之间.顶点着色器以顶点作为输入数据,而几何着色器的输入数据则是完整的图元.退出几何着色器时,必须将顶点的位置转换到齐次裁剪空间.
## 编写几何着色器
```
[maxvertexcount(4)]
void GS(point VertexOut gin[1], 
        uint primID : SV_PrimitiveID, 
        inout TriangleStream<GeoOut> triStream)
{
    // 几何着色器的具体实现
}
```
1. [maxvertexcount(N)] : N 是几何着色器单次调用所输出的顶点数量最大值。
2. 输入参数：  有以下几种：
   point： 输入的图元为点。
   line： 输入的图元为线列表或线条带
   triangle： 输入的图元为三角形列表或三角形带
   lineadj： 输入的图元为线列表及其邻接图元，或线条带及其邻接图元
   triangleadj：输入的图元为三角形列表及其邻接图元，或者三角形带及其邻接图元。
3. 输出参数：输出参数一定标有inout修饰符。另外必须是一种流类型。
   PointStream<OutputVertexType> : 一系列顶点所定义的点列表
   LineStream<OutputVertexType>: 一系列顶点所定义的线列表
   TriangleStream<OutputVertexType>: 一系列顶点所定义的三角形带。
4. SV_PrimitiveID: 若指定了该语义，则输入装配器阶段会自动为每个图元生成图元ID。

 ## <center> [项目地址](https://github.com/tsxy2007/MyDirectx12)