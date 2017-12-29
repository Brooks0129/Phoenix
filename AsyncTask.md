# AsyncTask

## 简介
AsyncTask是一个轻量级的异步任务类。它可以在线程池中执行后台任务，然后把执行的进度和最终结果传递给朱线程中更新UI。从实现上来说，AsyncTask封装了Thread和Handler，通过AsyncTask可以更加方便地执行后台任务以及在主线程中访问UI，但是AsyncTask并不适合进行特别耗时的后台任务。
```
public abstract class AsyncTask<Params, Progress, Result> {}
```

三种泛型类型分别代表“启动任务执行的输入参数”、“后台任务执行的进度”、“后台计算结果的类型”。在特定场合下，并不是所有类型都被使用，如果没有被使用，可以用java.lang.Void类型代替。

一个异步任务的执行一般包括以下几个步骤：

1. execute(Params... params)，执行一个异步任务，需要我们在代码中调用此方法，触发异步任务的执行。

1. onPreExecute()，在execute(Params... params)被调用后立即执行，一般用来在执行后台任务前对UI做一些标记。

1. doInBackground(Params... params)，在onPreExecute()完成后立即执行，用于执行较为费时的操作，此方法将接收输入参数和返回计算结果。在执行过程中可以调用publishProgress(Progress... values)来更新进度信息。

1. onProgressUpdate(Progress... values)，在调用publishProgress(Progress... values)时，此方法被执行，直接将进度信息更新到UI组件上。

1. onPostExecute(Result result)，当后台操作结束时，此方法将会被调用，计算结果将做为参数传递到此方法中，直接将结果显示到UI组件上。 


## 源码解析：
从AsyncTask的execute方法开始：

```

    /**
     * Executes the task with the specified parameters. The task returns
     * itself (this) so that the caller can keep a reference to it.
     * 
     * <p>Note: this function schedules the task on a queue for a single background
     * thread or pool of threads depending on the platform version.  When first
     * introduced, AsyncTasks were executed serially on a single background thread.
     * Starting with {@link android.os.Build.VERSION_CODES#DONUT}, this was changed
     * to a pool of threads allowing multiple tasks to operate in parallel. Starting
     * {@link android.os.Build.VERSION_CODES#HONEYCOMB}, tasks are back to being
     * executed on a single thread to avoid common application errors caused
     * by parallel execution.  If you truly want parallel execution, you can use
     * the {@link #executeOnExecutor} version of this method
     * with {@link #THREAD_POOL_EXECUTOR}; however, see commentary there for warnings
     * on its use.
     *
     * <p>This method must be invoked on the UI thread.
     *
     * @param params The parameters of the task.
     *
     * @return This instance of AsyncTask.
     *
     * @throws IllegalStateException If {@link #getStatus()} returns either
     *         {@link AsyncTask.Status#RUNNING} or {@link AsyncTask.Status#FINISHED}.
     *
     * @see #executeOnExecutor(java.util.concurrent.Executor, Object[])
     * @see #execute(Runnable)
     */
    @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
    
    @MainThread
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
        onPreExecute();
        mWorker.mParams = params;
        exec.execute(mFuture);
        return this;
    }
```

execute方法和executeOnExecutor方法必须在主线程中调用，在Android1.6之前，AsyncTask是串行执行任务的。1.6开始采用线程池处理并行任务，但是从3.0开始，为了避免AsyncTask所带来的并发错误，Asynctask又采用了一个线程来串行执行任务。尽管如此，在Android3.0以及以后，我们可以通过AsyncTask的executeOnExecutor方法来并行的执行任务。
execute方法调用了executeOnExecutor方法，sDefaultExecutor是一个串行的线程池，它的定义如下：

```
    /**
     * An {@link Executor} that executes tasks one at a time in serial
     * order.  This serialization is global to a particular process.
     */
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```


在executorOnExecutor中，onPreExecute方法最先执行，然后线程池开始执行。

```
  private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }

```

//ToDo FutureTask解析

AsyncTask的Params参数封装为FutueTask对象，FutureTask是一个并发类。SerialExecutor的execute方法首先会吧FutureTask对象插入到任务队列mTasks中，如果这个时候没有正在活动的AsyncTask任务，那么就会调用SerialExecutor的scheduleNext来执行下一个AsyncTask任务，同时当一个AsyncTask任务执行完成后，AsyncTask会继续执行其他任务直到所有的任务都会被执行，默认情况下，AsyncTask是串行执行的。


AsyncTask中有两个线程池（SerialExecutor和`THREAD_POOL_EXECUTOR`）和一个Handler（InternalHandler），其中线程池SerialExecutor用于任务的排队，而线程池`THREAD_POOL_EXECUTOR`用于真正的执行任务，InternalThread用于将执行环境从线程池切换到主线程。

由于FutureTask的run方法会调用mWorker的call方法，因此mWorker的call方法最终会在线程池中执行。

```
       mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };
```


在mWorker的call方法中，首先mTaskInvoked设为true，表示当前任务已经被调用过了，然后执行doInBackground方法，接着将其返回值传递给postResult方法。

```
    private static InternalHandler sHandler;
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }

    private Handler getHandler() {
        return mHandler;
    }
    

      
```

mHandler赋值如下：

```
    mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);
    private static Handler getMainHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler(Looper.getMainLooper());
            }
            return sHandler;
        }
    }

```


我们这里获取的mHandler一定是主线程的handler的吗？通过

```
public AsyncTask(@Nullable Looper callbackLooper) {
    mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);
    ...
}
            
```
我们可以看到如果当前的callbackLooper为null或者是主线程的Looper的话，那么对应的handler就是主线程的handler，callbackLooper有没有可能为其他值呢？

```
    /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     *
     * @hide
     */
    public AsyncTask(@Nullable Handler handler) {
        this(handler != null ? handler.getLooper() : null);
    }
```

该构造方法并不对外开放，所以handler一定为主线程的handler。这也就做到了线程的切换。

InternalHandler的定义如下：

```
    private static class InternalHandler extends Handler {
        public InternalHandler(Looper looper) {
            super(looper);
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

当sHandler收到`MESSAGE_POST_RESULT`消息时，会调用AsyncTask的finish方法。

```

    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```


## 注意事项：
1.AsyncTask中的所有任务将以串行的方式执行；

2.How to cancel task？
调用cancel方法：

	```
	public final boolean cancel(boolean mayInterruptIfRunning) {
        mCancelled.set(true);
        return mFuture.cancel(mayInterruptIfRunning);
    }
	```
	
	```
	// 终止更新进度
	    @WorkerThread
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
    // 回调onCancelled方法
    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
	```
`mFuture.cancel(mayInterruptIfRunning);`会调用线程的interrupt方法。即使我们正确地调用了cancle() 也未必能真正地取消任务。因为如果在doInBackgroud里有一个不可中断的操作，比如BitmapFactory.decodeStream()，那么这个操作会继续下去。

   ```
	@Override
	public void onProgressUpdate(Integer... value) {
	   // 判断是否被取消
	   if(isCancelled()) return;
		.........
	}
	
	@Override
	protected Integer doInBackground(Void... mgs) {
		// Task被取消了，马上退出
		if(isCancelled()) return null;
		}
	
   ```

3.内存泄漏

 如果AsyncTask被声明为Activity的非静态的内部类，那么AsyncTask会保留一个对创建了AsyncTask的Activity的引用。如果Activity已经被销毁，AsyncTask的后台线程还在执行，它将继续在内存里保留这个引用，导致Activity无法被回收，引起内存泄露。
 
 
 
4.Android中线程对比

		
		Thread name         | Function         |
		--------------------|------------------|
		AsyncTask | Helps get work on/off the UI thread   | 
		HandlerThread       | Dedicated thread for API callbacks   |
		[ThreadPool](https://github.com/Brooks0129/Phoenix/blob/master/ThreadPool.md)  | Running lots of parallel work      | 
		IntentService     | Helps get intents off the UI thread  | 




