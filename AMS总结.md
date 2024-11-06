#### Activity启动流程分析

通过startActivity启动Activity或者说某个App时，会经过以下调用链：
```
Activity startActivityForResult(intent, -1)
Instrumentation  execStartActivity(...)
				 ActivityTaskManager.getService().startActivity(...)
ActivityTaskManagerService startActivity(...)
				 startActivityAsUser(...)
getActivityStartController().obtainStarter(intent, "startActivityAsUser")
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
                
                //在此做了大量工作，生成对应的ActivityRecord
				executeRequest(mRequest);
				startActivityUnchecked(...);
				//启动Activity，并计算Activity是否应该添加到现有Task或执行顶部
				//Activity的onNewIntent
				startActivityInner(...);
				mRootWindowContainer.resumeFocusedStacksTopActivities(
                        mTargetStack, mStartActivity, mOptions);
RootWindowContainer resumeFocusedStacksTopActivities(...)
				focusedStack.resumeTopActivityUncheckedLocked(...)
				
ActivityStack   resumeTopActivityInnerLocked(...)
				mStackSupervisor.startSpecificActivity(next, true, true);
ActivityStackSupervisor 
				//在此判断该进程是否已经启动 已启动的执行realStartActivityLocked
				//未启动的执行 mService.startProcessAsync
				startSpecificActivity(...)
				realStartActivityLocked(...)
				mService.getLifecycleManager().scheduleTransaction(clientTransaction);
				
ClientLifecycleManager	scheduleTransaction(...)			
				  transaction.schedule();
ClientTransaction  
				//此处的client为IApplicationThread，所以这里进行了跨进程调用
				mClient.scheduleTransaction(this);
ActivityThread.ApplicationThread scheduleTransaction
ClientTransactionHandler scheduleTransaction

    void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
ActivityThread H handleMessage(...)
				final ClientTransaction transaction = (ClientTransaction) msg.obj;
                mTransactionExecutor.execute(transaction);
TransactionExecutor		
				// 通过以下两个函数执行完从当前生命周期到目标生命周期的整个流程
				// 执行LaunchActivityItem、ResumeActivityItem等等的excute函数
				// 并通过cycleToPath函数执行执行完所以中间状态	
				executeCallbacks(transaction);
                executeLifecycleState(transaction);

ActivityThread  handleLaunchActivity(...)
				handleStartActivity(...)
				handleResumeActivity(...)
```


1，在ActivityThread的performLaunchActivity中，通过反射创建了对应的Activity实例，通过调用activity的attach方法，创建了对应的PhoneWindow，将其与Activity关联起来。

2，在ActivityThread的handleResumeActivity中，通过wm.addView(decor, l) 将decor添加到window中，wm是WindowManagerImpl,最终会调用WindowManagerGolbal的addView，并将页面视图添加到PhoneWindow之上，也是因为这个原因，所以Activity的onResume回调函数中，并不能立即得到各个子View的实际宽高。









