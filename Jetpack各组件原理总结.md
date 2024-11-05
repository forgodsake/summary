
# Lifecycle原理分析

##### 核心原理：通过在父Activity的onCreate()生命周期方法添加没有布局的Fragment,来监听Activity的生命周期函数变化。并在各个生命周期函数中通过dispatch()进行分发。

生命周期的五种状态及流转事件
	
	Destroyed    Initilized     Created        Started          Resumed
	
	                 ON_CREATE------>ON_START------->ON_RESUME
	                 
		<-------------ON_DESTROY------ON_STOP<--------ON_PAUSE
		
		以上为各种状态及生命周期事件
		
---
想要让LiveData不发送黏性事件，在它的considerNotify()函数里，需要想办法让它的lastVersion和mVersion相等，否则会使得代码往下继续执行发送数据。

# LiveData原理分析 

 LiveData的observe函数会将observer对象包装为LifecycleBoundObserver。该类的构造方法接收owner和observer，从而将观察者和宿主绑定到一起，同时将包装类添加到activity的lifecycle的观察者里面。从而在生命周期活跃时通过 
 onStateChanged(source,event) ==> 
 activeStateChanged ==>
 dispatchingValue(this) ==> 
 considerNotify(initiator); => 
 observer.mObserver.onChanged((T) mData); 调用链来通知变化。
 在调用set和post函数时，会通过
 dispatchingValue(null); ==> 
 considerNotify(iterator.next().getValue()); ==> 
 observer.mObserver.onChanged((T) mData); 调用链来通知变化。
