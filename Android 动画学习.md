# Android 动画深入分析
1. 帧动画  
AnimationDrawable \<animation-list/>  xml方式创建在res目录下的drawable目录下

2. View动画  
. Animation 抽象类,可继承该类,自定义View动画,重写initialize方法和applyTransformation方法即可.  
. ScaleAnimation \<scale/>  
. TranslateAnimation   \<translate/>  
. AlphaAnimation   \<alpha/>  
. RotateAnimation \<rotate/>  
xml方式创建在res目录下的anim目录下,通过AnimationUtils.loadAnimation()方法加载  
View通过startAnimation(Animation animation)方法开始动画.可以通过AnimationSet来组合  
上面4种动画同时执行.     
原理:  
1.View.startAnimation(Animation animation):  
    ```
        public void startAnimation(Animation animation) {
            animation.setStartTime(Animation.START_ON_FIRST_FRAME);
            
            //这里将mCurrentAnimation赋值为animation,并对animation初始化
            setAnimation(animation);
            
            //这里面给parentView设置PFLAG_INVALIDATED标识
            invalidateParentCaches();
            invalidate(true);
        }
    ```
    2.invalidate方法一直向上调用,最后调用ViewRootImpl的invalidateChildInParent方法  
    3.然后会调用到ViewRootImpl的invalidateRectOnScreen方法,最终调用scheduleTraversals方法.  
    4.最终会执行performTraversals方法.这个方法很重要:包含performMeasure,performLayout,performDraw.  
    5.我们知道最终的过程还是会回调回View的draw方法.在3个参数的draw方法中有这样一段代码:
    ```
        final Animation a = getAnimation();
        if (a != null) {
            more = applyLegacyAnimation(parent, drawingTime, a, scalingRequired);
            concatMatrix = a.willChangeTransformationMatrix();
            if (concatMatrix) {
                mPrivateFlags3 |= PFLAG3_VIEW_IS_ANIMATING_TRANSFORM;
            }
            transformToApply = parent.getChildTransformation();
        } 
    ```
    ```
     private boolean applyLegacyAnimation(ViewGroup parent, long drawingTime,
                Animation a, boolean scalingRequired) {
            Transformation invalidationTransform;
            final int flags = parent.mGroupFlags;
            final boolean initialized = a.isInitialized();
            if (!initialized) {
            
                //对Animation进行初始化
                a.initialize(mRight - mLeft, mBottom - mTop, parent.getWidth(), parent.getHeight());
                a.initializeInvalidateRegion(0, 0, mRight - mLeft, mBottom - mTop);
                
                //这里mAttachInfo.mHandler是ViewRootImpl中的ViewRootHandler
                if (mAttachInfo != null) a.setListenerHandler(mAttachInfo.mHandler);
                onAnimationStart();
            }
    
            final Transformation t = parent.getChildTransformation();
            
            //这里根据动画计算变化内容,保存到Transformation t中.这里面主要会调用到Animation的applyTransformation方法.
            boolean more = a.getTransformation(drawingTime, t, 1f);
            
            if (scalingRequired && mAttachInfo.mApplicationScale != 1f) {
                if (parent.mInvalidationTransformation == null) {
                    parent.mInvalidationTransformation = new Transformation();
                }
                invalidationTransform = parent.mInvalidationTransformation;
                a.getTransformation(drawingTime, invalidationTransform, 1f);
            } else {
                invalidationTransform = t;
            }
    
            if (more) {
                if (!a.willChangeBounds()) {//动画是否会改变大小或位置,AlphaAnimation动画执行这里
                    if ((flags & (ViewGroup.FLAG_OPTIMIZE_INVALIDATE | ViewGroup.FLAG_ANIMATION_DONE)) ==
                            ViewGroup.FLAG_OPTIMIZE_INVALIDATE) {
                        parent.mGroupFlags |= ViewGroup.FLAG_INVALIDATE_REQUIRED;
                    } else if ((flags & ViewGroup.FLAG_INVALIDATE_REQUIRED) == 0) {
                        // The child need to draw an animation, potentially offscreen, so
                        // make sure we do not cancel invalidate requests
                        parent.mPrivateFlags |= PFLAG_DRAW_ANIMATION;
                        
                        //让parent重新绘制指定区域的内容
                        parent.invalidate(mLeft, mTop, mRight, mBottom);
                    }
                } else {//ScaleAniamtion,RoateAniamtion,TranslateAnimation都会执行这里
                    if (parent.mInvalidateRegion == null) {
                        parent.mInvalidateRegion = new RectF();
                    }
                    final RectF region = parent.mInvalidateRegion;
                    
                    //获取需要重新绘制的区域
                    a.getInvalidateRegion(0, 0, mRight - mLeft, mBottom - mTop, region,
                            invalidationTransform);
    
                    // The child need to draw an animation, potentially offscreen, so
                    // make sure we do not cancel invalidate requests
                    parent.mPrivateFlags |= PFLAG_DRAW_ANIMATION;
    
                    final int left = mLeft + (int) region.left;
                    final int top = mTop + (int) region.top;
                    
                    //让parent重新绘制指定的区域
                    parent.invalidate(left, top, left + (int) (region.width() + .5f),
                            top + (int) (region.height() + .5f));
                }
            }
            return more;
        }
    ```
 
3. Property动画  
Animator 抽象类:系统提供的实现类如下:  
. ValueAnimator  
. AnimatorSet  
. ObjectAnimator:继承自 ValueAnimator  
. TimeAnimator: 继承自 ValueAnimator  

对于与属性动画而言:系统还提供了几个类用于方式使用:ViewPropertyAnimator,PropertyValuesHolder.  
  
    
ViewPropertyAnimator使用:  
```
        view.animate().rotation(30).alpha(1.0f).translationX(100).scaleY(1.0f);  
        ViewPropertyAnimator提供了旋转,透明度,位移,缩放的操作.对于其他自定义的属性,该类无法使用.  
```
  
ObjectAnimator使用:  
```
        ObjectAnimator.ofFloat(view,"translationX",100).setDuration(100).start();
        
        PropertyValuesHolder translationX = PropertyValuesHolder.ofFloat("translationX",100);
        PropertyValuesHolder translationY = PropertyValuesHolder.ofFloat("translationY", 100);
        ObjectAnimator.ofPropertyValuesHolder(view,translationX,translationY).start();

        PropertyValuesHolder keyframe = PropertyValuesHolder.ofKeyframe("translationX", Keyframe.ofFloat(0, 0));
        ObjectAnimator.ofPropertyValuesHolder(view,keyframe);
        
        ObjectAnimator translationX1 = ObjectAnimator.ofFloat(view, "translationX", 100);
        ObjectAnimator translationY1 = ObjectAnimator.ofFloat(view, "translationY", 100);
        AnimatorSet animatorSet = new AnimatorSet();
        animatorSet.play(translationY1).before(translationX1);
        animatorSet.start();
```

ValueAnimator使用:  
```
        ValueAnimator valueAnimator = ValueAnimator.ofInt(0,100);
        valueAnimator.setInterpolator(new DecelerateInterpolator());
        valueAnimator.setEvaluator(new IntEvaluator());
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                //做view的属性改变...
            }
        });
```

ViewPropertyAnimation原理分析:      

1.内部都是统一执行animateProperty方法
```
    //当调用该方法时,会调用animateProperty(..)方法.对于rotate,scale.alpha等变化也是如此
    public ViewPropertyAnimator translationX(float value) {
        animateProperty(TRANSLATION_X, value);
        return this;
    }
```
2.来看animateProperty方法:
```
    private void animateProperty(int constantName, float toValue) {
        float fromValue = getValue(constantName);//这里getValue通过属性名获取初始值.
        float deltaValue = toValue - fromValue;
        animatePropertyBy(constantName, fromValue, deltaValue);
    }
    
    private void animatePropertyBy(int constantName, float startValue, float byValue) {
            // First, cancel any existing animations on this property
            if (mAnimatorMap.size() > 0) {
               .......
            }
            
            //创建一个NameValuesHolder对象,保存了属性名称以及属性初始值,和变化值.
            NameValuesHolder nameValuePair = new NameValuesHolder(constantName, startValue, byValue);
            mPendingAnimations.add(nameValuePair);
            
            mView.removeCallbacks(mAnimationStarter);
            mView.postOnAnimation(mAnimationStarter);
     }  
```
3.接下来执行View的postOnAnimation方法:
```
    public void postOnAnimation(Runnable action) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
        //通过mChoreographer向其CALLBACK_ANIMATION类型的回调列表中添加一个callback.当下一帧绘制来时,则调用action的run方法.
            attachInfo.mViewRootImpl.mChoreographer.postCallback(
                    Choreographer.CALLBACK_ANIMATION, action, null);
        } else {
            // Postpone the runnable until we know
            // on which thread it needs to run.
            getRunQueue().post(action);
        }
    }
```
4.ViewPropertyAnimator中的mAnimationStarter的run方法:其中只有一句代码,就是执行startAnimation方法
```
 private void startAnimation() {
        .......
        mView.setHasTransientState(true);
        //从这里可以知道,ViewProperyAnimator其实也是根据ValueAniamtor来实现的.
        ValueAnimator animator = ValueAnimator.ofFloat(1.0f);
        
        ArrayList<NameValuesHolder> nameValueList =
                (ArrayList<NameValuesHolder>) mPendingAnimations.clone();
        mPendingAnimations.clear();
        int propertyMask = 0;
        int propertyCount = nameValueList.size();
        for (int i = 0; i < propertyCount; ++i) {
            NameValuesHolder nameValuesHolder = nameValueList.get(i);
            propertyMask |= nameValuesHolder.mNameConstant;
        }
        //这个map中保存当前animator所需要改变的属性变化集合.
        mAnimatorMap.put(animator, new PropertyBundle(propertyMask, nameValueList));
        
        ......
        
        animator.addUpdateListener(mAnimatorEventListener);
        animator.addListener(mAnimatorEventListener);
        if (mStartDelaySet) {
            animator.setStartDelay(mStartDelay);
        }
        if (mDurationSet) {
            animator.setDuration(mDuration);
        }
        if (mInterpolatorSet) {
            animator.setInterpolator(mInterpolator);
        }
        //执行ValueAnimator的start方法
        animator.start();
    }

```
5.ValueAnimator的start方法:会调用start(false).
```
 private void start(boolean playBackwards) {
        if (Looper.myLooper() == null) {
            throw new AndroidRuntimeException("Animators may only be run on Looper threads");
        }
    .........
        mStarted = true;
        mPaused = false;
        mRunning = false;
        mAnimationEndRequested = false;
  
        mLastFrameTime = 0;
        
        //获取AnimationHandler.该值是ThredLocal变量.线程共享的
        AnimationHandler animationHandler = AnimationHandler.getInstance();
        //添加一个回调
        animationHandler.addAnimationFrameCallback(this, (long) (mStartDelay * sDurationScale));

        if (mStartDelay == 0 || mSeekFraction >= 0) {
            ......
            startAnimation();
            if (mSeekFraction == -1) {
                ......
                setCurrentPlayTime(0);
            } else {
                setCurrentFraction(mSeekFraction);
            }
        }
    }
```
6.AnimationHandler的addAnimationFrameCallback方法:
```
    public void addAnimationFrameCallback(final AnimationFrameCallback callback, long delay) {
        //如果当前回调数量为0.调用MyFrameCallbackProvider的postFrameCallback方法.其实内部就是
        //调用Choreographer的postFrameCallback方法.添加一个CALLBACK_ANIMATION回调.
        if (mAnimationCallbacks.size() == 0) {
            getProvider().postFrameCallback(mFrameCallback);
        }
        if (!mAnimationCallbacks.contains(callback)) {
            mAnimationCallbacks.add(callback);
        }

        if (delay > 0) {
            mDelayedCallbackStartTime.put(callback, (SystemClock.uptimeMillis() + delay));
        }
    }
    
    private AnimationFrameCallbackProvider getProvider() {
        if (mProvider == null) {
            mProvider = new MyFrameCallbackProvider();
        }
        return mProvider;
    }
```
7.当下一帧来时,会回调mFrameCallback的doFrame方法:  
```
    private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
        @Override
        public void doFrame(long frameTimeNanos) {
            //执行这一帧的操作
            doAnimationFrame(getProvider().getFrameTime());
            //如果mAnimationCallbacks不为空,则继续通过Choreographer接收下一帧来临回调.
            if (mAnimationCallbacks.size() > 0) {
                getProvider().postFrameCallback(this);
            }
        }
    };
```  

8.AnimationHandler的doAnimationFrame方法:
```
    private void doAnimationFrame(long frameTime) {
        int size = mAnimationCallbacks.size();
        long currentTime = SystemClock.uptimeMillis();
        for (int i = 0; i < size; i++) {
            final AnimationFrameCallback callback = mAnimationCallbacks.get(i);
            if (callback == null) {
                continue;
            }
            
            if (isCallbackDue(callback, currentTime)) {
                //执行valueAnimatior的doAnimationFrme方法.
                callback.doAnimationFrame(frameTime);
                
                if (mCommitCallbacks.contains(callback)) {
                    getProvider().postCommitCallback(new Runnable() {
                        @Override
                        public void run() {
                            commitAnimationFrame(callback, getProvider().getFrameTime());
                        }
                    });
                }
            }
        }
        cleanUpList();
    }
```
9.ValueAnimator的doAnimationFrame方法
```
  public final void doAnimationFrame(long frameTime) {
        .......
        final long currentTime = Math.max(frameTime, mStartTime);
        boolean finished = animateBasedOnTime(currentTime);

        if (finished) {//结束动画
            endAnimation();
        }
    }
```
10.ValueAnimator的animateBasedOnTime方法:
```
    boolean animateBasedOnTime(long currentTime) {
        boolean done = false;
        if (mRunning) {
            final long scaledDuration = getScaledDuration();
            //计算时间过去的百分比.
            final float fraction = scaledDuration > 0 ?
                    (float)(currentTime - mStartTime) / scaledDuration : 1f;
            .....
            mOverallFraction = clampFraction(fraction);
            float currentIterationFraction = getCurrentIterationFraction(mOverallFraction);
            animateValue(currentIterationFraction);
        }
        return done;
    }
```
11.ValueAnimator的animateValue方法:
```
    void animateValue(float fraction) {
        //根据插值器,算出动画完成度的百分比
        fraction = mInterpolator.getInterpolation(fraction);
        mCurrentFraction = fraction;
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
            //对于ObjectAnimator就是通过这里计算下一帧的属性值
            mValues[i].calculateValue(fraction);
        }
        
        //对于ViewPropertyAnimator,是通过这里的监听器,更新属性.
        if (mUpdateListeners != null) {
            int numListeners = mUpdateListeners.size();
            for (int i = 0; i < numListeners; ++i) {
                mUpdateListeners.get(i).onAnimationUpdate(this);
            }
        }
    }
```
12.ViewPropertyAnimator内部的AnimatorEventListener的onAnimationUpdate方法:
```
       public void onAnimationUpdate(ValueAnimator animation) {
                   ......
                   ArrayList<NameValuesHolder> valueList = propertyBundle.mNameValuesHolder;
                   if (valueList != null) {
                       int count = valueList.size();
                       for (int i = 0; i < count; ++i) {
                           NameValuesHolder values = valueList.get(i);
                           float value = values.mFromValue + fraction * values.mDeltaValue;
                           if (values.mNameConstant == ALPHA) {
                                //改变alpha值
                               alphaHandled = mView.setAlphaNoInvalidation(value);
                           } else {
                               //改变属性值.
                               setValue(values.mNameConstant, value);
                           }
                       }
                   }
                    ......
                   if (alphaHandled) {
                       mView.invalidate(true);
                   } else {
                       mView.invalidateViewProperty(false, false);
                   }
                   if (mUpdateListener != null) {
                       mUpdateListener.onAnimationUpdate(animation);
                   }
               }
```
整体流程:  
->ViewPropertyAnimation.animateProperty()  
->View.postOnAnimation()  
->ViewPropertyAnimation.startAnimation()  
->ValueAnimator.start()  
->AnimationHandler.addAnimationFrameCallback()  
->Choreographer.postFrameCallback()  
->Choreographer.scheduleFrameLocked()  
->Choreographer.scheduleVsyncLocked()  
->Choreographer.FrameDisplayEventReceiver.scheduleVsync()  
-> jni 调用: DisplayEventDispatcher.cpp 中的scheduleVsync().在这里面会执行requestNextVsync.
-> jni层回调 Choreographer.FrameDisplayEventReceiver.onVsync()
->Choreographer.doFrame()
->Choreographer.doCallback()
->Choreographer.FrameCallback.doFrame()  
->AnimationHandler.doAnimationFrame()  
->ValueAnimator.doAnimationFrame()  
->ValueAnimator.animateValue()  
->ViewPropertyAnimator.AnimatorEventListener.onAnimationUpdate()  
->ViewPropertyAnimator.setValue()

同样对于ObjectAnimator的流程从ValueAnimator.start开始也差不多,因为它本身也是基于ValueAnimator完成的.不同的地方在于  
->ValueAnimator.animateValue()  中  
->ProperValuesHolder.calculateValue()  
->Keyframes.getValue() 在该方法链中通过插值器和估值器计算下一帧属性值.  
->ObjectAnimator.animateValue中  
```
  void animateValue(float fraction) {
        final Object target = getTarget();
        if (mTarget != null && target == null) {
            // We lost the target reference, cancel and clean up. Note: we allow null target if the
            /// target has never been set.
            cancel();
            return;
        }

        super.animateValue(fraction);
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
            mValues[i].setAnimatedValue(target);
        }
    }
```
-> ProperValuesHolder.setAnimatedValue():通过反射设置新的属性值.




