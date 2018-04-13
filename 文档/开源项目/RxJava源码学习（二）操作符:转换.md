#   RxJava源码学习（二）操作符：转换

转换Observable发出的项目的操作符：

1. Buffer操作符：周期性地将由Observable发出的item收集成组并发射这些组，而不是一次一个地发射item
```
    //使用：

     //对于该例子：原本fromArray操作符按顺序一个一个的发射数据，当经过buffer作用后，对于订阅者来说，
     //接受的数据是一组一组的。
     Observable.fromArray(1,2,3,4,5,6,7,8,9,10)
                    .buffer(3)
                    .subscribe(new Consumer<List<Integer>>() {
                        @Override
                        public void accept(List<Integer> integers) throws Exception {
                            System.out.println(integers);
                        }
                    });

     //buffer操作符源码：
     public final Observable<List<T>> buffer(int count) {
         return buffer(count, count);
     }

     public final Observable<List<T>> buffer(int count, int skip) {
         return buffer(count, skip, ArrayListSupplier.<T>asCallable());
     }

     //第一个参数：count 代表多少个item为一组
     //第二个参数：skip 代表在启动新缓冲器之前应该跳过源ObservableSource发出的项目数量
     //第三个参数：bufferSuppiler：作为缓冲区提供者
     public final <U extends Collection<? super T>> Observable<U> buffer(int count, int skip, Callable<U> bufferSupplier) {
         //最终返回的是ObservableBuffer实例
         return RxJavaPlugins.onAssembly(new ObservableBuffer<T, U>(this, count, skip, bufferSupplier));
     }

     //ObservableBuffer类：的subscribeActual方法：

      protected void subscribeActual(Observer<? super U> t) {
             //当外界观察者订阅该ObservableBuffer时，根据skip和count是否相等做不同的逻辑处理
             if (skip == count) {
                //如果相等，创建BufferExactObserver观察者包装类包装外界的观察者t
                 BufferExactObserver<T, U> bes = new BufferExactObserver<T, U>(t, count, bufferSupplier);

                 //通过bes创建buffer成功，就执行源observable的订阅方法，触发数据的发射，将bes作为观察者
                 //传递给源observable
                 if (bes.createBuffer()) {
                     source.subscribe(bes);
                 }
             } else {
                //如果不等，则创建新的BufferSkipObserver作为新的观察者，订阅源observable
                 source.subscribe(new BufferSkipObserver<T, U>(t, count, skip, bufferSupplier));
             }
         }

     //BufferExactObserver: onNext方法
     public void onNext(T t) {
         U b = buffer;

         //如果buffer不为null，每接受到一个数据就存到buffer中
         if (b != null) {
             b.add(t);

             //当buffer中的元素个数大于等于限制的每个组元素的个数时，执行外界观察者的onNext，发射的是当前buffer。
             //然后创建新的buffer
             if (++size >= count) {
                 actual.onNext(b);

                 size = 0;
                 createBuffer();
             }
         }
     }

     //BufferExactObservable: onComplete方法
     @Override
     public void onComplete() {

        //当源Observable调用它的观察者的onComplete时
         U b = buffer;
         buffer = null;
         //如果buffer不为null，且不为空，那么就是调用外界观察者的onNext方法，发射最后一个buffer，然后调用其onComplete方法
         if (b != null && !b.isEmpty()) {
             actual.onNext(b);
         }
         actual.onComplete();
     }
```

2.  flatMap操作符：将Observable发射的items转化为Observables，然后将这些items的变成单个的Observable发射
```
    例子：
    Observable.fromArray(1,2,3,4,5)
        .buffer(5)//这里buffer变换后发射出来的一个集合item
        //通过flatMap就是将这个集合展平，把集合中的每个元素单独的发射
        .flatMap(new Function<List<Integer>, ObservableSource<Integer>>() {
            @Override
            public ObservableSource<Integer> apply(List<Integer> integers) throws Exception {
                return Observable.fromIterable(integers);
            }
        })
        .subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer o) throws Exception {
                System.out.println(o);
            }
        });

    //flatMap源码：
    public final <R> Observable<R> flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper) {
        return flatMap(mapper, false);
    }

    public final <R> Observable<R> flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper, boolean delayErrors) {
        return flatMap(mapper, delayErrors, Integer.MAX_VALUE);
    }

    public final <R> Observable<R> flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper, boolean delayErrors, int maxConcurrency) {
        return flatMap(mapper, delayErrors, maxConcurrency, bufferSize());
    }

    public final <R> Observable<R> flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper,
            boolean delayErrors, int maxConcurrency, int bufferSize) {

        //当前Observable是SaclarCallable类型时，执行下面逻辑
        if (this instanceof ScalarCallable) {
            T v = ((ScalarCallable<T>)this).call();
            if (v == null) {
                return empty();
            }
            return ObservableScalarXMap.scalarXMap(v, mapper);
        }


        //大多数情况，都是执行这里
        return RxJavaPlugins.onAssembly(new ObservableFlatMap<T, R>(this, mapper, delayErrors, maxConcurrency, bufferSize));
    }

    //ObservableFlatMap：

    public void subscribeActual(Observer<? super U> t) {

        if (ObservableScalarXMap.tryScalarXMapSubscribe(source, t, mapper)) {
            return;
        }

        //这里新创建一个MergeObserer包装外面的观察者t，用它来订阅源observable
        source.subscribe(new MergeObserver<T, U>(t, mapper, delayErrors, maxConcurrency, bufferSize));
    }

    //MergeObserver：

      //在MergeObserver的onNext方法被源Obervable执行时，会执行mapper的apply将源Observable发射出来的
      //数据t转换为一个新的ObservableSource对象，然后会执行ObservableSource的subscribe方法
      public void onNext(T t) {
            if (done) {
                return;
            }
            ObservableSource<? extends U> p;
            try {
                //转换成新的ObservableSource，用于生产数据
                p = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper returned a null ObservableSource");
            } catch (Throwable e) {
               ....
               return；
            }
            .....
            //在这里面订阅新的ObservableSource
            subscribeInner(p);
        }

       void subscribeInner(ObservableSource<? extends U> p) {
            ....
          //InnerObserver的onNext回调当前MergeObserver的tryEmit，其方法内部会执行actul的onNext
          InnerObserver<T, U> inner = new InnerObserver<T, U>(this, uniqueId++);
          ...
          p.subscribe(inner);
          ....
        }
```

3.  groupBy操作符：将一个Observable发射出的数据分成多组Observables，每个Observable发出一个来自原始Observable的不同子集
```
 //例子：正常是发射1，2，3，4，5，6，7，8，9，10.
 //进过groupBy操作符后，会分成多组发射，根据我们传递的Function的applay方法进行分组
 //订阅者，接收到的依然是一个Observable，因此需要通过take或者subscribe，获取Observable的元素值

 Observable.fromArray(1,2,3,4,5,6,7,8,9,10).groupBy(new Function<Integer, Object>() {
            @Override
            public Object apply(Integer integer) throws Exception {
                return integer%2==0?"a":"b";
            }
        }).subscribe(new Consumer<GroupedObservable<Object, Integer>>() {
            @Override
            public void accept(GroupedObservable<Object, Integer> objectIntegerGroupedObservable) throws Exception {
                if(objectIntegerGroupedObservable.getKey().equals("b")){
                    objectIntegerGroupedObservable.take(10).subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            System.out.println(integer);
                        }
                    });
                }
            }
        });

```

4. map操作符:通过对每个项目应用函数来转换Observable发出的项目
```
    例子：
    //源Observable发射1，2，3 经过map处理，对每个发射的元素做了乘以2的处理，所以最后的结果是2，4，6
    Observable.just(1,2,3).map(new Function<Integer, Integer>() {
                @Override
                public Integer apply(Integer integer) throws Exception {
                    return integer*2;
                }
            }).subscribe(new Consumer<Integer>() {
                @Override
                public void accept(Integer integer) throws Exception {
                    System.out.println(integer);
                }
            });

    //map操作符源码：
    public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
        //返回的是ObservableMap实例
        return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
    }

    //ObservableMap:

    public void subscribeActual(Observer<? super U> t) {
        //source是源Observable，t是外部观察者，function是map变换
        //这里新建了一个MapObservable观察者，订阅源Observable，该观察者包装了外部的观察者t
        source.subscribe(new MapObserver<T, U>(t, function));
    }

    //MapObserver:
     public void onNext(T t) {
                ....

                U v;

                try {
                    //通过mapper变换源Observable发射的数据为v
                    v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
                } catch (Throwable ex) {
                    fail(ex);
                    return;
                }

                //调用外部的观察者onNext方法，传递新值v
                actual.onNext(v);
            }
```

5.  Scan操作符：对Observable发出的每个项目按顺序应用一个函数，并发出每个连续的值
```
    //例子：源Observable按1，2，3顺序发射数据，scan操作符对第一个数据不处理，对后面的每一个数据都与前一个数据
    //进行一个函数计算生成新的数据发射给订阅者
    Observable.just(1,2,3).scan(new BiFunction<Integer, Integer, Integer>() {
            @Override
            public Integer apply(Integer integer, Integer integer2) throws Exception {
                return integer*integer2;
            }
        })
        .subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                System.out.println(integer);
            }
        });

    //scan操作符源码：
     public final Observable<T> scan(BiFunction<T, T, T> accumulator) {
          //返回的是ObservableScan实例
         return RxJavaPlugins.onAssembly(new ObservableScan<T>(this, accumulator));
     }

    //ObservableScan:
    public void subscribeActual(Observer<? super T> t) {

        //这是source是源Observable，够着一个新的ScanObservable观察者包装外面的观察者，订阅源Observable
        source.subscribe(new ScanObserver<T>(t, accumulator));
    }

    //ScanObserver：
    public void onNext(T t) {
        if (done) {
            return;
        }
        final Observer<? super T> a = actual;

        //value保存的上一次发射的值
        T v = value;

        //第一次的时候v为null，所以直接调用外部观察者的onNext方法传递t
        if (v == null) {
            //并将t赋值给value
            value = t;
            a.onNext(t);
        } else {

            //value不为null时

            T u;

            try {
                //通过scan操作符传递的函数，传递v和t作为参数，变化为新的值u
                u = ObjectHelper.requireNonNull(accumulator.apply(v, t), "The value returned by the accumulator is null");
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
                s.dispose();
                onError(e);
                return;
            }

            //将u保存到value中
            value = u;

            //调用外部观察者的onNext方法，传递新值
            a.onNext(u);
        }
    }
```

6.  Window操作符：定期将Observable中的项目细分为Observable窗口并发出这些窗口，而不是一次发送一个项目
和buffer操作符有点像
```
    //例子：源Observable发射1，2，3，4，5，6 ，widnow操作符将其分组打包成Observable，然后发射多个Observable
    Observable.just(1,2,3,4,5,6)
                .window(2)
                .subscribe(new Consumer<Observable<Integer>>() {
                    @Override
                    public void accept(Observable<Integer> integerObservable) throws Exception {

                    }
                });

    //window操作符源码：

     public final Observable<Observable<T>> window(long count) {
         return window(count, count, bufferSize());
     }

     public final Observable<Observable<T>> window(long count, long skip, int bufferSize) {

        //返回的是一个ObservableWindow实例，包装了当前Observable
         return RxJavaPlugins.onAssembly(new ObservableWindow<T>(this, count, skip, bufferSize));
     }

     //ObservableWindow
     public void subscribeActual(Observer<? super Observable<T>> t) {
         if (count == skip) {
             source.subscribe(new WindowExactObserver<T>(t, count, capacityHint));
         } else {
             source.subscribe(new WindowSkipObserver<T>(t, count, skip, capacityHint));
         }
     }

     //WindowExactObservable：
      public void onNext(T t) {
         UnicastSubject<T> w = window;

         //当前w为null，且没有取消，则创建Subject，Subject是继承了Observable的，且也实现了Obverser
         if (w == null && !cancelled) {
             w = UnicastSubject.create(capacityHint, this);
             window = w;

             //将这个Subject发射给外部订阅者
             actual.onNext(w);
         }

         if (w != null) {
            //执行Subject的onNext方法
             w.onNext(t);
             //当Subject发射了限制大小的数据后，清空当前window，size，执行Subject的onComplet方法
             if (++size >= count) {
                 size = 0;
                 window = null;
                 w.onComplete();

                 //如果外面的观察者取消了观察，则执行源Observable传递出来用于取消观察的s的dispose方法
                 if (cancelled) {
                     s.dispose();
                 }

                 //否则，当源observable继续发射数据时，再次创建新的Subject
             }
         }
     }


     //UnicastSubject：

     public void onNext(T t) {
         ObjectHelper.requireNonNull(t, "onNext called with null. Null values are generally not allowed in 2.x operators and sources.");
         if (done || disposed) {
             return;
         }

         //queue是个队列，将源observable发射出来的数据保存在queue中
         queue.offer(t);
         drain();
     }

```

