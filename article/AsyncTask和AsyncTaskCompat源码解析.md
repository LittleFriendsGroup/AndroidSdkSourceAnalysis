## 一、AsyncTask源码分析
### 1.1、简介
 AsyncTask是android提供的一种异步消息处理的解决方案，能简化我们在子线程中更新UI控件，使用AsyncTask你将看不到任何关于操作线程的代码。
### 1.2、版本差别
1、线程池配置
* android3.0以前线程池配置，代码如下所示：
```java
private static final int CORE_POOL_SIZE = 5;//核心线程数量
private static final int MAXIMUM_POOL_SIZE = 128;//线程池中允许的最大线程数目
private static final it KEEP_ALIVE = 10;//当线程数目大于核心线程数目时，如果超过这个keepAliveTime时间，那么空闲的线程会被终止。
……    
private static final ThreadPoolExecutor sExecutor = new ThreadPoolExecutor(CORE_POOL_SIZE,    
        MAXIMUM_POOL_SIZE, KEEP_ALIVE, TimeUnit.SECONDS, sWorkQueue, sThreadFactory);  
```
 android3.0以后更加灵活，根据cpu核数配置`CORE_POOL_SIZE`和`MAXIMUM_POOL_SIZE`
```java
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();//根据cpu的大小来配置核心的线程
private static final int CORE_POOL_SIZE = CPU_COUNT + 1;//核心线程数量
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;//线程池中允许的最大线程数目
private static final int KEEP_ALIVE = 1;//空闲线程的超时时间
```
2、串行和并行,引用来自[这篇文章](http://www.jianshu.com/p/a8b1861f2efc)
* android 1.5以前的时候`execute`是串行执行的
* android 1.6直到android 2.3.2被修改为并行执行，执行任务的线程池就是THREAD_POOL_EXECUTOR
* android 3.0以后，默认任务是串行执行的，如果想要并行执行任务可调用`executeOnExecutor(Executor exec, Params.. params)`。具体用法可参照[Android AsyncTask的骗术](http://glanwang.com/post/android/android-asynctaskde-pian-zhu)

## 二、基本用法
### 2.1、继承AsyncTask，设置子类三个泛型的参数
 ```java
 public abstract class AsyncTask<Params, Progress, Result>//java
 ```
 * Params     异步任务处理的参数
 * Progress   异步任务执行过程中返回给主线程的进度值，通过publishProgress()方法发送出去
 * Result     异步任务执行结束返回的结果的结果类型，可以为boolean，或者bitmap等类型
 
###2.2、子类必须实现的抽象方法
（1）onPreExecute<br>执行后台耗时操作前被调用，通常用于完成一些初始化操作，比如2.3例子中初始化dialog的操作，或者一些集合容器<br>
（2）doInBackGround<br>必须实现，异步执行后台线程将要完成的任务(该方法在子线程运行,下面源码会分析到)<br>
（3）onProgressUpdate<br>在doInBackGround方法中调用publishProgress方法，AsyncTask就会主动调用onProgressUpdate实现更新
任务的执行进度<br>
（4）onPostExecute<br>当doInBackGround完成后，系统会自动调用，销毁一些dialog的操作，并将doInBackGround﻿方法返回的值传给该方法<br>
 执行的大致流程是：

`onPreExecute`-> `doInBackGround`->`onProgressUpdate(调用publishProgress的时候)`->`onPostExecute`
###2.3、用法案例
1、实例化子类
```java
new DownAsyncTask().extcute();
```
2、代码案例
```java
public class AsyntaskActivity extends AppCompatActivity {

    private ProgressDialog progressDialog;
    private int progress = 0;
    private int MAX_PROGRESS = 100;
    private TextView tvProgress;
    private DownAsyncTask task;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_asyntask);
        tvProgress = (TextView) findViewById(R.id.tvProgress);
        progressDialog = new ProgressDialog(this);
        task = new DownAsyncTask();
        task.execute();
    }

    class DownAsyncTask extends AsyncTask<Void, Integer, Boolean> {

        @Override
        protected void onCancelled() {
            super.onCancelled();
        }
        @TargetApi(Build.VERSION_CODES.HONEYCOMB)
        @Override
        protected void onCancelled(Boolean aBoolean) {
            super.onCancelled(aBoolean);
            Log.i("onCancelled", "取消");
        }
        @Override
        protected void onPostExecute(Boolean aBoolean) {
            super.onPostExecute(aBoolean);
            progressDialog.dismiss();
            Log.i("onPostExecute", "提交成功");
            if (aBoolean) {
                Toast.makeText(AsyntaskActivity.this, "下载成功", Toast.LENGTH_SHORT).show();
            } else {
                Toast.makeText(AsyntaskActivity.this, "下载失败", Toast.LENGTH_SHORT).show();
            }
        }
        @Override
        protected void onPreExecute() {
            super.onPreExecute();
            progressDialog.show();
        }
        @Override
        protected void onProgressUpdate(Integer... values) {
            super.onProgressUpdate(values);
            tvProgress.setText("当前下载进度：" + values[0] + "%");
            progressDialog.setMessage("当前下载进度：" + values[0] + "%");
        }

        @Override
        protected Boolean doInBackground(Void... params) {
            while (progress < 10000) {
                progress = progress + 2;
                Log.i("当前的进度", "progress==" + progress);
                publishProgress(progress);
//                try {
//                    Thread.sleep(2000);
//                } catch (InterruptedException e) {
//                    e.printStackTrace();
//                }
            }
            return true;
        }
    }
    public void cancelTask(View v) {
        task.cancel(true);
    }
}
//java
```

例子中doInBackground返回的结果Boolean最终会传递给onPostExecute(Boolean aBoolean)，而这一系列的操作是由AsyncTask底层实现的，通过handler发结果发送到主线程

###2.4、取消异步任务
```java
AsyncTask.cancel(mayInterruptIfRunning);
```
mayInterruptIfRunning是boolean类型的，注意这里true和false的区别
```java
 try {
         Thread.sleep(2000);
     } catch (InterruptedException e) 
     {
         e.printStackTrace();
     }
```
如果线程处于休眠状态，为true则正在执行的线程将会中断，抛出异常，但执行的任务线程会继续执行完毕调用`onCanceled()`。为false则正在执行的线程不会中断，任务线程执行完毕调用`onCanceled()`。
如果线程不处于休眠状态，为true和false都没有区别，任务线程执行完毕后调用`onCanceled()`。
正确地取消要在`doInBackground(Void... params)`使用`isCancelled()`来判断，退出循环操作。如下面的
```java
        @Override
        protected Boolean doInBackground(Void... params) {
            while (progress < 10000) {
                progress = progress + 2;
                Log.i("当前的进度", "progress==" + progress);
                publishProgress(progress);
                //判断是不是调用了AsyncTask.cancel(mayInterruptIfRunning)，如果已经调用了，
                if(isCancelled())
                {
                   break;//跳出循环，马上调用onCancelled()方法，不需要等doInBackground执行完任务
                }
            }
            return true;
        }
```
## 三、AsyncTask源码分析(基于android3.0以后分析)
### 3.1、AsyncTask构造函数
```java
   /**AsyncTask的构造函数源码片段**/
   public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            //异步任务执行的时候调用call方法
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
```
AsyncTask的实例化在UI线程中。构造函数初始化了两个成员变量mWorker和mFuture。mWorker为WorkerRunnable类型的匿名内部类实例对象（实现了Callable接口），mFuture为FutureTask类型的匿名内部类实例对象，将mWorker作为mFuture的形参（重写了FutureTask类的done方法）。当执行了execute方法的时候在会回调`call()`方法，`call()`方法调用了`doInBackground(mParams)`,这部分是在子线程中完成的。
* WorkerRunnable是一个实现了Callable的抽象类,扩展了Callable多了一个Params参数
```java
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> 
{
        Params[] mParams;
}//java
```
下面讲述下Callable和Runnable的区别。
1、Callable的接口方法是call，Runnable是run<br>
2、Callable可以带返回值，Runnable不行,这个结果是Future获取的<br>
3、Callable可以捕获异常，Runnable不行<br>
```java
public class CallableAndFuture {
    public static void main(String[] args) {
        Callable<Integer> callable = new Callable<Integer>() {
            public Integer call() throws Exception {
                return new Random().nextInt(100);
            }
        };
        //那WorkerRunnable的回调方法call肯定是在FutureTask中调用的
        FutureTask<Integer> future = new FutureTask<Integer>(callable)
      ```java
      ```);
        new Thread(future).start();
        try {
            Thread.sleep(5000);// 可能做一些事情
            System.out.println(future.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}//java
```
* FutureTask实现了接口Runnable，它既可以作为Runnable被线程执行，同时将Callable作为构造函数的参数传入，那么这个组合的使用有什么好处呢？假设有一个很耗时的返回值需要计算，并且这个返回值不是立刻需要的话，那么就可以使用这个组合，用另一个线程去计算返回值，而当前线程在使用这个返回值之前可以做其它的操作，等到需要这个返回值时，再通过Future得到。
```java
public class FutureTask<V> implements RunnableFuture<V>//java
实现了RunnableFuture接口
```
当我们初始化FutureTask的时候传入callable，FutureTask的run方法要开始回调WorkerRunable的call方法了，call里面调用doInBackground(mParams),终于回到我们后台任务了，调用我们AsyncTask子类的`doInBackground()`,由此可以看出`doInBackground()`是在子线程中执行的，如下图所示
![](https://github.com/white37/AndroidSdkSourceAnalysis/blob/master/images/FutureTask(run).png)

### 3.2、核心方法
* execute()方法
```java
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

 /** AsyncTask类的execute方法**/
 public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    //调用executeOnExecutor方法
    return executeOnExecutor(sDefaultExecutor, params);
}
```
当执行execute方法时，实际上是调用了executeOnExecutor方法。这里传递了两个参数，一个是sDefaultExecutor，一个是params。从上面的源码可以看出，sDefaultExecutor其实是一个SerialExecutor对象，实现了串行线程队列。params其实最终会赋给doInBackground方法去处理。

* executeOnExecutor()方法
```java
//exec执行AsyncTask.execute()方法时传递进来的参数sDefaultExecutor，这个sDefaultExecutor其实就是SerialExecutor对象。
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;
	//实际是调用子类里面的onPreExecute
        onPreExecute();

        mWorker.mParams = params;
        //execute是调用SERIAL_EXECUTOR的execute，mFuture就是之前AsyncTask构造初始化赋值的FutureTask。
        exec.execute(mFuture);
        return this;
    }

```
* SerialExecutor的execute方法
```java
  private static class SerialExecutor implements Executor {
  	//循环数组实现的双向Queue。大小是2的倍数，默认是16。有队头队尾两个下标
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        //当前正在运行的runnable
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            //添加到双端队列里面去
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                    	//执行的是mFuture就是之前AsyncTask构造初始化赋值的FutureTask的run()方法
                        r.run();
                    } finally {
                    	//取出下一个任务执行
                        scheduleNext();
                    }
                }
            });
            //如果没有活动的runnable，从双端队列里面取出一个runnable放到线程池中运行
            //第一个请求任务过来的时候mActive是空的
            if (mActive == null) {
            	//取出下一个任务来执行
                scheduleNext();
            }
        }
	
        protected synchronized void scheduleNext() {
            //从双端队列中取出一个任务
            if ((mActive = mTasks.poll()) != null) {
            	//线程池执行取出来的任务，真正执行任务的
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }//java
```
exec.execute(mFuture)执行时，SerialExecutor将FutureTask作为参数执行execute方法。在SerialExecutor的execute方法中，这里通过一个任务队列mTasks把FutureTask插入进了队列中，执行r.run，其实就是执行FutureTask的run方法，因为传递进来的r参数就是mFuture，执行完无论什么情况都是会scheduleNext()取出下一个任务来执行的。由此可知道一个串行的线程池，同一时刻只会有一个线程正在执行，其余的均处于等待状态，等到上一个线程执完r.run()完之后，scheduleNext()取出下一个任务执行。如果再有新的任务被执行时就等待上一个任务执行完毕后才会得到执行，实现了串行的任务队列正是通过SerialExecutor核心类。


```java
mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);//调用子类的doInBackground
                Binder.flushPendingCommands();
                return postResult(result);//执行完后通过postResult结果传递出去
            }
        };
```
* AsyncTask的postResult方法
```java
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        //获取一个handler，等到一个消息，将结果封装在Message
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));//new AsyncTaskResult<Result>(this, result)将得到的结果再做了一层封装
        //将消息发送到主线程，会回调handleMessage()方法
        message.sendToTarget();
        return result;
    }

```

```java
  private static Handler getHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
            	//初始化一个InternalHandler，用与将结果发送给主线程
                sHandler = new InternalHandler();
            }
            return sHandler;
        }
    }
```
```java
     private static class InternalHandler extends Handler {
        public InternalHandler() {
            // 在主线程中调用
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```
执行postResult的时候，obtainMessage传递的参数是：MESSAGE_POST_RESULT和AsyncTaskResult(this, result))，然后message.sendToTarget()开始发消息，并在InternalHandler的handleMessage中开始处理消息。
当我们子类调用publishProgress(progress)的时候

```java
    @WorkerThread
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
```
调用AysncTask里面publishProgress,，obtainMessage传递的参数是：MESSAGE_POST_PROGRESS和AsyncTaskResult(this, result))，然后message.sendToTarget()开始发消息，并在InternalHandler的handleMessage中开始处理消息。
当我们子类调用publishProgress(progress)的时候
```java
 private void finish(Result result) {
        if (isCancelled()) {
            //如果已经取消了就会调用onCancelled方法
            onCancelled(result);
        } else {
            //执行完异步任务后，调用子类的onPostExecute，关闭一些dialog、释放资源等操作
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```
总结前面大致的流程是：·`new DownAsynTask().execute()->AsyncTask.executeOnExecutor(Executor exec,Params... params)->onPreExecute()->SerialExecutor.execute(mFuture)->mFuture插入队列->THREAD_POOL_EXECUTOR.execute(mActive)取出任务串行执行->mWorker.call()->doInBackground(mParams)->postResult(result)->onPostExecute(result)或者onCancelled(result)`
