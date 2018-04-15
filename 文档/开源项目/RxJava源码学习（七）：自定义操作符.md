#   RxJava源码学习（七）：自定义操作符
*   如果自定义的操作符被用来对源Observable发射的单个item起作用，实现ObservableOperator接口，然后通过lift操作符与
系统的操作符连接。
*   如果自定义的操作符被用来对源Observable整体其作用，则实现ObservableTransformer接口，然后通过compose操作符
与系统的操作符连接。


1. 定义操作符：通过实现ObservableOperator接口
```
public class MyOperator<T> implements ObservableOperator<T,T> {

    @Override
    public Observer<? super T> apply(final Observer<? super T> observer) throws Exception {
        return new Observer<T>() {
            @Override
            public void onSubscribe(Disposable d) {
                observer.onSubscribe(d);
            }

            @Override
            public void onNext(T t) {
                System.out.println("MyOperation:  onNext");
                observer.onNext(t);
                System.out.println("MyOperation:  afterOnNext");
            }

            @Override
            public void onError(Throwable e) {
                observer.onError(e);
            }

            @Override
            public void onComplete() {
                observer.onComplete();
            }
        };
    }
}
```

2. 使用自定义操作符：
```
    Observable.just(1,2,3)
        //通过lift操作符完成自定义操作符与系统操作符连接
        .lift(new MyOperator<Integer>())
        .subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                System.out.println(integer);
            }
        });

```

3.  lift操作符源码分析：

```
        public final <R> Observable<R> lift(ObservableOperator<? extends R, ? super T> lifter) {

            返回的是一个包装了源Observable的ObservableLift实例
            return RxJavaPlugins.onAssembly(new ObservableLift<R, T>(this, lifter));
        }

        //ObservableLift
         public void subscribeActual(Observer<? super R> s) {
                Observer<? super T> observer;
                try {

                    //通过自定义的操作符operator.apply转为外部的observable为自定义操作符中生成
                    //的包装Observer
                    observer = ObjectHelper.requireNonNull(operator.apply(s), "Operator " + operator + " returned a null Observer");

                } catch (NullPointerException e) { // NOPMD
                    throw e;
                } catch (Throwable e) {
                    Exceptions.throwIfFatal(e);
                    // can't call onError because no way to know if a Disposable has been set or not
                    // can't call onSubscribe because the call might have set a Disposable already
                    RxJavaPlugins.onError(e);

                    NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
                    npe.initCause(e);
                    throw npe;
                }


                //然后用包装Observer来订阅源Observable，这样，源Observable发射数据时，首先接到数据的则是我们自定义操作符中
                //的包装Observer
                source.subscribe(observer);
        }
```

4.  自定义操作符：实现ObservableTransformer接口：
```
    public class MyOperator2<T> implements ObservableTransformer<T,T> {
        @Override
        public ObservableSource<T> apply(Observable<T> upstream) {
            System.out.println("my operator apply");
            return upstream;
        }
    }

```

5.  通过compose操作符使用：
```
    Observable.just(1,2,3)
                .compose(new MyOperator2<Integer>())
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        System.out.println(integer);
                    }
                });
```

6. compose操作符源码：
```
    public final <R> Observable<R> compose(ObservableTransformer<? super T, ? extends R> composer) {
        //1.首先将当前源Observable，传递给composer进行变化，返回一个新的ObservableSource
        //2.对新的ObservableSource执行wrap操作
        return wrap(((ObservableTransformer<T, R>) ObjectHelper.requireNonNull(composer, "composer is null")).apply(this));
    }

    //wrap操作：
    public static <T> Observable<T> wrap(ObservableSource<T> source) {

        //如果当前source是一个Observable，则执行返回这个Observable
        if (source instanceof Observable) {
            return RxJavaPlugins.onAssembly((Observable<T>)source);
        }

        //否则通过ObservaleFromUnSafeSource包装source，然后返回
        return RxJavaPlugins.onAssembly(new ObservableFromUnsafeSource<T>(source));
    }

    //ObservableFromUnSafeSource： 其继承了Observable
    protected void subscribeActual(Observer<? super T> observer) {
        source.subscribe(observer);
    }
```