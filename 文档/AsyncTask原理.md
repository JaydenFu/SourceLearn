#   AsyncTask原理学习
AsyncTask可以简单的让开发者在后台线程执行操作.然后将结果传输到UI线程.一个AsyncTask只可以被执行一次.否则会抛出异常
AsyncTask适合用来处理在后台执行的操作时间比较短的过程.如果是长时间的执行,还是更建议使用线程池.

AsyncTask本身是个抽象类.所以需要继承之后才能使用.通常继承类需要实现doInBackground方法来执行自己的操作.以及onPostExecute方法来执行接收到结果后的操作.

简单介绍:
```
   //1.创建自己的Task子类,在doInBackground中执行操作.并返回需要传递会UI线程的结果
 * private class DownloadFilesTask extends AsyncTask<URL, Integer, Long>; {
 *     protected Long doInBackground(URL... urls) {
 *         int count = urls.length;
 *         long totalSize = 0;
 *         for (int i = 0; i < count; i++) {
 *             totalSize += Downloader.downloadFile(urls[i]);
 *             publishProgress((int) ((i / (float) count) * 100));
 *             // Escape early if cancel() is called
 *             if (isCancelled()) break;
 *         }
 *         return totalSize;
 *     }
 *
 *     protected void onProgressUpdate(Integer... progress) {
 *         setProgressPercent(progress[0]);
 *     }
 *
 *     protected void onPostExecute(Long result) {
 *         showDialog("Downloaded " + result + " bytes");
 *     }
 * }

    //2.创建实例,调用excute开始执行.
    new DownloadFilesTask().execute(url1, url2, url3);
```

1.  AsyncTask使用了泛型.Params代表执行时传入的参数类型.Progress代表进度类型.Result代表返回的结果值类型.
```
public abstract class AsyncTask<Params, Progress, Result>
```

2.  一个AsyncTask被执行了.会相应的有4个方法被回调
```
    *   onPreExecute(): 该方法会在任务开始执行之前,在UI线程被回调.该方法通常被用来做一些任务开始前的准备工作,比如在界面上展示一个进度条
    *   doInBackground():   发方法会在onPreExecute执行完后,立即执行.该方法执行在后台线程.该方法中主要用来执行后台耗时任务.
                        在该方法内部,可以通过调用publishProgress用来通知外界任务执行的进度.
    *   onPressgressUpdate():   当在doInBackground中调用了publishProgress方法.该方法就会在主线程中被回调.
    *   onPostExecute():    当doInBackground方法在后台线程中执行完成,该方法会在UI线程中被回调.doInBackground中返回的值,会被作为参数
                            传给onPostExecute.
```
3.  取消任务
```
    通过调用AsyncTask的cancle(boolean)方法来取消任务的执行.当该方法被调用了后,AsyncTask的isCancelled()方法被返回为true.
    当调用的cancle后,doInBackground方法执行后,onPostExecute方法不会被回调.而是回调onCancelled方法.
```

源码分析:
1.  AsyncTask的初始化
```
   //1.创建WorkerRunnable实例.实现了Callable接口.WorkRunnable只有一个成员变量.Params[] 数组mParams.
       Callable接口和Runnable接口很想.都是用来当做线程中执行的任务.但是Callback有返回值.Runnable没有返回值.
   public AsyncTask() {
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

        //2.创建FutureTask实例.该实例包裹了前面创建的WorkerRunnable实例
            FutureTask可以异步执行一些任务
            FutureTask实现了RunnableFuture接口.而RunnableFuture接口继承了Runnable和Future两个接口
            Future接口代表一个异步计算的结果.提供了isDone方法检查计算的结果.get方法获取计算结果.
            get方法会阻塞,直到计算完成.当计算完成了,计算就不能被cancel.

        mFuture = new FutureTask<Result>(mWorker) {
            //done方法会在FutureTask的finishComplete方法中被调用.
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


2.  AsyncTask.execute方法:
```
    //直接调用execute方法其实内部调用的是executeOnExecutor方法.我们自己同样的也可以直接调用executeOnExecutor方法.
    //二者区别:
        1.excute方法:使用内部默认的线程池.单个线程串行执行任务
        2.executeOnExcutor.可以自己指定线程池.从而并行执行任务

    //该线程池是静态的.也就是说任何AsyncTask实例都是用的同一个线程池.
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
```
3.  先分析SERIAL_EXECUTOR这个默认采用的线程池.对于多个AsyncTask实例调用了execute方法时,按串行方式执行这些任务.
```
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

    private static class SerialExecutor implements Executor {
        //是一个任务队列
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            //当线程池接收到要执行一个Runnable时.首先是将该Runnable封装成一个新的Runnable,并添加到任务队列中.
            //在封装的Runnable的run方法内部调用任务Runnable的run方法.然后需要调用scheduleNext方法.
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            //判断当前是否有任务在执行.如果没有就调用下一个任务.
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            //调度任务队列中的下一个任务.如果不为null.就赋值给mActive.并且使用THREAD_POOL_EXECUTOR执行该任务.
            //其实这里是用THREAD_POOL_EXECUTOR执行任务,THREAD_POOL_EXECUTOR是个线程池,可以并行执行任务,但是在这里也并
            //没有卵用.因为SerialExecutor控制只能串行执行任务.也就是说当一个任务执行完成之后,采用将下一个任务
            //通过THREAD_POOL_EXECUTOR来执行(THREAD_POOL_EXECUTOR不一定分配同一个线程来执行所有任务).
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```

4.  THREAD_POOL_EXECUTOR: 这是一个可以并行执行任务的线程池.在AsyncTask内部是静态的.所以对于任何AsyncTask实例都是使用的同一个线程池.
```
    //线程池的内容单独学习
    public static final Executor THREAD_POOL_EXECUTOR;
    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);
    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    static {
        //CORE_POOL_SIZE:   2个到4个之间.
        //MAXIMUM_POOL_SIZE;    CPU个数*2+1,线程池中允许的最大线程数量.
        //KEEP_ALIVE_SECONDS:   30秒
        //sPoolWorkQueue:是一个最大容量128的队列,表示等待队列,如果线程池中的线程数量已经最大数量了.并且所有线程都在执行,
        //那么,任务会被保存在这个工作队列中.这个工作队列最大只能保存128个.当超出128个的时候,就会抛出异常.
        //sThreadFactory:   创建线程的工厂类.
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }
```
5.  再回到executeOnExecutor方法上:
```
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        //mStatus标识当前AsyncTask的状态.
        //Status.PENDING:未执行
        //Status.RUNNING:正在执行
        //Status.FINISHED:执行完成

        //当一个AsyncTask的状态不是未执行状态.就会抛出异常.因此一个AsyncTask只可以执行一次.
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

        //将当前AsyncTask的状态标识位正在执行状态.
        mStatus = Status.RUNNING;

        //回调通知即将执行任务.
        onPreExecute();

        //将参数赋值给WorkerRunnable的成员变量mParams 数组.
        mWorker.mParams = params;
        //采用参数中传入的线程池执行mFuture.
        //如果我们执行执行的AsyncTask的execute方法.那这里exec是默认的SerialExecutor.
        //该方法会将mFuture添加到SerialExecutor的mTask队列中.然后串行依次执行.
        exec.execute(mFuture);

        return this;
    }
```
6.  FutureTask的run方法:
```
     public void run() {
         //检测状态是否为初始状态.否则不可以在继续执行.
         //检测runner是否为null.如果不为null则不执行后续内容.
         //同时这里还是当前线程赋值给runner.
         //这是为了阻止并发的执行该FutureTask的run方法.
         if (state != NEW ||
             !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
             return;

         try {
             Callable<V> c = callable;
             //state为NEW,也就是说当前FutureTask没有被取消
             if (c != null && state == NEW) {
                 V result;
                 boolean ran;
                 try {
                    //执行Callable的call.其实就是执行任务.这里的Callable其实是我们初始化AsyncTask时的WorkerRunnable.
                     result = c.call();
                     //标识执行完成.
                     ran = true;
                 } catch (Throwable ex) {
                    //在任务处理过程中发生异常的情况
                     result = null;
                     ran = false;
                     setException(ex);
                 }
                 //如果执行顺利完成.则将调用set方法将Callable返回的结果设置到outcome变量上.并且将state设置为COMPLETING,
                 //同时调用finishCompletion. finishCompletion方法具体干什么参看后面第12步.
                 if (ran)
                     set(result);
             }
         } finally {
             // runner must be non-null until state is settled to
             // prevent concurrent calls to run()
             runner = null;
             // state must be re-read after nulling runner to prevent
             // leaked interrupts
             int s = state;
             if (s >= INTERRUPTING)
                 handlePossibleCancellationInterrupt(s);
         }
     }


     protected void set(V v) {
         //将STATE状态设置为COMPLETING.
         if (U.compareAndSwapInt(this, STATE, NEW, COMPLETING)) {
             outcome = v;
             //将STATE状态设置为NORMAL.标识执行结束.
             U.putOrderedInt(this, STATE, NORMAL); // final state
             finishCompletion();
         }
     }
```
7.  WorkerRunnable的的call方法:
```
 public Result call() throws Exception {
                //mTaskInvoked是AtomicBoolean.表示更新是原子性的.
                //mTaskInvoked标识当前任务开始执行,这里设置为true.
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    //设置线程优先级为后台线程
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    //调用我们覆写的doInBackground方法.执行具体的任务.
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    //出现异常.会将mCancelled设置为true
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    //只要WorkRunnable的call方法被执行,一定会调用postResult方法.
                    //该方法通过Handler将结果切换到主线程.并且调用onPostExecute方法或者onCancelled方法.
                    postResult(result);
                }
                return result;
            }
```
8.  AsyncTask的postResult方法
```
    private Result postResult(Result result) {
        //通过Handler将当前任务的结果与当前任务所属的AsyncTask实例封装成一个AsyncTaskResult实例.切换到主线程.
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
    //getHandler()方法返回的是单例Handler.而且是与UI线程关联的Handler
    private static Handler getHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler();
            }
            return sHandler;
        }
    }

```
8.  InternalHandler:静态的.所有AsyncTask共享.
```
    private static class InternalHandler extends Handler {
        //这里通过主线程的looper创建Handler.所以该Handler的handleMessage最终是运行在主线程中的
        public InternalHandler() {
            super(Looper.getMainLooper());
        }

        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    //这里result.mTask就是我们AsyncTask实例.执行其finish方法.
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    //回调AsyncTask的onProgressUpdate方法.
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```
9.  AsyncTask的finish方法
```
    private void finish(Result result) {
        if (isCancelled()) {
            //如果前面执行doInBackground过程中出现了异常.
            //或者在执行过程中调用的cancel方法.则回调onCancelled方法
            onCancelled(result);
        } else {
            //如果正常执行,并返回结果.则会调用onPostExecute方法.
            onPostExecute(result);
        }
        //标记当前AsyncTask执行状态为结束.
        mStatus = Status.FINISHED;
    }

```
10. AsyncTask.cancel方法:
```
    //mayInterruptIfRunning是否该任务已经在执行过程中是否打断执行
    public final boolean cancel(boolean mayInterruptIfRunning) {
        //将mCancel标志设置为true.
        mCancelled.set(true);
        //因为实际是将mFuture添加进线程池的.所以要取消执行,需要调用mFuture的cancel方法.
        return mFuture.cancel(mayInterruptIfRunning);
    }
```
11. FutureTask的cancel方法:
```
    public boolean cancel(boolean mayInterruptIfRunning) {
        //当任务的状态为NEW时,也就是说还没开始执行或者未执行完成.则会将标志改为CANCELLED.或者INTERRUPTIING.
        //当任务的状态不为NEW时,也就是说任务已经执行过了.就不会接着处理了.
        if (!(state == NEW &&
              U.compareAndSwapInt(this, STATE, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;

        //如果任务还在执行过程中,根据mayInterruptIfRunning变量决定是否需要中断线程.
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    //在将状态设置为INTERRUPTED.
                    U.putOrderedInt(this, STATE, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
    }
```
12. FutureTask的finishCompletion方法:
```
    private void finishCompletion() {
        // assert state > COMPLETING;
        //判断是否有等待的节点.
        for (WaitNode q; (q = waiters) != null;) {
            //如果有等待节点,则移除节点.并且唤醒响应的等待线程.知道所有节点都被处理.
            if (U.compareAndSwapObject(this, WAITERS, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }


        //执行FutureTask的done方法.在done方法中调用AsyncTask的postResultIfNotInvoked(get());
        done();

        callable = null;        // to reduce footprint
    }

```
13. FutureTask的get方法:
```
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        //判断当前任务的状态是否小于等于COMPLETING. 只有两个状态NEW,COMPLETING满足
        //也就是说任务还没执行完成.
        if (s <= COMPLETING)
            //就调用awaitDone方法阻塞当前线程.等待结果返回.
            s = awaitDone(false, 0L);
        //当返回了状态.则根据相应状态返回结果.
        return report(s);
    }

    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }

    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {

        long startTime = 0L;    // Special value 0L means not yet parked
        WaitNode q = null;
        boolean queued = false;

        //循环执行任务状态的判断.
        for (;;) {
            int s = state;
            //如果状态值大于COMPLETING,直接返回当前节点.
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING)
                //当当前状态时COMPLETING时,说明任务执行完了.正在赋值结果.
                //所以通过Thread.yield方法告诉cpu.暂时自己的线程状态从运行状态调整为就绪状态.等待下一次cpu调度执行.
                Thread.yield();
            else if (Thread.interrupted()) {
                //当线程被打断了.抛出异常,肯定获取不到执行结果了.
                removeWaiter(q);
                throw new InterruptedException();
            }
            else if (q == null) {
                //如果当前q为null.
                //1.设置了超时,但是超时时间却是小于等于0.则立刻返回当前状态.
                //2.其他情况,都会创建一个等待节点赋值给q.
                if (timed && nanos <= 0L)
                    return s;
                q = new WaitNode();
            }
            else if (!queued)
                //如果还没有被添加到等待队列.则将节点添加到等到队列的头部.
                queued = U.compareAndSwapObject(this, WAITERS,
                                                q.next = waiters, q);
            else if (timed) {//如果设置了超时.
                final long parkNanos;
                if (startTime == 0L) { // first time
                    startTime = System.nanoTime();
                    if (startTime == 0L)
                        startTime = 1L;
                    parkNanos = nanos;
                } else {
                    long elapsed = System.nanoTime() - startTime;
                    if (elapsed >= nanos) {
                        removeWaiter(q);
                        return state;
                    }
                    parkNanos = nanos - elapsed;
                }
                // nanoTime may be slow; recheck before parking
                if (state < COMPLETING)
                    LockSupport.parkNanos(this, parkNanos);
            }
            else
                //阻塞线程.
                LockSupport.park(this);
        }
    }
```
14. AsyncTask的postResultIfNotInvoked方法:在FutureTask的done方法中调用
```
   private void postResultIfNotInvoked(Result result) {
        //根据任务是否被执行了.调用postResult方法.mTaskInvoked方法中被设置为true.
        final boolean wasTaskInvoked = mTaskInvoked.get();
        if (!wasTaskInvoked) {
            postResult(result);
        }
    }
    也就是说,只有当WorkerRunnable的call方法没有被执行时,就调用了AsyncTask的cancel方法.才会通过FutureTask的done方法调用
    postResultIfNotInvoked方法,执行postResult方法.
```

```
AsyncTask缺点:
1.  cancel:只能保证不会回调onPostExecute方法.并不保证doInBackground中正在执行的操作被打断.因为它最终是通过Thread.interrupt方法.
    首先要清楚该方法的作用:它并不是真的中断线程.只是给线程设置一个中断标识而已.通知线程应该结束了.所以在doInBackground中.
    我们需要自己不断的通过isInterrupted方法或者interrupted方法判断是否接受到了中断标识.如果接收到了.自己结束线程工作.(比如抛出异常,finilly中
    执行清理工作)或者简单点直接通过isCanceled方法判断是否取消了任务.
    isInterrupted():返回是否设置了中断标识,不会清楚中断标识.
    interrupted():返回是否设置了中断标识,会清楚中断标识.

    1.当线程处于被阻塞状态(比如sleep.wait.join),interrupt方法会是线程立刻退出.线程会清除中断状态,并报出 InterruptedException异常.
    2.当线程处于阻塞 I/O操作在 interruptible channel 上.会关闭channel.线程会设置中断状态,然后报出ClosedByInterruptException异常.
```

```
AsyncTask正常调用流程:

->  AsyncTask.execute()
->  AsyncTask.executeOnExecutor()
->  SerialExecutor.execute()
->  SerialExecutor.scheduleNext()
->  THREAD_POOL_EXECUTOR.execute(mActive)
->  FutureTask.run()
->  WorkerCallback.call()
->  AsyncTask.doInBackground()
->  AsyncTask.postResult()
->  InnternalHandler.handleMessage()
->  AsyncTask.finish()
->  AsyncTask.onPostExecute()/onCancelled()


```