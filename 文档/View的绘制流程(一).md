#   View 的绘制流程分析(一)
1.  我们都知道View的绘制分为3个步骤,measure,layout.draw.现在来分析这3个步骤时何时执行,是被谁触发执行的.
    这里又得提到ViewRootImpl了.因为这些流程是由ViewRootImpl来控制的.不管是当Activity第一次显示执行makeVisible,
    或者由View调用invalidate(),requestLayout()方法.其最后都是执行ViewRootImpl的scheduleTraversals()方法.通过该
    方法才调用到performTraversals()方法.
2. 先从第一次绘制来分析.在Activity的makeVisible方法中:
```
    void makeVisible() {
        //如果还没添加到wnindow中,则会执行WindowManger的addView方法,WindowManager涉及到IPC,其实这里wm是一个WindowManagerIml
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }
```
3.  WindowManagerImpl的addView方法.在该方法中又调用了
```
    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }

```
4. WindowManagerGlobal的addView方法.在该方法内部会调用ViewRootImpl的setView方法
```
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ......

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        if (parentWindow != null) {
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        } else {
            final Context context = view.getContext();
            if (context != null
                    && (context.getApplicationInfo().flags
                            & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
                wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
            }
        }

        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            ......
            //在这里创建ViewRootImpl的实例
            root = new ViewRootImpl(view.getContext(), display);
            view.setLayoutParams(wparams);

            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);

            try {
                //这里执行ViewRootImpl的setView方法
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {
                // BadTokenException or InvalidDisplayException, clean up.
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
                throw e;
            }
        }
    }

```
5.  ViewRootImpl的setView方法:在该方法内部会在添加到window之前,执行requestLayout.
```
  public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;
               .....
                // Schedule the first layout -before- adding to the window
                // manager, to make sure we do the relayout before receiving
                // any other events from the system.
                //这里执行第一次layout过程
                requestLayout();
                 ......
                try {
                  .......
                    //这里执行添加到window的操作
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
                } catch (RemoteException e) {
                   .......
                } finally {
                    if (restore) {
                        attrs.restore();
                    }
                }
            .......
                view.assignParent(this);
              .......
            }
        }
    }
```
6.  ViewRootImpl的requestLayout方法:
```
    @Override
    public void requestLayout() {
        //mHandlingLayoutInLayoutRequest标志意义参见第13步的A3
        //当在处理那些在layout过程中发起requestLayout的vie过程中发起的requestLayout执行到这里就不用管了.
        if (!mHandlingLayoutInLayoutRequest) {

            //检查线程是否正确,触发方法的线程是否是创建该view的线程,否则不合法
            checkThread();

            //mLayoutRequested标志表示是否发起了layout请求.执行requestLayout时被设置为true.
            //当执行performLayout的时会被设置为false.
            mLayoutRequested = true;

            //执行调度
            scheduleTraversals();
        }
    }
```
7.  ViewRootImpl的scheduleTraversals方法:执行过该方法,最后会执行方法内部发出的runnable.最后执行的是doTraversal()方法.
```
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
```
8.  ViewRootImpl的doTraversal()方法
```
    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            //最重要的方法来了.该方法内部触发了View的measure,layout,draw过程.
            performTraversals();
           ....
        }
    }
```
9.  ViewRootImpl的performTraversals方法:在该方法内部会视具体情况依次调用performMeasure,performLayout,performDraw方法.
    每个方法可能被调用一次或多次,也可能不会调用.但是对于第一次执行performTraversal方法而言,前面3个方法至少会调用一次.
```
    private void performTraversals() {

            .....
        boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
        if (layoutRequested) {
            final Resources res = mView.getContext().getResources();
             .......
            // 1. 通过measureHierarchy方法
            windowSizeMayChange |= measureHierarchy(host, lp, res,
                    desiredWindowWidth, desiredWindowHeight);
        }
        ......

        if (mFirst || windowShouldResize || insetsChanged ||
                viewVisibilityChanged || params != null || mForceNextWindowRelayout){
                ....
             //2.当窗口大小发生变化了,会发起重新测量.
             if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                                    || mHeight != host.getMeasuredHeight() || contentInsetsChanged ||
                                    updatedConfiguration) {

                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

                    int width = host.getMeasuredWidth();
                    int height = host.getMeasuredHeight();
                    boolean measureAgain = false;

                    //如果布局参数中设置了权重属性,则需要再次重新测量
                    if (lp.horizontalWeight > 0.0f) {
                        width += (int) ((mWidth - width) * lp.horizontalWeight);
                        childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(width,
                                MeasureSpec.EXACTLY);
                        measureAgain = true;
                    }
                    if (lp.verticalWeight > 0.0f) {
                        height += (int) ((mHeight - height) * lp.verticalWeight);
                        childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height,
                                MeasureSpec.EXACTLY);
                        measureAgain = true;
                    }

                    //更具前面是否需要重新测量,如果需要则执行performMesaure.
                    if (measureAgain) {
                        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                    }
                    layoutRequested = true;
             }
         }


        final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
         ....
        //3. 如果需要layout就执行performLayout过程
        if (didLayout) {
             performLayout(lp, mWidth, mHeight);
            ......
        }

         boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
         //如果可见且不是新surface.就执行performDraw
         if (!cancelDraw && !newSurface) {
             ......
             performDraw();
         } else {
            .....
         }
         .....
    }

```
10. ViewRootImpl的measureHierarchy方法:
```
    private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
            final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {

        int childWidthMeasureSpec;
        int childHeightMeasureSpec;
        boolean windowSizeMayChange = false;

        boolean goodMeasure = false;
        if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
            //对于Activity而言的不会执行这里,只有Dialog才会执行这里.
           .....
        }

        if (!goodMeasure) {
            //生成根MeasureSpec,对于分析的Actiivty而言,这里返回的就是大小为窗口大小,模式为EXACTLY的measureSpec.
            childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
            childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
            //执行测量
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

            //判断DecorView测量宽高是否和Window宽高不一致.如果不一致,则window大小可能变化了.该值为true时,会在后续流程中发起重新测量
            if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
                windowSizeMayChange = true;
            }
        }
        return windowSizeMayChange;
    }
```
11. ViewRootImpl的getRootMeasureSpec方法:
```
    private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            // 当WindowLayout.LayoutParams为MATCH_PARENT时,说明Window的大小不需要改变.大小为windowSize,模式为EXACTLY.
            // 对于我们主要研究的Actiivty而言,就只执行这里.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // 当WindowLayout.LayoutParams为MATCH_PARENT时,说明Window的大小可能改变.窗口的最大值为windowSize,模式为AT_MOST.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // 窗口固定大小.模式为EXACTLY.
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }

    //这里先来了解下MeasureSpec:
    MeasureSpec.makeMeasureSpec()方法返回一个int值.32位.
    高2位会设置成模式.低30位设置具体的值.
    有3种模式:
    *   UNSPECIFIED: 表示不指定子View的大小,想要多大就多大,通常不会使用到.
    *   EXACTLY:    表示指定子View的准确大小
    *   AT_MOST:    表示给定子View的最大大小.子View大小不能超过给定的值.

```
12. ViewRootImpl的performMeasure方法:
```
    private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        try {
            //执行DecorView的measure方法.从上面的分析中我们可以知道,对于顶级DecorView而言.
            //传递给它的measureSpec是根据window的大小和WindowManager.LayoutParams决定的.
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            ...
        }
    }
```
13. ViewRootImpl的performLayout方法:
```
 private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
        mLayoutRequested = false;
        mScrollMayChange = true;
        mInLayout = true;

        final View host = mView;
        try {
            //A1. 执行DecorView的layout过程
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

            mInLayout = false;

            //mLayoutRequesters是个ArrayList集合,里面存储的是在ViewRootImpl正在执行上面A1处的layout过程中,又发起了
            //requestLayout请求的子View.
            int numViewsRequestingLayout = mLayoutRequesters.size();

            if (numViewsRequestingLayout > 0) {
                //从mLayoutRequesters集合中选出正真需要重新执行layout的View
                //选择规则:
                //1.    view的不为null,view的attachInfo不为null,view的parent不为null
                //2.    view设置了View.PFLAG_FORCE_LAYOUT标识
                //3.    view以及其上面的所有父容器都不是Gone状态的
                ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters,
                        false);

                if (validLayoutRequesters != null) {

                    // Set this flag to indicate that any further requests are happening during
                    // the second pass, which may result in posting those requests to the next
                    // frame instead
                    //A2. 该标识表示正在处理那些在在A1步骤layout过程中发起layout的操作的View.
                    //该标志只在这里被设置为true,当该值被设置为true时,子View再发起reqeustLayout过程将不会立即按正常
                    //流程执行,而是会将发起reqeuestLayout的子View添加到mLayoutRequesters集合中.
                    mHandlingLayoutInLayoutRequest = true;

                    // 执行这些需要重新layout的View的requstLayout过程.
                    int numValidRequests = validLayoutRequesters.size();
                    for (int i = 0; i < numValidRequests; ++i) {
                        final View view = validLayoutRequesters.get(i);
                        //这一步其实主要是给这个发起reqeustLayout的View设置PFLAG_FORCE_LAYOUT标识
                        //因为当前mHandlingLayoutInLayoutRequest为true.,从第6步可以看出,是不会执行schduleTraversals()的
                        view.requestLayout();
                    }

                    //重新测量全部
                    measureHierarchy(host, lp, mView.getContext().getResources(),
                            desiredWindowWidth, desiredWindowHeight);

                    //该标识再次被设置,表示又在重新layout了.
                    mInLayout = true;

                    //A3.再次重新执行DecorView的layout过程,当在这里执行的过程中,子View再发起requestLayout.将不会
                    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

                    //该标志也只在这里设置为false.当该值为false时,子View发起requestLayout就会按正常流程被执行.
                    mHandlingLayoutInLayoutRequest = false;

                    //再次检测,在A2的layout执行过程中,又发起了reqeustLayout的子View.对这些View的reqeustLayout放到下一帧在进行处理
                    validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters, true);
                    if (validLayoutRequesters != null) {
                        final ArrayList<View> finalRequesters = validLayoutRequesters;
                        getRunQueue().post(new Runnable() {
                            @Override
                            public void run() {
                                int numValidRequests = finalRequesters.size();
                                for (int i = 0; i < numValidRequests; ++i) {
                                    final View view = finalRequesters.get(i);
                                    view.requestLayout();
                                }
                            }
                        });
                    }
                }

            }
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        //设置为false,表示layout过程结束
        mInLayout = false;
    }
```
14. ViewRootImpl的performDraw方法:在该方法内部会调用ViewRootImpl的draw方法
```
  private void performDraw() {
        ......
        final boolean fullRedrawNeeded = mFullRedrawNeeded;
        mFullRedrawNeeded = false;

        mIsDrawing = true;
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");
        try {
            //执行draw
            draw(fullRedrawNeeded);
        } finally {
            mIsDrawing = false;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        .........
    }

```
15. ViewRootImpl的draw方法:
```
  private void draw(boolean fullRedrawNeeded) {
        Surface surface = mSurface;
        boolean animating = mScroller != null && mScroller.computeScrollOffset();
        ......

        if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
            if (mAttachInfo.mHardwareRenderer != null && mAttachInfo.mHardwareRenderer.isEnabled()) {
                    .......
                //对于开启了硬件加速的执行这里
                //当我们在Manifest文件中对Application或者Actiivty设置了hardwareAccelerateds属性为true时,就是开启了窗口硬件加速
                //对于4.0以上的手机,系统是默认开始Application级的硬件加速的.所以我们分析这里就行了.对于非硬件加速模式就不分析了.

                //mHardwareRenderer是一个ThreadRenderer
                mAttachInfo.mHardwareRenderer.draw(mView, mAttachInfo, this);
            } else {
                //没有开启硬件加速.执行这里
                if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
                    return;
                }
            }
        }
        //如果在执行窗口动画,则继续调度直到动画结束
        if (animating) {
            mFullRedrawNeeded = true;
            scheduleTraversals();
        }
    }
```
16. ThreadedRender的draw方法:在内部执行updateRootDisplayList方法
```
    至于ThreadedRender是个啥?后面在学习Android的硬件加速渲染机制的时候再了解.

    void draw(View view, AttachInfo attachInfo, HardwareDrawCallbacks callbacks) {
        .....
        updateRootDisplayList(view, callbacks);
        .......
    }
```
17. ThreadedRender的updateRootDisplayList方法:在方法内部执行updateViewTreeDisplayList方法
```
    private void updateRootDisplayList(View view, HardwareDrawCallbacks callbacks) {
        ......
        updateViewTreeDisplayList(view);
         if (mRootNodeNeedsUpdate || !mRootNode.isValid()) {
                DisplayListCanvas canvas = mRootNode.start(mSurfaceWidth, mSurfaceHeight);
                try {
                    final int saveCount = canvas.save();
                    canvas.translate(mInsetLeft, mInsetTop);
                    callbacks.onHardwarePreDraw(canvas);

                    canvas.insertReorderBarrier();
                    canvas.drawRenderNode(view.updateDisplayListIfDirty());
                    canvas.insertInorderBarrier();

                    callbacks.onHardwarePostDraw(canvas);
                    canvas.restoreToCount(saveCount);
                    mRootNodeNeedsUpdate = false;
                } finally {
                    mRootNode.end(canvas);
                }
         }
        ......
    }

```
18. ThreadedRender的updateViewTreeDisplayList方法:
```
       private void updateViewTreeDisplayList(View view) {
           //这里的View其实就是DecorView.给DecorView设置View.PFLAG_DRAWN标识,表示可以被draw
           view.mPrivateFlags |= View.PFLAG_DRAWN;

           //当DecorView被设置了PFLAG_INVALIDATED标识,则表示要重建DisplayList
           //PFLAG_INVALIDATED该标识在调用invalidate时传入invalidateCache参数为true时被设置.同时在通过invalidate向上传递上,
           //1. 其直接父容器会根据子View是否设置了layerType.来设置该参数.如果发起invalidate的View的LayerType为TYPE_NONE,则其直接父容器不设置参数.
           //否则设置.(在父容器的invalidateChild方法中)
           //view.requestLayout正常流程时,也会被设置该参数.
           view.mRecreateDisplayList = (view.mPrivateFlags & View.PFLAG_INVALIDATED)
                   == View.PFLAG_INVALIDATED;

           //取消PFLAG_INVALIDATED标识
           view.mPrivateFlags &= ~View.PFLAG_INVALIDATED;
           //更新DisplayList
           view.updateDisplayListIfDirty();
           view.mRecreateDisplayList = false;
       }

```
19. View的updateDisplayListIfDirty方法:
```
 public RenderNode updateDisplayListIfDirty() {

        //RenderNode 持有View的所有属性,可能还通过DisplayList持有View的内容
        //每个View的RenderNode在View的构造方法中被初始化.
        final RenderNode renderNode = mRenderNode;
        .....

        if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
                || !renderNode.isValid()
                || (mRecreateDisplayList)) {

            //当前View的RenderNode是合法的,并且当前节点的DisplayList不需要被重建的情况
            if (renderNode.isValid() && !mRecreateDisplayList) {

                mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
                //清除dirty标识.
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;

                //该View如果是个ViewGroup.这该方法内部会让它的所有子View执行恢复或重建DisplayList.
                //对于View而言,该方法是个空方法
                dispatchGetDisplayList();

                return renderNode; // no work needed
            }

            // If we got here, we're recreating it. Mark it as such to ensure that
            // we copy in child display lists into ours in drawChild()

            //需要重建当前View节点的DisplayList
            mRecreateDisplayList = true;

            int width = mRight - mLeft;
            int height = mBottom - mTop;
            int layerType = getLayerType();

            //申请一个DisplayListCanvas去记录draw操作.
            //DisplsyListCanvas是一个GL Canvas的实现类,该类持有它要绘制的所有画笔/Bitmap对象.
            final DisplayListCanvas canvas = renderNode.start(width, height);
            canvas.setHighContrastText(mAttachInfo.mHighContrastText);

            try {
                if (layerType == LAYER_TYPE_SOFTWARE) {
                    //如果当前View被设置了View层级的LAYER_TYPE_SOFTWARE.默认layerType是为LYER_TYPE_NONE的.
                    //通过buildDrawingCache内部会重新采用不同的canvas绘制,然后将内容保存到一个bitmap上.
                    buildDrawingCache(true);

                    //拿到重新绘制好的缓存数据.绘制到DisplayListCanvas上面.
                    Bitmap cache = getDrawingCache(true);
                    if (cache != null) {
                        canvas.drawBitmap(cache, 0, 0, mLayerPaint);
                    }
                } else {
                    //如果没有设置layerType或者layerType设置为LAYER_TYPE_HARDWARD,则直接操作DisplayListCanvas
                    computeScroll();

                    canvas.translate(-mScrollX, -mScrollY);
                    mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
                    mPrivateFlags &= ~PFLAG_DIRTY_MASK;

                    // Fast path for layouts with no backgrounds
                    //当前View是否可以跳过绘制
                    if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                        //如果可以跳过绘制,则直接通过dispatchDraw往下分发绘制.
                        dispatchDraw(canvas);
                        if (mOverlay != null && !mOverlay.isEmpty()) {
                            mOverlay.getOverlayView().draw(canvas);
                        }
                    } else {
                        //不可跳过绘制时,执行View的draw方法.
                        draw(canvas);
                    }
                }
            } finally {
                //记录DisplayList的操作结束,在该方法内部会释放recycle Canvas
                //只有结束操作了.renderNode.isValid()才会返回true
                renderNode.end(canvas);
                setDisplayListProperties(renderNode);
            }
        } else {
            mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
        }
        return renderNode;
    }

```
20. ViewGroup的dispatchGetDisplayList方法:
```
    protected void dispatchGetDisplayList() {
        final int count = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < count; i++) {
            final View child = children[i];
            if (((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null)) {
                recreateChildDisplayList(child);
            }
        }
            .....
    }

```
21. ViewGroup的d recreateChildDisplayList方法:
```
    private void recreateChildDisplayList(View child) {
        child.mRecreateDisplayList = (child.mPrivateFlags & PFLAG_INVALIDATED) != 0;
        child.mPrivateFlags &= ~PFLAG_INVALIDATED;
        //执行子View的updateDisplayListIfDirty过程.和上面第19步一样.
        child.updateDisplayListIfDirty();
        child.mRecreateDisplayList = false;
    }
```
22. 对于我们本身分析的第一次绘制而言.前面第19步中View对应的是DecorView.因为是第一次绘制.所以肯定是需要执行draw方法的.也就是说,在
    DecorView的updateDisplayListIfDirty中执行DecorView的draw方法开始绘制流程.

```
分析到这里,我们终于知道DecorView的meausre,layout,draw方法是何时被执行的了.
总结一波流程:

->  ViewRootImpl.scheduleTraversals()
->  ViewRootImpl.doTraversal()
->  ViewRootImpl.performTraversals()
->  ViewRootImpl.measureHierarchy()
->  ViewRootImpl.performMeasure()//在performLayout之前,performMeasure可能会被多次调用.如:当前面的测量结果和窗口大小不一致,设备Configuration发生变化都会引用重新测量
->  ViewRootImpl.performLayout()
    ->在performLayout过程中的第一次调用DecorView的layout过程中,有子View调用了requestLayout.则会可能会导致再次执行measureHierarchy过程和layout过程.
    //至于是否会触发再次执行测量和layout过程,要看发起requestLayout的子View的状态.具体状态看第13步中.
->  ViewRootImpl.performDraw()
->  ViewRootImpl.draw()
->  ThreadedRender.draw() 硬件加速,4.0系统以上默认开启.
->  ThreadedRender.updateRootDisplayList()
->  TheadedRender.updateViewTreeDisplayList()
->  DecorView.updateDisplayListIfDirty()    DecorView继承自FrameLayout,其实执行的是View的updateDisplayListIfDirty方法
->  DecorView.draw()    DecorView继承自FrameLayout,其实执行的是View的draw方法.
```

