#   Choreographer 源码学习
该类是用来协调动画,输入,绘制的时间.它从底层显示子系统接收定时脉冲(timing pulses)比如垂直脉冲(vertical synchronization),
然后协调工作准备内容给下一帧绘制的时候使用.
应用通常通过动画框架.View 层级和该类打交道.
比如:
* View.postOnAnimation(Runnable callback) 当接收到下一帧的脉冲时,回调callback的run方法
* View.postOnAnimationDelay(Runnable callback,long delayMillis) 当接收到下一帧的脉冲时,延迟delayMillis再回调callback的run方法
* View.postInvalidateOnAnimation().当接收到下一帧脉冲时,调用view的invalidate方法
* Choreographer.postFrameCallback(FrameCallback callback),当接收到下一帧脉冲时,回调FrameCallback的doFrame方法.通过该方法,我们可以自己
实现监控是否丢帧的问题.
通过查看上述方法的源码,最后其实都是通过ViewRootImpl中的Choreographer来完成工作调度的.

1.  ViewRootImpl的构造方法中获取Choreographer实例.
```
    public ViewRootImpl(Context context, Display display) {
        ....
        mChoreographer = Choreographer.getInstance();
        ...
    }
```
2.  Choreographer.getInstance()方法:
```
    //其是ThreadLocal的.所以每个线程的实例都是独有的.线程一定要有Looper实例,Choreographer才能够构造成功.
    public static Choreographer getInstance() {
        return sThreadInstance.get();
    }

    private static final ThreadLocal<Choreographer> sThreadInstance =
            new ThreadLocal<Choreographer>() {
        @Override
        protected Choreographer initialValue() {
            Looper looper = Looper.myLooper();//对于主线程而言,Looper是在ActivityThread的main方法中被创建.
            if (looper == null) {
                throw new IllegalStateException("The current thread must have a looper!");
            }
            return new Choreographer(looper);
        }
    };
```
3.  ViewRootImpl中有个很重要的方法performTraversals()方法,对于View的measure,layout,draw过程都是在该方法发起.而这个方法只有doTraversal()
方法调用.而doTraversal()则是在TraversalRunnable的run方法中执行.而这个TraversalRunnable则在scheduleTraversals方法中被使用.

4.  ViewRootImpl的scheduleTraversals()方法:
```
    void scheduleTraversals() {

        //这个mTraversalScheduled标志位用与控制避免在同一帧类重复回调执行performTraversals()方法.
        if (!mTraversalScheduled) {

            //在scheduleTraversals中被设置为true,在unscheduleTrasversal()或doTraversal()方法中被设置为false.
            mTraversalScheduled = true;

            //向消息队列中发送一个同步屏障,作用就是阻塞消息队列的.普通消息会被阻塞,异步消息可以被执行.只要移除了该同步屏障
            //消息队列才会继续循环
            //mTraversalBarrier为该同步屏障的一个唯一识别标志,移除的需要使用该标志,才能正确移除.
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();

            //向Choreographer中添加一个CAHLLBACK_TRAVERSAL的callback.当下一帧脉冲接受时,回调.
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);

            .....
        }
    }
```
5.  MessageQueue的postSyncBarrier方法:插入同步屏障消息后,它之后的消息都会被阻塞.等分析Looper.Handler.MessageQueue原理时在详细说.
```
    public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

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
            //当when不为0的时候,会更具when时间排序,将新message插入到队列中.
            if (when != 0) {
                //查找when时间当前message的when之前的message.
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            //如果找到了,就插入到中间,否则,把当前message放在对列头.
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
6.  Choreographer的postCallback方法:更据不同的callbackType将action添加到不同的对列.
```
        public void postCallback(int callbackType, Runnable action, Object token) {
            //其实这里不止postCallback会调用该方法,postFrameCallback最后调用的也是方法.
            postCallbackDelayed(callbackType, action, token, 0);
        }

        //该方法主要做参数的校验,action为能为null.cahllbackType必须要是合法的.
        public void postCallbackDelayed(int callbackType,
                Runnable action, Object token, long delayMillis) {
            if (action == null) {
                throw new IllegalArgumentException("action must not be null");
            }
            if (callbackType < 0 || callbackType > CALLBACK_LAST) {
                throw new IllegalArgumentException("callbackType is invalid");
            }

            postCallbackDelayedInternal(callbackType, action, token, delayMillis);
        }

```
Choreographer的callbackType:
```
    Choreographer的callbackType有4种类型:这4种类型回调存在先后顺序:CALLBACK_INPUT > CALLBACK_ANIMATION > CALLBACK_TRAVERSAL > CALLBACK_COMMIT
        * CALLBACK_INPUT 值为0
        * CALLBACK_ANIMATION 值为1
        * CALLBACK_TRAVERSAL 值为2
        * CALLBACK_COMMIT 值为3
     4个值对应对应4个队列: 存在CallbackQueue的数组 长度为4.CallbackQueue中保存的是CallbackRecord.
     CallbackRecord:是个链表结构.

         private static final class CallbackRecord {
             public CallbackRecord next;
             public long dueTime;
             public Object action; // Runnable or FrameCallback
             public Object token;

             public void run(long frameTimeNanos) {
                 if (token == FRAME_CALLBACK_TOKEN) {
                     ((FrameCallback)action).doFrame(frameTimeNanos);
                 } else {
                     ((Runnable)action).run();
                 }
             }
         }
```
7.  Choreographer的postCallbackDelayedInternal方法:
```
    private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {

        synchronized (mLock) {
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;

            //将action回调添加到callbackType相对应的对列中,同样的页是根据dueTime的事件排序.
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

            if (dueTime <= now) {
                scheduleFrameLocked(now);
            } else {
                //这里其实最后也是去调用scheduleFrameLocked方法
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 = callbackType;
                msg.setAsynchronous(true);//以为前面添加了同步屏障,所以这里要设置异步,该message才会被处理.否则会被阻塞.
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }

```
8. Choreographer的scheduleFrameLocked方法:
```
        private void scheduleFrameLocked(long now) {
            //mFrameScheduled标志用于是否已经发起(结束)接受帧脉冲.当为true时,说明当次调度还没结束.不做其他处理
            if (!mFrameScheduled) {
                mFrameScheduled = true;
                if (USE_VSYNC) {//默认为true.
                    // If running on the Looper thread, then schedule the vsync immediately,
                    // otherwise post a message to schedule the vsync from the UI thread
                    // as soon as possible.
                    if (isRunningOnLooperThreadLocked()) {
                        scheduleVsyncLocked();
                    } else {
                        //这里如果线程没有looper,则会切换到mHandler的looper的线程中.然后执行scheduleVsyncLocked方法.
                        Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                        msg.setAsynchronous(true);
                        mHandler.sendMessageAtFrontOfQueue(msg);
                    }
                } else {
                    //当没有启用vsync时,默认10ms一帧,发出MSG_DO_FRAME消息
                    final long nextFrameTime = Math.max(
                            mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);//这里sFrameDelay默认值是10ms

                    //异步发送MSG_DO_FRAME消息,该消息实际执行的是Choreographer的doFrame方法.
                    Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtTime(msg, nextFrameTime);
                }
            }
        }
```
9.  Choreographer的scheduleVsyncLocked方法: 执行内部的FrameDisplayEventReceiver的scheduleVsync方法.
```
        private void scheduleVsyncLocked() {
            mDisplayEventReceiver.scheduleVsync();
        }

      //mDisplayEventReceiver是FrameDisplayEventReceiver,继承自DisplayEventReceiver.并且实现了Runnable接口.
      //该类重写了DisplayEventReceiver的onVsync方法.其在Choreographer的构造方法中被初始化.

      private Choreographer(Looper looper) {
          mLooper = looper;
          mHandler = new FrameHandler(looper);
          mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper) : null;
          mLastFrameTimeNanos = Long.MIN_VALUE;

          mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());

          mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
          for (int i = 0; i <= CALLBACK_LAST; i++) {
              mCallbackQueues[i] = new CallbackQueue();
          }
      }

```
10. DisplayEventReceiver的scheduleVsync方法:
```
    public void scheduleVsync() {
        if (mReceiverPtr == 0) {//mReceiverPtr在DisplayEventReceiver构造方法中被初始化.
            Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                    + "receiver has already been disposed.");
        } else {
            //native层去申请接收下一帧vsync,只有通过该方法申请了.才会接收到信号.
            //这里具体是否是这样,并不是从代码从分析出的.而是我在主线程sleep几百毫秒,发现没有打印跳帧的日志.
            //从而判断onVsync方法没有被调用.
            nativeScheduleVsync(mReceiverPtr);
        }
    }

    public DisplayEventReceiver(Looper looper) {
        if (looper == null) {
            throw new IllegalArgumentException("looper must not be null");
        }

        mMessageQueue = looper.getQueue();
        //通过native层初始化.
        mReceiverPtr = nativeInit(new WeakReference<DisplayEventReceiver>(this), mMessageQueue);

        mCloseGuard.open("dispose");
    }

```
11. 来看native层的nativeInit方法:该方法在android_view_DisplayEventReceiver.cpp文件中.
```
    static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak,
            jobject messageQueueObj) {
        sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
        if (messageQueue == NULL) {
            jniThrowRuntimeException(env, "MessageQueue is not initialized.");
            return 0;
        }

        //创建NativeDisplayEventReceiver实例
        sp<NativeDisplayEventReceiver> receiver = new NativeDisplayEventReceiver(env,
                receiverWeak, messageQueue);
        //执行初始化
        status_t status = receiver->initialize();

        if (status) {
            String8 message;
            message.appendFormat("Failed to initialize display event receiver.  status=%d", status);
            jniThrowRuntimeException(env, message.string());
            return 0;
        }

        //获取DisplayEventReceiver对象的引用,gDisplayEventReceiverClassInfo是个结构体,clazz保存的是java层的DisplayEventReciever.class
        receiver->incStrong(gDisplayEventReceiverClassInfo.clazz); // retain a reference for the object
        return reinterpret_cast<jlong>(receiver.get());
    }
```
12. 创建NativeDisplayEventReceiver:
```
    //NativeDisplayEventReciever继承自DisplayEventDispatcher. 而DisplayEventDispatch又继承自LooperCallback.

    NativeDisplayEventReceiver::NativeDisplayEventReceiver(JNIEnv* env,
            jobject receiverWeak, const sp<MessageQueue>& messageQueue) :
            DisplayEventDispatcher(messageQueue->getLooper()),
            mReceiverWeakGlobal(env->NewGlobalRef(receiverWeak)),
            mMessageQueue(messageQueue) {
        ALOGV("receiver %p ~ Initializing display event receiver.", this);
    }

```
13. NativeDisplayEventReceiver的initialize过程:    在DisplayEventDispatch.cpp中
```
    status_t DisplayEventDispatcher::initialize() {
        status_t result = mReceiver.initCheck();
        if (result) {
            ALOGW("Failed to initialize display event receiver, status=%d", result);
            return result;
        }
        //监听mReceiver获取的文件句柄,一旦有数据到来,则回调this的handEvent的方法.
        int rc = mLooper->addFd(mReceiver.getFd(), 0, Looper::EVENT_INPUT,
                this, NULL);
        if (rc < 0) {
            return UNKNOWN_ERROR;
        }
        return OK;
    }

```
14. DisplayEventDispatcher的handEvent方法: 在DisplayEventDispatcher.cpp中:
```
    int DisplayEventDispatcher::handleEvent(int, int events, void*) {
        .....
        // Drain all pending events, keep the last vsync.
        nsecs_t vsyncTimestamp;
        int32_t vsyncDisplayId;
        uint32_t vsyncCount;

        if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount)) {
            ALOGV("dispatcher %p ~ Vsync pulse: timestamp=%" PRId64 ", id=%d, count=%d",
                    this, ns2ms(vsyncTimestamp), vsyncDisplayId, vsyncCount);
            mWaitingForVsync = false;

            //发出脉冲信号.
            dispatchVsync(vsyncTimestamp, vsyncDisplayId, vsyncCount);
        }

        return 1; // keep the callback
    }

```
15. NativeDisplayEventReceiver重写了dispatchVsync方法: 在android_view_DisplayEventReceiver.cpp中:
```
    void NativeDisplayEventReceiver::dispatchVsync(nsecs_t timestamp, int32_t id, uint32_t count) {
        JNIEnv* env = AndroidRuntime::getJNIEnv();

        ScopedLocalRef<jobject> receiverObj(env, jniGetReferent(env, mReceiverWeakGlobal));
        if (receiverObj.get()) {

            //回调java层DisplayEventReceiver的dispatchVsync方法
            env->CallVoidMethod(receiverObj.get(),
                    gDisplayEventReceiverClassInfo.dispatchVsync, timestamp, id, count);

        }
        mMessageQueue->raiseAndClearException(env, "dispatchVsync");
    }

```
16. java层DisplayEventReceiver的dispatchVsync方法:
```
    private void dispatchVsync(long timestampNanos, int builtInDisplayId, int frame) {
        //调用onVsync方法,因为前面我们实际是FrameDislayEventReceiver.且该类重写了onVsync方法,
        onVsync(timestampNanos, builtInDisplayId, frame);
    }
```
17. Choreographer内部类: FrameDisplayEventReceiver的onVsync方法:
```
        public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
            //忽略来自第二屏幕的信号
            if (builtInDisplayId != SurfaceControl.BUILT_IN_DISPLAY_ID_MAIN) {
                scheduleVsync();
                return;
            }

            long now = System.nanoTime();
            if (timestampNanos > now) {
                Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
                        + " ms in the future!  Check that graphics HAL is generating vsync "
                        + "timestamps using the correct timebase.");
                timestampNanos = now;
            }

            if (mHavePendingVsync) {
                Log.w(TAG, "Already have a pending vsync event.  There should only be "
                        + "one at a time.");
            } else {
                mHavePendingVsync = true;
            }

            mTimestampNanos = timestampNanos;
            mFrame = frame;

            //这条消息异步发送,因为前面可能存在同步屏障,这条消息最后实际执行的是FrameDisplayEventReceiver的run方法,
            //因为它实现了Runnable.
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }
```
18. FrameDisplayEventReceiver的run方法: 执行doFrame方法mTimestampNanos为脉冲生成时间
```
        @Override
        public void run() {
            mHavePendingVsync = false;
            doFrame(mTimestampNanos, mFrame);
        }
```
19. Choreographer的doFrame方法:
```
    void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
          .....
            //当mFrameScheduled为false时,不做任何处理
            if (!mFrameScheduled) {
                  return; // no work to do
            }

            //frameTimeNanos为绘制帧的时间点.
            long intendedFrameTimeNanos = frameTimeNanos;
            //当前时间点
            startNanos = System.nanoTime();
            //当前时间和绘制帧之前的事件就是准备下一帧内容的时长
            final long jitterNanos = startNanos - frameTimeNanos;

            //如果当这个时间大于16.67ms 则就会跳帧.
            if (jitterNanos >= mFrameIntervalNanos) {
                //计算出跳过的帧数.
                final long skippedFrames = jitterNanos / mFrameIntervalNanos;

                //当系统发现跳帧超过30帧时就说明,主线程做了太多了事件,会通过日志告诉开发者.
                if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                    Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                            + "The application may be doing too much work on its main thread.");
                }

                //计算上一帧的偏移量.
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
                if (DEBUG_JANK) {
                    Log.d(TAG, "Missed vsync by " + (jitterNanos * 0.000001f) + " ms "
                            + "which is more than the frame interval of "
                            + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                            + "Skipping " + skippedFrames + " frames and setting frame "
                            + "time to " + (lastFrameOffset * 0.000001f) + " ms in the past.");
                }

                //计算出上一帧的实际时间.
                frameTimeNanos = startNanos - lastFrameOffset;
            }


            if (frameTimeNanos < mLastFrameTimeNanos) {
                if (DEBUG_JANK) {
                    Log.d(TAG, "Frame time appears to be going backwards.  May be due to a "
                            + "previously skipped frame.  Waiting for next vsync.");
                }
                scheduleVsyncLocked();
                return;
            }

            mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);

            //将mFrameScheduled标志设置false.可以再次接受scheduleFrameLocked调用
            mFrameScheduled = false;
            //保存上一帧绘制的时间.
            mLastFrameTimeNanos = frameTimeNanos;
        }

        try {

            AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

            //执行CALLBACK_INPUT队列中的回调
            mFrameInfo.markInputHandlingStart();
            doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

            //执行CALLBACK_ANIMATION的回调
            mFrameInfo.markAnimationsStart();
            doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

            //执行CALLBACK_TRAVERSAL会中回调
            mFrameInfo.markPerformTraversalsStart();
            doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

            //执行CALLBACK_COMMIT中的回调
            doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
        } finally {
            AnimationUtils.unlockAnimationClock();
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```
20. Choreographer的doCallbacks方法:
```
void doCallbacks(int callbackType, long frameTimeNanos) {
        CallbackRecord callbacks;
        synchronized (mLock) {

            final long now = System.nanoTime();

            //取出队列中所有执行时间在当前时间之前的callbacks
            callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                    now / TimeUtils.NANOS_PER_MS);

            if (callbacks == null) {
                return;
            }
            mCallbacksRunning = true;

            if (callbackType == Choreographer.CALLBACK_COMMIT) {
                final long jitterNanos = now - frameTimeNanos;

                if (jitterNanos >= 2 * mFrameIntervalNanos) {
                    final long lastFrameOffset = jitterNanos % mFrameIntervalNanos
                            + mFrameIntervalNanos;

                    frameTimeNanos = now - lastFrameOffset;
                    mLastFrameTimeNanos = frameTimeNanos;
                }
            }
        }
        try {
            //按顺序执行CallRecord的run方法.
            for (CallbackRecord c = callbacks; c != null; c = c.next) {
                c.run(frameTimeNanos);
            }
        } finally {
            //这里释放callback回对象池
            synchronized (mLock) {
                mCallbacksRunning = false;
                do {
                    final CallbackRecord next = callbacks.next;
                    recycleCallbackLocked(callbacks);
                    callbacks = next;
                } while (callbacks != null);
            }
        }
    }
```
21. CallbackRecord的run方法:
```
        public void run(long frameTimeNanos) {
            //这里区分FrameCallback和普通callback.回调方式不一样.
            if (token == FRAME_CALLBACK_TOKEN) {
                ((FrameCallback)action).doFrame(frameTimeNanos);
            } else {
                ((Runnable)action).run();
            }
        }

     //对于前面第4步中来说,这里就是回调ViewRootImpl中的TranversalRunnable的run方法.执行其内部的doTranversal方法.

```
22. ViewRootImpl的doTraversal方法
```
    void doTraversal() {
        if (mTraversalScheduled) {//mTraversalScheduled标志是在scheduleTranversals方法中设置为true的.
            mTraversalScheduled = false;

            //根据scheduleTranversals方法中生成的mTraversalBarrier,移除该同步屏障.
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }

```

```
对于Choregrapher的调用流程总结:以postCallback为例.调用流程中的参数省略了.

->  Choregrapher.postCallback(), postFrameCallback()
->  Choreographer.postCallbackDelayInternal()
->  CallbackQueue.addCallbackLocked()
->  Choregrapher.scheduleFrameLocked()  在这里会设置mFrameScheduled值为true
->  Choregrapher.scheduleVsyncLocked()
->  FrameDisplayEventReceiver.scheduleVsync()
->  DisplayEventReceiver.nativeScheduleVsync() 一定需要通过该放法申请了,才会接受到event.
->  native层的DisplayEventDispatcher.handEvent()
->  nvative层的DisplayEventDispatcher.processPendingEvents()
->  native层的NativeDisplayEventReceiver.dispatchVsync()
->  这里回到java层:DisplayEventReceiver.dispatchVsync()
->  Choregrapher.FrameDisplayEventReceiver.onVsync()
->  FrameDisplayEventReceiver.run()
->  Choreographer.doFrame() 在这里会判断mFrameScheduled值,如果不为true,则不做任何处理,如果为true,在执行doCallback前设置mFrameScheduled为false.
->  Choreographer.doCallbacks()
->  ChallbackRecord.run()
->  开始添加的为Runnbale则回调其run方法.为FrameCallback,则回调其doFrame方法.
```