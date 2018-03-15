#   View 的绘制流程分析
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
        //mHandlingLayoutInLayoutRequest标志表示是否正在执行layout
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
        //
        boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
        if (layoutRequested) {
            final Resources res = mView.getContext().getResources();
             .......
            // 通过measureHierarchy方法
            windowSizeMayChange |= measureHierarchy(host, lp, res,
                    desiredWindowWidth, desiredWindowHeight);
        }
        ......
        //当第一次执行performTraversals()方法时或者window大小变化等会执行这里方法
        if (mFirst || windowShouldResize || insetsChanged ||
                viewVisibilityChanged || params != null || mForceNextWindowRelayout){
                ....
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


        final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
         ....
        //如果需要layout就执行performLayout过程
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

        if (DEBUG_ORIENTATION || DEBUG_LAYOUT) Log.v(mTag,
                "Measuring " + host + " in display " + desiredWindowWidth
                + "x" + desiredWindowHeight + "...");

        boolean goodMeasure = false;
        if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
            // On large screens, we don't want to allow dialogs to just
            // stretch to fill the entire width of the screen to display
            // one line of text.  First try doing the layout at a smaller
            // size to see if it will fit.
            final DisplayMetrics packageMetrics = res.getDisplayMetrics();
            res.getValue(com.android.internal.R.dimen.config_prefDialogWidth, mTmpValue, true);
            int baseSize = 0;
            if (mTmpValue.type == TypedValue.TYPE_DIMENSION) {
                baseSize = (int)mTmpValue.getDimension(packageMetrics);
            }
            if (DEBUG_DIALOG) Log.v(mTag, "Window " + mView + ": baseSize=" + baseSize
                    + ", desiredWindowWidth=" + desiredWindowWidth);
            if (baseSize != 0 && desiredWindowWidth > baseSize) {
                childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
                childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                if (DEBUG_DIALOG) Log.v(mTag, "Window " + mView + ": measured ("
                        + host.getMeasuredWidth() + "," + host.getMeasuredHeight()
                        + ") from width spec: " + MeasureSpec.toString(childWidthMeasureSpec)
                        + " and height spec: " + MeasureSpec.toString(childHeightMeasureSpec));
                if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                    goodMeasure = true;
                } else {
                    // Didn't fit in that size... try expanding a bit.
                    baseSize = (baseSize+desiredWindowWidth)/2;
                    if (DEBUG_DIALOG) Log.v(mTag, "Window " + mView + ": next baseSize="
                            + baseSize);
                    childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                    if (DEBUG_DIALOG) Log.v(mTag, "Window " + mView + ": measured ("
                            + host.getMeasuredWidth() + "," + host.getMeasuredHeight() + ")");
                    if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                        if (DEBUG_DIALOG) Log.v(mTag, "Good!");
                        goodMeasure = true;
                    }
                }
            }
        }

        if (!goodMeasure) {
            childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
            childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
                windowSizeMayChange = true;
            }
        }

        if (DBG) {
            System.out.println("======================================");
            System.out.println("performTraversals -- after measure");
            host.debug();
        }

        return windowSizeMayChange;
    }
```
