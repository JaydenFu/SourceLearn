#   View的绘制流程(四)--draw过程

1.  View的draw方法:通常对于我们自定义View只需要完成onDraw部分即可.并且知道在draw中的绘制顺序.
```
    简单点说:
    //1.如果需要绘制背景
    //2.绘制当前view的内容
    //3.绘制子View, 通过dispatchDraw方法完成.
    //4.绘制装饰性的内容(比如前景)
    public void draw(Canvas canvas) {
        ....
        // Step 1, draw the background, if needed
        int saveCount;

        if (!dirtyOpaque) {
            drawBackground(canvas);
        }
        // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            ......

            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);

            // we're done...
            return;
        }
            ......
    }
```
2.  View的dispatchDraw方法是空实现.因为它不是容器View,不可能有子View.
```
    protected void dispatchDraw(Canvas canvas) {
        //对于默认情况,应用或Activity开启硬件加速情况,这里的Canvas是DisplayListCanvas.返回值更具这里的mRenderNode
        //和申请DisplayListCanvas时传入的RenderNode是否是同一个来判断.是同一个返回ture.否则返回false.
        //普通的Canvas返回值始终是false.
        boolean usingRenderNodeProperties = canvas.isRecordingFor(mRenderNode);
         .....
        if (usingRenderNodeProperties) canvas.insertReorderBarrier();
        .....

        for (int i = 0; i < childrenCount; i++) {
            ......
            final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
            final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);

            //当子View是VISIBLE或者在执行animation动画.执行drawChild
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                more |= drawChild(canvas, child, drawingTime);
            }
        }
       ......
        if (usingRenderNodeProperties) canvas.insertInorderBarrier();

        ......
    }

```
3.  ViewGroup的drawChild方法: 在其内部调用子View的3个参数的draw方法.
```
    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
    }
```
4.  View的3个参数的draw方法:在该方法内部会最终调用单个参数的draw方法.
