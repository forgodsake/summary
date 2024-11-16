	在使用安卓的Service进行跨进程通信时，我们一般的实践步骤是首先创建通信的aidl文件，make module 生成对应的java文件之后，在我们的service里面实现该java文件的stub类，完成实际业务逻辑，并在service类的onBind函数中将该stub返回。之后在对应的客户端进程里面，使用bindService后，在ServiceConnection的onServiceConnected回调里，通过Ixxxx.Proxy().asInterface(binder)得到对应的代理类，进行跨进程调用。
	那么系统是如何帮助我们完成的整个跨进程调用呢？

首先，在启动Activity时，在ActivityThread的performLaunchActivity函数里面会使用以下代码创建Context：
```
ContextImpl appContext = createBaseContextForActivity(r);
```

之后会使用反射创建的activity的activity.attach使得该Activity和Context进行绑定。
那么当追踪bindService函数调用，会发现调用到ContextWrapper的bindService函数，并在该函数转到mBase.bindService(service, conn, flags);这里的mBase就是上面的ContextImpl。

查看ContextImpl的bindService函数，有以下函数调用链：
```
ContextImpl  bindService ===>
			 bindServiceCommon ===>
			 //在这个函数里面，使用以下代码对ServiceConnection进行了包装：
			 IServiceConnection sd;
			 sd = mPackageInfo
			 .getServiceDispatcher(conn,...);
			 //这里的mPackageInfo是LoadedApk类的实例
			 //所以返回的ServiceDispatcher是LoadedApk
			 //内部的ServiceDispatcher.InnerConnection
			 //实现了Stub，方便进行跨进程调用
AMS          ActivityManager.getService().bindIsolatedService
			 //进行启动服务进程等一系列系统函数
ServiceDispatcher.InnerConnection
			 connected	
ServiceDispatcher
			 connected
			 doConnected
			 //在这里，最开始的ServiceConnection收到了已连接的绑定
			 mConnection.onServiceConnected(name, service);
			 
			 
```
