---
title: UE4-调试AutomationTool
date: 2020-07-08 10:49:36
tags: 虚幻4,ue4
---

UE4 COOK过程中失败,但是资源没有错误,需要我们对AutomationTool进行调试,查找到程序崩溃点;
1. 找到调试命令:
```
-ScriptsForProject=D:/Projects/xxx/xxx.uproject BuildCookRun -nocompileeditor -nop4 -project=D:/Projects/xxx/xxx.uproject -cook -skipstage -ue4exe=D:\Projects\UnrealEngine4_25\Engine\Binaries\Win64\UE4Editor-Win64-Debug-Cmd.exe -targetplatform=Win64 -utf8output -compile
```
2. 设置VS启动项为AutomationTool
3. 把命令粘贴到UnrealVS命令框里
4. 找到ThrowEx位置我的位置如下:
~~~
IProcessResult RunResult = Run(EditorExe, Args, Options: Opts, Identifier: Commandlet); // 此为调用D:\Projects\UnrealEngine4_25\Engine\Binaries\Win64\UE4Editor-Win64-Debug-Cmd.exe
PopDir();
~~~
通过调用关系发现:
调用命令为D:\Projects\UnrealEngine4_25\Engine\Binaries\Win64\UE4Editor-Win64-Debug-Cmd.exe D:\Projects\XXX\xxx.uproject -run=Cook  -TargetPlatform=WindowsNoEditor -fileopenlog -unversioned  -stdout -CrashForUAT -unattended -NoLogTimes  -UTF8Output
5. 设置启动项为UE4
设置Command:D:\Projects\UnrealEngine4_25\Engine\Binaries\Win64\UE4Editor-Win64-Debug-Cmd.exe
设置命令参数为D:\Projects\XXX\xxx.uproject -run=Cook  -TargetPlatform=WindowsNoEditor -fileopenlog -unversioned  -stdout -CrashForUAT 
6. 启动,会在错误处断住;