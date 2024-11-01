#### Activity启动流程分析

##### 1，在ActivityThread的performLaunchActivity中，通过反射创建了对应的Activity实例，通过调用activity的attach方法，创建了对应的PhoneWindow，将其与Activity关联起来。
##### 2，在ActivityThread的handleResumeActivity中，通过wm.addView(decor, l) 将decor添加到window中，wm是WindowManagerImpl,最终会调用WindowManagerGolbal的addView，并将页面视图添加到PhoneWindow之上，也是因为这个原因，所以Activity的onResume回调函数中，并不能立即得到各个子View的实际宽高。




# Jetpack Lifecycle原理分析

##### 核心原理：通过在父Activity的onCreate()生命周期方法添加没有布局的Fragment,来监听Activity的生命周期函数变化。并在各个生命周期函数中通过dispatch()进行分发。

生命周期的五种状态及流转事件
	
	Destroyed    Initilized     Created        Started          Resumed
	
	                 ON_CREATE------>ON_START------->ON_RESUME
	                 
		<-------------ON_DESTROY------ON_STOP<--------ON_PAUSE
		
		以上为各种状态及生命周期事件
		
---
想要让LiveData不发送黏性事件，在它的considerNotify()函数里，需要想办法让它的lastVersion和mVersion相等，否则会使得代码往下继续执行发送数据。








