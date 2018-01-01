# IntentService
IntentService是一种特殊的Service，它继承了Service并且它是一个抽象类，可以用于执行后台耗时的任务，当任务执行后它会自动停止，同时由于IntentService是服务的原因，它的优先级要比单纯的线程高，所以IntentService比较适合执行一些高优先级的后台任务。它的onCreate方法如下：

```
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.

        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

```

ServiceHandler的定义如下：

```
    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
```

IntentService内部封装了HanderThread和Handler，当它被第一次启动时，它的onCreate方法会被调用，onCreate方法会创建一个HandlerThread，然后使用它的Looper来构造一个Handler对象mServiceHandler，这样通过mServiceHandler发送的消息最终都会在HandlerThread中执行。
每次启动IntentService，它的onStartCommand方法就会调用一次，IntentService在onStartCommand中处理每个后台任务的Intent。onStartCommand方法调用了onStart，onStart如下：

```
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

```

IntentService通过mServiceHandler发送了一条消息，mServiceHandler收到消息后，会将Intent对象传递给onHandleIntent处理。当onHandleIntent方法结束后，IntentService会通过stopSelf(int startId)来尝试停止服务。stopSelf(int startId)会等待所有的消息都处理完毕后才终止服务；stopSelf()则会立即停止服务。
IntentService的onHandlerIntent方法是一个抽象方法，它的作用是从Intent的参数中区分具体的任务并执行这些任务。如果当前只存在一个任务，那么onHandleIntent方法执行完这个任务后，stopSelf(int startId)就会直接停止服务；如果目前存在多个任务，那么当onHandleIntent方法执行完最后一个任务，stopSelf(int startId)才会直接停止服务。