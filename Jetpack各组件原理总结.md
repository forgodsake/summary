
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
 ```
 onStateChanged(source,event) ==> 
 activeStateChanged ==>
 dispatchingValue(this) ==> 
 considerNotify(initiator); => 
 observer.mObserver.onChanged((T) mData); 
```
 调用链来通知变化。
 
 在调用set和post函数时，会通过
 ```
 dispatchingValue(null); ==> 
 considerNotify(iterator.next().getValue()); ==> 
 observer.mObserver.onChanged((T) mData); 
```
 调用链来通知变化。


# DataBinding原理分析

Databinding框架会把databinding类型的xml布局文件进行拆分，生成两个xml，一个正常的布局xml供我们的布局解析器正常解析加载，另一份类型为Layout包含Variables和Targets的是供databinding框架自己使用，其中变量部分为声明的变量，目标则包含根布局和使用绑定的View，并为它们生成特定的tag（根布局为layout/layout文件名_0,View为binding_xx(xx为数字)），这样即使不需要id也可以进行赋值操作。

当我们调用DataBindingUtil.setContentView时，除了将页面布局设置给activity，还会通过以下函数调用链将用到动态数据的View和对应数据绑定起来。
```
DataBindingUtil.setContentView(this,R.layout.activity_another) ==>
setContentView(activity, layoutId, sDefaultComponent) ==>
bindToAddedViews(bindingComponent, contentView, 0, layoutId) ==>
bind(component, children, layoutId) ==>
sMapper.getDataBinder(bindingComponent, roots, layoutId) ==>
DataBinderMapperImpl.getDataBinder 
return new ActivityAnotherBindingImpl(component, view)
```

在ActivityAnotherBindingImpl的构造方法中，会生成绑定视图的Object[]数组，同时在其父类
ViewDataBinding的静态代码块中通过以下代码
```
ROOT_REATTACHED_LISTENER = new View.OnAttachStateChangeListener() {
                @TargetApi(19)
                public void onViewAttachedToWindow(View v) {
                    ViewDataBinding binding = ViewDataBinding.getBinding(v);
                    binding.mRebindRunnable.run();
                    v.removeOnAttachStateChangeListener(this);
                }

                public void onViewDetachedFromWindow(View v) {
                }
            };

```
注册了视图状态监听，当视图附着到窗口时，会通过binding.mRebindRunnable.run()调用到子类
ActivityAnotherBindingImpl的executeBindings()函数，最终进行View数据的设置和双向绑定时 
视图绑定对象更新监听器的设置。

当我们设置数据时，再次通过ActivityAnotherBindingImpl的super.requestRebind经过层次调用再次调用到executeBindings()函数，实现对数据的更新。