#   RxJava源码学习（六）Scheduler

RxJava给我们提供了6个创建不同Scheduler的方法：
```
1.  Schedulers.computation();
    用于计算工作，如事件循环和回调处理; 请不要将此调度程序用于I / O（请使用Schedulers.io（））;
    默认情况下，线程的数量等于处理器的数量

2.  Schedulers.from(Executors.newCachedThreadPool());
    用一个指定的线程池作为调度器

3.  Schedulers.io();
    意味着I / O绑定的工作，例如阻塞I / O的异步性能，这个调度程序由一个可根据需要增长的线程池支持;
    对于普通的计算工作，切换到Schedulers.computation（）;
    Schedulers.io（）默认情况下是一个CachedThreadScheduler.

4.  Schedulers.newThread();
    始终创建一个新的线程

5.  Schedulers.single();
    单个后台线程，按顺序执行任务

6.  Schedulers.trampoline();
    工作任务入队列，在当前线程执行队列中的任务
```


1. Schedulers.computation:  内部线程池是ScheduledThreadPoolExecutor，核心线程数为1
```
    Schedulers类中的默认computation：
    static final class ComputationHolder {
        static final Scheduler DEFAULT = new ComputationScheduler();
    }

    //ComputationScheduler
    THREAD_FACTORY = new RxThreadFactory(THREAD_NAME_PREFIX, priority, true);

    public ComputationScheduler() {
        this(THREAD_FACTORY);
    }

    public ComputationScheduler(ThreadFactory threadFactory) {
        this.threadFactory = threadFactory;
        this.pool = new AtomicReference<FixedSchedulerPool>(NONE);
        start();
    }

    public void start() {
        //MAX_THREADS默认是cpu的核数
        FixedSchedulerPool update = new FixedSchedulerPool(MAX_THREADS, threadFactory);
        if (!pool.compareAndSet(NONE, update)) {
            update.shutdown();
        }
    }


    //FixedSchedulerPool：内部拥有maxThreads个PoolWorker，每个PoolWorker拥有自己的ScheduledThreadPoolExecutor
    //每个ScheduledThreadPoolExevutor的核心线程数为1
    FixedSchedulerPool(int maxThreads, ThreadFactory threadFactory) {
        // initialize event loops
        this.cores = maxThreads;
        this.eventLoops = new PoolWorker[maxThreads];
        for (int i = 0; i < maxThreads; i++) {
            this.eventLoops[i] = new PoolWorker(threadFactory);

    }

    //PoolWorker：内部含有一个ScheduledThreadPoolExecutor。
    static final class PoolWorker extends NewThreadWorker {
        PoolWorker(ThreadFactory threadFactory) {
            super(threadFactory);
        }
    }

    //NewThreadWorker
    public NewThreadWorker(ThreadFactory threadFactory) {

        //executor是一个ScheduledExecutorService，其实是ScheduledThreadPoolExecutor，该类继承自
        //ThreadPoolExecutor，主要用来在给定的延迟之后运行任务，或者定期执行任务。
        executor = SchedulerPoolFactory.create(threadFactory);
    }

    //SchedulerPoolFactory
    public static ScheduledExecutorService create(ThreadFactory factory) {

        //可以发现，我们创建的ScheduledThreadPoolExecutor的核心线程数为1.
        final ScheduledExecutorService exec = Executors.newScheduledThreadPool(1, factory);
        if (PURGE_ENABLED && exec instanceof ScheduledThreadPoolExecutor) {
            ScheduledThreadPoolExecutor e = (ScheduledThreadPoolExecutor) exec;
            POOLS.put(e, exec);
        }
        return exec;
    }


    //CompuetationScheduler:
    public Disposable scheduleDirect(@NonNull Runnable run, long delay, TimeUnit unit) {
        PoolWorker w = pool.get().getEventLoop();
        //PoolWorker通过内部的线程池执行run
        return w.scheduleDirect(run, delay, unit);
    }
```

2.  Schedules.from: 由我们提供的线程池作为线程池
```
    public static Scheduler from(@NonNull Executor executor) {
        return new ExecutorScheduler(executor);
    }


    //ExecutorScheduler
    public ExecutorScheduler(@NonNull Executor executor) {
        this.executor = executor;
    }
```

3.  Schedulers.io：内部通过CacheWorkPool的一个LinkedQueue管理空闲的ThreadWorker，每个ThreadWorker
内部有一个核心线程数为1的ScheduledThreadPoolExecutor，同时CacheWorkerPool内部还有一个ScheduledThreadPoolExecutor定期执行
清理LinkedQueue中过期的ThreadWorker。

```
    static final class IoHolder {
        static final Scheduler DEFAULT = new IoScheduler();
    }

    //IOScheduler:
    public IoScheduler() {
        this(WORKER_THREAD_FACTORY);
    }

     public IoScheduler(ThreadFactory threadFactory) {
            this.threadFactory = threadFactory;
            this.pool = new AtomicReference<CachedWorkerPool>(NONE);
            start();
        }

    @Override
    public void start() {
        //创建CacheWorkerPool，非核心线程默认生存60s
        CachedWorkerPool update = new CachedWorkerPool(KEEP_ALIVE_TIME, KEEP_ALIVE_UNIT, threadFactory);
        if (!pool.compareAndSet(NONE, update)) {
            update.shutdown();
        }
    }

    //CacheWorkerPool：
    CachedWorkerPool(long keepAliveTime, TimeUnit unit, ThreadFactory threadFactory) {
        this.keepAliveTime = unit != null ? unit.toNanos(keepAliveTime) : 0L;

        //存工作线程的队列
        this.expiringWorkerQueue = new ConcurrentLinkedQueue<ThreadWorker>();
        this.allWorkers = new CompositeDisposable();
        this.threadFactory = threadFactory;

        ScheduledExecutorService evictor = null;
        Future<?> task = null;
        if (unit != null) {
            evictor = Executors.newScheduledThreadPool(1, EVICTOR_THREAD_FACTORY);
            task = evictor.scheduleWithFixedDelay(this, this.keepAliveTime, this.keepAliveTime, TimeUnit.NANOSECONDS);
        }
        evictorService = evictor;
        evictorTask = task;
    }

    ThreadWorker get() {
        if (allWorkers.isDisposed()) {
            return SHUTDOWN_THREAD_WORKER;
        }

        //如果工作线程队列中有线程，返回该threadWorkder
        while (!expiringWorkerQueue.isEmpty()) {
            ThreadWorker threadWorker = expiringWorkerQueue.poll();
            if (threadWorker != null) {
                return threadWorker;
            }
        }

        //否则创建ThradWorker，ThreadWorker内有一个核心线程数为1的ScheduledThradPoolExecutor线程池
        ThreadWorker w = new ThreadWorker(threadFactory);
        allWorkers.add(w);
        return w;
    }
```

4.  Schedulers.newThread: 每次都创建一个新的NewThreadWorker来执行任务，NewThreadWorker内部是通过一个
核心线程熟练为1的ScheduledThreadPoolExecutor来执行任务
```
    static final class NewThreadHolder {
        static final Scheduler DEFAULT = new NewThreadScheduler();
    }

    //NewThreadScheduler:

    public NewThreadScheduler() {
        this(THREAD_FACTORY);
    }

    public NewThreadScheduler(ThreadFactory threadFactory) {
        this.threadFactory = threadFactory;
    }

    @NonNull
    @Override
    public Worker createWorker() {
        return new NewThreadWorker(threadFactory);
    }

```

5.  Schedulers.single：内部始终是通过同一个ScheduledThreadPoolExecutor线程池完成任务
```
    static final class SingleHolder {
        static final Scheduler DEFAULT = new SingleScheduler();
    }

    //SingleScheduler
    public SingleScheduler() {
        this(SINGLE_THREAD_FACTORY);
    }

    public SingleScheduler(ThreadFactory threadFactory) {
        this.threadFactory = threadFactory;
        executor.lazySet(createExecutor(threadFactory));
    }

    static ScheduledExecutorService createExecutor(ThreadFactory threadFactory) {
        //返回的是1个核心线程数为1的ScheduleThreadPoolExecutor
        return SchedulerPoolFactory.create(threadFactory);
    }
```


6.  Observable.subscribeOn
```
        public final Observable<T> subscribeOn(Scheduler scheduler) {
            //返回的是一个包装了源Observable的ObservableSubscribeOn实例
            return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
        }

        //ObservableSubscribeOn：

        public void subscribeActual(final Observer<? super T> s) {
            final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s);

            s.onSubscribe(parent);

            //通过调度器，调度执行SubscribeTask
            parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
        }

        //SubscribeTask：
         public void run() {
            //该方法将在调度器的线程中执行，也就是说源Observable的发射数据将通过自定的调度器在相应的
            //线程中执行。
            source.subscribe(parent);
         }

        //SubscribeOnObserver：
        public void onNext(T t) {
            //调用外部observer的onNext方法
            actual.onNext(t);
        }

```

7.  Observable.observerOn
```
    public final Observable<T> observeOn(Scheduler scheduler) {
        //bufferSize()默认为128
        return observeOn(scheduler, false, bufferSize());
    }

    public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
        //内部返回的是ObservableObserverOn，源Observable的一个包装类
        return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
    }

    //ObservableObserverOn：
    protected void subscribeActual(Observer<? super T> observer) {
        if (scheduler instanceof TrampolineScheduler) {
            //如果观察调度器制定的是Trampoline，则在当前线程观察
            source.subscribe(observer);
        } else {
            //否侧创建工作线程，
            Scheduler.Worker w = scheduler.createWorker();

            //将工作线程封装到ObserverOnObserver中。
            source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
        }
    }

    //ObserveOnObserver：
     public void onSubscribe(Disposable s) {
            if (DisposableHelper.validate(this.s, s)) {
                this.s = s;
                if (s instanceof QueueDisposable) {
                    @SuppressWarnings("unchecked")
                    QueueDisposable<T> qd = (QueueDisposable<T>) s;

                    int m = qd.requestFusion(QueueDisposable.ANY | QueueDisposable.BOUNDARY);

                    if (m == QueueDisposable.SYNC) {
                        sourceMode = m;
                        queue = qd;
                        done = true;
                        actual.onSubscribe(this);
                        schedule();
                        return;
                    }
                    if (m == QueueDisposable.ASYNC) {
                        sourceMode = m;
                        queue = qd;
                        actual.onSubscribe(this);
                        return;
                    }
                }

                //创建队列，缓冲长度默认为128
                queue = new SpscLinkedArrayQueue<T>(bufferSize);

                actual.onSubscribe(this);
            }
        }

       public void onNext(T t) {
            if (done) {
                return;
            }

            if (sourceMode != QueueDisposable.ASYNC) {
                queue.offer(t);
            }
            schedule();
       }

      void schedule() {
          if (getAndIncrement() == 0) {
              //通过线程调度，外界Observer的onNext操作将在worker线程中执行
              worker.schedule(this);
          }
      }

```