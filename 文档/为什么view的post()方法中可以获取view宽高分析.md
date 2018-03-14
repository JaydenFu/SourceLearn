#   View的post方法为什么可以正确获取到View的宽高?

要问这个问题,其实就是问View的post方法究竟是什么时候被执行的?

1.  View的post方法:
```
    public boolean post(Runnable action) {
        final AttachInfo attachInfo = mAttachInfo;

        //当attchInfo不为null时,采用attachInfo中的mHandler的post执行,这个mHandler其实是ViewRootImpl中的ViewRootHandler实例mHandler.
        //attchInfo是View 的dispatchAttachedToWindow中被赋值.

        if (attachInfo != null) {
            return attachInfo.mHandler.post(action);
        }

        //如果attachInfo为null,则将action添加进一个队列当中.这里添加了何时执行呢?
        //其实是在dispatchAttachToWindow中
        getRunQueue().post(action);
        return true;
    }

```
2.  View的dispatchAttachToWindow方法:
```
    //AttachInfo是从ViewRootImpl中传递下来的.
    void dispatchAttachedToWindow(AttachInfo info, int visibility) {
        //对mAttachInfo赋值
        mAttachInfo = info;
         ....
        // 执行队列中的action.用的页是AttachInfo中的mHandler
        if (mRunQueue != null) {
            mRunQueue.executeActions(info.mHandler);
            mRunQueue = null;
        }
         ...
        onAttachedToWindow();
        ....
    }

```
3.  View的dispatchAttachToWindow方法是何时调用的.分两种情况
    *   父容器已经执行过dispatchAttachToWindow方法.则子View在被添加进父容器的过程中执行子View的dispatchAttachToWindow方法
    *   父容器还未执行过.则子View的dispatchAttachToWindow方法是在父View的dispatchAttachToWindow执行时被执行的.
    ```
    //ViewgGroup的addViewInner方法
    private void addViewInner(View child, int index, LayoutParams params,
            boolean preventRequestLayout) {
        ....
        AttachInfo ai = mAttachInfo;

        //当attachInfo不为null,且该view没有设置FLAG_PREVENT_DISPATCH_ATTACHED_TO_WINDOW标志
        //FLAG_PREVENT_DISPATCH_ATTACHED_TO_WINDOW标志只在ViewGroup的dispatchAttachToWindow的方法中被设置,同时也会被取消.
        if (ai != null && (mGroupFlags & FLAG_PREVENT_DISPATCH_ATTACHED_TO_WINDOW) == 0) {
            .....
            child.dispatchAttachedToWindow(mAttachInfo, (mViewFlags&VISIBILITY_MASK));
            ....
        }
        ....
    }

    //ViewGroup的dispatchAttachedToWindow方法.
        @Override
        void dispatchAttachedToWindow(AttachInfo info, int visibility) {
            mGroupFlags |= FLAG_PREVENT_DISPATCH_ATTACHED_TO_WINDOW;
            super.dispatchAttachedToWindow(info, visibility);
            mGroupFlags &= ~FLAG_PREVENT_DISPATCH_ATTACHED_TO_WINDOW;

            final int count = mChildrenCount;
            final View[] children = mChildren;
            for (int i = 0; i < count; i++) {
                final View child = children[i];
                child.dispatchAttachedToWindow(info,
                        combineVisibility(visibility, child.getVisibility()));
            }
           .....
        }
    ```
4.  对于顶级view DecorView的dispatchAttachToWindow方法是在何时被调用的呢?是在
```
    //在ViewRootImpl的performTranversals()方法中:
    private void performTraversals(){
        .....
         if (mFirst) {//mFirst标志表示是否是第一次执行performTraversals方法
                ....
                //这里的host就是我们的DecroView.由于DecorView是继承与FrameLayout.所以该方法会一直传递到所有子view上.
                host.dispatchAttachedToWindow(mAttachInfo, 0);
                mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(true);
                dispatchApplyInsets(host);
                .....
         }
        ....
    }
```
5.  由于执行performTraversals是在主线程上执行.View中的post最后是通过ViewRootImpl中的mHandler发送消息的.所以需要等当前线程执行完了.才会
    执行消息中的事件.由于performTraversals中会执行performMeasure,performLayout.performDraw方法.因此当mHandler方法发出的消息被执行时,
    view的绘制流程已经结束.因此它可以正确获取到View的宽高.