>在Android事件处理中单击，双击，长按，滑动都是很常用的手势操作。GestureDetector对这些操作进行了封装，方便使用

**1. 看一下GestureDetector的构造方法:**
```
public GestureDetector(Context context, OnGestureListener listener)
  
public GestureDetector(Context context, OnGestureListener listener, Handler handler)
    
public GestureDetector(Context context, OnGestureListener listener, Handler handler,
            boolean unused)
```
GestureDetector有3个构造方法，源码不展开，只做说明：第一种构造内部默认调用了第二种构造,handler传了null，第二种比第一种需要多传一个handler，第三种多了一个参数，但是看源码并没有用到，还是调用的第二个构造，所以没有意义。

所以重点在第二个构造上，看一下它的源码：
```
    public GestureDetector(Context context, OnGestureListener listener, Handler handler) {
       	//part1
      	if (handler != null) {
            mHandler = new GestureHandler(handler);
        } else {
            mHandler = new GestureHandler();
        }
      	
        mListener = listener;
      	//part2
        if (listener instanceof OnDoubleTapListener) {
            setOnDoubleTapListener((OnDoubleTapListener) listener);
        }
        if (listener instanceof OnContextClickListener) {
            setContextClickListener((OnContextClickListener) listener);
        }
        init(context);
    }
```
part1中可以看到不管有没有传入handler，都会创建一个GestureHandler()，只是传没传入handler的区别,看一下源码：
```
private class GestureHandler extends Handler {
        GestureHandler() {
            super();
        }
        GestureHandler(Handler handler) {
            super(handler.getLooper());
        }

        @Override
        public void handleMessage(Message msg) {
        }
    }
```
可以看到，如果传入了handler那么就获取它的looper，然后就没有区别了。

**小结：这里可以得出结论GestureDetector内部是使用handler处理消息的，而handler是要使用要looper的，那么如果在主线程使用的话内部创建的handler会直接获取到主线程的looper。而如果在子线程使用的话内部创建的handler是没有办法获取looper的，所以需要外部传入一个handler获取looper。所以如果是一般在主线程使用的话直接使用第一种构造就可以，如果在子线程使用，那么可以使用第二种构造，它必须传入一个handler，这个handler必须可以获取到looper。**

part2中可以看到如果传入的listener同时是OnDoubleTapListener或OnContextClickListener的话，也会自动设置回调，不用自己再在外部设置。

**2:再看一下GestureDetector的Listener**
>OnGestureListener
OnDoubleTapListener
OnContextClickListener
SimpleOnGestureListener

先看OnGestureListener：
```
public interface OnGestureListener {
				//down事件
        boolean onDown(MotionEvent e);
				//预按下
        void onShowPress(MotionEvent e);
				//单击确认
        boolean onSingleTapUp(MotionEvent e);
				//滑动
        boolean onScroll(MotionEvent e1, MotionEvent e2, float distanceX, float distanceY);
				//长按
        void onLongPress(MotionEvent e);
				//快速滑过，
        boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX, float velocityY);
    }
```
* onDown():
```
    //表示是否消费事件，如果需要响应后面的操作，这里必须返回true。
    @Override
    public boolean onDown(MotionEvent e) {
        return true;
    }
```

* onShowPress()：
```
//一般点击事件时用户按下后默认一段时间后会给用户一个反馈，如控件变色等。这个时间到了以后就会执行这个方法，大概是100毫秒
		@Override
    public void onShowPress(MotionEvent e) {

    }
```

* onSingleTapUp():
```
    //单击抬起时回调， 双击时第一次抬起时也会回调，如果响应长按时间此方法也不会触发，关闭长按事件长按此方法也会触发。返回值无意义
		@Override
    public boolean onSingleTapUp(MotionEvent e) {
        return false;
    }
```

* onScroll():
```
               //滑动监听，可以理解为onMove()
		//e1:代表手指按下时的Event，也就是down事件
		//e2:代表手指当前正在发生的事件,也就是move事件
		//distanceX：在x轴划过的距离，这个距离是某一时刻旧位置减去新位置的值，那么如果往左滑动distanceX就是正值，如果往右滑distanceX就是负值。
		//distanceY：在y轴划过的距离，这个距离是某一时刻旧位置减去新位置的值，那么如果往上滑动distanceY就是正值，如果往下滑distanceY就是负值
		//返回值无意义
		@Override
    public boolean onScroll(MotionEvent e1, MotionEvent e2, float distanceX, float distanceY) {
        return false;
    }
```

* onLongPress():
```
  	//长按回调
		@Override
    public void onLongPress(MotionEvent e) {

    }
```

* onFling():
```
  //快速滑过手指抬起后回调，一般快速滑动后会有一段惯性滑动，就是回调这个方法
	//e1:代表手指按下时的Event，也就是down事件
	//e2:代表手指当前正在发生的事件,也就是move事件
	//velocityX：滑行完成后在x轴上的速度，也就是移动的距离（像素）除以时间
	//velocityY：滑行完成后在y轴上的速度，也就是移动的距离（像素）除以时间
	//返回值无意义
	@Override
    public boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX, float velocityY){
        return false;
    }
```

再看OnDoubleTapListener：
```
   public interface OnDoubleTapListener {
				//单击确认
        boolean onSingleTapConfirmed(MotionEvent e);
 				//双击
        boolean onDoubleTap(MotionEvent e);
				//双击
        boolean onDoubleTapEvent(MotionEvent e);
    }
```

* onSingleTapConfirmed():
```
  //单击确认，确定是单击时才会触发，双击时的单次点击不会触发,返回值无意义
	public boolean onSingleTapConfirmed(MotionEvent e) {
        return false;
    }
```

*  onDoubleTap():
```
   	//双击时触发，第二次点击时即使不抬起手指也会触发
		@Override
    public boolean onDoubleTap(MotionEvent e) {
        return false;
    }
```

* onDoubleTapEvent()：
```
   //此方法和双击一样，会在第二次点击时点下时触发，但是它会响应随后的up事件，如果需要在第二次抬起时才响应时间，可以在此方法里做
   @Override
    public boolean onDoubleTapEvent(MotionEvent e) {
        return false;
    }
```

再看OnContextClickListener：
```
    //外部设备的按钮是否按下
    public interface OnContextClickListener {
        boolean onContextClick(MotionEvent e);
    }
```

再看：SimpleOnGestureListener：
```
 public static class SimpleOnGestureListener implements OnGestureListener, OnDoubleTapListener,
            OnContextClickListener {

        public boolean onSingleTapUp(MotionEvent e) {
            return false;
        }

        public void onLongPress(MotionEvent e) {
        }

        public boolean onScroll(MotionEvent e1, MotionEvent e2,
                float distanceX, float distanceY) {
            return false;
        }

        public boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX,
                float velocityY) {
            return false;
        }

        public void onShowPress(MotionEvent e) {
        }

        public boolean onDown(MotionEvent e) {
            return false;
        }

        public boolean onDoubleTap(MotionEvent e) {
            return false;
        }

        public boolean onDoubleTapEvent(MotionEvent e) {
            return false;
        }

        public boolean onSingleTapConfirmed(MotionEvent e) {
            return false;
        }

        public boolean onContextClick(MotionEvent e) {
            return false;
        }
    }
```
可以看到SimpleOnGestureListener实现了OnGestureListener和OnDoubleTapListener，重写了它们的所有方法，因为OnGestureListener和OnDoubleTapListener都是接口，如果都实现的话，必须每次重写所有方法，所以一般可以使用SimpleOnGestureListener()根据需要重写方法

**3:GestureDetector的使用**
```
GestureDetector gestureDetector = new GestureDetector(context,this);

  @Override
 public boolean onTouchEvent(MotionEvent event) {
        return gestureDetector.onTouchEvent(event);

    }
```
首先构建一个GestureDetector对象，传入Context和Listener,再在onTouchEvent()里把事件交给GestureDetector，再根据需要重写用到的方法就OK了