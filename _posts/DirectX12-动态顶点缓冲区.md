---
title: DirectX12-动态顶点缓冲区
date: 2022-01-19 08:28:50
tags: Direct3D
---

到目前为止我们始终将顶点数据存于默认的缓冲区资源当中，可借此存储静态几何体。但是我们不能动态的改变此资源中所存的几何体（只能一次性设置好数据），再以GPU读取其中的数据并进行绘制。为了解决我们此种需求，我们需要**动态顶点缓冲区**，它允许用户频繁的更改其中的顶点数据。另一种需要动态缓冲区情况是：**粒子系统**。

## 创建上传缓冲区
```
template<typename T>
class UploadBuffer
{
public:
    UploadBuffer(ID3D12Device* device, UINT elementCount, bool isConstantBuffer) : 
        mIsConstantBuffer(isConstantBuffer)
    {
        mElementByteSize = sizeof(T);

        if(isConstantBuffer)
            mElementByteSize = d3dUtil::CalcConstantBufferByteSize(sizeof(T));

        // 创建上传缓冲区
        ThrowIfFailed(device->CreateCommittedResource(
            &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD),
            D3D12_HEAP_FLAG_NONE,
            &CD3DX12_RESOURCE_DESC::Buffer(mElementByteSize*elementCount),
			D3D12_RESOURCE_STATE_GENERIC_READ,
            nullptr,
            IID_PPV_ARGS(&mUploadBuffer)));

        // 映射资源
        ThrowIfFailed(mUploadBuffer->Map(0, nullptr, reinterpret_cast<void**>(&mMappedData)));
    }

    UploadBuffer(const UploadBuffer& rhs) = delete;
    UploadBuffer& operator=(const UploadBuffer& rhs) = delete;
    ~UploadBuffer()
    {
        // 解除映射
        if(mUploadBuffer != nullptr)
            mUploadBuffer->Unmap(0, nullptr);

        mMappedData = nullptr;
    }

    ID3D12Resource* Resource()const
    {
        return mUploadBuffer.Get();
    }

    void CopyData(int elementIndex, const T& data)
    {
        // copy 资源
        memcpy(&mMappedData[elementIndex*mElementByteSize], &data, sizeof(T));
    }

private:
    Microsoft::WRL::ComPtr<ID3D12Resource> mUploadBuffer;
    BYTE* mMappedData = nullptr;

    UINT mElementByteSize = 0;
    bool mIsConstantBuffer = false;
};
```

## 缺点以及避免使用动态顶点缓冲区

使用动态缓冲区时会不可避免的产生一些开销，因为必须将新数据从CPU端内存回传至GPU端显存。

1. 简单的动画可以在顶点着色器中实现。
2. 可以使用渲染到纹理，或者计算着色器与顶点纹理拾取(vertex texture fetch)等技术实现上述波浪模拟，而且全程在GPU中实现
3. 几何着色器为GPU创建或销毁图元提供支持，在几何着色器出现之前，一半用CPU来处理相关任务。
4. 在曲面细分阶段中可以通过GPU来对几何体进行镶嵌化处理，在硬件曲面细分之前，通常由CPU负责。

