---
title: UE5初始化分析
date: 2023-0-06 14:43:14
tags: ue5
---
## 程序入口
```
// windows 程序入口；
int32 WINAPI WinMain(_In_ HINSTANCE hInInstance, _In_opt_ HINSTANCE hPrevInstance, _In_ char* pCmdLine, _In_ int32 nCmdShow)
{
	int32 Result = LaunchWindowsStartup(hInInstance, hPrevInstance, pCmdLine, nCmdShow, nullptr);
	LaunchWindowsShutdown();
	return Result;
}
```
## Launch入口
```
LAUNCH_API int32 LaunchWindowsStartup( HINSTANCE hInInstance, HINSTANCE hPrevInstance, char*, int32 nCmdShow, const TCHAR* CmdLine )
{
    // TRACE日志跟踪系统 
    //创建了一个名为“WinMain.Enter”的书签，并记录了该书签的时间等信息。
	TRACE_BOOKMARK(TEXT("WinMain.Enter"));

	// 命令行参数
	SetupWindowsEnvironment();

    // 错误等级
	int32 ErrorLevel			= 0;
    // UE5程序进程句柄
	hInstance				= hInInstance;

	if (!CmdLine)
	{
        // 获取命令行参数
		CmdLine = ::GetCommandLineW();

		//尝试使用标准的windows实现来处理命令行参数(这确保与使用argc和argv的其他平台的行为相同)。
		if ( ProcessCommandLine() )
		{
			CmdLine = *GSavedCommandLine;
		}
	}

	// 解析命令行参数时候包含“无人值守”选项
	if ( FParse::Param( CmdLine, TEXT("unattended") ) )
	{
		SetErrorMode(SEM_FAILCRITICALERRORS | SEM_NOGPFAULTERRORBOX | SEM_NOOPENFILEERRORBOX);
	}
    // 解析命令行参数时候包含“崩溃上报”选项
	if ( FParse::Param( CmdLine,TEXT("crashreports") ) )
	{
		GAlwaysReportCrash = true;
	}

    // 忽略异常捕捉功能
	bool bNoExceptionHandler = FParse::Param(CmdLine,TEXT("noexceptionhandler"));
	(void)bNoExceptionHandler;

    // 忽略调试功能
	bool bIgnoreDebugger = FParse::Param(CmdLine, TEXT("IgnoreDebugger"));
	(void)bIgnoreDebugger;

	bool bIsDebuggerPresent = FPlatformMisc::IsDebuggerPresent() && !bIgnoreDebugger;
	(void)bIsDebuggerPresent;

	// Using the -noinnerexception parameter will disable the exception handler within native C++, which is call from managed code,
	// which is called from this function.
	// The default case is to have three wrapped exception handlers 
	// Native: WinMain() -> Native: GuardedMainWrapper().
	// The inner exception handler in GuardedMainWrapper() catches crashes/asserts in native C++ code and is the only way to get the
	// correct callstack when running a 64-bit executable. However, XAudio2 sometimes (?) don't like this and it may result in no sound.
#ifdef _WIN64
	if ( FParse::Param(CmdLine,TEXT("noinnerexception")) || FApp::IsBenchmarking() || bNoExceptionHandler)
	{
		GEnableInnerException = false;
	}
#endif	

	// When we're running embedded, assume that the outer application is going to be handling crash reporting
#if UE_BUILD_DEBUG
	if (GUELibraryOverrideSettings.bIsEmbedded || !GAlwaysReportCrash)
#else
	if (GUELibraryOverrideSettings.bIsEmbedded || bNoExceptionHandler || (bIsDebuggerPresent && !GAlwaysReportCrash))
#endif
	{
		//当附加了调试器以准确捕获崩溃时，不要使用异常处理。这不会检查我们是否是第一个实例!
		ErrorLevel = GuardedMain( CmdLine );
	}
	else
	{
		// 当程序崩溃时捕获异常并上报；
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
		__try
#endif
 		{
			GIsGuarded = 1;
			// 执行主程序
			ErrorLevel = GuardedMainWrapper( CmdLine );
			GIsGuarded = 0;
		}
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
		__except( FPlatformMisc::GetCrashHandlingType() == ECrashHandlingType::Default
				? ( GEnableInnerException ? EXCEPTION_EXECUTE_HANDLER : ReportCrash(GetExceptionInformation()) )
				: EXCEPTION_CONTINUE_SEARCH )	
		{
			// Crashed.
			ErrorLevel = 1;
			if(GError)
			{
				GError->HandleError();
			}
			LaunchStaticShutdownAfterError();
			FPlatformMallocCrash::Get().PrintPoolsUsage();
            // 结束程序
			FPlatformMisc::RequestExit( true );
		}
#endif
	}

	TRACE_BOOKMARK(TEXT("WinMain.Exit"));

	return ErrorLevel;
}

```
## 主程序
```
int32 GuardedMain( const TCHAR* CmdLine )
{
    // FTrackedActivity跟踪活动用于在半高层上可视化流程中正在发生的事情*
    // 这是非常有用的，当想要显示在线状态，或者如果运行时正在等待加载的东西*
    // 启用新控制台时，跟踪的活动显示在日志窗口底部跟踪活动可以在多个线程中创建/更新/销毁
	FTrackedActivity::GetEngineActivity().Update(TEXT("Starting"), FTrackedActivity::ELight::Yellow);

    // 这个类可以用来标记一个执行上下文，也就是head或Job，并允许我们稍后在调用堆栈中查询状态。它通常用于IsinRendering/GamethreadFunctions。
	FTaskTagScope Scope(ETaskTag::EGameThread);

#if !(UE_BUILD_SHIPPING)

	// 如果指定了“-waitforattach”或“-waitForDebugger”，则停止启动并等待调试器加载后再继续
	if (FParse::Param(CmdLine, TEXT("waitforattach")) || FParse::Param(CmdLine, TEXT("WaitForDebugger")))
	{
		while (!FPlatformMisc::IsDebuggerPresent())
		{
			FPlatformProcess::Sleep(0.1f);
		}
		UE_DEBUG_BREAK();
	}

#endif

    //创建了一个名为“DefaultMain”的书签，并记录了该书签的时间等信息。
	BootTimingPoint("DefaultMain");

	// 初始化广播;
	FCoreDelegates::GetPreMainInitDelegate().Broadcast();

	// 把引擎退出放在析构函数里，确保能够调用
	struct EngineLoopCleanupGuard 
	{ 
		~EngineLoopCleanupGuard()
		{
			// Don't shut down the engine on scope exit when we are running embedded
			// because the outer application will take care of that.
			if (!GUELibraryOverrideSettings.bIsEmbedded)
			{
				EngineExit();
			}
		}
	} CleanupGuard;

	// Set up minidump filename. We cannot do this directly inside main as we use an FString that requires 
	// destruction and main uses SEH.
	// These names will be updated as soon as the Filemanager is set up so we can write to the log file.
	// That will also use the user folder for installed builds so we don't write into program files or whatever.
#if PLATFORM_WINDOWS
	FCString::Strcpy(MiniDumpFilenameW, *FString::Printf(TEXT("unreal-v%i-%s.dmp"), FEngineVersion::Current().GetChangelist(), *FDateTime::Now().ToString()));
#endif

	FTrackedActivity::GetEngineActivity().Update(TEXT("Initializing"));
    // 预初始化引擎
	int32 ErrorLevel = EnginePreInit( CmdLine );

	// 退出如果与初始化失败
	if ( ErrorLevel != 0 || IsEngineExitRequested() )
	{
		return ErrorLevel;
	}

	{
		FScopedSlowTask SlowTask(100, NSLOCTEXT("EngineInit", "EngineInit_Loading", "Loading..."));

		// EnginePreInit leaves 20% unused in its slow task.
		// Here we consume 80% immediately so that the percentage value on the splash screen doesn't change from one slow task to the next.
		// (Note, we can't include the call to EnginePreInit in this ScopedSlowTask, because the engine isn't fully initialized at that point)
		SlowTask.EnterProgressFrame(80);

		SlowTask.EnterProgressFrame(20);

#if WITH_EDITOR
        // 如果是编辑器模式初始化
		if (GIsEditor)
		{
			ErrorLevel = EditorInit(GEngineLoop);
		}
		else
#endif
		{
            // 游戏模式初始化
			ErrorLevel = EngineInit();
		}
	}
    // 获取当前时间减去起始时间；
	double EngineInitializationTime = FPlatformTime::Seconds() - GStartTime;
	UE_LOG(LogLoad, Log, TEXT("(Engine Initialization) Total time: %.2f seconds"), EngineInitializationTime);

#if WITH_EDITOR
	UE_LOG(LogLoad, Log, TEXT("(Engine Initialization) Total Blueprint compile time: %.2f seconds"), BlueprintCompileAndLoadTimerData.GetTime());
#endif

	ACCUM_LOADTIME(TEXT("EngineInitialization"), EngineInitializationTime);

    //创建了一个名为“Tick loop starting”的书签，并记录了该书签的时间等信息。
	BootTimingPoint("Tick loop starting");
	DumpBootTiming();

	FTrackedActivity::GetEngineActivity().Update(TEXT("Ticking loop"), FTrackedActivity::ELight::Green);

	// 主循环
	if (!GUELibraryOverrideSettings.bIsEmbedded)
	{
		while( !IsEngineExitRequested() )
		{
			EngineTick();
		}
	}

    //创建了一个名为“Tick loop end”的书签，并记录了该书签的时间等信息。
	TRACE_BOOKMARK(TEXT("Tick loop end"));

#if WITH_EDITOR
    // 如果是编辑器模式退出调用；
	if( GIsEditor )
	{
		EditorExit();
	}
#endif
	return ErrorLevel;
}
```