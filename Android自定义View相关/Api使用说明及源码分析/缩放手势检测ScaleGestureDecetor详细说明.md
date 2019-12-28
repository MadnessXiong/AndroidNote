**1. 看一下ScaleGestureDecetor的构造**
```
 //构造1
		public ScaleGestureDetector(Context context, OnScaleGestureListener listener) {
        this(context, listener, null);
    }
    
		//构造2
    public ScaleGestureDetector(Context context, OnScaleGestureListener listener,
                                Handler handler) {
        mContext = context;
        mListener = listener;
    
        if (targetSdkVersion > Build.VERSION_CODES.JELLY_BEAN_MR2) {
            setQuickScaleEnabled(true);
        }
    }
    
     public void setQuickScaleEnabled(boolean scales) {
        mQuickScaleEnabled = scales;
        if (mQuickScaleEnabled && mGestureDetector == null) {
          	//构造GestureDetector，传入handler
            mGestureDetector = new GestureDetector(mContext, gestureListener, mHandler);
        }
    }
```
ScaleGestureDetector有2个构造方法，第一个参数传入了context,OnScaleGestureListener,第二个比第一个多了一个handler参数，然后第一个构造默认调用了第二个构造，那么只需要关注第二个构造就可以了。

在第二个构造里调用了setQuickScaleEnabled(),在setQuickScaleEnabled()里把handler传给了GestureDetector，那么这个handler其实还是GestureDetector用的，只是为了获取一个looper。

**2. 再看一下ScaleGestureDetector的Listener**

>OnScaleGestureListener
SimpleOnScaleGestureListener

SimpleOnScaleGestureListener只是OnScaleGestureListener的空实现，主要关注下OnScaleGestureListener：

```
    public interface OnScaleGestureListener {
				//缩放中
        public boolean onScale(ScaleGestureDetector detector);
				//缩放开始
        public boolean onScaleBegin(ScaleGestureDetector detector);
				//缩放结束
        public void onScaleEnd(ScaleGestureDetector detector);
    }
```
* onScale():
```
 	//正在缩放，会多次回调
	//返回值如果返回true，代表事件已经处理，会重新累积缩放因子；如果返回false，则会继续累积缩放因子
	@Override
  public boolean onScale(ScaleGestureDetector detector) {
    return false;
   }
```

* onScaleBegin():
```
//缩放开始，如果要使用缩放这里要返回true，代表要消费事件
@Override
public boolean onScaleBegin(ScaleGestureDetector detector) {
   	return false;
}
```

* onScaleEnd():
```
//缩放结束
@Override
public void onScaleEnd(ScaleGestureDetector detector) {

 }
```

**3. ScaleGestureDecetor的使用**
```
 scaleGestureDetector = new ScaleGestureDetector(context, new ScaleGestureDetector.OnScaleGestureListener() {
                @Override
                public boolean onScale(ScaleGestureDetector detector) {
                  	//获取x轴的焦点
                    detector.getFocusX();
                  	//获取y轴的交点
                    detector.getFocusY();
                  	//获取缩放系数
                    detector.getScaleFactor();
                    return false;
                }

                @Override
                public boolean onScaleBegin(ScaleGestureDetector detector) {
                    return false;
                }

                @Override
                public void onScaleEnd(ScaleGestureDetector detector) {

                }
            });
            
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        return scaleGestureDetector.onTouchEvent(event);
        
    }
```
ScaleGestureDecetor的使用非常简单，首先构造ScaleGestureDetector，传入listener重写方法，然后在onTouchEvent()中将事件交给ScaleGestureDetector处理就可以了

在onScale()中可以通过detector.getFocusX()获取x轴的焦点，通过detector.getFocusY()获取y轴的焦点，通过detector.getScaleFactor()获取缩放系数,这个系数小于1代表缩放，大于1代表放大。