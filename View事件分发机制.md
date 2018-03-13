# View 事件分发机制  

1.  在ViewRootImpl的setView中初始化接收事件的InputChannel和WindowInputEventReceiver.
```
      public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView){

             .........
             //1.判断是否没有设置INPUT_FEATURE_NO_INPUT_CHANNEL标志.
             //如果设置了该标志,则整个view不再接受事件
             if ((mWindowAttributes.inputFeatures
                                    & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                 //初始化InputChannel,通过一个文件描述符来从另一个进程中给一个window发送输入事件.
                 mInputChannel = new InputChannel();
             }
             .......

             if (mInputChannel != null) {
                                 .....
                 //WindowInputEventReceiver继承自InputEventReceiver,用来接受输入事件.
                 mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,Looper.myLooper());
             }

      }
```
2.  WindowInputEventReceiver是ViewRootImpl的一个内部类.
```
final class WindowInputEventReceiver extends InputEventReceiver {
        public WindowInputEventReceiver(InputChannel inputChannel, Looper looper) {
            //调用InputEventReceiver的构造方法,native层建立联系.
            super(inputChannel, looper);
        }

        //当有输入事件时,native层回调该方法,当一个事件被消费掉后,一定需要调用finishInputEvent.否则无法接受后续事件.
        @Override
        public void onInputEvent(InputEvent event) {
            enqueueInputEvent(event, this, 0, true);
        }

        @Override
        public void onBatchedInputEventPending() {
            if (mUnbufferedInputDispatch) {
                super.onBatchedInputEventPending();
            } else {
                scheduleConsumeBatchedInput();
            }
        }

        @Override
        public void dispose() {
            unscheduleConsumeBatchedInput();
            super.dispose();
        }
    }
```
3.  InputEventReceiver的构造方法:
```
    public InputEventReceiver(InputChannel inputChannel, Looper looper) {
        if (inputChannel == null) {
            throw new IllegalArgumentException("inputChannel must not be null");
        }
        if (looper == null) {
            throw new IllegalArgumentException("looper must not be null");
        }

        mInputChannel = inputChannel;
        mMessageQueue = looper.getQueue();
        //通过native层完成输入事件接受注册.
        mReceiverPtr = nativeInit(new WeakReference<InputEventReceiver>(this),
                inputChannel, mMessageQueue);

        mCloseGuard.open("dispose");
    }
```
4.  InputEventReceiver的dispatchInputEvent方法.该方法有native层回调.然后调用onInputEvent.
```
    // Called from native code.
    @SuppressWarnings("unused")
    private void dispatchInputEvent(int seq, InputEvent event) {
        mSeqMap.put(event.getSequenceNumber(), seq);
        onInputEvent(event);
    }
```
5.  WindowInputEventReceiver的onInputEvent方法从第2中可以知道调用的是ViewRootImpl的enqueueInputEvent方法
```
     void enqueueInputEvent(InputEvent event, InputEventReceiver receiver, int flags, boolean processImmediately) {
            //这里会有将事件添加到队列的操作,代码省略了.
            .......
            if (processImmediately) {//根据第2步中传入参数来看,程序会执行立刻处理输入事件
                doProcessInputEvents();
            } else {
                scheduleProcessInputEvents();
            }
        }
```
6.  ViewRootImpl的doProcessInputEvents方法:
```
    //在该方法中循环处理所有待处理的输入事件
   void doProcessInputEvents() {
        // Deliver all pending input events in the queue.
        while (mPendingInputEventHead != null) {
            QueuedInputEvent q = mPendingInputEventHead;
            mPendingInputEventHead = q.mNext;
            if (mPendingInputEventHead == null) {
                mPendingInputEventTail = null;
            }
            q.mNext = null;

            mPendingInputEventCount -= 1;
            Trace.traceCounter(Trace.TRACE_TAG_INPUT, mPendingInputEventQueueLengthCounterName,
                    mPendingInputEventCount);

            long eventTime = q.mEvent.getEventTimeNano();
            long oldestEventTime = eventTime;
            if (q.mEvent instanceof MotionEvent) {
                MotionEvent me = (MotionEvent)q.mEvent;
                if (me.getHistorySize() > 0) {
                    oldestEventTime = me.getHistoricalEventTimeNano(0);
                }
            }
            mChoreographer.mFrameInfo.updateInputEventTime(eventTime, oldestEventTime);

            //通过该方法传递事件
            deliverInputEvent(q);
        }

        // We are done processing all input events that we can process right now
        // so we can clear the pending flag immediately.
        if (mProcessInputEventsScheduled) {
            mProcessInputEventsScheduled = false;
            mHandler.removeMessages(MSG_PROCESS_INPUT_EVENTS);
        }
    }
```
7.  ViewRootImpl的deliverInputEvent方法:
```
 private void deliverInputEvent(QueuedInputEvent q) {

        InputStage stage;
        if (q.shouldSendToSynthesizer()) {//这里其实判断的就是该事件是否设置了FLAG_UNHANDLED标志
            stage = mSyntheticInputStage;//mSyntheticInputStage = new SyntheticInputStage();在setView中被初始化
        } else {
            stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;//在setView中被初始化
        }

        if (stage != null) {
            stage.deliver(q);
        } else {
            finishInputEvent(q);
        }
    }

```
8.  InputStage会一层一层的传递.直到ViewPostImeInputStage的onProcess方法.该类会将事件传递给View层级
```
    protected int onProcess(QueuedInputEvent q) {
            if (q.mEvent instanceof KeyEvent) {
                return processKeyEvent(q);//keyEvent执行这里
            } else {
                final int source = q.mEvent.getSource();
                if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
                    return processPointerEvent(q);//点击事件执行这里
                } else if ((source & InputDevice.SOURCE_CLASS_TRACKBALL) != 0) {
                    return processTrackballEvent(q);
                } else {
                    return processGenericMotionEvent(q);
                }
            }
        }
```
9.  ViewPostImeInputStage的processPointEvent方法:
```
        private int processPointerEvent(QueuedInputEvent q) {
            final MotionEvent event = (MotionEvent)q.mEvent;

            mAttachInfo.mUnbufferedDispatchRequested = false;
            final View eventTarget =
                    (event.isFromSource(InputDevice.SOURCE_MOUSE) && mCapturingView != null) ?
                            mCapturingView : mView;
            mAttachInfo.mHandlingPointerEvent = true;

            //这里就是调用DecorView的的dispatchPointEvent
            boolean handled = eventTarget.dispatchPointerEvent(event);

            maybeUpdatePointerIcon(event);
            mAttachInfo.mHandlingPointerEvent = false;
            if (mAttachInfo.mUnbufferedDispatchRequested && !mUnbufferedInputDispatch) {
                mUnbufferedInputDispatch = true;
                if (mConsumeBatchedInputScheduled) {
                    scheduleConsumeBatchedInputImmediately();
                }
            }
            return handled ? FINISH_HANDLED : FORWARD;//返回处理结果
        }
```
10. DecorView继承自FrameLayout.但是并没有重写dispatchPointerEvent方法.所以调用的是View的dispatchPointerEvent方法.
```
    //View.dispatchPointerEvent方法

    public final boolean dispatchPointerEvent(MotionEvent event) {
        if (event.isTouchEvent()) {//如果是触摸事件则执行dispatchTouchEvent方法.对于DecorView而言,它重写了该事件
            return dispatchTouchEvent(event);
        } else {
            return dispatchGenericMotionEvent(event);
        }
    }
```
11. DecorView的dispatchTouchEvent方法:
```
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        //调用Window.Callback的dispatchTouchEvent方法.对于Activity分析,我们知道对于Activity而言,它实现了该接口.而且这里的cb就是Activity.
        final Window.Callback cb = mWindow.getCallback();
        return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
                ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
    }
```
12. Activity的dispatchTouchEvent方法:
```
    public boolean dispatchTouchEvent(MotionEvent ev) {
        //执行PhoneWindow的superDispatchTouchEvent方法.如果返回值为true,则表示事件被消费了.
        //否则执行Activity的onTouchEvent方法.
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```
13. PhoneWindow的superDispatchTouchEvent方法:
```
   @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        //这里又将事件分发会DecorView.执行它的superDispatchTouchEvent方法.
        return mDecor.superDispatchTouchEvent(event);
    }
```
14. DecorView的superDispatchTouchEvent方法:
```
   public boolean superDispatchTouchEvent(MotionEvent event) {
        //执行父类的dispatchTouchEvent.由于DecorView继承自FrameLayout.所以这里其实调用的是ViewGroup的dispatchTouchEvent.
        return super.dispatchTouchEvent(event);
    }
```
15. 分析到这里,终于知道事件是如何分发到我们熟知的ViewGroup身上了.对于后面的事件分发.主要就是dispatchTouchEvent.
onInterceptTouchEvent.onTouchEvent3个方法.
先来看ViewGroup的dispatchTouchEvent方法:
```
    @Override
        public boolean dispatchTouchEvent(MotionEvent ev) {
           .....
            boolean handled = false;
            if (onFilterTouchEventForSecurity(ev)) {
                final int action = ev.getAction();
                final int actionMasked = action & MotionEvent.ACTION_MASK;

                // 如果是MotionEvent.ACTION_DOWN事件,则需要重置一些状态
                if (actionMasked == MotionEvent.ACTION_DOWN) {
                    //主要是设置mFirstTouchTarget=null
                    cancelAndClearTouchTargets(ev);
                    //主要是将mGroupFlags上的FLAG_DISALLOW_INTERCEPT标志清除.
                    //所以对于Down事件,子View调用parent.requestDisallowInterceptTouchEvent是不起作用的
                    resetTouchState();
                }

                final boolean intercepted;

                //1.如果是Down事件,或者mFirstTouchTarget不为null,就会去判断是否需要拦截该事件.
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || mFirstTouchTarget != null) {

                    //判断子View是否要求当前View不拦截该事件
                    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;

                    if (!disallowIntercept) {
                        //子View允许拦截,则执行onInterceptTouchEvent判断是否主动拦截
                        intercepted = onInterceptTouchEvent(ev);
                        ev.setAction(action); // restore action in case it was changed
                    } else {
                        //子View不允许拦截,则一定不能拦截.
                        intercepted = false;
                    }
                } else {
                    //2.如果不为Down事件,且mFirstTouchTarget为null,说明没有子View处理该事件,所以一定会拦截该事件.
                    intercepted = true;
                }

         .....

                // Check for cancelation.
                final boolean canceled = resetCancelNextUpFlag(this)
                        || actionMasked == MotionEvent.ACTION_CANCEL;

                // Update list of touch targets for pointer down, if needed.
                final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
                TouchTarget newTouchTarget = null;
                boolean alreadyDispatchedToNewTouchTarget = false;

                //对于非取消且不拦截的情况:
                //一:是MotionEvent.ACTION_DOWN或者 MotionEvent.ACTION_POINTER_DOWN事件
                //1.循环查询可接受该事件的子View
                //  可接受原则: 1.VISIBLE或者在执行View动画. 2. 事件的坐标在该子View范围内
                //2.当查找到了可以接受的子View的情况,会调用dispatchTransformedTouchEvent方法看子View是否处理,如果返回true,则说明子View处理
                //  在这种情况下会通过addTouchTarget方法对mFirstTouchTarget赋值.
                //3.当子View不处理该事件.则mFirstTouchTarget = null,则会通过dispatchTransformedTouchEvent方法,
                //  调用父类View的dispatchTouchEvent方法.然后执行自己的onTouchEvent方法

                //二:如果是其他MOVE或者UP事件.则不会再执行查找过程,会直接判断mFirstTouchTarget
                //1.为空的情况,则会通过dispatchTransformedTouchEvent方法,
                //  调用父类View的dispatchTouchEvent方法.然后执行自己的onTouchEvent方法
                //2.不为空的情况,则会根据是否拦截来处理,如果拦截.则会通过dispatchTransformedTouchEvent给子View发送cancel事件.然后将mFirstTouchTarget设置为null.
                //  如果不拦截,则同样是通过dispatchTransformedTouchEvent将事件传递给子View.

                if (!canceled && !intercepted) {

                    View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                            ? findChildWithAccessibilityFocus() : null;

                    if (actionMasked == MotionEvent.ACTION_DOWN
                            || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                            || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                        final int actionIndex = ev.getActionIndex(); // always 0 for down
                        final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                                : TouchTarget.ALL_POINTER_IDS;

                        // Clean up earlier touch targets for this pointer id in case they
                        // have become out of sync.
                        removePointersFromTouchTargets(idBitsToAssign);

                        final int childrenCount = mChildrenCount;
                        if (newTouchTarget == null && childrenCount != 0) {
                            final float x = ev.getX(actionIndex);
                            final float y = ev.getY(actionIndex);
                            // Find a child that can receive the event.
                            // Scan children from front to back.
                            final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                            final boolean customOrder = preorderedList == null
                                    && isChildrenDrawingOrderEnabled();
                            final View[] children = mChildren;
                            for (int i = childrenCount - 1; i >= 0; i--) {
                                final int childIndex = getAndVerifyPreorderedIndex(
                                        childrenCount, i, customOrder);
                                final View child = getAndVerifyPreorderedView(
                                        preorderedList, children, childIndex);

                                // If there is a view that has accessibility focus we want it
                                // to get the event first and if not handled we will perform a
                                // normal dispatch. We may do a double iteration but this is
                                // safer given the timeframe.
                                if (childWithAccessibilityFocus != null) {
                                    if (childWithAccessibilityFocus != child) {
                                        continue;
                                    }
                                    childWithAccessibilityFocus = null;
                                    i = childrenCount - 1;
                                }

                                //判断子View是否可以处理该事件.否则继续循环查找
                                if (!canViewReceivePointerEvents(child)
                                        || !isTransformedTouchPointInView(x, y, child, null)) {
                                    ev.setTargetAccessibilityFocus(false);
                                    continue;
                                }

                                newTouchTarget = getTouchTarget(child);
                                if (newTouchTarget != null) {
                                    // Child is already receiving touch within its bounds.
                                    // Give it the new pointer in addition to the ones it is handling.
                                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                                    break;
                                }

                                resetCancelNextUpFlag(child);
                                if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                    // Child wants to receive touch within its bounds.
                                    mLastTouchDownTime = ev.getDownTime();
                                    if (preorderedList != null) {
                                        // childIndex points into presorted list, find original index
                                        for (int j = 0; j < childrenCount; j++) {
                                            if (children[childIndex] == mChildren[j]) {
                                                mLastTouchDownIndex = j;
                                                break;
                                            }
                                        }
                                    } else {
                                        mLastTouchDownIndex = childIndex;
                                    }
                                    mLastTouchDownX = ev.getX();
                                    mLastTouchDownY = ev.getY();
                                    newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                    alreadyDispatchedToNewTouchTarget = true;
                                    break;
                                }

                                // The accessibility focus didn't handle the event, so clear
                                // the flag and do a normal dispatch to all children.
                                ev.setTargetAccessibilityFocus(false);
                            }
                            if (preorderedList != null) preorderedList.clear();
                        }

                        if (newTouchTarget == null && mFirstTouchTarget != null) {
                            // Did not find a child to receive the event.
                            // Assign the pointer to the least recently added target.
                            newTouchTarget = mFirstTouchTarget;
                            while (newTouchTarget.next != null) {
                                newTouchTarget = newTouchTarget.next;
                            }
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                        }
                    }
                }

                // Dispatch to touch targets.
                if (mFirstTouchTarget == null) {
                    // No touch targets so treat this as an ordinary view.
                    handled = dispatchTransformedTouchEvent(ev, canceled, null,
                            TouchTarget.ALL_POINTER_IDS);
                } else {
                    // Dispatch to touch targets, excluding the new touch target if we already
                    // dispatched to it.  Cancel touch targets if necessary.
                    TouchTarget predecessor = null;
                    TouchTarget target = mFirstTouchTarget;
                    while (target != null) {
                        final TouchTarget next = target.next;
                        if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                            handled = true;
                        } else {
                            final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                    || intercepted;
                            if (dispatchTransformedTouchEvent(ev, cancelChild,
                                    target.child, target.pointerIdBits)) {
                                handled = true;
                            }
                            if (cancelChild) {
                                if (predecessor == null) {
                                    mFirstTouchTarget = next;
                                } else {
                                    predecessor.next = next;
                                }
                                target.recycle();
                                target = next;
                                continue;
                            }
                        }
                        predecessor = target;
                        target = next;
                    }
                }

                // Update list of touch targets for pointer up or cancel, if needed.
                if (canceled
                        || actionMasked == MotionEvent.ACTION_UP
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    resetTouchState();
                } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                    final int actionIndex = ev.getActionIndex();
                    final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                    removePointersFromTouchTargets(idBitsToRemove);
                }
            }

            if (!handled && mInputEventConsistencyVerifier != null) {
                mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
            }
            return handled;
        }
```
16. ViewGroup的dispatchTransformedTouchEvent方法:主要用于向子View分发事件(调用dispatchTouchEvent方法),或者自己分发事件(View的dispatchTouchEvent方法)
```
 private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        //如果设置了cancel,则将事件该为cancel事件发送给子VIEW
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }

        // Calculate the number of pointers to deliver.
        final int oldPointerIdBits = event.getPointerIdBits();
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;


        if (newPointerIdBits == 0) {
            return false;
        }

       .....
        //根据child的值,具体分发给子VIEW还是自己.
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }

            handled = child.dispatchTouchEvent(transformedEvent);
        }

        // Done.
        transformedEvent.recycle();
        return handled;
    }
```
17. View的dispatchTouchEvent方法:
```
    public boolean dispatchTouchEvent(MotionEvent event) {
        ...
        boolean result = false;
        final int actionMasked = event.getActionMasked();
        ....

        if (onFilterTouchEventForSecurity(event)) {
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            //如果设置了onTouchListener,则会执行它的onTouch方法.所以它是在onTouchEvent方法之前执行.
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            //如果onTouchListener没有或者其返回值为false.才会继续执行onTouchEvent方法.
            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
                ......
        return result;
    }
```
18. View的onTouchEvent方法:    只要View CLICKABLE或者LONG_CLICKABLE.则无论如何,都会返回true.
```
  public boolean onTouchEvent(MotionEvent event) {
        final float x = event.getX();
        final float y = event.getY();
        final int viewFlags = mViewFlags;
        final int action = event.getAction();

        if ((viewFlags & ENABLED_MASK) == DISABLED) {//如果是DISABLE的,则直接返回,
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return (((viewFlags & CLICKABLE) == CLICKABLE
                    || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                    || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
        }
        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

        //根据具体事件进行相应逻辑处理:
        //1.DOWN事件:
            1.设置PFLAG_PRESSED按下状态,并且保存坐标.
            2,发送一个500ms的延迟runnable.用来处理长按事件的.
          2.MOVE事件:
            1.用新的坐标和上次的坐标对比.如果偏移了系统TouchSlop距离,则认为不在原来的点上了.会移除长按Runnable,且清除PFLAG_PRESSED标志
          3.UP事件:
            1.如果PFLAG_PRESSED标识存在,则会执行performConlick方法.否则不会执行.
          4.CANCEL事件:重置所有状态.

        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
            switch (action) {
                case MotionEvent.ACTION_UP:
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        // take focus if we don't have it already and we should in
                        // touch mode.
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            // The button is being released before we actually
                            // showed it as pressed.  Make it show the pressed
                            // state now (before scheduling the click) to ensure
                            // the user sees it.
                            setPressed(true, x, y);
                       }

                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }

                        if (mUnsetPressedState == null) {
                            mUnsetPressedState = new UnsetPressedState();
                        }

                        if (prepressed) {
                            postDelayed(mUnsetPressedState,
                                    ViewConfiguration.getPressedStateDuration());
                        } else if (!post(mUnsetPressedState)) {
                            // If the post failed, unpress right now
                            mUnsetPressedState.run();
                        }

                        removeTapCallback();
                    }
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_DOWN:
                    mHasPerformedLongPress = false;

                    if (performButtonActionOnTouchDown(event)) {
                        break;
                    }

                    // Walk up the hierarchy to determine if we're inside a scrolling container.
                    boolean isInScrollingContainer = isInScrollingContainer();

                    // For views inside a scrolling container, delay the pressed feedback for
                    // a short period in case this is a scroll.
                    if (isInScrollingContainer) {
                        mPrivateFlags |= PFLAG_PREPRESSED;
                        if (mPendingCheckForTap == null) {
                            mPendingCheckForTap = new CheckForTap();
                        }
                        mPendingCheckForTap.x = event.getX();
                        mPendingCheckForTap.y = event.getY();
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                    } else {
                        // Not inside a scrolling container, so show the feedback right away
                        setPressed(true, x, y);
                        checkForLongClick(0, x, y);
                    }
                    break;

                case MotionEvent.ACTION_CANCEL:
                    setPressed(false);
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_MOVE:
                    drawableHotspotChanged(x, y);

                    // Be lenient about moving outside of buttons
                    if (!pointInView(x, y, mTouchSlop)) {
                        // Outside button
                        removeTapCallback();
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                            // Remove any future long press/tap checks
                            removeLongPressCallback();

                            setPressed(false);
                        }
                    }
                    break;
            }

            return true;
        }

        return false;
    }
```

总结调用流程: 对于一次点击事件
```
-> InputEventReceiver.dispatchTouchEvent()
-> ViewRootImpl.WindowInputEventReceiver.onInputEvent()
-> ViewRootImpl.enqueueInputEvent()
-> ViewRootImpl.doProcessInputEvent()
-> ViewRootImpl.deliverInputEvent()
->  ...
-> ViewPostImeInputStage.onProcess()
-> ViewPostImeInputStage.dispatchPointEvent()
-> View.dispatchPointEvent()
-> DecorView.dispatchTouchEvent()
-> Activity.dispatchTouchEvent()
-> PhoneWindow.superDispatchTouchEvent()
-> DecorView.superDispatchTouchEvent()
-> ViewGroup.dispatchTouchEvent()
-> ViewGroup.dispatchTransformTouchEvent()
-> View.dispatchTouchEvent()
-> View.onTouchEvent()

到这里就分析结束啦.
```


