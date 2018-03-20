#   HandlerThread源码分析
1.  HandlerThread的基本使用;
```
    //1.创建一个HandlerThread.并开启线程
    HandlerThread handlerThread = new HandlerThread("fxj");
    handlerThread.start();
    //2.使用HandlerThread内部的looper创建handler
    Handler handler = new Handler(handlerThread.getLooper());
    //3.通过handler发送消息.该消息会在我们创建的名为fxj的线程中执行.
    handler.post(new Runnable() {
       @Override
       public void run() {
           Log.d("fxj","currentThread1:"+Thread.currentThread());
           SystemClock.sleep(5000);
       }
    });
```
2.  HandlerThread的创建:两个构造方法
 ```
    //1.指定创建的线程名,线程优先级默认
    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }

    //2.指定线程名以及线程优先级
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }
```
3.  HandlerThread的run方法:因为HandlerThread是继承Thread.所以start方法最终会调用run方法:
```
    public void run() {
        mTid = Process.myTid();
        //创建Looper对象,并存入本地线程变量中.
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        //设置线程优先级.线程优先级从高到低:-20到19.对于默认而言是0.用户交互的前台线程:THREAD_PRIORITY_FOREGROUND = -2
        Process.setThreadPriority(mPriority);
        //执行looper准备好的回调.当继承HandlerThread时,可以覆写该方法接收相应停止.
        onLooperPrepared();
        //开始循环取消息
        Looper.loop();
        mTid = -1;
    }
```
4. HandlerThread的退出.
```
    一个线程要退出,就是run方法执行完.因此要退出线程实际还是通过looper控制退出循环.

    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }

    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }
```
5.  Looper的quit和quitSafely方法:
```
    looper.quit方法其内部调用的都是mQueue.quit(false);
    looper.quitSafely内部调用的是mQueue.quit(true)
```
6.  MessageQueue的quit方法:
```
    void quit(boolean safe) {
        if (!mQuitAllowed) {//mQuitAllowed控制是否循序退出,在创建时指定.
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) {//如果正在或已经退出则不重复执行.
                return;
            }
            mQuitting = true;

            //对于安全退出.会移除当前时间点之后的消息.对于当前时间点之前的消息会让其执行完.
            if (safe) {
                removeAllFutureMessagesLocked();
            } else {
            //对于非安全退出.会直接移除所有消息.
                removeAllMessagesLocked();
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
            //唤醒线程.在MessageQueue再次执行next时:因为mQuitting为true.会执行dispose操作.并返回null到Looper.loop中.
            nativeWake(mPtr);
        }
    }
```
7.  MessageQueue的next方法:
```
   Message next() {
        .....
        // Process the quit message now that all pending messages have been handled.
        //在这之前,如果之前消息已经全部执行完(对于安全退出而言).对于直接退出而言,直接没有消息.
        if (mQuitting) {
            dispose();
            return null;
        }
        .....
    }
```
8.  Looper的loop方法:
```
 public static void loop() {
        .....
        for (;;) {
            Message msg = queue.next(); // might block
            //当前返回的msg为null时,会退出循环.
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
            ......
        }
    }
```
9.  当Looper退出死循环后,HandlerThread线程即会运行结束.