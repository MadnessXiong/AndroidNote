### 1. Scroller的使用

**1.1 构造Scroller**
```
 //构造1
 scroller = new Scroller(context);
 //构造2
 scroller = new Scroller(context, new Interpolator() {
            @Override
            public float getInterpolation(float input) {
                return 0;
            }
        });
```
可以看到Scroller有2个构造器，其中第二个构造器需要传入Interpolator，它是一个插值器，决定了view在滑动中的效果。第一种构造会默认使用ViscousFluidInterpolator这个插值器。

**1.2 重写View的computeScroll()**
```
    @Override
    public void computeScroll() {
        super.computeScroll();
      	//判断滑动是否完成，true为未完成，false未完成
        if (scroller.computeScrollOffset()){
          	//将view移动到当前滑动都都位置
            scrollTo(scroller.getCurrX(),scroller.getCurrY());

        }
    }
```

**1.3 开始滑动**
```
//这里代表用时3000毫秒，从100，100的位置分别在X轴分别移动500个像素，负值代表向右，正值代表向左
scroller.startScroll(100,100,-500,-500,3000);
//重绘页面
invalidate();
```
以上就是Scroller的简单使用，需要注意的是Scroller移动的是View的内容而不是View本身，如一个LinearLayout里有2个Button，移动的是这2个Button而不是LinearLayout自己(原因在源码解析里)。

另外Scroller还有个快速滑过的方法：
```
//startX-起始x轴位置
//startY-起始Y轴位置
//velocityX-x轴初始速度
//velocityY-y轴初始速度
//minX-x轴方向最小值，scroller的滑动不会超过这个点
//maxX-x轴方向最打值，scroller的滑动不会超过这个点
//minY-y轴方向最小值，scroller的滑动不会超过这个点
//maxY-y轴方向最大值，scroller的滑动不会超过这个点
public void fling(int startX, int startY, int velocityX, int velocityY,
            int minX, int maxX, int minY, int maxY) {}
```

### 2. Scroller的源码分析
> 从Scroller的使用可以看出，View的移动其实是在computeScroll()中通过自身的scrollTo()完成的。

**2.1 先看下startScroll():**
```
    public void startScroll(int startX, int startY, int dx, int dy, int duration) {
        mMode = SCROLL_MODE;
        mFinished = false;
      	//动画时常
        mDuration = duration;
        mStartTime = AnimationUtils.currentAnimationTimeMillis();
      	//x轴起点
        mStartX = startX;
      	//y轴起点
        mStartY = startY;
      	//最终x轴的值
        mFinalX = startX + dx;
      	//最终y轴的值
        mFinalY = startY + dy;
        mDeltaX = dx;
        mDeltaY = dy;
        mDurationReciprocal = 1.0f / (float) mDuration;
    }
```
可以看到startScroll只是记录了传入了参数，并没有做什么实际操作。



**2.2 前面说过移动是通过scrollTo()完成的，那么看看它的源码**
```
    public void scrollTo(int x, int y) {
        if (mScrollX != x || mScrollY != y) {
            int oldX = mScrollX;
            int oldY = mScrollY;
            mScrollX = x;
            mScrollY = y;
            invalidateParentCaches();
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
            if (!awakenScrollBars()) {
                postInvalidateOnAnimation();
            }
        }
    }
```

这里可以看到scrollTo更新了mScrollX，和mScrollY的值，最终会触发重绘调用invalidate()，最终会调用到viewGroup的drawChild():
```
    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
    }
```

最终调用到了view的3个参数的draw()，再继续看
```
  boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
          if (!drawingWithRenderNode) {
            computeScroll();
            sx = mScrollX;
            sy = mScrollY;
        }
 		 canvas.translate(mLeft - sx, mTop - sy);
  }
```
在draw()中调用了computeScroll(),然后把最新的mScrollX，mScrollY赋值给sx,sy，最后完成移动。

那么就是这样一个流程：
startScroll()->invalidate()->draw()->computeScroll()-scrollTo()->invalidate()->draw()->computeScroll()-scrollTo()...
并一直循环，直到scroller.computeScrollOffset()返回false，滑动完成。

### 3：OverScroller：

>OverScroller和Scroller基本相同，不过它比Scroller更加完善，并且进行了一些扩展，主要添加了越界操作

**3.1 看一下OverScroller扩展的方法**

* isOverScrolled()：判断是否滑动出了边界

* springBack(int startX, int startY, int minX, int maxX, int minY, int maxY)：实现回弹效果,返回true代表控件不在指定范围内，false反之

* fling(int startX, int startY, int velocityX, int velocityY, int minX, int maxX, int minY, int maxY, int overX, int overY)：这里比Scroller的fling的多2个参数，实现了回弹效果，overX决定了x轴可以滑出屏幕多远再弹回来，overY决定了y轴可以滑出屏幕多远再弹回来。（回弹效果的实现和springBack()调用的是同样的方法）

### 4：总结

* OverScroller是Scroller的扩展，它实现了回弹效果，一般推荐使用OverScroller

* 调用startScroll()确定起始位置以及偏移量，然后调用invalidate()让view绘制在起始位置

* invalidate()最终会触发draw()，然后触发computeScroll()

* 重写computeScroll(),通过OverScroller获取最新计算出的x,y值，然后再通过View自身的scrollTo()更新x,y值

* 更新后再次invalidate()，以此重复，一直更新坐标，一直绘制到新的位置，完成滑动

* 最后通过OverScroller的computeScrollOffset()，判断滑动是否完成，完成后就不再调用scrollTo(),最终完成整个滑动过程