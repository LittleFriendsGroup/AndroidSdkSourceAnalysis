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
### 2.1、创建自定义的AsyncTask
 ```java
 public abstract class AsyncTask<Params, Progress, Result>//java
 ```
 * Params     
 异步任务处理的参数
 * Progress
 异步任务执行过程中返回给主线程的进度值，通过publishProgress()方法发送出去
 * Result
 异步任务执行结束返回的结果的结果类型，可以为boolean，或者bitmap等类型

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
