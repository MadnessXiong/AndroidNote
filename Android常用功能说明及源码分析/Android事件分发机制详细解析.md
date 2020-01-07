### 1. 事件分发机制的大概流程
**1.1** 当一个点击事件发生时，首先会传递到Activity的dispatchTouchEvent，看一下代码：
```
  public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```
可以看到这里做了3件事
* 当事件是MotionEvent.ACTION_DOWN时调用onUserInteraction()
* 把事件传递给winodw的superDispatchTouchEvent()，如果返回true，则结束方法
* 如果winodw的superDispatchTouchEvent()返回false，则执行onTouchEvent()

**1.2** 那么一个个来看下，先看下onUserInteraction()
```
    public void onUserInteraction() {
    }
```
可以看到此方法是一个空实现，**但是它是事件发生时第一个会被调用的方法**，也就是说用户只要开始交互就会调用此方法，那么可以根据需要重写此方法。
> onUserInteraction方法主要是用于管理状态栏通知，以及在恰当的时候取消通知。与该方法相关的还有另一个方法，onUserLeaveHint。该方法作为Activity生命周期回调的一部分，会在用户将Activity放到后台时调用（如用户点击Home键），该方法会在onPause方法之前调用。

**1.3** 再看getWindow().superDispatchTouchEvent(ev),PhoneWindow是Window所以这里实际调用的是PhoneWindow的方法
```
   @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }
```
PhoneWindow实际调用来mDecor的superDispatchTouchEvent(),这个mDecor实际是DecorView,再看
```
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }
```
可以看到最终调用里DecorView父类的dispatchTouchEvent(),DecorView继承自FrameLayout，而FrameLayout并没有实现父类的dispatchTouchEvent并且FrameLayout继承自ViewGroup，所以最终会调用ViewGroup的dispatchTouchEvent()。也就是说所有的时间最终都会交给ViewGroup的dispatchTouchEvent()处理。

**1.4** 再看onTouchEvent():
```
    public boolean onTouchEvent(MotionEvent event) {
        if (mWindow.shouldCloseOnTouch(this, event)) {
            finish();
            return true;
        }

        return false;
    }
```
在onTouchEvent()中调用Window的shouldCloseOnTouch(),判断是否该结束activity,此方法是系统的hide()方法不必关注,如果不结束,则返回false并没有做任何操作

**1.5 小结一下：
首先当用户按下时，会产生一个事件，这个事件首先会由Activity的dispatchTouchEvent()接收到，它主要做了三件事**
* 第一件事是判断是否是down事件，如果是则调用onUserInteraction()，此方法是空实现，一般用来管理状态栏通知
* 第二件事是将事件传递给ViewGroup的dispatchTouchEvent(),如果ViewGroup处理事件(也就是返回true),则事件就直接交给ViewGroup处理，结束方法
* 第三件事是如果ViewGroup并没有处理事件(也就是返回false)，则调用Activity的onTouchEvent(),这里系统进行里一些判断，并没有执行其他逻辑

**可以看出，事件处理的核心其实是在ViewGroup的dispatchTouchEvent()中**

### 2. 事件处理的核心
**2.1** 事件处理的核心在ViewGroup的dispatchTouchEvent()中，由于源码过长，将分为数个片段解读，先看片段1：
```
//片段1
public boolean dispatchTouchEvent(MotionEvent ev) {
  			//表示是否已经处理了事件
        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
           //获取事件类型
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

           	//如果是down事件，代表是一个全新的事件，则进行一些清除及重置工作
            if (actionMasked == MotionEvent.ACTION_DOWN) {
 								//清空targets
                cancelAndClearTouchTargets(ev);
              	//重制状态
                resetTouchState();
            }
            //表示是否拦截事件
            final boolean intercepted;
          	//如果是down事件或者mFirstTouchTarget不等于null，
          	//TouchTarget记录了已处理事件的组件
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
              	//获取子view是否通知父view不要拦截
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;							//如果子view没有通知过父类不要拦截
                if (!disallowIntercept) {
                  	//调用自己的拦截方法是否拦截
                    intercepted = onInterceptTouchEvent(ev);
                  	//恢复action，因为之前改变过
                    ev.setAction(action); // restore action in case it was changed
                } else {
                  //子view要求父view不要拦截
                    intercepted = false;
                }
            } else {
               	//不是down事件并且没有view处理事件，则这里拦截事件
                intercepted = true;
            }

        return handled;
    }
```
可以看到，在片段1中首先声明了boolean类似的handled字段，这个字段是dispatchTouchEvent()的返回值，决定了此view是否处理事件。然后获取了事件类型，如果是down事件，那么代表是一个全新的事件，则进行清理及重置工作。
>这里调用了cancelAndClearTouchTargets(),看名字可以知道是对TouchTargets进行处理,TouchTarget是一个链表，它里面记录了处理事件的view,一般只是单点事件的话只需要关注mFirstTouchTarget。mFirstTouchTarget标记了当前viewgroup的哪个子view消费了事件

重置完成后，声明了boolean类型的变量intercepted，此变量代表此viewGroup是否拦截事件，然后再判断是否是down事件，或者mFirstTouchTarget不为null，如果都不符合则直接将intercepted置为true，拦截事件。

> down事件代表新事件，mFirstTouchTarget不为null则代表已有view处理此事件，所以一般是move,up等其他事件

如果有一项符合，则先判断子view是否请求过不要拦截，如果子view请求过则直接将intercepted置为false,不拦截事件，如果子view没有请求过则调用自身的onInterceptTouchEvent(),赋值给intercepted，默认不拦截，可以通过重写方法进行拦截。
> 子view可以通过调用getParent().requestDisallowInterceptTouchEvent(true)来通知父view不要拦截

**小结一下：片段1首先进行了清理及重置工作，然后判断如果不是down事件，并且没有子view处理事件（mFirstTouchTarget为null）那么直接拦截事件。如果有一条符合则先判断子view是否请求过不要拦截事件，如果请求过则不拦截，如果没有请求过，则调用自身的onInterceptTouchEvent()决定是否拦截事件**

**2.2** 片段2：如果片段1的代码拦截了事件，那么片段2的代码不会执行
```
//仅保留相关代码  
//如果没有取消，并且没有被拦截
if (!canceled && !intercepted) {
  							//如果是down事件（其他条件可以不用关注），或者多指操作，或者鼠标
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
         							//判断子view是否处理事件,并进行事件分发
                  		//这里的child的参数是由未贴出的代码遍历得出
                      if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                          //对mFirstTouchTarget进行赋值   
                          newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                break;
                            }
                }
            }
```
如果不拦截事件，会来到片段2，如果是down事件，首先会遍历viewgroup的子view（代码未贴出）,会调用dispatchTransformedTouchEvent()来判断子view是否消费事件，如果子view消费事件（返回true），那么代表有view消费事件，通过addTouchTarget()给mFirstTouchTarget赋值。那么看一下dispatchTransformedTouchEvent():
```
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            handled = child.dispatchTouchEvent(transformedEvent);
        }
        return handled;
    }
```
这里可以看到，如果child为null，则调用super.dispatchTouchEvent(transformedEvent)，如果不为空则调用child的
dispatchTouchEvent()，viewGroup的父为View,child也是View，所以都是调用的View的dispatchTouchEvent()，那么看一下代码：
```
 public boolean dispatchTouchEvent(MotionEvent event) {
				//是否处理事件
        boolean result = false;

        if (onFilterTouchEventForSecurity(event)) {
            ListenerInfo li = mListenerInfo;
         		//mOnTouchListener不为null，并且是ENABLED状态，调用onTouch()
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
				//返回是否处理事件的结果
        return result;
    }
```
可以看到，这里声明了boolean类型的变量result,它代表是否处理事件。

如果设置了OnTouchListener,那么会调用它的onTouch()，并且返回true的话会将result置为true。

如果返回true，那么将不再调用onTouchEvent()方法。如果返回false,那么将根据onTouchEvent()的结果拉决定是否消费事件。

**小结一下：片段2中，首先遍历了viewGroup的view，并调用他们的dispatchTransformedTouchEvent(),在这个方法中，如果子view不为null，那么调用子view的dispatchTouchEvent()，这个方法是重写最顶层View的dispatchTouchEvent()。如果为null，那么会调用super.dispatchTouchEvent(),往上传递，如果一直没有view处理，也就是最终也会调用最顶层View的dispatchTouchEvent()，在顶层View的dispatchTouchEvent()中，如果设置过OnTouchListener，那么会调用它的onTouch()，如果onTouch()返回true，则处理事件，并不再响应onTouchEvent(),如果onTouch()返回false,则会调用onTouchEvent(),根据onTouchEvent()的返回值决定是否处理事件**

**2.3** 片段3: move,up等事件的分发
```
          //mFirstTouchTarget为null代表没有view处理事件，或者自身拦截了事件
					if (mFirstTouchTarget == null) {
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
               
                while (target != null) {
                    final TouchTarget next = target.next;
                  //alreadyDispatchedToNewTouchTarget排除了down事件
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                     		//上面排除了down事件，并且mFirstTouchTarget不为null，那么move,up事件就会走这里
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                      
                    }
                    predecessor = target;
                    target = next;
                }
            }
```
这里可以看到如果mFirstTouchTarget为null，那么没有view处理事件同样会调用dispatchTransformedTouchEvent()，如果不为null，那么会排除down事件，然后通过dispatchTransformedTouchEvent(),对move,up等事件进行分发。这里的dispatchTransformedTouchEvent()和片段2里是一样的。

### 3. 总结
* 当一个事件发生时首先会调用Activity的dispatchTouchEvent(),在这个方法中会调用onUserInteraction()，所以onUserInteraction()是第一个响应事件的方法。

* 然后会经过传递最终会调用viewGroup的dispatchTouchEvent(),如果返回false，代表viewGroup不处理事件，则会调用Activity自身的onTouchEvent()。
*  如果ViewGroup处理事件，那么在它的dispatchTouchEvent()中，首先如果是down事件，那么会清空TouTouchGets并重置状态
> TouchTarget是一个链表，它里面记录了处理事件的view,一般只是单点事件的话只需要关注mFirstTouchTarget。

* 状态重置完成后，会判断是否是down事件或者mFirstTouchTarget是否等于null，down代表新事件，mFirstTouchTarget代表已有子view处理事件，那么判断子view是否请求过不要拦截事件，如果请求过则直接不拦截，如果没请求过，则调用自身的onInterceptTouchEvent()决定是否拦截事件

* 如果不拦截并且是down事件，那么遍历viewGroup的子view，调用他们的dispatchTransformedTouchEvent()，在这个方法中，如果子view不为null,那么会调用子view的dispatchTouchEvent(),如果为null则调用父view的dispatchTouchEvent()，一直向上传递，都没有view处理的话，最终会来到顶层View的dispatchTouchEvent()

>顶层View和子view的dispatchTouchEvent()是一个，所以合并分析
* 在View的dispatchTouchEvent()中，会判断是否设置了OnTouchListener(),如果设置了，那么会调用它的onTouch(),并根据onTouch()的返回值决定是否处理事件。如果onTouch()返回true则直接处理事件返回。如果onTouch()返回false,则再会去调用onTouchEvent()。再根据onTouchEvent()的结果决定是否处理事件

>这里可以得出一个结论，onTouch()是先于onTouchEvent()执行的，并且onTouch()返回true的话，那么不再执行onTouchEvent();
也可以看出viewGroup是没有重写onTouchEvent()的，调用的都是View的OnTouchEvent(),从view和viewGroup的dispatchTouchEvent()的区别可以看出，viewGroup主要是分发事件，View主要是处理事件

* dispatchTransformedTouchEvent()的返回值决定了是否有view处理事件，如果有view处理返回true，那么就会调用addTouchTarget(),在这个方法里完成对mFirstTouchTarget的赋值。

* 继续回到viewGroup的dispatchTouchEvent(),如果mFirstTouchTarget为null，那么调用dispatchTransformedTouchEvent()完成分发(如果没有view处理。最终一般会通过这个方法层层往上传递，最终丢掉事件)

* 如果mFirstTouchTarget不为null，那么代表有view处理事件，那么先排除掉down事件，防止重复调用，然后再通过dispatchTransformedTouchEvent(),对move,up等事件进行分发。



