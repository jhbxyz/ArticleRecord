# AsyncTask 的基础用法和源码分析

AsyncTask内部封装了Thread和Handler，可以让我们在后台进行计算并且把计算的结果及时更新到UI上，而这些正是Thread+Handler所做的事情，没错，AsyncTask的作用就是简化Thread+Handler，让我们能够通过更少的代码来完成一样的功能。



### AsyncTask的基础用法

模拟一个后台请求，来分别解释各个方法的含义

```java
public class AsyncTaskActivity extends AppCompatActivity {
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
      //1
        MyAsyncTask asyncTask = new MyAsyncTask();
      //2
        asyncTask.execute();
    }

    class MyAsyncTask extends AsyncTask<Void, Integer, Integer> {
        private int value;

        @Override
        protected void onPreExecute() {
            super.onPreExecute();
            //开始前
            value = 0;
        }

        @Override
        protected Integer doInBackground(Void... voids) {
            //模拟耗时
            for (int i = 0; i < 100; i++) {
                value++;
                //通知更新
                publishProgress(value);
            }
            return value;
        }

        @Override
        protected void onProgressUpdate(Integer... values) {
            super.onProgressUpdate(values);
            //更新进度
            System.out.println("当前进度为: " + values[0]);
        }


        @Override
        protected void onPostExecute(Integer result) {
            super.onPostExecute(result);
            //最终结果
            System.out.println("结果为: " + result);
        }
      
        @Override
        protected void onCancelled() {
            super.onCancelled();
        }

    }
}
```

注释 1：实例化 MyAsyncTask 对象

注释 2：调用 MyAsyncTask 的 `execute`方法

说明一下 MyAsyncTask 类的各个回调方法

* onPreExecute：开始前执行，做一些初始化操作
* doInBackground：后台运行，执行耗时操作
* onProgressUpdate：更新进度的回调方法，通过 `publishProgress`配合使用
* onPostExecute：结果回调，处理操作UI
* onCancelled：用户调用取消时

> 除了`doInBackground`方法，运行在子线程，其余方法全都运行在主线程。











注释 1：

注释 2：

注释 3：

注释 4：

注释 5：

