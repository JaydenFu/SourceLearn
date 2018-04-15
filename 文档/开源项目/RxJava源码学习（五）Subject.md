#   RxJava源码学习：Subject
Subject 继承了Observable，同时又实现了Observer，因此它既可以当做观察者使用，也可以当做Observable使用，
当用一个Subject去订阅一个冷Observable时，这可以使得该Subject成为一个热Observable。

RxJava中有4种Subject：AsyncSubject、BehaviorSubject、PublishSubject、ReplaySubject

1.  AsyncSubject
```
AsyncSubject发出由源Observable发出的最后一个值（并且只发布最后一个值），
并且仅在该源Observable完成之后发出。（如果源Observable不发出任何值，
则AsyncSubject也会完成而不会发出任何值。）

源码：
        public void onNext(T t) {
            ObjectHelper.requireNonNull(t, "onNext called with null. Null values are generally not allowed in 2.x operators and sources.");
            if (subscribers.get() == TERMINATED) {
                return;
            }

            //始终只保存源Onservable发射的最近的一个item
            value = t;
        }

        public void onComplete() {
                if (subscribers.get() == TERMINATED) {
                    return;
                }
                T v = value;

                //当源Observable发射onComplete通知时，AsyncSubject判断是否有Observer订阅当前Subject，
                //通知所有订阅了的Observer最后一个value，同时发出onComplete通知
                AsyncDisposable<T>[] array = subscribers.getAndSet(TERMINATED);
                if (v == null) {
                    for (AsyncDisposable<T> as : array) {
                        as.onComplete();
                    }
                } else {
                    for (AsyncDisposable<T> as : array) {
                        as.complete(v);
                    }
                }
            }
```

2.  BehaviorSubject
```
当观察者订阅BehaviorSubject时，它首先发布源Observable最近发出的项目（或者种子/默认值，如果还没有发出的话），
然后继续发送源Observable发送的任何其他项目（ S）,如果源Observable发射错误后，才订阅subject，则只会接收到onError通知

```

3.  PublishSubject
```
PublishSubject仅向观察者发出由订阅时间之后的源Observable发出的那些项目。

请注意，一个PublishSubject可能会在创建后立即开始发布项目（除非您已采取措施来防止这种情况发生），因此存在创建主体和观察者订阅该项目之
间可能会丢失一个或多个项目的风险。 如果您需要保证从源Observable交付所有项目，则需要使用Create来形成Observable，
以便您可以手动重新引入“冷”Observable行为（在开始发射项目之前查看所有观察者已订阅 ），或者切换到使用ReplaySubject。
```

4.  ReplaySubject
```
ReplaySubject向任何观察者发出由源Observable（s）发出的所有项目，而不管观察者何时订阅。

还有ReplaySubject的版本会在重播缓冲区威胁超出特定大小，或者自项目最初发出后指定的时间间隔过去时丢弃旧项目。

如果您使用ReplaySubject作为观察者，请注意不要从多个线程调用其onNext方法（或其他方法），
因为这可能会导致重合（非顺序）调用，这违反了Observable合同并会产生歧义 在结果主题中，首先应该重播哪个项目
或通知。
```