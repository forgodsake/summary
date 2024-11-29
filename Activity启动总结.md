#### Activity启动流程分析

通过startActivity启动Activity或者说某个App时，会经过以下调用链：
```
Activity startActivityForResult(intent, -1)
Instrumentation  execStartActivity(...)
				 ActivityTaskManager
				 .getService()
				 .startActivity(...)
ActivityTaskManagerService 
				 startActivity(...)
				 // 这里传入了UserId 安卓支持多用户
				 startActivityAsUser(...)
				 // 通过ActivityStartController 
				 // 获取ActivityStarter来启动Activity
				 getActivityStartController()
				 .obtainStarter(intent, "startActivityAsUser")
                 .setCaller(caller)
                 .setCallingPackage(callingPackage)
                 .setCallingFeatureId(callingFeatureId)
                 .setResolvedType(resolvedType)
                 .setResultTo(resultTo)
                 .setResultWho(resultWho)
                 .setRequestCode(requestCode)
                 .setStartFlags(startFlags)
                 .setProfilerInfo(profilerInfo)
                 .setActivityOptions(bOptions)
                 .setUserId(userId)
                 .execute();
                
ActivityStarter //在此做了大量工作，生成对应的ActivityRecord
				executeRequest(mRequest);
				startActivityUnchecked(...);
				//启动Activity，并计算Activity是否应该添加到
				//现有Task或执行顶部Activity的onNewIntent
				startActivityInner(...);
				mRWC.resumeFocusedStacksTopActivities(...)
RootWindowContainer 
				focusedStack
				.resumeTopActivityUncheckedLocked(...)
				
ActivityStack   resumeTopActivityInnerLocked(...)
				mStackSupervisor
				.startSpecificActivity(next, true, true);
ActivityStackSupervisor 
				startSpecificActivity(...)
				//在此判断该进程是否已经启动
				//已启动的执行 realStartActivityLocked 
				//未启动的执行 mService.startProcessAsync
				realStartActivityLocked(...)
				// 在此创建了要启动的Activity的
				// LaunchActivityItem 以及
				// ResumeActivityItem
				// 交给 clientTransaction
				// 最终在activityThread里面得以执行
				mService
	           .getLifecycleManager()
	           .scheduleTransaction(clientTransaction);
				
ClientLifecycleManager	
				scheduleTransaction(...)			
				transaction.schedule();
ClientTransaction  
				//此处的client为IApplicationThread，
				//所以这里进行了跨进程调用
				mClient.scheduleTransaction(this);
ApplicationThread scheduleTransaction(...)
ClientTransactionHandler 
				scheduleTransaction(...)

    void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
ActivityThread H handleMessage(...)
				transaction = (ClientTransaction) msg.obj;
                mTransactionExecutor.execute(transaction);
TransactionExecutor		
				// 通过以下两个函数执行完
				// 从当前生命周期到目标生命周期的整个流程
				// 执行LaunchActivityItem、
				// ResumeActivityItem等等的excute函数
				// 并通过cycleToPath()函数执行执行完所有中间状态	
				executeCallbacks(transaction);
                executeLifecycleState(transaction);

ActivityThread  handleLaunchActivity(...)
				handleStartActivity(...)
				handleResumeActivity(...)
```

对于未创建进程的，会执行以下函数调用链：
```
ActivityManagerInternal::startProcess
ActivityManagerService.LocalService 
				startProcess(...)
				startProcessLocked(...)
				mProcessList.startProcessLocked(...)
ProcessList     startProcessLocked(...)
				startProcess(...)
				appZygote.getProcess().start(...)
Process         start(...)
				ZYGOTE_PROCESS.start(...)
ZygoteProcess   start(...)
				startViaZygote(...)
				zygoteSendArgsAndGetResult(...)
				attemptZygoteSendArgsAndGetResult(...)
ZygoteServer    runSelectLoop(...)
ZygoteConnection processOneCommand(...)
				Zygote.forkAndSpecialize(...)
				handleChildProc(...)
ZygoteInit      zygoteInit(...)
				RuntimeInit.commonInit();
		        ZygoteInit.nativeZygoteInit();
RuntimeInit     applicationInit(...)	
ZygoteInit      main(...)
				caller = ZygoteServer.runSelectLoop(...)
				// caller是一个runnable 
				// 封装了ActivityThread的main函数
				caller.run(...) 
ActivityThread  main(...)
AMS             attachApplication(...)
				// 执行创建Application、ContentProvider,
				// 调用App的attachBaseContext、onCreate
				thread.bindApplication
				// 将thread交给ProcessRecord保管
				app.makeActive(thread, mProcessStats);
				// 添加app到ProcessList的LRU列表
				mProcessList.updateLruProcessLocked(app, false, null);
ATMS            attachApplication(...)
RootWindowContainer attachApplication(...)
				startActivityForAttachedApplicationIfNeeded(...)
				// 到这里会再次进入之前的启动流程 此时Process已经创建，
				// 开始走接下来的流程
				mStackSupervisor.realStartActivityLocked(...)
```


1，在ActivityThread的performLaunchActivity中，通过反射创建了对应的Activity实例，通过调用activity的attach方法，创建了对应的PhoneWindow，将其与Activity关联起来。

2，在ActivityThread的handleResumeActivity中，通过wm.addView(decor, l) 将decor添加到window中，wm是WindowManagerImpl,最终会调用WindowManagerGolbal的addView，并将页面视图添加到PhoneWindow之上，也是因为这个原因，所以Activity的onResume回调函数中，并不能立即得到各个子View的实际宽高。









