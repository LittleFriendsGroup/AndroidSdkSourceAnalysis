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
//串行实例化
new DownAsyncTask().extcute();
//并行实例化
new DownAsyncTask().executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR,"");
```
2、点击查看，[代码案例](https://github.com/white37/AndroidSdkSourceAnalysis/blob/master/source/AsyntaskActivity.java)
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
* 如果线程处于休眠状态，为true则正在执行的线程将会中断，抛出异常，但执行的任务线程会继续执行完毕调用`onCanceled()`。为false则正在执行的线程不会中断，任务线程执行完毕调用`onCanceled()`。
* 如果线程不处于休眠状态，为true和false都没有区别，任务线程执行完毕后调用`onCanceled()`。
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
            //异步任务执行的时候回调call方法
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                //设置线程的优先级
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                //将结果发送出去
                return postResult(result);
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            //任务执行完毕后会调用done方法
            @Override
            protected void done() {
                try {
                    //get()表示获取mWorker的call的返回值，即Result.然后看postResultIfNotInvoked方法
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
AsyncTask的实例化在UI线程中。构造函数初始化了两个成员变量mWorker和mFuture。mWorker为WorkerRunnable类型的匿名内部类实例对象（实现了Callable接口），mFuture为FutureTask类型的匿名内部类实例对象，将mWorker作为mFuture的形参（重写了FutureTask类的done方法）。
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
* 
```java
public class FutureTask<V> implements RunnableFuture<V>//java
实现了RunnableFuture接口
```
FutureTask实现了接口Runnable，，它既可以作为Runnable被线程执行，同时将Callable作为构造函数的参数传入，这样组合的好处是，假设有一个很耗时的返回值需要计算，并且这个返回值不是立刻需要的话，就可以使用这个组合，用另一个线程去计算返回值，而当前线程在使用这个返回值之前可以做其它的操作，等到需要这个返回值时，再通过Future得到。FutureTask的run方法要开始回调WorkerRunable的call方法了，call里面调用doInBackground(mParams),终于回到我们后台任务了，调用我们AsyncTask子类的`doInBackground()`,由此可以看出`doInBackground()`是在子线程中执行的，如下图所示
![](https://github.com/white37/AndroidSdkSourceAnalysis/blob/master/images/FutureTask(run).png)

### 3.2、核心方法
1、execute()方法
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

2、executeOnExecutor()方法
```java
//exec执行AsyncTask.execute()方法时传递进来的参数sDefaultExecutor，这个sDefaultExecutor其实就是SerialExecutor对象。
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        //如果一个任务已经进入执行的状态，再执行就会抛异常。这就决定了一个AsyncTask只能执行一次
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
	//一旦executeOnExecutor调用了就标记为运行状态
        mStatus = Status.RUNNING;
	//实际是调用子类里面的onPreExecute
        onPreExecute();
	//将处理的参数类型赋值给mWorker
        mWorker.mParams = params;
        //execute是调用SERIAL_EXECUTOR的execute，mFuture就是之前AsyncTask构造初始化赋值的FutureTask。
        exec.execute(mFuture);
        return this;
    }

```
这里要说明一下，AsyncTask的异步任务有三种状态
* PENDING 待执行状态。当AsyncTask被创建时，就进入了PENDING状态。
* RUNNING 运行状态。当调用executeOnExecutor，就进入了RUNNING状态。
* FINISHED 结束状态。当AsyncTask完成(用户cancel()或任务执行完毕)时，就进入了FINISHED状态。
3、SerialExecutor的execute方法
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
                    	//无论执行结果如何都会取出下一个任务执行
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
exec.execute(mFuture)执行时，SerialExecutor将FutureTask作为参数执行execute方法。在execute方法中，假设FutureTask插入进了两个以上的任务队列到mTasks中，第一次过来mActive==null，通过`mTasks.poll()`取出一个任务丢给线程池运行，线程池执行r.run，其实就是执行FutureTask的run方法，因为传递进来的r参数就是mFuture。等到上一个线程执完r.run()完之后，这里是通过一个try-finally代码块，并在finally中调用了scheduleNext()方法，保证无论发生什么情况，scheduleNext()都会取出下一个任务执行。接着因为mActive不为空了，不会再执行``scheduleNext()`，由此可知道这是一个串行的执行过程，同一时刻只会有一个线程正在执行，其余的均处于等待状态。


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
可以看到如果回调了`call()`方法，就会调用了`doInBackground(mParams)`方法，这都是在子线程中执行的。执行完后，将结果通过`postResult(result)`发送出去。

4、AsyncTask的postResult方法
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
因为`postResult(Result result)`还是在子线程中调用的，如果要发送给主线程，必须通过Handler。源码中使用sHandler并带着MESSAGE_POST_RESULT和封装了任务执行结果的对象AsyncTaskResult，然后message.sendToTarget()开始发消息。
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
并在InternalHandler的handleMessage中开始处理消息，InternalHandler的源码如下所示：
```java
     private static class InternalHandler extends Handler {
        public InternalHandler() {
            // 这个handler是关联到主线程的
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
* 这里根据消息的类型进行了判断，如果是MESSAGE_POST_RESULT消息，就会去执行finish()方法，`finish()`源码如下文所示：
```java
private void finish(Result result) {  
    if (isCancelled()) {  
        onCancelled(result);  
    } else {  
        onPostExecute(result);  
    }  
    mStatus = Status.FINISHED;  
} 
```
如果任务已经取消了，调用`onCancelled`方法，如果没被取消，则调用onPostExecute()方法。
* 如果`doInBackground(Void... params)`调用`publishProgress()`方法，实际就是发送一条MESSAGE_POST_PROGRESS消息，就会去执行onProgressUpdate()方法。`publishProgress()`的源码如下文所示：
```java
    @WorkerThread
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
```
如果你还不够清晰，请看下面的这个流程图。
![](https://github.com/white37/AndroidSdkSourceAnalysis/blob/master/images/AsyncTask%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

## AsyncTask需要注意的坑
* AsyncTask的对象必须在主线程中实例化，execute方法也要在主线程调用
* AsyncTask任务只能被执行一次，即只能调用一次execute方法，多次调用时将会抛异常
* cancel()方法无法直接中断子线程，只是更改了中断的标志位。控制异步任务执行结束后不会回调onPostExecute()。正确的取消异步任务要cancel()方法+doInbacground()做判断跳出循环
