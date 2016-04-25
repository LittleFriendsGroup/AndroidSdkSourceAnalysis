# Toast简介
Toast是一种提供给用户简洁信息的视图，该视图已浮于应用程序之上的形式呈现给用户。
Toast源码位于：[frameworks\base\core\java\android\widget\Toast.java](https://github.com/android/platform_frameworks_base/blob/master/core/java/android/widget/Toast.java)

# Toast使用
Toast使用就一行代码：
```
 Toast.makeText(ToastActivity.this,"Toast源码解析",Toast.LENGTH_LONG).show();
```
Toast 提供了setView方法，可以自定义View：
```
Toast toast=new Toast(ToastActivity.this);
View view=View.inflate(ToastActivity.this,R.layout.toast_view,null);
toast.setView(view);
toast.setGravity(Gravity.CENTER,0,0);
toast.show();
```
toast_view.xml
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <ImageView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:background="@mipmap/ic_launcher" />
</LinearLayout>
```
# Toast源码分析
Toast源码分析有两个目标，知道Toast源码在哪里体现了Toast显示，又在哪里体现了Toast消失。首先从Toast的基本使用，作为入口。
```
 Toast.makeText(ToastActivity.this,"Toast源码解析",Toast.LENGTH_LONG).show();
```
## 1. makeText
```
public static Toast makeText(Context context, CharSequence text, @Duration int duration) {
        Toast result = new Toast(context);

        LayoutInflater inflate = (LayoutInflater)
                context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        View v = inflate.inflate(com.android.internal.R.layout.transient_notification, null);
        TextView tv = (TextView)v.findViewById(com.android.internal.R.id.message);
        tv.setText(text);
        
        result.mNextView = v;//传入下个view
        result.mDuration = duration;//Toast显示的时间长度

        return result;
    }
```
makeText中的transient_notification.xml，源码位于：[frameworks\base\core\res\layout\transient_notification.xml](https://github.com/android/platform_frameworks_base/blob/master/core/res/res/layout/transient_notification.xml)
```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:background="?android:attr/toastFrameBackground">

    <TextView
        android:id="@android:id/message"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:layout_gravity="center_horizontal"
        android:textAppearance="@style/TextAppearance.Toast"
        android:textColor="@color/bright_foreground_dark"
        android:shadowColor="#BB000000"
        android:shadowRadius="2.75"
        />

</LinearLayout>
```
从makeText方法看，就是Toast的自定义view的那部分代码。

# 2. show
```
public void show() {
		//解释自定义view，需要先setView
        if (mNextView == null) {
            throw new RuntimeException("setView must have been called");
        }
		//得到INotificationManager服务
        INotificationManager service = getService();
        String pkg = mContext.getOpPackageName();
		//TN对象
        TN tn = mTN;
        tn.mNextView = mNextView;

        try {
            service.enqueueToast(pkg, tn, mDuration);
        } catch (RemoteException e) {
            // Empty
        }
    }
```
看了show方法，发现涉及两个新的类，TN 和INotificationManager 。enqueueToast方法大概就是实现Toast显示和消失吧，让我们一步步探索。

## 2.1. TN 
```
 private static class TN extends ITransientNotification.Stub {
 
        final Runnable mShow = new Runnable() {
            @Override
            public void run() {
                handleShow();//显示处理
            }
        };

        final Runnable mHide = new Runnable() {
            @Override
            public void run() {
                handleHide();//消失处理
                // Don't do this in handleHide() because it is also invoked by handleShow()
                mNextView = null;
            }
        };
		//应用程序窗口
        private final WindowManager.LayoutParams mParams = new WindowManager.LayoutParams();
        final Handler mHandler = new Handler();    

        int mGravity;//出现在屏幕的位置
        int mX, mY;//分别是出现在屏幕的X、Y方向偏移量
        float mHorizontalMargin;//横向margin值
        float mVerticalMargin;//竖向margin值


        View mView;//当前view
        View mNextView;//下个Toast显示的view

        WindowManager mWM;
		//TN构造函数
        TN() {
            // XXX This should be changed to use a Dialog, with a Theme.Toast
            // defined that sets up the layout params appropriately.
            final WindowManager.LayoutParams params = mParams;
            params.height = WindowManager.LayoutParams.WRAP_CONTENT;
            params.width = WindowManager.LayoutParams.WRAP_CONTENT;
            // 不设置这个弹出框的透明遮罩显示为黑色
			params.format = PixelFormat.TRANSLUCENT;
			// 动画
            params.windowAnimations = com.android.internal.R.style.Animation_Toast;
            // 类型
			params.type = WindowManager.LayoutParams.TYPE_TOAST;
            params.setTitle("Toast");
			// 设置flag
            params.flags = WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON
                    | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                    | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;
        }

        /**
         * schedule handleShow into the right thread
         */
        @Override
        public void show() {
            if (localLOGV) Log.v(TAG, "SHOW: " + this);
            mHandler.post(mShow);
        }

        /**
         * schedule handleHide into the right thread
         */
        @Override
        public void hide() {
            if (localLOGV) Log.v(TAG, "HIDE: " + this);
            mHandler.post(mHide);
        }
		//显示Toast
        public void handleShow() {
            if (localLOGV) Log.v(TAG, "HANDLE SHOW: " + this + " mView=" + mView
                    + " mNextView=" + mNextView);
            if (mView != mNextView) {//判断下个view是否一样
                // remove the old view if necessary
                handleHide();//移除当前view
                mView = mNextView;
                Context context = mView.getContext().getApplicationContext();//获取当前view上下文
                String packageName = mView.getContext().getOpPackageName();
                if (context == null) {
                    context = mView.getContext();
                }
				//获得 WindowManager对象
                mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
                // We can resolve the Gravity here by using the Locale for getting
                // the layout direction
                final Configuration config = mView.getContext().getResources().getConfiguration();
				//获取绝对的Gravity
                final int gravity = Gravity.getAbsoluteGravity(mGravity, config.getLayoutDirection());
                mParams.gravity = gravity;
                if ((gravity & Gravity.HORIZONTAL_GRAVITY_MASK) == Gravity.FILL_HORIZONTAL) {
                    mParams.horizontalWeight = 1.0f;
                }
                if ((gravity & Gravity.VERTICAL_GRAVITY_MASK) == Gravity.FILL_VERTICAL) {
                    mParams.verticalWeight = 1.0f;
                }
                mParams.x = mX;
                mParams.y = mY;
                mParams.verticalMargin = mVerticalMargin;
                mParams.horizontalMargin = mHorizontalMargin;
                mParams.packageName = packageName;
                if (mView.getParent() != null) {
                    if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
                    mWM.removeView(mView);//如果当前view存在，先移除
                }
                if (localLOGV) Log.v(TAG, "ADD! " + mView + " in " + this);
                mWM.addView(mView, mParams);//通过WindowManager调用addView加载
                trySendAccessibilityEvent();
            }
        }
		
        private void trySendAccessibilityEvent() {
            AccessibilityManager accessibilityManager =
                    AccessibilityManager.getInstance(mView.getContext());
            if (!accessibilityManager.isEnabled()) {
                return;
            }
            // treat toasts as notifications since they are used to
            // announce a transient piece of information to the user
            AccessibilityEvent event = AccessibilityEvent.obtain(
                    AccessibilityEvent.TYPE_NOTIFICATION_STATE_CHANGED);
            event.setClassName(getClass().getName());
            event.setPackageName(mView.getContext().getPackageName());
            mView.dispatchPopulateAccessibilityEvent(event);
            accessibilityManager.sendAccessibilityEvent(event);
        }        
		//WindowManager调用removeView方法来将Toast视图移除
        public void handleHide() {
            if (localLOGV) Log.v(TAG, "HANDLE HIDE: " + this + " mView=" + mView);
            if (mView != null) {
                // note: checking parent() just to make sure the view has
                // been added...  i have seen cases where we get here when
                // the view isn't yet added, so let's try not to crash.
                if (mView.getParent() != null) {
                    if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
                    mWM.removeView(mView);
                }

                mView = null;
            }
        }
    }
```
TN类继承自ITransientNotification.Stub，ITransientNotification.aidl，用于进程间通信，源码位于[frameworks\base\core\java\android\app\ITransientNotification.aidl](https://github.com/android/platform_frameworks_base/blob/master/core/java/android/app/ITransientNotification.aidl)
```
/** @hide */
oneway interface ITransientNotification {
    void show();
    void hide();
}
```
具体实现就在TN类，其他进程回调TN类，来操作Toast的显示和消失：
```
 @Override
        public void show() {
            if (localLOGV) Log.v(TAG, "SHOW: " + this);
            mHandler.post(mShow);//显示
        }

        /**
         * schedule handleHide into the right thread
         */
        @Override
        public void hide() {
            if (localLOGV) Log.v(TAG, "HIDE: " + this);
            mHandler.post(mHide);//消失
        }
```
这里可以看出Toast显示和消失用的Handler机制实现的。

## 2.2. INotificationManager
调用了getService，如下：
```
private static INotificationManager sService;

    static private INotificationManager getService() {
        if (sService != null) {
            return sService;
        }
        sService = INotificationManager.Stub.asInterface(ServiceManager.getService("notification"));
        return sService;
    }
```
得到INotificationManager服务，再调用enqueueToast方法，参数有三个，包名，TN，时间。INofiticationManager接口的具体实现类是NotificationManagerService类，源码位置：[frameworks\base\services\core\java\com\android\server\notification\NotificationManagerService.java](https://github.com/android/platform_frameworks_base/blob/master/services/core/java/com/android/server/notification/NotificationManagerService.java)

## 2.3. enqueueToast
```
 @Override
        public void enqueueToast(String pkg, ITransientNotification callback, int duration)
        {
            if (DBG) {
                Slog.i(TAG, "enqueueToast pkg=" + pkg + " callback=" + callback
                        + " duration=" + duration);
            }

            if (pkg == null || callback == null) {
                Slog.e(TAG, "Not doing toast. pkg=" + pkg + " callback=" + callback);
                return ;
            }
			//(1)判断是否系统的Toast，如果当前包名是android则为系统
            final boolean isSystemToast = isCallerSystem() || ("android".equals(pkg));
			//判断当前toast所属的pkg是不是所阻止的
            if (ENABLE_BLOCKED_TOASTS && !noteNotificationOp(pkg, Binder.getCallingUid())) {
                if (!isSystemToast) {
                    Slog.e(TAG, "Suppressing toast from package " + pkg + " by user request.");
                    return;
                }
            }
			//入队列mToastQueue 
            synchronized (mToastQueue) {
                int callingPid = Binder.getCallingPid();
                long callingId = Binder.clearCallingIdentity();
                try {
                    ToastRecord record;
                    //(2)判断Toast是否在队列当中
                    int index = indexOfToastLocked(pkg, callback);
                    // If it's already in the queue, we update it in place, we don't
                    // move it to the end of the queue.
                    if (index >= 0) {
                        record = mToastQueue.get(index);
                        record.update(duration);
                    } else {
                        // Limit the number of toasts that any given package except the android
                        // package can enqueue.  Prevents DOS attacks and deals with leaks.
                        if (!isSystemToast) {
                            int count = 0;
                            final int N = mToastQueue.size();
                            for (int i=0; i<N; i++) {
                                 final ToastRecord r = mToastQueue.get(i);
                                 if (r.pkg.equals(pkg)) {
                                     count++;
                                     if (count >= MAX_PACKAGE_NOTIFICATIONS) {//限制toasts数，最大50
                                         Slog.e(TAG, "Package has already posted " + count
                                                + " toasts. Not showing more. Package=" + pkg);
                                         return;
                                     }
                                 }
                            }
                        }
						//获得ToastRecord对象
                        record = new ToastRecord(callingPid, pkg, callback, duration);
						//放入mToastQueue中                       
					    mToastQueue.add(record);
                        index = mToastQueue.size() - 1;
                        keepProcessAliveLocked(callingPid);//(3)设置该Toast为前台进程
                    }
                    // If it's at index 0, it's the current toast.  It doesn't matter if it's
                    // new or just been updated.  Call back and tell it to show itself.
                    // If the callback fails, this will remove it from the list, so don't
                    // assume that it's valid after this.
                    if (index == 0) {
                        showNextToastLocked();//(4)直接显示Toast
                    }
                } finally {
                    Binder.restoreCallingIdentity(callingId);
                }
            }
        }
```
(1)判断是否系统的Toast，源码：
```
 private static boolean isCallerSystem() {
        return isUidSystem(Binder.getCallingUid());
    }
private static boolean isUidSystem(int uid) {
        final int appid = UserHandle.getAppId(uid);
		//判断pid为系统进程使用的用户id，值为1000，或者为系统进程的手机的用户id，值为1001
        return (appid == Process.SYSTEM_UID || appid == Process.PHONE_UID || uid == 0);
    }
```
(2)判断Toast是否在队列当中，源码：
```
 // lock on mToastQueue
    int indexOfToastLocked(String pkg, ITransientNotification callback)
    {
	//
        IBinder cbak = callback.asBinder();
        ArrayList<ToastRecord> list = mToastQueue;
        int len = list.size();
        for (int i=0; i<len; i++) {
            ToastRecord r = list.get(i);
            if (r.pkg.equals(pkg) && r.callback.asBinder() == cbak) {
                return i;
            }
        }
        return -1;
    }
```
(3)设置该Toast为前台进程，源码：
```
 // lock on mToastQueue
    void keepProcessAliveLocked(int pid)
    {
        int toastCount = 0; // toasts from this pid
        ArrayList<ToastRecord> list = mToastQueue;
        int N = list.size();
        for (int i=0; i<N; i++) {
            ToastRecord r = list.get(i);
            if (r.pid == pid) {
                toastCount++;
            }
        }
        try {
		    //设置该Toast为前台进程
            mAm.setProcessForeground(mForegroundToken, pid, toastCount > 0);
        } catch (RemoteException e) {
            // Shouldn't happen.
        }
    }
```
(4)直接显示Toast，源码：
```
 void showNextToastLocked() {
		//直接取第一个
        ToastRecord record = mToastQueue.get(0);
        while (record != null) {
            if (DBG) Slog.d(TAG, "Show pkg=" + record.pkg + " callback=" + record.callback);
            try {
                record.callback.show();//回调TN类，显示Toast
                scheduleTimeoutLocked(record);//设置消失
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Object died trying to show notification " + record.callback
                        + " in package " + record.pkg);
                // remove it from the list and let the process die
                int index = mToastQueue.indexOf(record);
                if (index >= 0) {
                    mToastQueue.remove(index);
                }
                keepProcessAliveLocked(record.pid);
                if (mToastQueue.size() > 0) {
                    record = mToastQueue.get(0);
                } else {
                    record = null;
                }
            }
        }
    }
 private void scheduleTimeoutLocked(ToastRecord r)
    {
        mHandler.removeCallbacksAndMessages(r);//移除ToastRecord
        Message m = Message.obtain(mHandler, MESSAGE_TIMEOUT, r);
        //static final int LONG_DELAY = 3500; // 3.5 seconds
        //static final int SHORT_DELAY = 2000; // 2 seconds
        long delay = r.duration == Toast.LENGTH_LONG ? LONG_DELAY : SHORT_DELAY;
		//发送Toast消失的message
        mHandler.sendMessageDelayed(m, delay);
    }
```
从enqueueToast方法可知，先判断是不是系统和合法的Toast，然后判断是否在ToastQueue（这里解释了很多Toast，是一个个显示的），如果存在，只需要更新Toast显示的时间，如果不在，就直接显示，回调给TN类。`到这里，知道了Toast是如何显示的。`
还没有结束，继续追踪mHandler，来到WorkerHandler ：
```
private final class WorkerHandler extends Handler
    {
        @Override
        public void handleMessage(Message msg)
        {
            switch (msg.what)
            {
                case MESSAGE_TIMEOUT:
                    handleTimeout((ToastRecord)msg.obj);
                    break;
               ……
            }
        }

    }
 private void handleTimeout(ToastRecord record)
    {
        if (DBG) Slog.d(TAG, "Timeout pkg=" + record.pkg + " callback=" + record.callback);
        synchronized (mToastQueue) {
        //还是判断Toast是否在队列当中
            int index = indexOfToastLocked(record.pkg, record.callback);
            if (index >= 0) {
                cancelToastLocked(index);
            }
        }
    }
 void cancelToastLocked(int index) {
        ToastRecord record = mToastQueue.get(index);
        try {
            record.callback.hide();//回调TN类，Toast消失
        } catch (RemoteException e) {
            Slog.w(TAG, "Object died trying to hide notification " + record.callback
                    + " in package " + record.pkg);
            // don't worry about this, we're about to remove it from
            // the list anyway
        }
        mToastQueue.remove(index);//该ToastRecord对象从mToastQueue中移除
        keepProcessAliveLocked(record.pid);//设置该Toast为前台进程
        if (mToastQueue.size() > 0) {
            // Show the next one. If the callback fails, this will remove
            // it from the list, so don't assume that the list hasn't changed
            // after this point.
            showNextToastLocked();//继续show下个Toast
        }
    }
```
`到这里，知道了Toast是如何消失的。`
Toast核心显示和消失源码分析完毕，其他一些比如duration、gravity等设置，不难，一看就明白。

# 总结
Toast代码调用只有一行，了解这行代码的背后，了解Toast是怎样显示，又是怎样消失的。自定义Toast时，需要调用setView，不然show会抛异常，这个从show方法就能得知。至此，Toast源码解析告一段落。




