# ThreadPool

## ThreadPool
### 优点
1. 重用线程池中的线程，避免线程创建和销毁带来的性能开销。
2. 能有效控制线程池的最大并发数，避免大量线程之间因互相抢占系统资源而导致的阻塞现象。
3. 能够对线程进行简单的管理，并且提供定时执行以及指定间隔循环执等功能。

## ThreadPoolExecutor
线程池的真正实现。


``` 
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
    }
```

* corePoolSize：核心线程数，即使空闲状态也一直存活，除非allowCoreThreadTimeOut设置为true
* maximumPoolSize：线程池容纳的最大线程数，当活动线程数达到这个数值之后，后续的新任务会阻塞
* keepAliveTime：非核心线程闲置时的超时时长，超时后，会被回收
* workQueue：线程池中的任务队列
* threadFactory：线程工厂，为线程池提供新线程的功能。
* handler：当线程池无法执行新任务时，这可能是由于任务队列已满或者是无法成功执行任务，这个时候ThreadPoolExecutor会调用handler的rejectedExecution方法来通知调用者。

### ThreadPoolExecutor执行任务时大致遵循如下规则：

1. 如果线程池中的线程数量未达到核心线程的数量，那么会直接启动一个核心线程来执行任务。
2. 如果线程池中的线程数量已经达到或者超过核心线程的数量，那么任务会被插入到任务队列中排队等待执行。
3. 如果步骤2中无法将任务插入到任务队列中，这往往是由于任务队列已满，这个时候如果线程数量未达到线程池规定的最大值，那么会立即启动一个非核心线程来执行任务。
4. 如果步骤3中线程数量已经达到线程池规定的最大值，那么就拒绝执行此任务，ThreadPoolExecutor会调用handler的rejectedExecution方法来通知调用者。

### AsyncTask中线程池的应用：

```
public abstract class AsyncTask<Params, Progress, Result> {
    private static final String LOG_TAG = "AsyncTask";

    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    // We want at least 2 threads and at most 4 threads in the core pool,
    // preferring to have 1 less than the CPU count to avoid saturating
    // the CPU with background work
    private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    private static final int KEEP_ALIVE_SECONDS = 30;

    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    /**
     * An {@link Executor} that can be used to execute tasks in parallel.
     */
    public static final Executor THREAD_POOL_EXECUTOR;

    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }

```

### 线程池的分类：

1.FixedThreadPool

> 线程数量固定的线程池，当线程处于空闲状态时，他们并不会被回收，除非线程池被关闭。当所有线程都处于活动状态时，新任务都会处于等待状态，直到有线程空闲出来。

```

  public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    } 
    
```


2.CachedThreadPool
>线程数量不定的线程池，最大线程数为Integer.MAX_VALUE，这类线程池比较适合执行大量的耗时较少的任务。

```

    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

3.ScheduledThreadPool

>它的核心线程数量是固定的，而非核心线程数是没有限制的，并且当前非核心线程闲置时会被立即回收。主要用于执行定时任务和具有固定周期的重复任务。

```
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
        DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,new DelayedWorkQueue());
    }
```

4.SingleThreadExecutor
>线程池内部只有一个核心线程，它确保所有的任务都在同一个线程中按顺序执行。统一所有的外界任务到一个线程中，使得这些任务之间不需要处理线程同步的问题。

```
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```