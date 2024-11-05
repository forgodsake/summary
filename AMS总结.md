#### Activity启动流程分析

##### 1，在ActivityThread的performLaunchActivity中，通过反射创建了对应的Activity实例，通过调用activity的attach方法，创建了对应的PhoneWindow，将其与Activity关联起来。
##### 2，在ActivityThread的handleResumeActivity中，通过wm.addView(decor, l) 将decor添加到window中，wm是WindowManagerImpl,最终会调用WindowManagerGolbal的addView，并将页面视图添加到PhoneWindow之上，也是因为这个原因，所以Activity的onResume回调函数中，并不能立即得到各个子View的实际宽高。









