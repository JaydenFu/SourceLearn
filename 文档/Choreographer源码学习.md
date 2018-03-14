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
5.  MessageQueue的postSyncBarrier