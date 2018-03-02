Activity 源码解析
==

Activity是Android里非常重要的一个组件。东西非常多，如果本文有没有覆盖到，但你又觉得非常重要的部分，欢迎给我反馈。如果你发现了任何错误也欢迎给我反馈。（Email:zhxilong2015@gmail.com）

以下全部基于Android 8.0 Api26的源码分析。不同版本可能稍有不同。但整个基本原理其实是完全一样的。
既然是源码分析，所以这里不会去做功能分析，比如Activity都有什么功能，startActivity怎么调用，怎么给Activity添加专场动画等，这里会着重分析Activity从创建到结束，都经过了哪些过程，这些过程代码都在什么位置，Activity是在哪里new出来的，生命周期为什么能被回调等等这些，本文会跟随着Android系统的代码执行顺序，一步步从Activity被创建分析到Activity被销毁。大致会从以下几个部分来介绍，有一定的阅读门槛：

- Android消息机制
- ActivityThread介绍
- Activity如何收到消息
- Activity的生命周期事件
- Activity结束
- 总结


## 1. 简介
Activity是Android的四大组件之一，光注释都五六百行。大致从注释能看到Activity本身是为UI服务的，介绍了生命周期和一些常用功能。更详细的还需要从源码层面分析。
## 2. 源码分析
### 1.Android消息机制
除了类似单片机这种特殊的操作系统，大部分操作系统都会有消息机制。消息机制把软件需要执行的代码都包装成消息。有了消息机制可以很方便的让软件对外界操作进行实时响应。一个RecyclerView，你上滑一下，RecyclerView滑动还没有结束你这时候可以按住RecyclerView，RecyclerView会实时响应你的操作让滑动立刻停止。怎么做到的？其实就是消息机制。RecyclerView的滑动被分割成了若干个消息，每个消息都移动一小段距离，这时候你突然按住RecyclerView，你的操作会被立即插入到消息循环，这时候不再执行移动的操作，而是响应你按住这个操作，让RecyclerView立刻停止。用户看到的效果就是App实时响应了用户的请求，没有因为App当前正在滑动RecyclerView而用户等待动画结束。

在Android中实现消息机制的就是Handler、Message、MessageQueue、Looper这四兄弟。任何操作都被封装成Message被添加到MessageQueue中，Looper负责消息的循环，拿到消息后交给Handler来处理。
这里之所以要简单介绍消息机制，主要就是Activity的各种事件本质来源其实都是Handler消息。知道消息机制，接下来就会很容易理解Activity的一系列事件回调。

### 2.ActivityThread介绍
Java可以很方便的随时Dump当前的方法栈，可以看到当前代码是经过了怎样的方法栈被调到的。
```java
      at android.app.Activity.performCreate(Activity.java:6975)
      at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1214)
      at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2784)
      at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2906)
      at android.app.ActivityThread.-wrap11(ActivityThread.java:-1)
      at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1605)
      at android.os.Handler.dispatchMessage(Handler.java:105)
      at android.os.Looper.loop(Looper.java:172)
      at android.app.ActivityThread.main(ActivityThread.java:6637)
      at java.lang.reflect.Method.invoke(Method.java:-1)
      at com.android.internal.os.Zygote$MethodAndArgsCaller.run(Zygote.java:240)
      at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:767)
```
在任意一个Activity的onCreate加个断点，可以得到以上方法栈数据。或者每当App崩溃的时候也会有类似数据被输出到Log，再或者也可以通过
```java
        Thread.currentThread().getStackTrace();
```
得到方法栈自己输出到Logcat。除了关注自己代码部分，可能很少会有人去关注下面这些方法。下面这些方法你有看到Looper.loop吗？这不是我们的消息循环吗？loop是被谁调用的？ActivityThread.main。ActivityThread的main方法就是Android App的入口代码（这个入口是被ActivityManagerService定义的，那可以在ActivityManagerService中找到相关代码，这里不做详细介绍）。ActivityThread源码包含在SDK中（如果你双击Shift输入ActivityThread IDE没有找到这个类的话，你可以先找到Activity类，然后在Activity源码中find查找ActivityThread，然后通过快捷键跟进ActivityThread源码）。
定位到main方法：
```java
public class ActivityThread{

    ...

    private void attach(boolean system) {
        if (!system) {
            ...
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManager.getService();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            ...
        }
    }

    ...

    public static void main(String[] args) {
        ......
        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
}
```
基本上做了两件事，初始化ActivityThread（注意有个mgr.attachApplication(mAppThread)注册mAppThread），然后在main方法最后进入了Looper.loop的消息循环，因为这个循环是不会结束的，所以最后一个异常正常情况下是不会被执行到的。当然你可以通过
```java
        Looper.getMainLooper().quit();
```
强行让主线程消息循环结束。

看到这里我们应该明白，在App初始化完成后，执行的代码都会进入到Looper中执行，所以就很容易理解，其实我们的Activity创建，Activity生命周期回调也将是Handler消息的形式。

在ActivityThread类中有个内部类H，继承自Handler。H类定义了很多类型的int，看名字其实很容易跟生命周期回调对应起来。

### 3.Activity如何收到消息
前面我们知道Looper.loop之后，App要想执行代码，肯定是通过HandlerMessage的形式。那么Activity在创建前一定是收到了一个HandlerMessage。找到处理Message的H类，里面有个LAUNCH_ACTIVITY。
```java
private class H extends Handler {
    public static final int LAUNCH_ACTIVITY         = 100;
    ...
    public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
            case LAUNCH_ACTIVITY: {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                r.packageInfo = getPackageInfoNoCheck(
                        r.activityInfo.applicationInfo, r.compatInfo);
                handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            } break;
            ...
        }
        ...
    }
}
```
在handleMessage时候通过handleLaunchActivity来处理LAUNCH_ACTIVITY消息。
在分析处理部分的代码前，我们回过来想下LAUNCH_ACTIVITY消息是被谁发出的呢？find搜下。
```java
class ActivityThread{
    ...
    private class ApplicationThread extends IApplicationThread.Stub {
        ...
        @Override
        public final void scheduleLaunchActivity(...){
            ...
            sendMessage(H.LAUNCH_ACTIVITY, r);
        }
        ...
    }
    ...
}
```
保留关键代码就是上面的结构。到这里创建Activity的代码，其实都涉及到了。为了更清楚的理解，我们再把代码顺序梳理下，就是下面的代码从上到下。我用“>数字”标出了代码执行顺序。顺着数字顺序可以很清晰的理解这个过程。
```java
public class ActivityThread{

    ...
    // >1
    // App入口
    public static void main(String[] args) {
        ......
        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        // >2
        // 初始化一些必要数据
        thread.attach(false);
        // >5
        // 启动消息循环
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }

    ...
    // >3
    private void attach(boolean system) {
        if (!system) {
            ...
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManager.getService();
            try {
                // >4
                // 注册binder到远端ActivityManagerService
                // 注册完成后mAppThread就可以接收ActivityMangerService跨进程发过来的消息了。
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            ...
        }
    }

    ...

    private class ApplicationThread extends IApplicationThread.Stub {
        ...
        // >6
        // 远端ActivityManagerService通过Binder跨进程通信，
        // 调到这个方法，当前在App的Binder线程。
        @Override
        public final void scheduleLaunchActivity(...){
            ...
            // >7
            // 由于此时在Binder线程，所以不能直接调用handleLaunchActivity方法。
            // 需要通过Handler跨线程通信给主线程发LAUNCH_ACTIVITY消息。
            sendMessage(H.LAUNCH_ACTIVITY, r);
        }
        ...
    }
    ...

    private class H extends Handler {
        public static final int LAUNCH_ACTIVITY         = 100;

        ...
        // >8
        // 主线程收到LAUNCH_ACTIVITY消息。
        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    // 处理LAUNCH_ACTIVITY消息
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    // >9
                    // 打开Activity
                    handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;
                ...
            }
            ...
        }
        ...
    }
}
```
LAUNCH_ACTIVITY事件从scheduleLaunchActivity发出。那scheduleLaunchActivity被谁调用？还记得前面初始化ActivityThread我说的（注意有个setApplicationObject注册mAppThread）吗？ApplicationThread继承自IApplicationThread.Stub，看到Stub是不是觉得很熟悉？就是Binder跨进程通信。
在Looper进入loop循环前，调用了ActivityThread.attach方法，在attach中将ApplicationThread绑定到了系统的ActivityManagerService进程。此时只要ActivityManagerSercice通过跨进程通信给App发LAUNCH_ACTIVITY消息，那么ApplicationThread的scheduleLaunchActivity将收到LAUNCH_ACTIVITY消息，但是因为此时ApplicationThread是在Binder线程，所以此时将通过Handler跨线程给主线程发LAUNCH_ACTIVITY，主线程的H类的handlerMessage收到LAUNCH_ACTIVITY，然后调用handleLaunchActivity（这里梳理了下系统是如何收到打开Activity消息的，后面还会把怎么发出消息即startActivity也给分析下）。

### 3.Activity的生命周期事件
把App如何接收LAUNCH_ACTIVITY消息梳理完了，下面就要开始Activity是怎么被new的，以及Activity是如何收到各种声明周期事件的。还是老样子，我把Activity创建和生命周期相关关键代码贴出来，并用“>数字”来标注执行顺序。
```java
    // >1
    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
        ...

        // Initialize before creating the activity
        WindowManagerGlobal.initialize();
        // 创建Activity以及处理生命周期
        // >2
        Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            ...
            // >12
            // onResume
            handleResumeActivity(...);
            ...
            }
        } else {
            ...
        }
    }

    // >3
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            // >4
            // 创建Activity实例
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            ...
        } catch (Exception e) {
            ...
        }
        try {
            ...
            activity.attach(...);
            ...
            // >7
            // onCreate
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            ...
            // >8
            // onStart
            if (!r.activity.mFinished) {
                activity.performStart();
                r.stopped = false;
            }
            // >9
            // onRestore
            if (!r.activity.mFinished) {
                if (r.isPersistable()) {
                    if (r.state != null || r.persistentState != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                r.persistentState);
                    }
                } else if (r.state != null) {
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                }
            }
            // >10
            // onPostCreate
            if (!r.activity.mFinished) {
                activity.mCalled = false;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnPostCreate(activity, r.state,
                            r.persistentState);
                } else {
                    mInstrumentation.callActivityOnPostCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onPostCreate()");
                }
            }
            r.paused = true;
            // >11
            // 持有Activity实例的r被添加到mActivities中。
            mActivities.put(r.token, r);
            ...
        } catch (Exception e) {
            ...
        }
        ...
    }
    // >13
    final void handleResumeActivity(...) {
        ...
        // >14
        // onResume
        r = performResumeActivity(token, clearHide, reason);
        if (r != null) {
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                // 不可见，后面会通过activity.makeVisible()恢复可见
                decor.setVisibility(View.INVISIBLE);
                ...
                if (a.mVisibleFromClient) {
                    if (!a.mWindowAdded) {
                        a.mWindowAdded = true;
                        // >17
                        // 被添加到WindowManager
                        wm.addView(decor, l);
                    } else {
                        ...
                    }
                }
                ...
                        // 恢复Activity可见
                        r.activity.makeVisible();
                ...
            ...
            }
        }
    }
    ...
    // >15
    // onResume
    public final ActivityClientRecord performResumeActivity(...) {
        ...
        if (r != null && !r.activity.mFinished) {
            ...
            try {
                ...
                // >16
                // onResume
                r.activity.performResume();
                ...
            } catch (Exception e) {
                ...
            }
    }
```
```java
public class Instrumentation {
    ...
    // >5
    public Activity newActivity(ClassLoader cl, String className,
            Intent intent)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        // >6
        // 通过反射创建Activity实例
        return (Activity)cl.loadClass(className).newInstance();
    }
    ...
}
```
我已经用“>数字”把主要的事件执行顺序回调贴出来了。按照顺序可以很清晰的看到一个Activity是怎么被创建出来的，以及创建出来后比如onCreate、onStart等各个生命周期是如何收到回调事件的。收到LAUNCH_ACTIVITY消息后，ActivityManagerService不需要再发送RESUME_ACTIVITY消息，LAUNCH_ACTIVITY处理过程会直接处理掉onStart，onResume。
那假如我此时按下Home键后，App要收到onPause消息，我们来分析下这个过程。
按下Home后，ActivityManagerService将通过跨进程通信给App发送PAUSE_ACTIVITY的消息。
```java
public final class ActivityThread {
    private class ApplicationThread extends IApplicationThread.Stub {
        ...
        // >1
        // 当前在Binder线程收到ActivityManagerService跨进程通信
        public final void schedulePauseActivity(...) {
            ...
            // >2
            // 此时在Binder，给主线程发PAUSE_ACTIVITY的消息
            // 如果是finished就发送PAUSE_ACTIVITY_FINISHING消息。
            sendMessage(
                    finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
                    token,
                    (userLeaving ? USER_LEAVING : 0) | (dontReport ? DONT_REPORT : 0),
                    configChanges,
                    seq);
        }
        ...
    }
 
    private class H extends Handler {
        // >3
        // 此时进入主线程
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case PAUSE_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                    SomeArgs args = (SomeArgs) msg.obj;
                    // >4
                    // onPause∏
                    handlePauseActivity((IBinder) args.arg1, false,
                            (args.argi1 & USER_LEAVING) != 0, args.argi2,
                            (args.argi1 & DONT_REPORT) != 0, args.argi3);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;
            }
        }
    }
    // >5
    private void handlePauseActivity(...) {
        ...
        if (r != null) {
            //Slog.v(TAG, "userLeaving=" + userLeaving + " handling pause of " + r);
            if (userLeaving) {
                // >6
                // 见过这个生命周期吗？
                performUserLeavingActivity(r);
            }
            // >8
            performPauseActivity(token, finished, r.isPreHoneycomb(), "handlePauseActivity");
            ...
        }
    }

    // >7
    final void performUserLeavingActivity(ActivityClientRecord r) {
        mInstrumentation.callActivityOnUserLeaving(r.activity);
    }

    // >9
    final Bundle performPauseActivity(IBinder token, boolean finished,
            boolean saveState, String reason) {
        ActivityClientRecord r = mActivities.get(token);
        // >10
        // 重载performPauseActivity
        return r != null ? performPauseActivity(r, finished, saveState, reason) : null;
    }

    // >11
    final Bundle performPauseActivity(ActivityClientRecord r, boolean finished,
            boolean saveState, String reason) {
        ...
        // >12
        // 回调onSaveInstanceState
        // Next have the activity save its current state and managed dialogs...
        if (!r.activity.mFinished && saveState) {
            callCallActivityOnSaveInstanceState(r);
        }

        // >13
        performPauseActivityIfNeeded(r, reason);

        ...
        // >16
        return !r.activity.mFinished && saveState ? r.state : null;
    }

    // >14
    private void performPauseActivityIfNeeded(ActivityClientRecord r, String reason) {
        ...

        try {
            ...
            // >15
            // 回调onPause
            mInstrumentation.callActivityOnPause(r.activity);
            ...
        } catch (...) {
            ...
        }
        r.paused = true;
    }
}
```
好了，onPause结束。过程很简单，顺便还能学到一个回调performUserLeavingActivity。同样的onResume过程跟onPause过程基本差不多，这里就不再啰嗦了。
到了这里其实你应该也就非常清楚了所有生命周期回调的先后顺序，以及每个生命周期具体是怎么回调到Activity的。那么生命周期的介绍就算告一个段落了。

### 4.Activity的结束
这里再说下mActivies。
```java
public final class ActivityThread {
    ...
    // 在低版本上这里是用的HashMap
    final ArrayMap<IBinder, ActivityClientRecord> mActivities = new ArrayMap<>();
    ...
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
        mActivities.put(r.token, r);
        ...
    }
    ...
}
```
还记得前面在LaunchActivity的时候把持有Activity实例的ActivityClientRecord r被put到mActivities吗？Java不需要自己手动处理对象释放，Java会通过自己的GC来释放。那么Activity对象在创建后肯定需要一个对象来持有它，否则Activity实例在GC时候就会被销毁了。那么谁来持有Activity？就是这里到mActivities。所有Activity被创建后都会被添加进来，当然与之对应的就会在onDestory的时候从mActivities中移除。你可以尝试用上面分析的过程去找下什么时候被移除的。
## 3. 总结
Activity的东西非常多，但只要理解了其中的规律，就会很容易举一反三。如果你看明白了这整个过程，就可以很轻松的通过H类定义的msg.what，找到App都能接受到多少种不同类型的系统回调。这些回调有Activity的，也有Service的，也有Application的等等。同样的方法你可以很容易分析Service的消息处理过程，大家都在说Service onStartCommand是发生在主线程，那么为什么是主线程？同样的思路，你可以非常容易弄明白这个过程。
