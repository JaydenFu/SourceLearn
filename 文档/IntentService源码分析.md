#   IntentService源码分析
官方解释:IntentService是一个用来处理异步请求的service.通过startService开启IntentService.然后IntentService会在一个工作线程中按顺序处理接受的Intent.
当工作结束了.IntentService会结束掉自己.

1.  IntentService的onCreate方法:
````
    @Override
    public void onCreate() {
        super.onCreate();
        //创建一个HanderThread来作为工作线程.
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        //获取和工作线程关联的looper对象.并通过该looper构造处理Intent的Handler.
        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
````
2.  IntentService的onStart方法和onStartCommand方法;
```
    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        //当接收到Intent时,会构造一条消息.将Intent存在msg.obj字段中.发送给工作线程关联的消息队列.并通过looper取消息.然后交给相应的handler处理.
        //这里因为需要切换线程.onStart是在UI线程中调用.所以通过消息机制,将数据传送到工作线程.
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    //每次startService方法,onStartCommand都会被回调.
    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        //在这里简直的调用onStart方法,向消息队里中插入消息.
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }
```
3.  IntentService的mServiceHandler
```
    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            //通过onHandleIntent方法处理消息.该方法是我们继承IntentService时需要覆写的方法.
            onHandleIntent((Intent)msg.obj);
            //当处理完消息后,会执行stopSelf停掉当前service.这里msg.arg1代表的是onStartCommend方法中的startId参数.
            //startId.代表着Service被start了多少次.这里startId要对得上,才能停止掉Service.所以如果在任务处理过程中.又
            //调用的startService开启当前IntentService.则前面的stopSelf是停止不掉服务的.
            stopSelf(msg.arg1);
        }
    }
```
