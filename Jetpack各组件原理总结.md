
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

 LiveData的observe函数会将observer对象包装为LifecycleBoundObserver。该类的构造方法接收owner和observer，从而将观察者和宿主绑定到一起。同时observe函数会执行owner.getLifecycle().addObserver(wrapper);将包装类添加到activity的lifecycle的观察者里面。从而在生命周期活跃时通过 
 ```
 onStateChanged(source,event) ==> 
 activeStateChanged ==>
 dispatchingValue(this) ==> 
 considerNotify(initiator); => 
 observer.mObserver.onChanged((T) mData); 
```
 调用链来通知观察者更新数据。
 
 在调用set和post函数时，会通过
 ```
 dispatchingValue(null); ==> 
 considerNotify(iterator.next().getValue()); ==> 
 observer.mObserver.onChanged((T) mData); 
```
 调用链来通知观察者更新数据。


# DataBinding原理分析

Databinding框架会把databinding类型的xml布局文件进行拆分，生成两个xml，一个正常的布局xml供我们的布局解析器正常解析加载，另一份根布局为Layout包含Variables和Targets的，仅供databinding框架自己使用，其中Variables部分为原始layout的data部分声明的变量，Targets中则包含根布局和使用绑定的View，并为它们生成特定的tag（根布局为layout/layout文件名_0,View为binding_xx(xx为数字)），这样即使不需要id也可以进行赋值操作。

当我们调用DataBindingUtil.setContentView时，除了将页面布局设置给activity，还会通过以下函数调用链将用到动态数据的View和对应数据绑定起来(在生成的binding类的构造方法中通过调用mapBindings函数)。
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

# ViewModel实现原理

ViewModel之所以能够在Activity发生旋转等配置项变化时保留其中数据不被清理，是因为ComponentAcitivty实现了ViewModelStoreOwner接口，而在实现函数getViewModelStore中，生成了一个ViewModelStore来保存页面的ViewModel，这个ViewModelStore又保存在了页面的NonConfiguratioInstances当中。

```

@Override
    public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException
            ("Your activity is not yet attached to the "
            + "Application instance. You can't request "
            + "ViewModel before onCreate call.");
        }
        ensureViewModelStore();
        return mViewModelStore;
    }

    @SuppressWarnings("WeakerAccess") /* synthetic access */
    void ensureViewModelStore() {
        if (mViewModelStore == null) {
            NonConfigurationInstances nc =
	        getLastNonConfigurationInstance();
            if (nc != null) {
                // Restore the ViewModelStore from 
                // NonConfigurationInstances
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
    }

```


这个NonConfigurationInstances又是从何而来的呢，通过方法跟踪，可找到其在Activity的attach方法中通过方法参数传入。Activity的attach方法是在Activity的加载流程中由ActivityThread的performLaunchActivity调用的，调用时传入的是ActivityRecordClient中的NonConfigurationInstances对象，那ActivityRecordClient又是在什么时候保存的NonConfigurationInstances对象的呢，这就要从Activity因为配置变化被销毁时查起了。

*** 在TransactionExecutor内部执行handleLaunchActivity之前，会通过以下代码获取ActivityRecordClient,而getActivityClient的实现在ActivityThread中，是根据binder从
ArrayMap当中获取。

```
final IBinder token = transaction.getActivityToken();
ActivityClientRecord r = mTransactionHandler.getActivityClient(token);
```

当Activity因为配置变化被销毁时，在其销毁流程中ActivityThread会调用performDestroyActivity方法，该方法内部会回调Activity的retainNonConfigurationInstances方法获取NonConfigurationInstances并保存在ActivityRecordClient中以备之后Activity重建之需。
该函数内部又调用了onRetainNonConfigurationInstance()方法，而该方法由ComponentActivity进行了覆写。

```
public final Object onRetainNonConfigurationInstance() {
        // Maintain backward compatibility.
        Object custom = onRetainCustomNonConfigurationInstance();

        ViewModelStore viewModelStore = mViewModelStore;
        if (viewModelStore == null) {
            // No one called getViewModelStore(), so see if there was an existing
            // ViewModelStore from our last NonConfigurationInstance
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                viewModelStore = nc.viewModelStore;
            }
        }

        if (viewModelStore == null && custom == null) {
            return null;
        }

        NonConfigurationInstances nci = new NonConfigurationInstances();
        nci.custom = custom;
        nci.viewModelStore = viewModelStore;
        return nci;
    }

```

onRetainNonConfigurationInstances方法的主要逻辑就是创建了一个NonConfigurationInstances对象(此NonConfigurationInstances类与前头的NonConfigurationInstances类不是同一个类)，并将当前Activity的ViewModelStore保存到了所创建的对象的viewModelStore变量中，从而使得Activity在销毁后重建时能获取到销毁前的ViewModelStore，进而可获取到销毁前的ViewModel。onRetainNoConfigurationInstance方法返回的NonConfigurationInstance对象最终被存储到了retainNonConfigurationInstances方法中创建的NonConfigurationInstances对象的activity变量里。

```

NonConfigurationInstances retainNonConfigurationInstances() {
        Object activity = onRetainNonConfigurationInstance();
        HashMap<String, Object> children = onRetainNonConfigurationChildInstances();
        FragmentManagerNonConfig fragments = mFragments.retainNestedNonConfig();

        // We're already stopped but we've been asked to retain.
        // Our fragments are taken care of but we need to mark the loaders for retention.
        // In order to do this correctly we need to restart the loaders first before
        // handing them off to the next activity.
        mFragments.doLoaderStart();
        mFragments.doLoaderStop(true);
        ArrayMap<String, LoaderManager> loaders = mFragments.retainLoaderNonConfig();

        if (activity == null && children == null && fragments == null && loaders == null
                && mVoiceInteractor == null) {
            return null;
        }

        NonConfigurationInstances nci = new NonConfigurationInstances();
        nci.activity = activity;
        nci.children = children;
        nci.fragments = fragments;
        nci.loaders = loaders;
        if (mVoiceInteractor != null) {
            mVoiceInteractor.retainInstance();
            nci.voiceInteractor = mVoiceInteractor;
        }
        return nci;
    }

```

需要注意的是，如果Activity是正常的销毁，那么ViewModelStore会清空其保存的所有ViewModel，而如果是因为配置变化而被销毁，则不清空，这个逻辑可由ComponentActivity的构造函数中觅得：

```
getLifecycle().addObserver(new LifecycleEventObserver() {
            @Override
            public void onStateChanged(@NonNull LifecycleOwner source,
                    @NonNull Lifecycle.Event event) {
                if (event == Lifecycle.Event.ON_DESTROY) {
                    // Clear out the available context
                    mContextAwareHelper.clearAvailableContext();
                    // And clear the ViewModelStore
                    if (!isChangingConfigurations()) {
                        getViewModelStore().clear();
                    }
                }
            }
        });

```

Room使用总结

使用@Entity\@Dao\@Database 三个注解作为类的注解分别进行数据层、操作层、数据库实例层的编写。

Entity层
在字段上使用@ColumnInfo 可以对字段设置数据库内的别名，使用@Primarykey设置为主键。


Dao层
Dao层一般会在对应的Dao接口中使用@Insert、@Delete、@Update、@Query 四种注解用于函数上来进行数据表的增删改查操作。

DataBase层
@Database注解上需要entities参数设置数据表对应的实体类数组；version参数设置数据库版本号，当数据表发生变动时，需要更改版本号，并在构建Database时增加migration函数进行升级操作来更改旧版数据；exportSchema参数来设定是否将数据库的修改相关sql语句导出，方便直接拿来使用。

一般数据库类会使用静态单列函数来创建全局可用的数据库实例，方便进行直接获取。
需要在Database类里面声明abstract类型的Dao实例，Room框架会默认生成对应的Impl实现类。

