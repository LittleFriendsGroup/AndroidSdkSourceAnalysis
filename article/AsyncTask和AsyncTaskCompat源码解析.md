## 一、AsyncTask简介 
### 1.1、定义
 AsyncTask能让我们更方便容易的使用UI线程，这个类允许我们在UI线程执行后台耗时的操作和发布处理结果都在UI线程上。也就是你使用AsyncTask的时候你将操作到任何关于线程的代码。
### 1.2、背景
 执行异步耗时的任务的时候，初学者一般想到使用线程和Handler完成异步处理操作，但这样会导致一些问题，当开发中需要创建N个线程时，你可能会new N个Thread出来。过多的线程创建出来又缺乏统的管理，性能开销大，甚至你的activity销毁没及时取消停止线程的运行，你创建的线程仍然有可能在后台运行。为了更好的控制、提供性能，Android给我们提供了一个实现异步处理任务的工具类AsyncTask。
 
### 1.3、AysncTask优点[来源于anany的总结](http://anany.me/2015/08/31/2015_8_31_asynctask/)  

    内部采用线程池机制，统一管理线程池
    提供自定义的线程池，实现多个线程顺序同步执行，异步并发执行
    提供回调方法，及时监控后台执行任务的进度，更新主线程的UI控件以及获取异步执行结果
## 二、AsyncTask用法
### 2.1、AsyncTask是抽象类，需要被子类继承，需要指定三个泛型的参数
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
（4）onPostExecute<br>当doInBackGround完成后，系统会自动调用，销毁一些dialog的操作，并将doInBackGround﻿方法返回的值传给该方法

###2.3、创建自定义的AsyncTask，实现抽象方法

```java
public class AsyntaskActivity extends AppCompatActivity {

    private ProgressDialog progressDialog;
    private int progress = 0;
    private int MAX_PROGRESS = 100;
    private TextView tvProgress;
    private DownAsynTask task;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_asyntask);
        tvProgress = (TextView) findViewById(R.id.tvProgress);
        progressDialog = new ProgressDialog(this);
        task = new DownAsynTask();
        task.execute();
    }

    class DownAsynTask extends AsyncTask<Void, Integer, Boolean> {

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
![](https://github.com/white37/AndroidSdkSourceAnalysis/blob/master/images/Asyntask.png)

例子中doInBackground返回的结果Boolean最终会传递给onPostExecute(Boolean aBoolean)，而这一系列的操作是由AsyncTask底层实现的，通过handler发结果发送到主线程

###2.4、取消异步任务
AsyncTask是可以在任何时候被取消的，开篇我提到了当你activity销毁了，我们需要及时的释放资源，包括AsyncTask，你也可以取消正在执行的异步任务
```java
AsyncTask.cancel(mayInterruptIfRunning);
```
mayInterruptIfRunning是boolean类型的，可以是true，也可以是false,他们两个有什么区别呢，调用了cancel能马上取消doInBackground里面还在运行的异步任务，你可能会这样想<br>
我做了一个实验，当我设置AsyncTask.cancel(true)
![](https://github.com/white37/AndroidSdkSourceAnalysis/blob/master/images/cancel(true).png)
当我在progress更新到1000的时候我点击了，AsyncTask.cancel(true)，界面已经不在更了，但是Log还是会继续累加progress，一直输出到20000，执行完doInBackground才调用onCanceled方法<br>
当我设置AsyncTask.cancel(false)
当我在progress更新到1000的时候我点击了，AsyncTask.cancel(false)，界面已经不在更新了，但是Log还是会继续累加progress，一直输出到20000，执行完doInBackground才调用onCanceled方法<br>
那到底AsyncTask.cancel(true/false)有什区别呢，听我慢慢到来<br>
在你的doInBackground里面没有下面代码的时候，无论true或者false都是一样结果，也就是，界面已经不在更新了，但是doInBackground会继续累加progress，一直输出到20000，执行完doInBackground才调用onCanceled方法<br>
```java
 try {
         Thread.sleep(2000);
     } catch (InterruptedException e) 
     {
         e.printStackTrace();
     }
```

在你的doInBackground里面有上述休眠的代码时候<br>
* AsyncTask.cancel(true)出现的结果是界面已经不在更新了，但是Log还是会继续累加progress，期间不会抛出中断的异常，一直输出到20000，执行完doInBackground才调用onCanceled方法<br>

* AsyncTask.cancel(false)出现的结果是界面已经不在更新了，但是Log还是会继续累加progress，先抛出中断的异常，再一直累加输出到20000，执行完doInBackground才调用onCanceled方法,<br>
![](https://github.com/white37/AndroidSdkSourceAnalysis/blob/master/images/cacel(false).png)





