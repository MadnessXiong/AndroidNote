### 1. View绘制流程的概览。

- **1.1 绘制流程的开始是从ViewRootImpl类的performTraversals() 开始的，看看代码**
```
    private void performTraversals() {
      
      int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
      int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
      //measure
      mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
      //layout
      mView.layout(0, 0, mView.getMeasuredWidth(), mView.getMeasuredHeight());
      //draw
      mView.draw(canvas);
      
    }

```

这里可以看到依次执行了View的measure(),layout(),draw()。这就是绘制一个View的流程。
那么依次看一下View这3个方法的源码

- **1.2 measure():**

```
    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
           
      		onMeasure(widthMeasureSpec, heightMeasureSpec);
      
          if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException("View with id " + getId() + ": "
                        + getClass().getName() + "#onMeasure() did not set the"
                        + " measured dimension by calling"
                        + " setMeasuredDimension()");
            }
    }

```
可以看到，measure()是一个final方法,是我们无法操作的，所以onMeasure()要重点关注的，并且onMeasure()里必须调用setMeasuredDimension(),不然之后就会抛出异常，那么看一下onMeasure()的源码:

```
   protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
```

可以看到，这里只是单纯的调用了setMeasuredDimension(),并传入了getDefaultSize()，那么再看一下代码：

```
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
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int opticalWidth  = insets.left + insets.right;
            int opticalHeight = insets.top  + insets.bottom;

            measuredWidth  += optical ? opticalWidth  : -opticalWidth;
            measuredHeight += optical ? opticalHeight : -opticalHeight;
        }
    
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }

    private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;

        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    }
```

可以看到，在getDefaultSize()中通过MeasureSpec.getMode()获取了specMode，通过MeasureSpec.getSize()获取size,这里可以借鉴使用

然后在setMeasuredDimension()中调用了setMeasuredDimensionRaw(),然后在这个方法里最终完成对mMeasuredWidth，mMeasuredHeight的赋值，以及对mPrivateFlags的更新。所以测量的宽高是通过setMeasuredDimension()保存的，同时setMeasuredDimension()后测量的宽高就有值了，可以通过getMeasuredWidth()获取测量的宽，通过getMeasuredHeight()获得测量的高。

**小结：measure()是一个final方法，它的方法体里调用了onMeasure(),一般自定义View时可以通过onMeasure()对宽高进行修正,然后在onMeasure()里调用setMeasuredDimension()对测量结果进行保存，所以重写onMeasure()必须调用setMeasuredDimension()，否则无法保存测量结果，并会抛出异常。同时onMeasure()执行完后才能通过getMeasuredWidth()和getMeasuredHeight()获取测量的宽高,因为setMeasuredDimension()是在onMeasure()里执行的**

>可以通过MeasureSpec.getMode()获取模式，通过MeasureSpec.getSize()获取宽高

- **1.3 layout():**

```
    public void layout(int l, int t, int r, int b) {
	//根据传入的值判断布局是否改变过，并调用setFrame()
        boolean changed = setFrame(l, t, r, b);
	//如果布局改变过调用onLayout()
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);
        }

     
    }

```

可以看到，先是通过setFrame()，判断布局是否改变过，如果改变过就调用onLayout(),那么先看一下它们的代码：

```
protected boolean setFrame(int left, int top, int right, int bottom) {
        boolean changed = false;
	//如果左上右下任何一个值改变，代表布局发生了改变
        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
            changed = true;
	//记录改变后的左上右下的值
            mLeft = left;
            mTop = top;
            mRight = right;
            mBottom = bottom;
        return changed;
    }
  	
  	//空实现
	  protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    }

```

可以看到如果左右上下任何一个值发生了改变，那么代表布局发生了改变，改变后对新的左右上下的值进行了记录

然后onLayout是一个空实现。

**小结：layout()接收了父View传过来的左上右下四个值，然后在setFrame()里根据这个值是否改变判断了是否更新布局，并且记录了自身最新的左上右下值，如果发生改变那么会调用onLayout()重新布局，需要调用着自己去实现**

- **1.4 draw():**

```
    public void draw(Canvas canvas) {
        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */
       // Step 1, draw the background, if needed
		  //如果需要的话绘制背景
		  if (!dirtyOpaque) {
		      drawBackground(canvas);
 			 }
      
        // Step 2, save the canvas' layers
        //保存绘制图层
         if (drawTop) {
             canvas.saveLayer(left, top, right, top + length, null, flags);
         }

        // Step 3, draw the content
      	//绘制内容
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
      	//绘制子view
        dispatchDraw(canvas);
      
      // Step 5, draw the fade effect and restore layers
      //绘过渡效果和恢复涂层
        final Paint p = scrollabilityCache.paint;
        final Matrix matrix = scrollabilityCache.matrix;
        final Shader fade = scrollabilityCache.shader;

        if (drawTop) {
            matrix.setScale(1, fadeHeight * topFadeStrength);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, right, top + length, p);
        }
      
       // Step 6, draw decorations (scrollbars)
		 	 //绘制滚动条
 			 onDrawScrollBars(canvas);

    }

```

可以看到，在源码中官方给出了6个步骤，但是只需要关注步骤3和步骤4就行了，那么分别看一下它们的源码：

```
  	protected void onDraw(Canvas canvas) {
  	
    }
    
    protected void dispatchDraw(Canvas canvas) {

    }

```

都是空实现，onDraw()是用来绘制自己的，dispatchDraw()是用来分发绘制子view的。

**小结：draw()中对背景等内容进行了绘制，同时通过onDraw()绘制自身，通过dispatchDraw()分发绘制子View**

### 2. VIew绘制流程细节分析

>前面分析了View的大概流程，现在看一下一些细节

- **2.1 先看measure(), 前面分析过，measure()是一个final方法，那么只能通过onMeasure去完成测量**

>**测量的时候有一个很重要的概念MeasureSpec，它封装了specMode和specSize，specMode是父View对子VIiew的要求，它有3种模式**:
MeasureSpec.EXACTLY：确定模式，父View期望子View的大小是一个已确定的值，也就是这个specSize
MeasureSpec.AT_MOST：至多模式，父View期望子View最大是某个值
MeasureSpec.UNSPECIFIED：未限定模式，父View对子View不做限制，如scrollView

了解了测量模式那么再看一下自定义View时的测量，之前了解到测量是调用measure()，然后可以在onMeasure()中进行修改，现在有2种情况，1是自定义view对自身进行修改，2是测量子view，分别看一下:

**2.1.1 通过代码先看修改自身：**

```
  @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        //模拟自定义View自定义的宽高
        int realWidthSize=100;
        int realHeightSize=100;
      	//获取mode和size
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
				//根据父View对子View的期望mode
        switch (widthMode) {
            //如果父view希望子view是一个确定的值，那么realWidthSize的修改无效，只能是widthSize
            case MeasureSpec.EXACTLY:
                realWidthSize = widthSize;
                break;
            //如果父view希望子view最多是一个值，那么当realWidth最大只能是widthSize
            case MeasureSpec.AT_MOST:
                if (realWidthSize > widthSize) {
                    realWidthSize = widthSize;
                }
                break;
            //如果是未限定模式，那么realWidthSize不变，想多大就多大
            case MeasureSpec.UNSPECIFIED:
                realWidthSize = realWidthSize;
                break;
        }
				//修正完成后通过setMeasuredDimension()保存结果，这里realHeightSize的修正是一样的
        setMeasuredDimension(realWidthSize,realHeightSize);

    }

```
以上就是对自定义view后对宽高的修正过程，可以看出其实都是模版代码，其实Android已经提供了方法进行修正，看下代码：

```
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        //模拟自定义View
        int realWidthSize=100;
        int realHeightSize=100;
				//通过resolveSize()对结果进行修正
        int resolveWidthSize = resolveSize(realWidthSize, widthMeasureSpec);
        int resolveHeightSize = resolveSize(realHeightSize, heightMeasureSpec);
      	//保存修正后的结果
        setMeasuredDimension(resolveWidthSize,resolveHeithSize);

    }

```

**2.1.2 再看测量子view：**

```
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
				//获取子view
        View child = getChildAt(0);
      	//获取子view的layoutParams，获取childWidth，然后定义mode
        LayoutParams layoutParams = child.getLayoutParams();
        int childWidth = layoutParams.width;
        int childMode=0;
				//获取mode和size
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        switch (widthMode) {
            //如果父view的要求是EXACTLY
            case MeasureSpec.EXACTLY:
                if (childWidth>=0){
                  //如果子view自己设置了具体的值，那么childWidth就是它自身的值，childMode就是EXACTLY
                    childWidth=childWidth;
                    childMode=MeasureSpec.EXACTLY;
                }else if (childWidth==LayoutParams.MATCH_PARENT){
                  //如果子view设置了MATCH_PARENT，那么它的childWidth就等于父view的widthSize，并且父view的mode也是EXACTLY，那么子view的mode也应该是EXACTLY
                    childWidth=widthSize;
                    childMode=MeasureSpec.EXACTLY;

                }else if (childWidth==LayoutParams.WRAP_CONTENT){
                  //如果子view设置了WRAP_CONTENT，那么这里先把父view的widthSize赋值给它的childWidth，并且它是可以在onMeasure()对自己的值进行修改的，那么它的值最大是widthSize，所以它的mode是MeasureSpec.AT_MOST
                    childWidth=widthSize;
                    childMode=MeasureSpec.AT_MOST;
                }
                break;
            //如果父view的要求是AT_MOST
            case MeasureSpec.AT_MOST:
                if (childWidth>=0){
            //如果子view自己设置了具体的值，那么childWidth就是它自身的值，childMode就是EXACTLY
                    childWidth=childWidth;
                    childMode=MeasureSpec.EXACTLY;
                }else if (childWidth==LayoutParams.MATCH_PARENT){
            //如果子view设置了MATCH_PARENT，那么它的childWidth就等于父view的widthSize，但是父view的mode是AT_MOST，那么子view的mode也应该是AT_MOST
                    childWidth=widthSize;
                    childMode=MeasureSpec.AT_MOST;
                }else if (childWidth==LayoutParams.WRAP_CONTENT){
            //如果子view设置WRAP_CONTENT，那么这里先把父view的widthSize赋值给它的childWidth，并且它是可以在onMeasure()对自己的值进行修改的，那么它的值最大是widthSize，所以它的mode是MeasureSpec.AT_MOST
                    childWidth=widthSize;
                    childMode=MeasureSpec.AT_MOST;
                }
                break;
            //如果父view没有要求
            case MeasureSpec.UNSPECIFIED:
                if (childWidth>=0){
            //如果子view自己设置了具体的值，那么childWidth就是它自身的值，childMode就是EXACTLY
                    childWidth=childWidth;
                    childMode=MeasureSpec.EXACTLY;
                }else if (childWidth==LayoutParams.MATCH_PARENT){
            //如果子view设置了MATCH_PARENT，那么它的childWidth就等于父view的widthSize，但是父view的mode是UNSPECIFIED，那么子view的mode也应该是UNSPECIFIED
                    childWidth=widthSize;
                    childMode=MeasureSpec.UNSPECIFIED;
                }else if (childWidth==LayoutParams.WRAP_CONTENT){
            //如果子view设置WRAP_CONTENT，那么这里先把父view的widthSize赋值给它的childWidth，并且它是可以在onMeasure()对自己的值进行修改的，那么它的值最大是widthSize，但是父view的widthSiz并没有限制，所以这里子view的mode也应该是UNSPECIFIED
                    childWidth=widthSize;
                    childMode=MeasureSpec.UNSPECIFIED;
                }
                break;
        
      //根据计算出的size和mode生成MeasureSpec
     int childWidthMeasureSpec=MeasureSpec.makeMeasureSpec(widthMode, widthSize);
     //height的步骤同理，这里省略
     int childWidthMeasureSpec=MeasureSpec.makeMeasureSpec(heightMode, HeightSize)
      //传入MeasureSpec完成测量
      child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```
以上就是子view的size和mode的计算，最后完成测量的过程，可以看出代码很繁琐，并且也是模版代码，Android也提供了方法:
```
 @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
                View child = getChildAt(0);
      	//带margin的测量,一般用这种，这里的参数0是已用空间，可以通过计算已测量的子view进行计算，这里为了方便写为0
        measureChildWithMargins(child,widthMeasureSpec,0,heightMeasureSpec,0);
        //普通测量子view
      	measureChild(child,widthMeasureSpec,heightMeasureSpec);
      	//测量所有子view,这里里面其实就是遍历了子view然后调用measureChild()
        measureChildren(widthMeasureSpec,heightMeasureSpec);
    }

```

这里有3个测量方法，其实内里逻辑差不多只是measureChildWithMargins()带了margin，那么在看下这个方法：

```
    protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
      	//这里用到了MarginLayoutParams
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
				//通过此方法完成mode和size的计算,这个方法不再展开
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);
				//通过计算出的
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }

```
可以看出内部也是计算出了MeasureSpec，最后通过measure()完成测量。需要主要的是如果使用measureChildWithMargins()测量，它里面用到了MarginLayoutParams，需要去实现一个方法,如果不实现，就无法使用layout_margin：

```
    @Override
    public LayoutParams generateLayoutParams(AttributeSet attrs) {
        return new MarginLayoutParams(getContext(),attrs);
    }

```

**小结：**

**1:自定义View时，可以在onMeasure()中通过resolveSize()对实际的宽高进行修正。如果不对宽高进行修改，那么就是它的父View调用child.measure()测量出的值，这里有个比较特殊的情况就是如果自身设置了wrap_content()，然后不做任何修正的话，那么父view的宽高就是自身的宽高，这个可以从上面的计算过程中看出来。**

**2:测量子View时可以通过measureChildWithMargins()等3个方法完成对子view的测量，需要注意的是如果使用measureChildWithMargins()需要实现generateLayoutParams()并返回new MarginLayoutParams(getContext(),attrs)**

- **2.2 再看layout():前面分析过，layout()第一个重要方法是setFrame():**

```
  protected boolean setFrame(int left, int top, int right, int bottom) {
        boolean changed = false;
			//如果左上右下任意一个值改变，代表布局改变
        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
          changed = true;
						//判断大小是否改变，布局改变不代表大小改变
          boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

						//记录新的左上右下的值
            mLeft = left;
            mTop = top;
            mRight = right;
            mBottom = bottom;
						
          	//如果布局大小改变，调用sizeChange()，然后在这个方法了里又会调用onSizeChanged()
            if (sizeChanged) {
                sizeChange(newWidth, newHeight, oldWidth, oldHeight);
            }

        }
        return changed;
    }

```

这里如果如果布局发生了改变，那么再判断大小是否发生改变，然后给新的左上右下赋值

这里有一个点是左上右下位置的改变，并不代表大小改变，有可能只是位置改变，如果只是位置改变，大小不变那么不会触发onSizeChanged()，否则就会触发onSizeChanged()，并且onSizeChanged()执行时新的位置已经赋值完成，那么这时候这个布局的实际宽高已经有值了，也就是getWidth()和getHeight()可以获取到最新的宽高信息了。

setFrame()完后，如果布局发生了改变或者调用requestLayout()，那么就会调用onLayout()，这是一个空实现，可以在这个方法里通过child.layout()，去对子view进行布局，子view又会重复走以上layout()流程。 

**小结：layout()主要通过它的setFrame()方法判断了布局是否改变，并且记录了最新的左上右下的值，如果布局发生改变，并且大小改变，那么会调用onSizeChange()，所以一般如果不需要对子view进行操作的话，自身的操作可以放到这个方法里，因为这时已经可以通过getWidth()和getHeigh()拿到真实的宽高值了。如果需要对子view进行操作，那么就重写onLayout(),通过child.layout()去摆放子view的位置，如此一直循环**

- **2.3 再看onDraw():**

```
    public void draw(Canvas canvas) {
        // Step 3, draw the content
      	//绘制内容
      	//背景不为空且不为透明时才会执行onDraw，viewgroup默认不会执行此方法
        if (!dirtyOpaque) onDraw(canvas)

    }

```
一般在onDraw()绘制自身内容，另外viewGroup默认不回执行onDraw(),同时onDraw()是一个会调用很多次的方法，所以不要在onDraw()做一些计算或频繁分配内存的工作，一面造成内存抖动。

### 3:总结

- view的绘制流程是从ViewRootImpl的performTraversals()开始的，在这个方法里依次调用了measure(),layout(),draw()

- 自定义view时父view通过调用子view的measure()可以测量子view。然后在子view可以通过重写onMeasure()，对自己进行修改，修改完成后可以通过resolveSize()方法修正宽高，修正后再通过MeasureSpec.makeMeasureSpec()生成MeasureSpec，最后通过setMeasuredDimension()保存结果，这个方法后就可以通过getMeasuredWidth()和getMeasuredHeight()获取测量的宽高了。

- 在onMeasure()调用子view的measure()方法可以对子view进行测量，可以通过measureChildWithMargins()生成需要的参数，调用这个方法必须实现generateLayoutParams()并返回new MarginLayoutParams(getContext(),attrs)

- 测量完成后就是layout(),它先是调用setFrame()，在这个方法里如果左上右下的值任何一个改变，那么代表布局改变，同时会记录最新的值，再会判断布局大小是否改变，如果改变，就会调用onSizeChange()，一般如果布局大小改变需要进行的操作可以重写这个方法，它执行时已经能通过getHeight()和getWidth()获取实际的宽高了。

- setFrame()执行完后会把结果返回给layout()，如果布局改变，那么会去执行onLayout()，这是个空方法，可以在这个方法里调用子view的layout()对子view进行布局。

- 最后是onDraw()这个方法就是绘制具体内容的方法，所以它会频繁执行，所以需要避免在此频繁分配内存的操作。viewgroup默认不会执行ondraw()

### 附：invalidate()和requestLayout()的区别
- invalidate()不会执行measure()和layout()，只执行draw(),如果开启了硬件加速则只进行调用者View的重绘。

- requestLayout()会执行自身及父view的measure(),layout(),一般情况下不会执行draw流程，如果要重绘必须调用invalidate()，或者当布局大小发生改变时layout()中的setFrame()会触发重绘。子View不一定会重新measure()和layout()。
