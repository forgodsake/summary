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
			 //紧接着调用了AMS的bindIsolatedService
			 //将创建服务及绑定的操作交给AMS
AMS          ActivityManager.getService().bindIsolatedService
			 //进行启动服务进程等一系列框架层调用
ActiveService
			 bindServiceLocked
			 bringUpServiceLocked
			 realStartServiceLocked
ApplicationThread
			 scheduleCreateService
ActiveService
			 requestServiceBindingsLocked
ApplicationThread
			 scheduleBindService
			 handleBindService
ActivityManagerService
			 publishService
ActiveService
			 publishServiceLocked
ServiceDispatcher.InnerConnection
			 connected	
ServiceDispatcher
			 connected
			 doConnected
			 //在这里，最开始的ServiceConnection收到了已连接的绑定
			 mConnection.onServiceConnected(name, service);
			 
			 
```

ONEWAY的概念
默认情况下，跨进程调用是需要等待对端返回的，所以也就是阻塞调用。对于系统进程而言（以及少部分三方接口），设计时是不需要等待对端返回，所以使用ONEWAY标记表示该aidl函数是单向的，执行完无需等待，继续执行接下来的代码。

in\out的概念
AIDL中的定向 tag 表示了在跨进程通信中数据的流向，其中 in 表示数据由客户端流向服务端， out 表示数据由服务端流向客户端，而 inout 则表示数据可在服务端与客户端之间双向流通。
（注意：针对in\out,客户端观察的应该是对象本身，而不是return返回的值）
