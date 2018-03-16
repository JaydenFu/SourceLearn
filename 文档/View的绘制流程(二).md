#   View的绘制流程(二)
在第一节中已经分析顶级View的measure,layout.draw方法是如何被触发的.现在来分析从DecorView一层一层如何measure,layout.draw

1.  View的measure方法:该方法是final的,所以View的任何子类都是不能覆写该方法的.因此,对于DecorView而言,measure方法也是执行的
View本身measure方法.
```
   public final void measure(int widthMeasureSpec, int heightMeasureSpec) {

        //将宽/高 MeasureSpec组合成64为的一个long值(宽MeasureSpec作为高32位,高MeasureSpec作为低32位),作为
        //测量缓存的key值.
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
        if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);

        //是否被设置了PFLAG_FORCE_LAYOUT标志,调用reqeustLayout就会可能被设置该标识.
        //至于是否设置该标志可以参考View绘制第一节中第13步performLayout.
        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;

        //当前传递下来的MesaureSpec和父容器之前传递下来的MeasureSpec是否改变了.
        final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec
                || heightMeasureSpec != mOldHeightMeasureSpec;

        //父容器对当前View的测量模式是否限定为EXACTLY.
        final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
                && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;

        //上一次测量值和当前父容器传递下来的可用值是否一致.
        final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
                && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);

        //sAlwaysRemeasureExactly该参数可以忽略.默认是false的.
        //判断是否需要layout.需要满足2个条件
        //1.父容器传递的MeasureSpec改变了.不管是大小,还是模式.
        //2.当次的模式不为Exactly或者当次的大小和上次测量完成的大小不一致.
        final boolean needsLayout = specChanged
                && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);

        if (forceLayout || needsLayout) {
            // first clears the measured dimension flag
            //清除PFLAG_MEASURED_DIMENSION_SET,该标识表示已经给该View设定测量大小,在setMeasuredDimensionRaw方法中设置.
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;
           ......

            //如果View并没有发起reqeustLayout.则可以查找是否有可用的测量缓存.这里缓存可能是多次的.
            //forceLayour为false的情况下,能进入这里,首先肯定当次的MeasureSpec和上次的不一致.但是不代表和更之前的不一致.
            //所以是可能存在缓存的
            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);

            //sIgnoreMeasureCache可以忽略,默认为false
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                //如果没有缓存,就调用onMeasure去实际的完成View的大小测量.
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                //清除PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT标识.在View的layout操作具体执行前,不需要再次测量了.
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                //如果查找到了缓存.就直接使用缓存的测量值就好了.
                long value = mMeasureCache.valueAt(cacheIndex);
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                //这个缓存值不一定对,所以会设置PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT,表示View layout之前需要重新测量.
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }

            if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException("View with id " + getId() + ": "
                        + getClass().getName() + "#onMeasure() did not set the"
                        + " measured dimension by calling"
                        + " setMeasuredDimension()");
            }
            //设置PFLAG_LAYOUT_REQUIRED,表示要求执行onLayout方法
            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }

        //将当次的MeasureSpec保存.作为下次measure被调用时的上次参考值.
        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;

        //保存测量缓存值,值也是有宽高的实际测量值组合成64为的值.宽的值在高32为,高的值在低32位.
        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }

```
2.  View的onMeasure方法:
* 对于View的子类而言其实都是需要自己去覆写该方法的.因为如果是容器View.当是WRAP_CONTENT时,需要根据子View的宽高去设置自己的宽高,
  而对于普通的View而言,也同样需要自己去根据自己的内容来设置自己的大小.否则WRAP_CONTENT会失效.
```
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        //setMeasureDimmsion方法通过调用setMeasureDimmsionRaw方法设置测量值.并设置PFLAG_MEASURED_DIMENSION_SET标识.
        //表示测量值已经确定了.
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }

    //getSuggestedMinimumHeight方法与该方法类似
    protected int getSuggestedMinimumWidth() {
        //1.没有设置背景Drawable,则返回的是设置给View的最小宽度.
        //2.设置了背景Drawable,则返回的是最小宽度和Drawable本身的宽的最大值.只有BitmapDrawable这样的drawable才有本身宽高.
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }

    //从getDefaultSize中我们可以知道:
    //1.当父容器指定的模式为UMSPECIFIED时,则View的大小就是最小值.
    //2.当父容器指定的模式为EXACTLP时,则View的大小就是父容器传递下来的大小.
    //3.当父容器指定的模式为ATMOST是,则View的大小也是父容器传递下来的大小.因此就存在一个问题.当View的WRAP_CONTENT时,
    //如果我们自己没有重写onMeasure方法时,View大小始终是父容器传递下来的值.
    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }

    protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
            .....
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }

    private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;
        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    }

```
3.  这里以FrameLayout为列:来看它实现的onMeasure方法:
```
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int count = getChildCount();

        //根据父容器限定的测量模式判断是否需要再次测量宽高不会MATCH_PARENT的子View.
        //因为当父容器指定的测量模式是EXACTLY时,则不管子View如何,自己的大小总是固定的.所以对于子View是MATCH_PARENT而言,它的
        //宽高也会相应的是固定的.就不会要再测量.
        final boolean measureMatchParentChildren =
                MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
                MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
        mMatchParentChildren.clear();

        int maxHeight = 0;
        int maxWidth = 0;
        int childState = 0;

        //循环测量子View.
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);

            //mMeasureAllChildren默认为false.当改值被设置为true时,不管子View是不是GONE状态.都会被测量.
            //所以这里通常只是更具child的显示状态如果不是GONE,则就会对其进行测量.
            if (mMeasureAllChildren || child.getVisibility() != GONE) {

                //对子View进行测量,该方法解析查看第4步
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);

                final LayoutParams lp = (LayoutParams) child.getLayoutParams();

                //取当前最大值与子View的宽高+margin和的最大值赋值给maxWidth,maxHeight
                maxWidth = Math.max(maxWidth,
                        child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
                maxHeight = Math.max(maxHeight,
                        child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);

                childState = combineMeasuredStates(childState, child.getMeasuredState());

                //如果需要再次测量布局参数为MATCH_PARENT的子View.就将符合条件的子View.添加到mMatchParentChildren集合中.后面再次对
                //该集合中的View进行测量.
                if (measureMatchParentChildren) {
                    if (lp.width == LayoutParams.MATCH_PARENT ||
                            lp.height == LayoutParams.MATCH_PARENT) {
                        mMatchParentChildren.add(child);
                    }
                }
            }
        }

        //计算出最大宽度,最大高度
        // Account for padding too
        maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
        maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

        //最大宽度,最大高度检测是否大于最小宽高,如果是,则重新赋值最大宽高为View设置的最小宽高.
        // Check against our minimum height and width
        maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
        maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

        //最大宽度,最大高度检测是否大于foreground的drawable的原始宽高.,如果是,则重新赋值最大宽高为View设置的foregroun的drawable的原始宽高
        // Check against our foreground's minimum height and width
        final Drawable drawable = getForeground();
        if (drawable != null) {
            maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
            maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
        }

        //设置当前View的测量大小.View.resolveSizeAndState方法参考第6步.
        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));

        //如果有需要重新测量的View.就重新测量.因为这个时候当前View的宽高是固定的了.所以传递下去的测量模式页一定是EXACLTY.
        count = mMatchParentChildren.size();
        if (count > 1) {
            for (int i = 0; i < count; i++) {
                final View child = mMatchParentChildren.get(i);
                final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

                final int childWidthMeasureSpec;
                if (lp.width == LayoutParams.MATCH_PARENT) {
                    //如果子View的宽是MATCH_PARENT,则将用当前View的测量宽高减去子View的margin,以及当前View的padding当做子View的宽高生成MeasureSpec
                    final int width = Math.max(0, getMeasuredWidth()
                            - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                            - lp.leftMargin - lp.rightMargin);
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                            width, MeasureSpec.EXACTLY);
                } else {
                    //如果子View的宽不是MATCH_PARENT的,则按正常的流程生成MeasureSpec,
                    //对于ViewGroup的getChildMeasureSpec方法参见第5步.
                    childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                            getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                            lp.leftMargin + lp.rightMargin,
                            lp.width);
                }

                //高度的处理和宽度处理一致
                final int childHeightMeasureSpec;
                if (lp.height == LayoutParams.MATCH_PARENT) {
                    final int height = Math.max(0, getMeasuredHeight()
                            - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                            - lp.topMargin - lp.bottomMargin);
                    childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                            height, MeasureSpec.EXACTLY);
                } else {
                    childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                            getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                            lp.topMargin + lp.bottomMargin,
                            lp.height);
                }
                //发起测量
                child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
            }
        }
    }

```
4.  ViewGroup的measureChildWithMargins方法:
```
    //参数意义:
    //child 表示子View
    //parentWidthMeasureSpec 表示当前View的宽measureSpec
    //widthUsed 表示已经消耗了的宽度
    //parentHeightMeasureSpec heightUsed 同宽意义

    protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        //计算子View的measureSpec.
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        //发起子View的测量
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }

```
5.  ViewGroup的getChildMeasureSpec方法:    该方法会根据当前容器的MeasureSpec,和边距,子View的布局参数.确定子View最终的MeasureSpec.
```
    //参数意思:
    //spec: 当前容器View的MeasureSpec.
    //padding:  相对于子View来说,就是当前容器的内边距.相对于当前容器来说,该值通常是当前容器的内边距以及子View的外边距之和.
    //childDimension:   子View期望的大小.
    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        //计算出理论上在父容器中可使用的大小.
        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY://如果当前容器是EXACTLY的.
            if (childDimension >= 0) {
                //1. 子View如果有确定的期望大小值,则子View的size就是它期望的大小.且模式为EXACTLY.
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                //2.子View是MATCH_PARENT的,则子View的size就是理论上可使用的大小.模式为EXACTLY.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                //3.子View是WRAP_CONTENT的,则子View的size会是理论上可使用的大小,但是模式是AT_MOST.也就是不能超过可使用的大小
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST://如果当前容器是AT_MOST的
            if (childDimension >= 0) {
                //1.子View有确定的期望大小.则子View的size就是它期望的大小.且模式为EXACTLY.
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                //2.子View的布局参数为MATCH_PARENT时,则子View的size就是可使用的最大大小.且模式是AT_MOST.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                //3.子View的布局参数为WRAP_CONTENT是,则子View的size是可使用的最大小.且模式是AT_MOST.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED://当父容器的模式是UNSPECIFIED时:
            if (childDimension >= 0) {
                //1.子View有确定的期望大小.则子View的size就是它期望的大小.模式为EXACTLY
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                //2.子View布局参数是MATCH_PARENT的,则子View的大小是0或者最大可使用值.模式为UNSPECIFIED.
                //sUseZeroUnspecifiedMeasureSpec默认是false.
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                //3. 子View布局参数是WRAP_CONTENT时,则子View的大小是0或者最大可是用值,模式为UNSPECFIED.
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //noinspection ResourceType
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }

```
6.  View的resolveSizeAndState方法:
```
    //参数意思:
    //1.size: View想要的大小.只在View是WRAP_COENET时会用到.
    //2.measureSpec: 当前View的measureSpec.
    public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
        final int specMode = MeasureSpec.getMode(measureSpec);
        final int specSize = MeasureSpec.getSize(measureSpec);
        final int result;
        switch (specMode) {
            case MeasureSpec.AT_MOST://如果当前View测量模式是AT_MOST.那么最终大小是就是期望大小和可使用大小中小的那个值.
                if (specSize < size) {
                    //当可使用大小是小于期望大小的时候,同时还会给MEASURED_STATE_TOO_SMALL标识.
                    result = specSize | MEASURED_STATE_TOO_SMALL;
                } else {
                    result = size;
                }
                break;
            case MeasureSpec.EXACTLY://如果当前View的测量模式是EXACTLY的,那么最终大小就是MeasureSpec中的值.
                result = specSize;
                break;
            case MeasureSpec.UNSPECIFIED://如果当前View的测量模式是UNSPECIFIED的,则最终大小就是期望大小.
            default:
                result = size;
        }
        return result | (childMeasuredState & MEASURED_STATE_MASK);
    }
```

```
分析到这里.View的measure流程基本完成了.
总结一波:
View的measure方法:
    1.如果存在PFLAG_FORCE_LAYOUT标识,无论如何,都会执行onMeasuere方法.
    2.如果不存在PFLAG_FORCE_LAYOUT标识,则看当前measureSpec和上次MeasureSpec是否不一致.
        不一致:查看是否缓存当前MeasuereKey的缓存,存在则直接设置测量大小,并且会设置PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT.
        该标识会导致在layout实际之前再次测量.  不存在缓存,执行onMeasure方法.
    3.上述都不满足.则不需要测量.
    4.将测量值存入缓存.

onMeasure方法:
    各种View根据自己的规则去测量子View.并且设置自己的测量值.

```
