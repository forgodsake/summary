	在使用安卓的Service进行跨进程通信时，我们一般的实践步骤是首先创建通信的aidl文件，make module 生成对应的java文件之后，在我们的service里面实现该java文件的stub类，完成实际业务逻辑，并在service类的onBind函数中将该stub返回。之后在对应的客户端进程里面，使用bindService后，在ServiceConnection的onServiceConnected回调里，通过Ixxxx.Stub().asInterface(binder)得到对应的代理类，进行跨进程调用。
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
			 // 在bringUp函数的最后
			 // 会进行判断，如果进程不存在
			 // 会执行开启进程的代码
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
（注意：针对in\out,客户端观察的应该是传参对象本身，而不是return返回的值）

##### BinderProxy!!!
	在我们使用Service配合aidl进行自定义服务的开发时，因为系统的良好设计，甚至会觉得跟平时在App内部主进程操作函数没有太大区别，以至于对于具体如何跨进程调用基本无感。其实这是因为系统和编译器帮我们做了太多事情。
	当我们查看定义好的aidl文件经过编译生成的java类里面的Stub或者Proxy。会发现它们要么继承自Binder，要么持有Binder的引用。所以可以说是Binder在帮我们进行跨进程的调用。
	在Binder通信过程中，客户端涉及到传递binder的，在java层都会被系统转化为BinderProxy，而这，也是aidl自动生成的java文件中，Proxy类持有的mRemote的实际java类型。
	已知bindService过程中，通过Ixxxx.Stub().asInterface(binder)得到对应的Proxy对象。而我们的实际函数调用，都是通过Proxy内部的mRemote使用transact函数传递不同参数来实现的。那么，这个BinderProxy是在哪里生成的呢？
	追踪整个bindService流程，会发现在handleBindService时，有如下代码：

```
// 这里手动调用创建的Service的onBind函数
IBinder binder = s.onBind(data.intent);
// 告诉SystemServer该服务已准备就绪
ActivityManager.getService().publishService(
data.token, data.intent, binder);
```

这里调用了s.onBind(data.intent)函数，熟悉自定义Service的都明白，这个函数返回的正是我们实现的完整Stub类，而接下来的ActivityManager.getService().publishService正好进行了跨进程操作，该函数传递的最后一个参数正是我们的stub。继续查看ActivityManager.getService()，发现该函数返回的是IActivityManager.aidl生成的Proxy类：

```
final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
final IActivityManager am = IActivityManager.Stub.asInterface(b);
```

如果尝试自己编译该系统源码的aidl文件，会发现生成的java文件中包含以下代码：

```
  public void publishService(IBinder token, Intent intent, IBinder service) throws RemoteException {
    Parcel _data = Parcel.obtain();
    Parcel _reply = Parcel.obtain();
    try {
      _data.writeInterfaceToken("android.app.IActivityManager");
      _data.writeStrongBinder(token);
      if (intent != null) {
        _data.writeInt(1);
        intent.writeToParcel(_data, 0);
      } else {
        _data.writeInt(0);
      }
      // IBinder对象是通过writeStrongBinder方法写入的
      // 注意这个IBinder就是Service实现的onBind方法中返回的, 
      // 就是Ixxx.Stub类型
      _data.writeStrongBinder(service);
      boolean _status = this.mRemote.transact(32, _data, _reply, 0);
      if (!_status && IActivityManager.Stub.getDefaultImpl() != null) {
        IActivityManager.Stub.getDefaultImpl().publishService(token, intent, service);
        return;
      } 
      _reply.readException();
    } finally {
      _reply.recycle();
      _data.recycle();
    } 
  }

```

可以看到，通过_data.writeStrongBinder将我们的stub写入到了Parcel对象，紧接着执行了mRemote.transact来进行跨进程调用。这里的_data.writeStrongBinder最终到native层。

```
parcel->writeStrongBinder(ibinderForJavaObject(env, object))
//ibinderForJavaObject(env, object)返回JavaBBinder对象
sp<IBinder> ibinderForJavaObject(JNIEnv* env, jobject obj)
{
    if (obj == NULL) return NULL;

    // Instance of Binder?
    if (env->IsInstanceOf(obj, gBinderOffsets.mClass)) {
        JavaBBinderHolder* jbh = (JavaBBinderHolder*)
            env->GetLongField(obj, gBinderOffsets.mObject);
        // 、返回一个IBinder
        return jbh->get(env, obj);
    }

    // Instance of BinderProxy?
    if (env->IsInstanceOf(obj, gBinderProxyOffsets.mClass)) {
        return getBPNativeData(env, obj)->mObject;
    }

    ALOGW("ibinderForJavaObject: %p is not a Binder object", obj);
    return NULL;
}

sp<JavaBBinder> get(JNIEnv* env, jobject obj)
{
    AutoMutex _l(mLock);
    sp<JavaBBinder> b = mBinder.promote();
    if (b == NULL) {
        //  创建JavaBBinder
        b = new JavaBBinder(env, obj);
        if (mVintf) {
            ::android::internal::Stability::markVintf(b.get());
        }
        if (mExtension != nullptr) {
            b.get()->setExtension(mExtension);
        }
        mBinder = b;
        ...
    }
    // 直接返回的JavaBBinder,说明JavaBBinder继承了IBinder
    return b;
}

flattenBinder(val)
// 由于这里的val是JavaBBinder类型。继承自BBinder，
// 所以判断localBinder不为空，走以下流程：
status_t Parcel::flattenBinder(const sp<IBinder>& binder)
{
    flat_binder_object obj;
    // ......
    if (binder != nullptr) {
        // 是local还是remote？
        BBinder *local = binder->localBinder();
        if (!local) {
            BpBinder *proxy = binder->remoteBinder();
            if (proxy == nullptr) {
                ALOGE("null proxy");
            }
            const int32_t handle = proxy ? proxy->handle() : 0;
            obj.hdr.type = BINDER_TYPE_HANDLE;
            obj.binder = 0; 
            obj.handle = handle;
            obj.cookie = 0;
        } else {
            // 进入！
            if (local->isRequestingSid()) {
                obj.flags |= FLAT_BINDER_FLAG_TXN_SECURITY_CTX;
            }
            // 注意这里的type，是BINDER_TYPE_BINDER！
            obj.hdr.type = BINDER_TYPE_BINDER;
            // 保存BBinder的弱引用，这个是干啥用的?
            obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
            // 保存BBinder对象
            obj.cookie = reinterpret_cast<uintptr_t>(local);
        }
    } else {
        obj.hdr.type = BINDER_TYPE_BINDER;
        obj.binder = 0;
        obj.cookie = 0;
    }

    // 完成IBinder的写入
    return finishFlattenBinder(binder, obj);
}

finishFlattenBinder(binder, obj)
status_t Parcel::finishUnflattenBinder(
    const sp<IBinder>& binder, sp<IBinder>* out) const
{
    int32_t stability;
    status_t status = readInt32(&stability);
    if (status != OK) return status;

    status = internal::Stability::set(binder.get(), stability, true /*log*/);
    if (status != OK) return status;

    // out指向这个内存区域
    *out = binder;
    return OK;
}

// 经过以上操作，将IBinder保存到内存中的某个特定区域
```

接着到了AMS的接收部分,通过看编译后生成的IActivityManager.Stub.Class文件中对应的publishService方法：

```
public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        // ......
        data.enforceInterface(descriptor);
        // 读取IBinder对象，不过这个是token, 不是我们想要的Service的IBinder
        // 用过Binder通信知道，是按顺序来读写的，所以我们看最后一个IBinder
        iBinder11 = data.readStrongBinder();
        if (0 != data.readInt()) {
          intent6 = (Intent)Intent.CREATOR.createFromParcel(data);
        } else {
          intent6 = null;
        } 
        // 这里就是我们想要的Service的IBinder了
        iBinder26 = data.readStrongBinder();
        publishService(iBinder11, intent6, iBinder26);
        reply.writeNoException();
        return true;
        // ......
    }
```

这里的data.readStrongBinder()就是我们的binder了，依然是native函数调用。

```
static jobject android_os_Parcel_readStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        // Parcel->readStrongBinder
        // 将IBinder转换成jobject
        return javaObjectForIBinder(env, parcel->readStrongBinder());
    }
    return NULL;
}

sp<IBinder> Parcel::readStrongBinder() const
{
    sp<IBinder> val;
    // 暂时不明确IBinder指代的是具体哪个子类吼
    readNullableStrongBinder(&val);
    return val;
}

status_t Parcel::readNullableStrongBinder(sp<IBinder>* val) const
{
    // 来了，解压的过程
    return unflattenBinder(val);
}

status_t Parcel::unflattenBinder(sp<IBinder>* out) const
{
    // 从内存区域中读取当前位置的数据
    const flat_binder_object* flat = readObject(false);
    // 存在
    if (flat) {
        switch (flat->hdr.type) {
            
            case BINDER_TYPE_BINDER: {
                sp<IBinder> binder = reinterpret_cast<IBinder*>(flat->cookie);
                return finishUnflattenBinder(binder, out);
            }
            case BINDER_TYPE_HANDLE: {
                sp<IBinder> binder =
                    ProcessState::self()->getStrongProxyForHandle(flat->handle);
                return finishUnflattenBinder(binder, out);
            }
        }
    }
    return BAD_TYPE;
}

status_t Parcel::finishUnflattenBinder(
    const sp<IBinder>& binder, sp<IBinder>* out) const
{
    int32_t stability;
    status_t status = readInt32(&stability);
    if (status != OK) return status;

    status = internal::Stability::set(binder.get(), stability, true /*log*/);
    if (status != OK) return status;

    // out指向这个内存区域
    *out = binder;
    return OK;
}

jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
{
    // ......
    // 这里决定是不是直接返回JavaBBinder的类型
    if (val->checkSubclass(&gBinderOffsets)) {
        // It's a JavaBBinder created by ibinderForJavaObject. 
        // Already has Java object.
        jobject object = static_cast<JavaBBinder*>(val.get())->object();
        LOGDEATH("objectForBinder %p: it's our own %p!\n", val.get(), object);
        return object;
    }

    // 终于看到和BinderProxy相关的了, 至少名字上看都有关系
    BinderProxyNativeData* nativeData = new BinderProxyNativeData();
    nativeData->mOrgue = new DeathRecipientList;
    // 将从内存中读取的IBinder对象存起来
    nativeData->mObject = val;

    // 调用到java方法mGetInstance，生成BinderProxy！
    jobject object = env->CallStaticObjectMethod(gBinderProxyOffsets.mClass,
            gBinderProxyOffsets.mGetInstance, (jlong) nativeData, (jlong) val.get());
    // ......
    // 所以这里最终返回是BinderProxy对象对应JNI的jobject.
    return object;
}

```

以上的代码判断val->checkSubclass，如下：

```
bool IBinder::checkSubclass(const void* /*subclassID*/) const
{
    return false;
}

// JavaBBinder:
bool    checkSubclass(const void* subclassID) const
{
    //  gBinderOffsets的初始化
    return subclassID == &gBinderOffsets;
}

```

由于内核传递时会将BINDER_TYPE_BINDER转为BINDER_TYPE_HANDLE类型，所以在上面unflattenBinder函数中，得到的是BpBinder类型的IBinder，又由于BpBinder没有实现checkSubclass函数，调用的是父类IBinder的checkSubclass，返回false，代码往下走。其中gBinderOffsets是在进程的初始化注册jni函数时定义的：

```
static int int_register_android_os_Binder(JNIEnv* env)
{
    jclass clazz = FindClassOrDie(env, kBinderPathName);

    gBinderOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    gBinderOffsets.mExecTransact = GetMethodIDOrDie(env, clazz, "execTransact", "(IJJI)Z");
    gBinderOffsets.mGetInterfaceDescriptor = GetMethodIDOrDie(env, clazz, "getInterfaceDescriptor",
        "()Ljava/lang/String;");
    gBinderOffsets.mObject = GetFieldIDOrDie(env, clazz, "mObject", "J");

    return RegisterMethodsOrDie(
        env, kBinderPathName,
        gBinderMethods, NELEM(gBinderMethods));
}

```

所以javaObjectForIBinder()函数最终返回的是BinderProxy类型对象。而此时，我们还在AMS当中（SystemServer进程）。最终返回Client端，还需要进行一次传输。这一次与以上调用过程略有不同,主要体现在writeStrongBinder时，ibinderForJavaObject返回的是一个BpBinder类型对象。导致最终写入的obj.hdr.type = BINDER_TYPE_HANDLE：

```
sp<IBinder> ibinderForJavaObject(JNIEnv* env, jobject obj)
{
    if (obj == NULL) return NULL;

    // 是否为Binder对象
    if (env->IsInstanceOf(obj, gBinderOffsets.mClass)) {
        JavaBBinderHolder* jbh = (JavaBBinderHolder*)
            env->GetLongField(obj, gBinderOffsets.mObject);
        return jbh->get(env, obj);
    }

    // 是否为BinderProxy对象
    if (env->IsInstanceOf(obj, gBinderProxyOffsets.mClass)) {
        // SystemServer向Client App发送Server App的BinderProxy对象，走这里
        return getBPNativeData(env, obj)->mObject;
    }

    ALOGW("ibinderForJavaObject: %p is not a Binder object", obj);
    return NULL;
}

status_t Parcel::flattenBinder(const sp<IBinder>& binder)
    // ......
        // 注意此时我们身处SystemServer进程，传入的IBinder实际上是对应BinderProxy
        BBinder *local = binder->localBinder();
        if (!local) {
            BpBinder *proxy = binder->remoteBinder();
            if (proxy == nullptr) {
                ALOGE("null proxy");
            }
            const int32_t handle = proxy ? proxy->handle() : 0;
            obj.hdr.type = BINDER_TYPE_HANDLE;
            obj.binder = 0; /* Don't pass uninitialized stack data to a remote process */
            obj.handle = handle;
            obj.cookie = 0;
        } else {
            if (local->isRequestingSid()) {
                obj.flags |= FLAT_BINDER_FLAG_TXN_SECURITY_CTX;
            }
            obj.hdr.type = BINDER_TYPE_BINDER;
            obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
            obj.cookie = reinterpret_cast<uintptr_t>(local);
        }
    // ......
}

```

在内核传输中，当传输的是BINDER_TYPE_BINDER的话，会转换成BINDER_TYPE_HANDLE：
```
static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply,
			       binder_size_t extra_buffers_size)
{
    // ......
    for (buffer_offset = off_start_offset; buffer_offset < off_end_offset;
            buffer_offset += sizeof(binder_size_t)) {
        // ......
        switch (hdr->type) {
            case BINDER_TYPE_BINDER:
            case BINDER_TYPE_WEAK_BINDER: {
                struct flat_binder_object *fp;

                fp = to_flat_binder_object(hdr);
                ret = binder_translate_binder(fp, t, thread);
                // ......
        }
    // ......
}

static int binder_translate_binder(struct flat_binder_object *fp,
				   struct binder_transaction *t,
				   struct binder_thread *thread)
{
    // ......
    if (fp->hdr.type == BINDER_TYPE_BINDER)
        // 这里，如果读取的hdr.type是BINDER_TYPE_BINDER
        // 将会被改成BINDER_TYPE_HANDLE！！！
        fp->hdr.type = BINDER_TYPE_HANDLE;
    else
        fp->hdr.type = BINDER_TYPE_WEAK_HANDLE;
    // ......
}

```

所以上面AMS解包时，实际走的是BINDER_TYPE_HANDLE的路线：
```
status_t Parcel::unflattenBinder(sp<IBinder>* out) const
{
    // 从内存区域中读取当前位置的数据
    const flat_binder_object* flat = readObject(false);
    // 存在
    if (flat) {
        switch (flat->hdr.type) {
            case BINDER_TYPE_BINDER: {
                sp<IBinder> binder = reinterpret_cast<IBinder*>(flat->cookie);
                return finishUnflattenBinder(binder, out);
            }
            case BINDER_TYPE_HANDLE: {
                // 注意此时收到的数据是经过Binder驱动加工过的，
                // 我们现在是在SystemServer进程
                // 所以这个hdr.type从BINDER_TYPE_BINDER转成了BINDER_TYPE_HANDLE！
                // 这里解包，注意flat_binder_object这个结构体，
                // 他里面的binder和handle是被组合成union结构的!
                // 所以封包的时候写存入的binder就是此时读取的handle，
                // 这也代表了Server App中对应的BBinder
                sp<IBinder> binder =
                    ProcessState::self()->getStrongProxyForHandle(flat->handle);
                // 这里我们就可以知道，其实SystemServer自始至终
                // 都是保存了来自Server App的BBinder相同数据,
                // 但是转成了BpBinder的IBinder对象
                return finishUnflattenBinder(binder, out);
            }
        }
    }
    return BAD_TYPE;
}

```

至此了解了IBinder转换过程的原理，BinderProxy的生成过程，也知道了BBinder和BpBinder的映射关系，接下来就是在Client App和Server App之间的通信了。