#   RxJava源码学习(一)操作符：创建Observable

一：Creating Observables：用来创建一个新的Observable对象

1.  Create 操作符: 用于直接创建一个Observable对象
```
    //1. create 操作符使用
    Observable<Object> observable = Observable.create(new ObservableOnSubscribe<Object>() {
        @Override
        public void subscribe(ObservableEmitter<Object> emitter) throws Exception {
            if (emitter.isDisposed()) {
                //...外界已经取消订阅
            } else {
                //
            }
        }
    });

    //2. create操作符源码：
    public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
        //在没有额外配置的情况下，可以忽略RxJavaPlugins的调用，它内部直接返回的就是传给它
        //的参数，在有额外配置的情况下，会根据配置对传入的参数变换后返回

        //也就是说，我们Creat操作符返回的是ObservableCreate对象
        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
    }

    //ObservableCreate类继承了Observable类
    public final class ObservableCreate<T> extends Observable<T> {
        final ObservableOnSubscribe<T> source;

        public ObservableCreate(ObservableOnSubscribe<T> source) {
            this.source = source;
        }

        @Override
        protected void subscribeActual(Observer<? super T> observer) {

            //这里用CreateEmitter包装了订阅当前Observable的observer，
            //当CreateEmitter的onNext，onError，onComplete被回调时，
            //如果没有取消订阅，其内部会相应的执行observer的onNext，onError，onComplete方法
            CreateEmitter<T> parent = new CreateEmitter<T>(observer);

            //这里observer是对这一级observable进行观察的观察者，通知观察者，已经订阅上观察源
            //这里回传的是CreateEmitter，其实现了Disposable借口，订阅者可以通过这个Disposable取消观察。
            observer.onSubscribe(parent);

            try {
                //这里触发消息的产生，执行的是我们create方法传入的参数的subscribe方法，
                //这里穿的参数同样也是CreateEmitter，其还实现了ObservableEmitter接口
                //事件生产者，也就是ObservableOnSubscribe类在其subscribe方法中可以通过ObservableEmmitter
                //进行事件的发射：onNext，onError，onComplet。
                source.subscribe(parent);
            } catch (Throwable ex) {
                Exceptions.throwIfFatal(ex);
                parent.onError(ex);
            }
        }
        ....
    }

```

2. Defer操作符: 用于当subscribe真正产生时，才生生成生产数据的数据源对象,每次subscribe,都生成新的ObservableSource：
经测试，defer操作符产生的数据源，在subscribe后，没办法直接取消订阅

```
    //1.defer 操作符使用
    Observable<Object> observableDefer = Observable.defer(new Callable<ObservableSource<?>>() {
        @Override
        public ObservableSource<?> call() throws Exception {

            return new ObservableSource<Object>() {
                @Override
                public void subscribe(Observer<? super Object> observer) {
                    observer.onNext(new Object());
                    observer.onComplete();
                }
            };
        }
    });

    //2. defer操作符源码
    public static <T> Observable<T> defer(Callable<? extends ObservableSource<? extends T>> supplier) {
        //同样的忽略RxJavaPlugins，这里看到返回的是ObservableDefer对象
        return RxJavaPlugins.onAssembly(new ObservableDefer<T>(supplier));
    }

    //ObservableDefer类
    public final class ObservableDefer<T> extends Observable<T> {
        final Callable<? extends ObservableSource<? extends T>> supplier;

        public ObservableDefer(Callable<? extends ObservableSource<? extends T>> supplier) {
            this.supplier = supplier;
        }

        @Override
        public void subscribeActual(Observer<? super T> s) {
            ObservableSource<? extends T> pub;
            try {
                //从这里可以看到，当订阅者真正执行subscribe订阅时，才通过supplier.call（）生成最新的
                //ObservableSource
                pub = ObjectHelper.requireNonNull(supplier.call(), "null ObservableSource supplied");
            } catch (Throwable t) {
                Exceptions.throwIfFatal(t);
                EmptyDisposable.error(t, s);
                return;
            }

            //然后执行ObservableSource的subscribe方法，生产数据
            pub.subscribe(s);
        }
    }
```

3.  Empty,Never,Error 操作符
```
    //1.Empty操作符使用：
    Observable<Object> empty = Observable.empty();

   //2.Empty操作符源码：
   public static <T> Observable<T> empty() {
       //返回的是ObservableEmpty
       return RxJavaPlugins.onAssembly((Observable<T>) ObservableEmpty.INSTANCE);
   }

   //ObservableEmpty类
   public final class ObservableEmpty extends Observable<Object> implements ScalarCallable<Object> {
       public static final Observable<Object> INSTANCE = new ObservableEmpty();

       private ObservableEmpty() {
       }

       @Override
       protected void subscribeActual(Observer<? super Object> o) {
           //在被订阅的时候，不做任何处理，而是直接触发订阅者的onComplete方法
           EmptyDisposable.complete(o);
       }

       @Override
       public Object call() {
           return null; // null scalar is interpreted as being empty
       }
   }


   //3. Never操作符使用：
   Observable<Object> never = Observable.never();

   //4. Never操作符源码：
   public static <T> Observable<T> never() {
       //返回的是ObservableNever实例
       return RxJavaPlugins.onAssembly((Observable<T>) ObservableNever.INSTANCE);
   }

   //ObservableNever类：
   public final class ObservableNever extends Observable<Object> {
       public static final Observable<Object> INSTANCE = new ObservableNever();

       private ObservableNever() {
       }

       @Override
       protected void subscribeActual(Observer<? super Object> o) {
           //可以发现，当订阅者订阅后，永远不会执行其onNext，onError，onComplete方法，
           //只会出发订阅者的onSubscribe方法，通知订阅成功
           o.onSubscribe(EmptyDisposable.NEVER);
       }
   }

   //5. Error操作符使用
   Observable<Object> error = Observable.error(new Throwable());//参数传递一个错误

   //6. Error操作符源码:
   public static <T> Observable<T> error(final Throwable exception) {
       return error(Functions.justCallable(exception));
   }

   public static <T> Observable<T> error(Callable<? extends Throwable> errorSupplier) {

       //追后返回的是ObservableError实例
       return RxJavaPlugins.onAssembly(new ObservableError<T>(errorSupplier));
   }

   //ObservableError类
   public final class ObservableError<T> extends Observable<T> {
       final Callable<? extends Throwable> errorSupplier;
       public ObservableError(Callable<? extends Throwable> errorSupplier) {
           this.errorSupplier = errorSupplier;
       }

       @Override
       public void subscribeActual(Observer<? super T> s) {
           Throwable error;
           try {
               //这里获取到前面通过error操作符传入的参数
               error = ObjectHelper.requireNonNull(errorSupplier.call(), "Callable returned null throwable. Null values are generally not allowed in 2.x operators and sources.");
           } catch (Throwable t) {
               Exceptions.throwIfFatal(t);
               error = t;
           }

           //通过这里将error操作符传入的参数，执行订阅者的onSubscribe方法，然后立即执行订阅者的onError方法
           EmptyDisposable.error(error, s);
       }
   }
```

4.  From操作符：将各种其他数据类型的对象转换为Observable

```
    from操作符有5个：fromArray，fromIterable，fromFuture，fromCallback，formPublisher

    //1.fromArray操作符：将数组数据转为Observable
    Observable.fromArray(1,2,3,4);

    //fromArray源码：
    public static <T> Observable<T> fromArray(T... items) {

        //如果数组的长度为0，说明为空，则返回的是empty操作符生成的ObservableEmpty
        if (items.length == 0) {
            return empty();
        } else
        if (items.length == 1) {
            //如果长度为1，则通过just操作符代替，这里先不考虑
            return just(items[0]);
        }

        //当执行到这里时，返回的是ObservableFromArray对象
        return RxJavaPlugins.onAssembly(new ObservableFromArray<T>(items));
    }

    //ObservableFromArray类：
    public final class ObservableFromArray<T> extends Observable<T> {
        final T[] array;
        public ObservableFromArray(T[] array) {
            this.array = array;
        }
        @Override
        public void subscribeActual(Observer<? super T> s) {

            //当订阅者执行subscribe时，该数组Observable会将数据源以及订阅者二者作为参数
            //创建一个FromArrayDisposable对象
            FromArrayDisposable<T> d = new FromArrayDisposable<T>(s, array);

            //通知订阅者，已订阅，并传入上面创建的FromArrayDispsable，用于取消订阅
            s.onSubscribe(d);

            if (d.fusionMode) {
                return;
            }

            //执行FromArrayDisposeable的run方法
            d.run();
        }
        ...
    }

    //FromArrayDisposable的run方法：
    void run() {
        T[] a = array;
        int n = a.length;

        //按顺序发射数组中的元素给订阅者，发射前判断是否取消订阅
        for (int i = 0; i < n && !isDisposed(); i++) {
            T value = a[i];
            //数组元素不能为null，否者会执行订阅者的onError，不再发射后续元素
            if (value == null) {
                actual.onError(new NullPointerException("The " + i + "th element is null"));
                return;
            }
            actual.onNext(value);
        }

        //当元素发射完后，如果还没取消订阅，则调用订阅者的onComplet方法
        if (!isDisposed()) {
            actual.onComplete();
        }
    }


    //2.fromIterable操作符：将支持迭代器的集合转化为Observable
     List<String> list = new ArrayList<>();
     list.add("a");
     list.add("b");
     Observable.fromIterable(list);

     //fromIterable源码：
     public static <T> Observable<T> fromIterable(Iterable<? extends T> source) {
         //返回的是ObservableFromIterable
         return RxJavaPlugins.onAssembly(new ObservableFromIterable<T>(source));
     }

     //ObservableFromIterable类：
     public final class ObservableFromIterable<T> extends Observable<T> {
         final Iterable<? extends T> source;
         public ObservableFromIterable(Iterable<? extends T> source) {
             this.source = source;
         }

         @Override
         public void subscribeActual(Observer<? super T> s) {
             Iterator<? extends T> it;
             try {
                //获取迭代器
                 it = source.iterator();
             } catch (Throwable e) {
                 Exceptions.throwIfFatal(e);
                 EmptyDisposable.error(e, s);
                 return;
             }
             boolean hasNext;
             try {
                //是否还有下一个元素
                 hasNext = it.hasNext();
             } catch (Throwable e) {
                 Exceptions.throwIfFatal(e);
                 EmptyDisposable.error(e, s);
                 return;
             }

             //当没有下一个元素，直接调用订阅者的onComplete方法
             if (!hasNext) {
                 EmptyDisposable.complete(s);
                 return;
             }



             //在内部创建FromItrableDisposable对象
             FromIterableDisposable<T> d = new FromIterableDisposable<T>(s, it);

             //通知订阅者，订阅完成，传入Disposable对象，用于取消订阅
             s.onSubscribe(d);

             if (!d.fusionMode) {
                 //执行run方法，内部循环迭代，执行订阅者的onNext方法，期间如果取消订阅了就退出循环，完成后，如果没取消订阅，执行订阅者onComplete
                 d.run();
             }
         }
         ...
     }

     //对于From操作符：还有几个就不再详细分析，原理是一样的
     Observable.fromCallable(new Callable<Object>() {
         @Override
         public Object call() throws Exception {
             return null;
         }
     });

     Observable.fromFuture(new FutureTask<Object>(new Callable<Object>() {
         @Override
         public Object call() throws Exception {
             return null;
         }
     }),0,null,Schedulers.newThread());


     Observable.fromPublisher(new Publisher<Object>() {
         @Override
         public void subscribe(Subscriber<? super Object> s) {

         }
     });

```

5.  interval操作符：用于创建发射数据之间有一定间隔时间的Observable,可以用来处理一些定时任务
```
    //使用：
    Observable.interval(2,TimeUnit.SECONDS);

    //intrvable源码
    public static Observable<Long> interval(long period, TimeUnit unit) {
        //内部调用了带有5个参数的interval方法，其默认调度器使用的是computation
        return interval(period, period, unit, Schedulers.computation());
    }

    public static Observable<Long> interval(long initialDelay, long period, TimeUnit unit, Scheduler scheduler) {
        //返回的是ObservableInterval实例
        return RxJavaPlugins.onAssembly(new ObservableInterval(Math.max(0L, initialDelay), Math.max(0L, period), unit, scheduler));
    }

    //ObservableInterval:

     public ObservableInterval(long initialDelay, long period, TimeUnit unit, Scheduler scheduler) {
            this.initialDelay = initialDelay;
            this.period = period;
            this.unit = unit;
            this.scheduler = scheduler;
        }

        @Override
        public void subscribeActual(Observer<? super Long> s) {

            //创建一个IntrvalObservable观察者，包装观察当前ObservableInterval的观察者
            IntervalObserver is = new IntervalObserver(s);

            //将包装类IntrvalObservable通过订阅者的onSubscribe方法传递出去，用于取消订阅
            s.onSubscribe(is);

            Scheduler sch = scheduler;

            if (sch instanceof TrampolineScheduler) {
                Worker worker = sch.createWorker();
                is.setResource(worker);
                worker.schedulePeriodically(is, initialDelay, period, unit);
            } else {
                //默认情况，我们的线程调度器是computation，所以执行这里
                //这里通过响应的调度器执行is。
                Disposable d = sch.schedulePeriodicallyDirect(is, initialDelay, period, unit);
                is.setResource(d);
            }
        }
        ....
     }

     //IntervalObserver实现了Runnable接口，当在线程池中执行时，run方法会被调用
     @Override
     public void run() {
        //在内部判断当前是否取消订阅，没有则回调订阅者的onNext方法，count表示的是序号，发射了多少次
         if (get() != DisposableHelper.DISPOSED) {
             actual.onNext(count++);
         }
     }

```

6.  Just操作符：用于创建一个发射特定项目的Observable
```
    //使用
    Observable.just(1);
    Observable.just(1,2,3,4);//对于多个item的情况，内部通过fromArray操作完成，不再多讲
    这里来看看单个item的情况

   public static <T> Observable<T> just(T item) {

        //返回的是ObservableJust对象
        return RxJavaPlugins.onAssembly(new ObservableJust<T>(item));
   }

   public final class ObservableJust<T> extends Observable<T> implements ScalarCallable<T> {

       private final T value;
       public ObservableJust(final T value) {
           this.value = value;
       }

       @Override
       protected void subscribeActual(Observer<? super T> s) {

            //当该Observable被订阅时，会创建一个ScalarDisposable对象，其实现了Disposable
            //传入实际订阅者和实际数据
           ScalarDisposable<T> sd = new ScalarDisposable<T>(s, value);
           //所以可以传递给订阅者，可以使用该对象来取消订阅
           s.onSubscribe(sd);

           //执行其run方法发射数据
           sd.run();
       }

       @Override
       public T call() {
           return value;
       }
   }

   //ScalarDisposable的run方法
       @Override
       public void run() {
           //判断当前状态是否是START,通过CAS操作将状态设置为ON_NEXT
           if (get() == START && compareAndSet(START, ON_NEXT)) {

               //然后执行订阅者的onNext方法,将数据value发射出去
               observer.onNext(value);

               //因为上面设置了ON_NEXT,所以执行onComplete方法，并且设置状态为ON_COMPLETE
               if (get() == ON_NEXT) {
                   lazySet(ON_COMPLETE);
                   observer.onComplete();
               }
           }
       }
```

7.  Range操作符: 创建一个发出特定范围的整数的Observable
```
    //使用:第一个参数标识起始序号，第二个参数表示发射的个数
    Observable.range(n,m);//发射的序列是：n n+1 n+2 .... n+m-1

    //range操作符源码：
    public static Observable<Integer> range(final int start, final int count) {
        if (count < 0) {
            throw new IllegalArgumentException("count >= 0 required but it was " + count);
        }
        //如何个数为0，当做empty操作处理
        if (count == 0) {
            return empty();
        }

        //个数为1，当做just操作处理
        if (count == 1) {
            return just(start);
        }

        //超出了Intger的最大值，抛异常
        if ((long)start + (count - 1) > Integer.MAX_VALUE) {
            throw new IllegalArgumentException("Integer overflow");
        }

        //正常返回ObservableRange对象
        return RxJavaPlugins.onAssembly(new ObservableRange(start, count));
    }


    //ObservableRange类：
    public final class ObservableRange extends Observable<Integer> {
        private final int start;
        private final long end;

        public ObservableRange(int start, int count) {
            this.start = start;
            this.end = (long)start + count;
        }

        @Override
        protected void subscribeActual(Observer<? super Integer> o) {

            //当订阅者订阅时，内部创建一个RangeDisposable对象，用于包装订阅者
            RangeDisposable parent = new RangeDisposable(o, start, end);

            //将包装类通过订阅者的onSubscribe方法传递出去，订阅者就可以通过该类取消订阅
            o.onSubscribe(parent);

            //执行RangeDisposable的run方法，发射数据
            parent.run();
        }
        .....
    }

    //RangeDisposable的run方法：
    void run() {
        if (fused) {
            return;
        }
        Observer<? super Integer> actual = this.actual;
        long e = end;
        //循环发射数据，这里当订阅者取消订阅了，get（）方法返回的是1，因此如果取消订阅了，是不会在执行onNext操作
        for (long i = index; i != e && get() == 0; i++) {
            actual.onNext((int)i);
        }

        //如果还没有取消订阅，则继续调用订阅者的onComplete方法
        if (get() == 0) {
            lazySet(1);
            actual.onComplete();
        }
    }
```

8.  Repeat操作符：创建一个可以多次发射特定项目的Observable(将一个已存在的observable对象进行包装)

```
    //使用:repeat操作符参数：times：表示重复执行的次数，不传值得情况，timex内部默认为Long.Long.MAX_VALUE

    //对应下面这个列子，就会打印a字符串3次
     Observable.just("a").repeat(3).subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                System.out.println(s);
            }
        });

     //repeat源码：
    public final Observable<T> repeat(long times) {
        //次数不能小于0，否则会抛参数异常
        if (times < 0) {
            throw new IllegalArgumentException("times >= 0 required but it was " + times);
        }
        //当次数为0时，当成empty操作符处理
        if (times == 0) {
            return empty();
        }

        //返回的是ObservableRepeat对象，它对原始Observable进行了包装
        return RxJavaPlugins.onAssembly(new ObservableRepeat<T>(this, times));
    }

    //ObservableRepeat类：
    public final class ObservableRepeat<T> extends AbstractObservableWithUpstream<T, T> {
        final long count;
        public ObservableRepeat(Observable<T> source, long count) {
            super(source);
            this.count = count;
        }

        @Override
        public void subscribeActual(Observer<? super T> s) {

            //当订阅者订阅时，该Observable内部会创建一个SequentialDisposable实例
            SequentialDisposable sd = new SequentialDisposable();

            //将该实例通过订阅者的onSubscribe传递出去，可以用于取消订阅
            s.onSubscribe(sd);

            //创建一个新的观察者RepeatObsever，包装了订阅了当前Observable的观察者
            RepeatObserver<T> rs = new RepeatObserver<T>(s, count != Long.MAX_VALUE ? count - 1 : Long.MAX_VALUE, sd, source);
            //执行新的观察者的subscribeNext方法
            rs.subscribeNext();
        }
        ....
    }

    //RepeatObserver的subscribeNext方法：
    void subscribeNext() {
        //获取当前的值，然后加1
        if (getAndIncrement() == 0) {
            int missed = 1;

            //无限循环执行
            for (;;) {
                //判断观察当前Observable的观察者是否取消了订阅，如果取消了订阅，就不再执行
                if (sd.isDisposed()) {
                    return;
                }

                //如果没有取消，则执行repeat作用的源Observable的subscribe方法，把当前RepeatObserver观察者传给源Observable
                //在源Observable内部，会执行订阅它的观察者的onNext方法，完成后会执行onComplete方法
                source.subscribe(this);

                //加上-missed后，再获取新的值
                missed = addAndGet(-missed);

                //因此正常情况（单线程调用subscribeNext），这里为0，因此会跳出循环
                if (missed == 0) {
                    break;
                }
            }
        }
    }

    @Override
    public void onNext(T t) {
        //当onNext会源Obsrvable调用时，直接调用订外部的观察者的onNext方法
        actual.onNext(t);
    }


    @Override
    public void onComplete() {
        //onComplete调用时，会根据需要执行的次数决定回调外界订阅者的onComplete方法，或者继续执行subscribeNext方法
        long r = remaining;
        if (r != Long.MAX_VALUE) {
            remaining = r - 1;
        }
        if (r != 0L) {
            subscribeNext();
        } else {
            actual.onComplete();
        }
    }
```

9.  Start操作符：创建一个可以发出函数式指令的返回值的Observable
在RxJava中其就是：fromCallback操作符
```
 Observable.fromCallable(new Callable<Object>() {
            @Override
            public Object call() throws Exception {
                return "abc";
            }
        }).subscribe(new Consumer<Object>() {
            @Override
            public void accept(Object o) throws Exception {
                System.out.println(o);
            }
        });
```

10. timer操作符:创建一个Observable，在给定的时延后发出一个特定的项目
```
    Observable.timer(3,TimeUnit.SECONDS,Schedulers.io());//可以会制定调度器，默认实在computation上

    //源码：
    public static Observable<Long> timer(long delay, TimeUnit unit, Scheduler scheduler) {

        //返回的是ObservableTimer实例
        return RxJavaPlugins.onAssembly(new ObservableTimer(Math.max(delay, 0L), unit, scheduler));
    }

    //ObservableTimer类：
    public final class ObservableTimer extends Observable<Long> {
        final Scheduler scheduler;
        final long delay;
        final TimeUnit unit;
        public ObservableTimer(long delay, TimeUnit unit, Scheduler scheduler) {
            this.delay = delay;
            this.unit = unit;
            this.scheduler = scheduler;
        }

        @Override
        public void subscribeActual(Observer<? super Long> s) {

            //当其被订阅的时候，会创建一个TimerObserver包装外界的观察者s
            //TimerObserver实现了Disposable和Runnable
            TimerObserver ios = new TimerObserver(s);

            //将这个包装观察者通过外部观察者的onSubscribe方法传递出去，用来取消订阅。
            s.onSubscribe(ios);

            //通过调度器调度执行任务
            Disposable d = scheduler.scheduleDirect(ios, delay, unit);

            ios.setResource(d);
        }
        ....
    }

    //TimerObserver的run方法：
    public void run() {
        //如果没有取消订阅，就调用onNext方法，然后onComplete方法
        if (!isDisposed()) {
            actual.onNext(0L);
            lazySet(EmptyDisposable.INSTANCE);
            actual.onComplete();
        }
    }

```
