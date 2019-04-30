---
title: UE4网络分析
date: 2019-04-20 12:52:14
tags:
---
## 网络功能：
1.**UNetDriver**：网络驱动;
2.**UReplicationGraph**： 网络表格;
3.**FSocket**： 创建Socket;
4.**PacketHandler** ;
5.**channel**： 数据通道;
6.**PlayerController**： 玩家控制器，对应一个LocalPlayer,同时对应一个connenction,记录了当前的连接信息。
7.**UPendingNetGame**：
8.**PacketHandler**：该类维护所有PacketHandler组件的数组，并将每个组件的传入包和传出包转发
9.**UNetConnection**：