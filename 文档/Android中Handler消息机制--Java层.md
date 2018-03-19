#   Android中Handler消息机制--Java层

涉及到的类:Handler Looper MessageQueue Message.以下分析都是基于主线程的Handler,Looper.messageQueue分析
1.  先看Handler:  对于我们常用一般就是默认构造方法.
```
    Handler类对外提供了4种构造方法:
        public Handler() {
            this(null, false);
        }

        public Handler(Callback callback) {
            this(callback, false);
        }

        public Handler(Looper looper) {
            this(looper, null, false);
        }

        public Handler(Looper looper, Callback callback) {
            this(looper, callback, false);
        }

        //隐藏的
        public Handler(Callback callback, boolean async) {
            if (FIND_POTENTIAL_LEAKS) {
                final Class<? extends Handler> klass = getClass();
                if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                        (klass.getModifiers() & Modifier.STATIC) == 0) {
                    Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                        klass.getCanonicalName());
                }
            }

            mLooper = Looper.myLooper();
            if (mLooper == null) {
                throw new RuntimeException(
                    "Can't create handler inside thread that has not called Looper.prepare()");
            }
            mQueue = mLooper.mQueue;
            mCallback = callback;
            mAsynchronous = async;
        }

        //隐藏的
        public Handler(Looper looper, Callback callback, boolean async) {
            mLooper = looper;
            mQueue = looper.mQueue;
            mCallback = callback;
            mAsynchronous = async;
        }
```
2.  对于默认的构造函数.我们可以知道.其最后调用的实际是Handler(Callback callback, boolean async)方法:
```
    会通过以下获取当前线程的Looper对象.以及当前线程的MessageQueue.
    mLooper = Looper.myLooper();
    mQueue = mLooper.mQueue;

    //对于主线程而言.主线程的Looper对象在ActivityThread的main方法中就被初始化了,通过:Looper.prepareMainLooper().
    public static void prepareMainLooper() {
        prepare(false);//主线程false是因为不允许退出.
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));//每个线程只允许创建一个Looper对象.
    }

    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }

    在ActivityThread的main方法中.当主线程的Looper对象呗创建后,会执行Looper.loop()方法.该方法是个死循环.一致从looper对应的消息队列中取出消息.
```
3.  Looper的构造函数:
```
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);在Looper的构造方法中,会创建对应的MessageQueue.
        mThread = Thread.currentThread();
    }
```
4.  Handler发送消息:有send方法以及post方法.
```
    send方式: 发送的都是消息
        1.sendMessage
        2.sendMessageAtFrontOfQueue 该方法内部直接调用enqueueMessage
        3.sendMessageAtTime
        4.sendMessageDelayed
        5.sendEmptyMessage
        6.sendEmptyMessageDelayed
        7.sendEmptyMessageAtTime
     post方式: post都是Runnable,内部会将Runnable包装成消息
        1.post
        2.postAtFrontOfQueue: 该方法内容调用sendMessageAtFrontOfQueue.
        3.postAtTime
        4.postDelayed

     除了sendMessageAtFrontOfQueue和postAtFrontOfQueue以外.其他方法最后都调用的sendMessageAtTime方法.
```
5.  Handler的sendMessageAtTime方法: 内部也是通过enqueueMessage将消息放进消息队列
```
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```
6.  MessageQueue的enqueueMessage方法:该方法通常是在子线程中被调用.
```
 boolean enqueueMessage(Message msg, long when) {
        ......
        synchronized (this) {//因为通常在子线程执行,所以需要同步锁.
            ......
            msg.markInUse();
            msg.when = when;
            Message p = mMessages;//mMessage表示当前消息队列中的消息头,Message本身是个链表结构,持有next Message(如果存在).
            boolean needWake;
            //消息头为空,或者被操作的消息的处理时间点为0.或者处理时间点小于当前消息头的处理时间.则将消息添加进消息头.
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                //如果之前队列被阻塞,则需要唤醒,在MessageQueue的next方法中的nativePollOnce方法会可能阻塞主线程.
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                //当消息是被插入到消息队列中间的后,通常我们不需要去唤醒队列.因为之前阻塞的时候是否设置超时时长的.
                //但是如果消息头是个同步屏障的话.那么当前阻塞是没有超时时间的.而这个时候传进来的又是个异步消息(且在队列中是最早的).则需要唤醒线程.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;

                //把消息插入进队列,更具时间排序
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                //通过native层去唤醒主线程.
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
8.  Looper的loop方法:在主线程执行.
```
 public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        //这里无限循环从消息队列中取消息.
        for (;;) {
            //从消息队列中取出下一个消息.该方法可能会阻塞当前线程.
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            final long traceTag = me.mTraceTag;

            try {
                //Message的target其实就是我们发送消息的handler.也就是说,在这里,将消息回传给我们的handler来执行
                msg.target.dispatchMessage(msg);
            } finally {
                ....
            }
            ....
            //消息回收,放回消息池中.
            msg.recycleUnchecked();
        }
    }
```
9.  Handler的dispatchMessage方法:
```
    //1.当msg中有callback时,执行msg的callback(其实是个Runnable)方法的run方法.
    //2.当msg中没有callback,但是handler本身设置有callback时,执行handler的callback的handMessage方法.
    //3.当以上两者都没有时,才会回调Handler的handMessage方法.
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```
10. MessageQueue的next方法:
```
 Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            //该方法可能会阻塞当前线程.nextPollTimeoutMillis为超时时间,当为0时,会立即返回,并不阻塞.当为-1时,会一直阻塞(是否一直阻塞也取决于native层的消息的执行时间).
            //当为其他值时,阻塞达到超时时长也会返回.
            //当循环被第一次执行时,nextPollTimeoutMillis为0.
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.

                //从消息队列中取消息.有几个规则要注意:
                //1.消息队列中消息头如果是个同步屏障消息,则会往后查找需要异步执行的消息.
                    如果找到了异步消息:
                        *   如果找到了,并且执行时间小于当前时间.则异步消息被返回到Looper.loop中被执行.
                        *   如果执行时间大于当前时间.则会对nextPollTimeoutMillis设值为两个时间差值.表示,等待这么长时间后,再来执行.
                    如果没找到异步消息:
                        *   nextPollTimeoutMillis为被赋值为-1.再循环时,就会一直阻塞当前线程.直到被wake唤醒.

                //2.消息队列中消息头不是同步屏障消息.
                    * 如果消息的时间小于当前时间.则结束当前循环,返回到Looper的loop方法中.将消息传给相应的handler处理消息.
                    * 如果消息的时间大于当前时间.则设置nextPollTimeoutMillis当前时间和消息的时间的差值.表示要阻塞的时长.

                //3.要理解MessageQueue的next方法的循环的意义:就是将达到执行时间的消息全部返回去执行.直到没有消息或者没有到消息的执行时间,阻塞线程.

                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.

                //程序能够执行到这里,说明没有找到需要当前处理的消息了.当前线程即将被阻塞.
                //会回调我们给MessageQueue设置的IdleHandler.

                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {//当没有设置IdleHandler时,会通过mBlocked标识当前线程阻塞.
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;//继续循环.在再次执行nativePollOnce时阻塞.
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```
11. MessageQueue的postSyncBarrier方法:向队列中添加同步屏障.用来阻塞主线程.在ViewRootImpl的scheduleTraversals方法中被用到了.
```
    public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

    //根据时间,插入队列中.同步屏障消息没有target.也就是说没有handler.
    private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
```
12. MessageQueue的removeSyncBarrier方法:
```
   //根据同步屏障的token值,移除当前消息队列中的同步屏障.
   public void removeSyncBarrier(int token) {
        // Remove a sync barrier token from the queue.
        // If the queue is no longer stalled by a barrier then wake it.
        synchronized (this) {
            Message prev = null;
            Message p = mMessages;
            while (p != null && (p.target != null || p.arg1 != token)) {
                prev = p;
                p = p.next;
            }
            if (p == null) {
                throw new IllegalStateException("The specified message queue synchronization "
                        + " barrier token has not been posted or has already been removed.");
            }
            final boolean needWake;

            if (prev != null) {//如果移除的同步屏障消息不是消息头.则没必要去主动唤醒,否则.需要我们通过nativeWake去唤醒线程.
                prev.next = p.next;
                needWake = false;
            } else {
                mMessages = p.next;
                needWake = mMessages == null || mMessages.target != null;
            }
            p.recycleUnchecked();

            // If the loop is quitting then it is already awake.
            // We can assume mPtr != 0 when mQuitting is false.
            if (needWake && !mQuitting) {
                nativeWake(mPtr);
            }
        }
    }
````


