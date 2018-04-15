#   RxJava源码学习（四）操作符：打包

1. Join操作符：类似于将两个Observable发射出的item进行组合，当第二个Observable发射某一个数据时，与第一个Observable发射已经发射过的数据进行组合
```
 //结果为1a，2a
 Observable.just(1, 2)
                .subscribeOn(Schedulers.computation())
                .join(Observable.just("a","b","c").subscribeOn(Schedulers.computation()), new Function<Integer, ObservableSource<String>>() {
                    @Override
                    public ObservableSource<String> apply(Integer integer) throws Exception {
                        return Observable.just(integer+"");
                    }
                }, new Function<String, ObservableSource<String>>() {
                    @Override
                    public ObservableSource<String> apply(String s) throws Exception {
                        return Observable.just(s);
                    }
                }, new BiFunction<Integer, String, String>() {
                    @Override
                    public String apply(Integer integer, String s) throws Exception {
                        return integer+s;
                    }
                })
                .subscribeWith(new DisposableObserver<String>() {
                    @Override
                    public void onNext(String s) {
                        System.out.println(s);
                    }

                    @Override
                    public void onError(Throwable e) {
                        System.out.println("onError:"+e.getMessage());
                    }

                    @Override
                    public void onComplete() {
                        System.out.println("onComplete");
                    }
                });
```

2.  Merge操作符：将多个Observable按顺序合并成一个Observable，然后发射数据
```
    Observable.just(1,2)
                .mergeWith(Observable.just(3,4))
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        System.out.println(integer);
                    }
                });

    源码：
        public final Observable<T> mergeWith(ObservableSource<? extends T> other) {
            return merge(this, other);
        }

        public static <T> Observable<T> merge(ObservableSource<? extends T> source1, ObservableSource<? extends T> source2) {
            //内部通过fromArray操作符，合并成数组，然后再通过flatMap展开
            return fromArray(source1, source2).flatMap((Function)Functions.identity(), false, 2);
        }
```

3.  zip操作符：通过指定的函数将多个Observable的发射结合在一起，并为每个组合发射单个项目
```
    使用：注意，这里以两个Observable发射的个数少的为界，比如这里结果就是1a，2b
    Observable.just(1,2)
                    .zipWith(Observable.just("a", "b","c"), new BiFunction<Integer, String, Object>() {
                        @Override
                        public Object apply(Integer integer, String s) throws Exception {
                            return integer+s;
                        }
                    })
            .subscribe(new Consumer<Object>() {
                @Override
                public void accept(Object o) throws Exception {
                    System.out.println(o);
                }
            });


    源码：
        public final <U, R> Observable<R> zipWith(ObservableSource<? extends U> other,
                BiFunction<? super T, ? super U, ? extends R> zipper) {

            //内部调用zip静态方法
            return zip(this, other, zipper);
        }

        public static <T1, T2, R> Observable<R> zip(
                ObservableSource<? extends T1> source1, ObservableSource<? extends T2> source2,
                BiFunction<? super T1, ? super T2, ? extends R> zipper) {

            //内部通过zipArray完成
            return zipArray(Functions.toFunction(zipper), false, bufferSize(), source1, source2);
        }

```

4.  StartWith操作符：在开始从源Observable发射项目之前发射指定的项目序列
```
    使用：结果为-1，-2，1，2，3
    Observable.just(1,2,3)
                .startWithArray(-1,-2)
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        System.out.println(integer);
                    }
                });

    原理：内部是通过fromArray操作符实现
```

5.  Switch操作符：将源Observable发射的item（也是一个Observable），将这些item Observabl发射的元素按顺序发射给外部观察者
```
    使用：结果为 1，2，3，2，4，8
      Observable.just(Observable.just(1,2,3),Observable.just(2,4,8))
                     .switchMap(new Function<Observable<Integer>, ObservableSource<?>>() {
                         @Override
                         public ObservableSource<?> apply(Observable<Integer> integerObservable) throws Exception {
                             return integerObservable;
                         }
                     }).subscribe(new Consumer<Object>() {
                 @Override
                 public void accept(Object o) throws Exception {
                     System.out.println(o);
                 }
             });

```