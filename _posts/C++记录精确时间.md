---
title: c++记录精确时间
date: 2020-05-21 09:53:00
tags: c++
category:
---

需要包含头文件
## 精确时间
bool QueryPerformanceFrequency(_Out_ LARGE_INTEGER *lpFrequency);
返回硬件支持的高精度计数器频率; 


## 获取机器内部时钟计数
QueryPerformanceCounter(LARGE_INTEGER* lpPerformanceCount);

代码如下
``` c++
#include <windows.h>

int main()
{

  // 记录频率
    __int64 countsPerSec;
    QueryPerformanceFrequency((LARGE_INTEGER*&countsPerSec);

    LARGE_INTEGER nBeginTime;
    LARGE_INTEGER nEndTime;

// 获取计数器
    QueryPerformanceCounter(&nBeginTime);

    Sleep(1000);
// 结束计数器
      QueryPerformanceCounter(&nEndTime);

//转换获取时间差
      double Time = (double)(nEndTime.QuadPart - nBeginTime.QuadPart) / countsPerSec;


    return 0;
}

```