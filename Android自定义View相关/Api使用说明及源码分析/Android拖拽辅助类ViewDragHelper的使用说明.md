**1.先看一下如何构造一个ViewDragHelper：**

```
  
    public static ViewDragHelper create(@NonNull ViewGroup forParent, @NonNull ViewDragHelper.Callback cb) {
        return new ViewDragHelper(forParent.getContext(), forParent, cb);
    }

    public static ViewDragHelper create(@NonNull ViewGroup forParent, float sensitivity, @NonNull ViewDragHelper.Callback cb) {
        ViewDragHelper helper = create(forParent, cb);
        helper.mTouchSlop = (int)((float)helper.mTouchSlop * (1.0F / sensitivity));
        return helper;
    }
    
    
    
    private ViewDragHelper(@NonNull Context context, @NonNull ViewGroup forParent, @NonNull ViewDragHelper.Callback cb) {
   					//获取触发移动的最小距离
            this.mTouchSlop = vc.getScaledTouchSlop();
						//构造OverScroller
            this.mScroller = new OverScroller(context, sInterpolator);
       
    }
```
ViewDragHelper的构造是私有的，是通过create()构造对象的。

它有2个create(),第一个传入了parent,也就是使用ViewDragHelper的View本身，第二个是Callback。然后默认调用第二个create()，第二个多了一个参数sensitivity，这个sensitivity是用来设置mTouchSlop的，它值越大mTouchSlop就会越小，就会越敏感，也就是滑动的时候判断move的间距越短。

create()最后调用了构造，可以在构造方法中看到初始化了一个OverScroller，可以判断ViewDragHelper的滑动计算等操作也是通过OverScroller计算的（参考：[Android中Scroller的使用及原理解析](https://github.com/MadnessXiong/AndroidNote/blob/master/Android自定义View相关/Api使用说明及源码分析/Android中Scroller的使用及原理解析.md)）

**2. 再看一下ViewDragHelper的Callback**

```
 public abstract static class Callback {
        public Callback() {
        }
				//View的拖拽状态改变时触发
        //STATE_IDLE:　未被拖拽
        //STATE_DRAGGING：正在被拖拽
        //STATE_SETTLING:　被安放到一个位置中的状态
        public void onViewDragStateChanged(int state) {
        }
				//拖拽时的（开始移动）触发
      	//changeView:当前被拖拽的view
      	//left:拖动时left坐标
      	//top:拖动时top坐标
      	//dx:拖拽时x轴偏移量
      	//dy:拖拽时y轴偏移量
        public void onViewPositionChanged(@NonNull View changedView, int left, int top, @Px int dx, @Px int dy) {
        }
				//view被捕获时触发（也就是按下）
      	//capturedChild：捕获的view
      	//activePointerId：按下手指的id,多指触控时会用到
      	//一般用于做准备初始化工作
        public void onViewCaptured(@NonNull View capturedChild, int activePointerId) {
        }
				//view被放下时触发
      	//releasedChild被放下的view
      	//xvel：释放View的x轴方向上的加速度
      	//yvel：释放View的y轴方向上的加速度
      	//一般用于收尾工作
        public void onViewReleased(@NonNull View releasedChild, float xvel, float yvel) {
        }
				//边缘触摸时触发（需开启边缘触摸）
      	//edgeFlags：触摸的位置EDGE_LEFT,EDGE_TOP,EDGE_RIGHT,EDGE_BOTTOM
     		//pointerId： 按下手指的id,多指触控时会用到
      	//使用较少，一般不重写
        public void onEdgeTouched(int edgeFlags, int pointerId) {
        }
				//是否开启边缘触摸，true代表开启，默认不开启
      	//edgeFlags：触摸的位置EDGE_LEFT,EDGE_TOP,EDGE_RIGHT,EDGE_BOTTOM
       //使用较少，一般不重写
        public boolean onEdgeLock(int edgeFlags) {
            return false;
        }
				//边缘触摸时触发（需开启边缘触摸）
        //edgeFlags：触摸的位置EDGE_LEFT,EDGE_TOP,EDGE_RIGHT,EDGE_BOTTOM
     		//pointerId： 按下手指的id,多指触控时会用到
      	//使用较少，一般不重写
        public void onEdgeDragStarted(int edgeFlags, int pointerId) {
        }
				//寻找当前触摸点下的子View时会调用此方法，寻找到的View会提供给tryCaptureViewForDrag()来尝试捕获。
        //如果需要改变子View的遍历查询顺序可改写此方法，例如让下层的View优先于上层的View被选中。
      	//使用较少，一般不重写
        public int getOrderedChildIndex(int index) {
            return index;
        }
				//暂不明确（返回任何值都可以移动，网上说的都是错的）
       //使用较少，一般不重写
        public int getViewHorizontalDragRange(@NonNull View child) {
            return 0;
        }
				//暂不明确（返回任何值都可以移动，网上说的都是错的）
      	//使用较少，一般不重写
        public int getViewVerticalDragRange(@NonNull View child) {
            return 0;
        }
				//尝试捕获被拖拽的view，如果返回true代表可以被拖拽，返回false代表不可以被拖拽
      	//var1:被拖拽的view
      	//使用时判断需要被拖拽的view是否等等于var1。
      	//一般判断很多view其中哪些是否可以移动时使用
        public abstract boolean tryCaptureView(@NonNull View var1, int var2);
				//返回view在水平方向的位置，
      	//left:当前被拖拽的的view要移动到的的left值
      	//dx:移动的偏移量
      	//返回0则无法移动，通常直接返回left
      	//一般必须重写此方法返回left
        public int clampViewPositionHorizontal(@NonNull View child, int left, int dx) {
            return 0;
        }
      
      	//返回view在竖直方向的位置，
      	//top:当前被拖拽的的view要移动到的的left值
      	//dy:移动的偏移量
      	//返回0则无法移动，通常直接返回top
      	//一般必须重写此方法返回top
        public int clampViewPositionVertical(@NonNull View child, int top, int dy) {
            return 0;
        }
    }
```

**3. ViewDragHelper的使用**

* 先看一下常用到的方法
```
//以松手前的滑动速度为初速动，让捕获到的View自动滚动到指定位置。只能在Callback的onViewReleased()中调用。
settleCapturedViewAt(int finalLeft, int finalTop)
//以松手前的滑动速度为初速动，让捕获到的View在指定范围内fling。只能在Callback的onViewReleased()中调用。
flingCapturedView(int minLeft, int minTop, int maxLeft, int maxTop)
//指定某个View自动滚动到指定的位置，初速度为0，可在任何地方调用。
smoothSlideViewTo(View child, int finalLeft, int finalTop)
```

* 再看使用:
初始化并重写需要用到的方法：
```
 viewDragHelper=  ViewDragHelper.create(this, new ViewDragHelper.Callback() {
            @Override
            public boolean tryCaptureView(@NonNull View view, int i) {

                return true;
            }

            @Override
            public int clampViewPositionHorizontal(@NonNull View child, int left, int dx) {
               return left;
           }

            @Override
            public int clampViewPositionVertical(@NonNull View child, int top, int dy) {
                return top;
            }

           @Override
           public void onViewReleased(@NonNull View releasedChild, float xvel, float yvel) {
               super.onViewReleased(releasedChild, xvel, yvel);
                            super.onViewReleased(releasedChild, xvel, yvel);
             		//让捕获到的View自动滚动到100，300位置，只能在这里使用这个方法
               viewDragHelper.settleCapturedViewAt(100,300);
             	 //让捕获到的View在100，100，500，500这个范围内fling	，只能在这里使用这个方法
               viewDragHelper.flingCapturedView(100,100,500,500);
               //指定某个View自动滚动到500，500，初速度为0，可在任何地方调用。 
               viewDragHelper.smoothSlideViewTo(releasedChild,500,500);
               //以上方法必须手动去刷新页面
							 invalidate();
           }

       });


```

以上四个为常用的方法，然后获取事件权限：
```
    @Override
    public boolean onInterceptHoverEvent(MotionEvent event) {
        return viewDragHelper.shouldInterceptTouchEvent(event);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        viewDragHelper.processTouchEvent(event);
        return true;

    }
```

之前说了，ViewDragHelper内部是overScroller完成计算的,那么和overScroller一样需要重写computeScroll()一直刷新页面：
```
    @Override
    public void computeScroll() {
        super.computeScroll();
        //使用continueSettling(true)判断拖拽是否完成
        if (viewDragHelper.continueSettling(true)){
            invalidate();
        }
    }
```

