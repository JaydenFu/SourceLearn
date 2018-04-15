#   RxJava源码学习（三）操作符:过滤

1. Debounce操作符：如果特定的时间跨度已经过去而没有发射另一个物品，则只从Observable发射一个物品,默认在computation调度器执行
```
    例子：我们设值得debounce时长是1s，所以对于Oservable来说，1s内发射的多个item，只会发射出去这1s内最后一个item，其余item会被丢掉
    //下面这个例子的结果是：1，3
      Observable.create(new ObservableOnSubscribe<Integer>() {
                @Override
                public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                    emitter.onNext(1);
                    sleepWait(1100);
                    emitter.onNext(2);
                    sleepWait(100);
                    emitter.onNext(3);
                }
            })
                    .debounce(1, TimeUnit.SECONDS)
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            System.out.println(integer);
                        }
                    });

      //debounce操作符源码：
      public final Observable<T> debounce(long timeout, TimeUnit unit) {
          //默认在computation调度器执行
          return debounce(timeout, unit, Schedulers.computation());
      }

      public final Observable<T> debounce(long timeout, TimeUnit unit, Scheduler scheduler) {
          //最后返回的是一个包装源Observable的ObservableDebounceTimed实例
          return RxJavaPlugins.onAssembly(new ObservableDebounceTimed<T>(this, timeout, unit, scheduler));
      }

      //ObservableDebounceTimed：
      public void subscribeActual(Observer<? super T> t) {

          //当外界的观察者订阅了ObservableDeounceTImed时，它在subscribeActual中，会创建一个包装了
          //外部观察者的DebounceTimedObserver用于订阅源Observable
          source.subscribe(new DebounceTimedObserver<T>(
                  new SerializedObserver<T>(t),
                  timeout, unit, scheduler.createWorker()));
      }

      //DebounceTimedObserver 其实现了Observer，也实现了Dispose
       public void onSubscribe(Disposable s) {
          //当源Observable回调onSubscribe时，内部回调外部观察者的onSubscribe，传出去的是当前
          //DebounceTimeObserver
          if (DisposableHelper.validate(this.s, s)) {
              this.s = s;
              actual.onSubscribe(this);
          }
      }

      @Override
      public void onNext(T t) {
            //这里done会在onComplete或者onError中赋值为true
          if (done) {
              return;
          }

          //这个index值，会用来对比当前实际接受到的源Observable发射出来的item序号，以及经过timeout时间调度之后
          //的DebounceEmiiter所带的idx进行对比，一致的情况下，才会发射给外部观察者
          long idx = index + 1;
          index = idx;

          Disposable d = timer.get();
          if (d != null) {
              //取消之前的发射
              d.dispose();
          }


          DebounceEmitter<T> de = new DebounceEmitter<T>(t, idx, this);
          if (timer.compareAndSet(d, de)) {
              //通过内部调度线程池，调度延迟timeout执行de，在DebounceEmitter的run方法中，会执行DebounceTimedObserver
              //的emit方法
              d = worker.schedule(de, timeout, unit);

              de.setResource(d);
          }

      }

       void emit(long idx, T t, DebounceEmitter<T> emitter) {
              //对比当前index和debounceemitter所带的inx是否相同，相同就发射，然后dipose掉。
              if (idx == index) {
                  actual.onNext(t);
                  emitter.dispose();
              }
          }

```

2.  Distinct操作符：取出Observable发射的重复的item
```
    使用：取出重复的元素
     Observable.just(1,2,2,3,4,4,5)
                    .distinct()
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            System.out.println(integer);
                        }
                    });

    源码：
        public final Observable<T> distinct() {
            return distinct(Functions.identity(), Functions.createHashSet());
        }

        public final <K> Observable<T> distinct(Function<? super T, K> keySelector, Callable<? extends Collection<? super K>> collectionSupplier) {
            //返回的是ObservableDistinct实例，keySelector用于去重对比的key值生成器，collectionSupplier提供缓存元素key的集合，默认是hashSet
            return RxJavaPlugins.onAssembly(new ObservableDistinct<T, K>(this, keySelector, collectionSupplier));
        }

    //ObservableDistinct：

        protected void subscribeActual(Observer<? super T> observer) {
             Collection<? super K> collection;

             try {
                //生成一个缓存元素key的集合
                 collection = ObjectHelper.requireNonNull(collectionSupplier.call(), "The collectionSupplier returned a null collection. Null values are generally not allowed in 2.x operators and sources.");
             } catch (Throwable ex) {
                 Exceptions.throwIfFatal(ex);
                 EmptyDisposable.error(ex, observer);
                 return;
             }

            //创建一个包装了外部observer的DistinctObserver观察者，订阅源Observable
            source.subscribe(new DistinctObserver<T, K>(observer, keySelector, collection));
        }

    //DistinctObserver:
     public void onNext(T value) {
        if (done) {
            return;
        }
        if (sourceMode == NONE) {
            K key;
            boolean b;

            try {
                //将源Observable发射的value生成对应的key
                key = ObjectHelper.requireNonNull(keySelector.apply(value), "The keySelector returned a null key");
                //将key值保存在集合种，如果原来集合不包含key，则返回true
                b = collection.add(key);
            } catch (Throwable ex) {
                fail(ex);
                return;
            }

            //如果b为true，也就是原来集合不包含key，者执行onNext
            if (b) {
                actual.onNext(value);
            }
        } else {
            actual.onNext(null);
        }
    }
```

3.  ElementAt操作符：从Observable发射的item中，选出固定位置的item发送给观察者
```
    使用：
     Observable.just(1,2,3,4)
                    .elementAt(2)
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            System.out.println(integer);
                        }
                    });

     源码：
     public final Maybe<T> elementAt(long index) {
        //index不能小于0
         if (index < 0) {
             throw new IndexOutOfBoundsException("index >= 0 required but it was " + index);
         }

         //返回的实际是包装了源Observable的ObservableElementAtMaybe实例
         return RxJavaPlugins.onAssembly(new ObservableElementAtMaybe<T>(this, index));
     }

     //ObservableElementAtMaybe

         public void subscribeActual(MaybeObserver<? super T> t) {
             source.subscribe(new ElementAtObserver<T>(t, index));
         }

     //ElementAtObserver:

         public void onNext(T t) {
             if (done) {
                 return;
             }
             //count表示当前计数
             long c = count;

             //当count和我们要求的index位置相同时，发射数据
             if (c == index) {
                 done = true;
                 s.dispose();
                 actual.onSuccess(t);
                 return;
             }
             count = c + 1;
         }
```

4.  filter操作符：只发射通过条件判断的item
```
    使用：只通过test返回true的item
     Observable.just(1,2,3,4,5)
                    .filter(new Predicate<Integer>() {
                        @Override
                        public boolean test(Integer integer) throws Exception {
                            return integer>3;
                        }
                    })
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            System.out.println(integer);
                        }
                    });

     实现原理：和其他操作符一样，都是通过返回包装了包装的源Observable的ObservableFIlter，其内部订阅的时候，
     生成一个FilterObserver包装外部的observer，在执行onNext调用Predicate，如果返回true，就调用
     外部observer的onNext方法，否则丢弃当前item

```

5.  First操作符：只发出Observable发出的第一个项目（或符合某些条件的第一个项目）
```
    //使用
    Observable.just(1,2,3,4)
                .firstElement()
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        System.out.println(integer);
                    }
                });
```

6.  ignoreElements操作符：不发射任何item，但是要发射complete通知
```
     Observable.just(1,2,3,4)
                    .ignoreElements()
                    .subscribe(new Action() {
                        @Override
                        public void run() throws Exception {
                            //当complete时，执行这里
                        }
                    });
```

7.  Last操作符：发射最后一个item
```
     Observable.just(1,2,3,4)
                    .lastElement()
                    .subscribe(new Consumer<Integer>() {
                        @Override
                        public void accept(Integer integer) throws Exception {
                            System.out.println(integer);
                        }
                    });

     //原理：内部会保存当前源Observable发射的item，当源Observable发射complete通知时，发射item给外部的观察者，然后发送complete通知

```

8.  Sample操作符：在周期性的时间间隔内发出Observable发出的最近的项目,当源Observable调用的onComplete后，周期性调度停止，同时通知外部观察者，onComplete
```
    Observable.just(1,2,3,4)
                .sample(1,TimeUnit.SECONDS)
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        System.out.println(integer);
                    }
                });
```

9.  Skip操作符：丢弃源Observable发射的前n个item
```
    Observable.just(1,2,3,4)
                .skip(2)
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        System.out.println(integer);
                    }
                });

    原理：在skip操作符生成的SkipObserver中有个计数器，当接受到源Observaer发射item时，计数器-1.知道计数器
    为0是，将item，发射给外部的观察者
```

10. SkipLast操作符：压制源Observable发出的最后n个项目
```
    Observable.just(1,2,3,4)
                .skipLast(2)
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        System.out.println(integer);
                    }
                });

    原理：
        内部有个ArrayDeque用来保存数据，当接受到源Observabe发的数据时，先判断队列中的元素个数是否等于
        skip个数，不等于，将元素添加到队列尾部，等于，则取出队列头部的数据，发射给外部的观察者，然后再将数据
        放进队列的尾部
```

11. take操作符：只发射前源observable发射的前n个item
```
    Observable.just(1,2,3,4,5)
                .take(3)
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        System.out.println(integer);
                    }
                });

    原理：
        内部也是有个计数器，当计数器不为0时，才发射item给外部观察者，同时，当计数器为0的那一个itme时，还要发送onComplet给外部观察者，
```

12. takeLast操作符：发射源Observable发射的后n个item
```
    原理：内部也是通过ArrayDeque队列实现的，当源Observable发射数据来时，判断队列中元素个个数是否为n，不为n，
    则将元素添加到队列尾部，为n的时候，先把队列头部的数据出队，然后再把新数据添加到队列。当源Observable调用
    onComplete时，操作符生成的Observer的onCompete内部循环发射队列中的item给外部观察者。
```