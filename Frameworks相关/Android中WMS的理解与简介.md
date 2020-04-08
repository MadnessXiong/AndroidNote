###  1. WMS的概念

从名字可以看出，window表明它是与窗口相关的，Manager表明它具有管理者的身份。简单来讲，它是窗口管理员。窗口是一个抽象的概念，从用户的角度来讲，它是一个界面。从SufaceFlinger的角度来讲，它是一个Layer,承载着和界面有关的数据和属性。所以它是一个WindowState，用于管理和界面有关的状态。

WMS也是系统服务，由SystemServer启动。直到关机时才会退出。发生异常时必须重启。

- **1.1 WMS涉及的元素：**

  - WindowManagerPolicy-mPolicy:窗口策略类。

    Android提供了WindowManagerPolicy接口定义Ui显示策略，手机的实现类为PhoneWindowManager。也可以自己实现WindowManagerPolicy来自定义策略。

  > 窗口策略：这里的initPolicy()初始化的PhoneWindowManager代表的是一种窗口策略。应用到WMS中则代表了Android显示系统所遵循的统一的窗口显示规则。针对不同的产品，UI显示策略通常是不一样的。如手机一般都有Status Bar，但是平板没有。所以要设定不同的策略。

  - ArraySet<Session>-mSessions:主要用于进程间通信，其他应用程序想要和WMS通信就需要经过Session，每个应用程序进程都会对应一个Session，WMS保存这些Session用来记录所有向WMS提出窗口管理服务的客户端。

  - WindowHashMap<IBinder,WindowState>-mWindowMap:WindowState用于保存窗口信息，用来描述一个窗口。mWindowMap其实就是用来保存WMS中各种窗口的集合

  - ArrayList<AppWindowToken>-mFinishedStarting:AppWindowToken为WindowToken的子类，WindowToken主要有2个作用：

    - 窗口令牌，当应用程序想要向WMS申请新创建一个窗口，则需要向WMS出示有效的WindowToken。AppWindowToken主要用来描述应用程序的WindowToken结构，应用程序中每个Activity都对应一个AppWindowToken
    - WindowToken会将同一个组件(比如同一个Activity)的窗口(WindowState)集合在一起，方便管理

    mFinishedStarting就是用于存储已经完成启动的应用程序窗口(比如Activity)的AppWindowToken列表。

  - ArrayList<WindowState>-mResizingWindows:用来存储正在调整大小的窗口列表。

  - WindowAnimator-mAnimator:用于管理窗口的动画以及特效动画

  - H-h:系统的Handler类，用于将任务加入到主线程消息队列中

  - InputManagerService-mInputManager:输入系统的管理者。会对触摸事件进行处理，他会寻找一个合适的窗口来处理触摸返回信息，WMS是窗口的管理者，所以需要持有IMS引用

  - 显示图形的需求

    - Application Window：普通应用程序显示申请所产生的window，和系统窗口相比，它们的窗口层级值比较低。
    - System Window：系统顶部的系统状态栏，壁纸等
    - Sub Window：Toast等弹窗

  - ActivityManagerService：AMS管理者所有的Activity，而Activity的变化通常会带来界面上的改变。那么界面上产生的变化（淡出等动画效果）也要涉及WMS

- **1.2 WMS包含的功能**
  - 窗口的添加与删除：当某个进程由显示需求时，它可以请求WMS添加一个窗口，并于不再需要显示时移除该窗口
  - 启动窗口：当增了一个窗口时，在某些条件下需要添加一个启动窗口
  - 窗口动画：窗口切换时，采用窗口动画可以加强UI特效。可以定制
  - 窗口大小： Android系统支持显示不同大小的窗口，如StatusBar就只是屏幕最顶层的一条Bar，还有对话框，浮窗等
  - 窗口层级：由Z-Order决定，值越大，排序越靠后，会被值小的窗口盖住
  - 事件派发

- **1.3 内部组织方式**

  当一个新的Activity被启动时，它首先需要在AMS中注册，此时AMS会在内部生成一个ActivityRecord来记录这个Activity；另外因为Activity是四大组件中专门用于UI显示的，所以WMS也会对它以WindowState的形式进行记录。

  所以Activity在AMS中的表现形式为ActivityRecord，在WMS中的表现形式为WindowState，其中WMS还有一个变量AppWindowToken和AMS中的ActivityRecord相对应，这样它们就关联了起来。

### 2. WMS的启动

WMS由SystemServer进程启动，在SystemServer的main()中执行了 startOtherServices()，WMS就在这里启动，看下代码：

```java
       private void startOtherServices() {
         			 //part1
               final Watchdog watchdog = Watchdog.getInstance();
               watchdog.init(context, mActivityManagerService);
         			 //part2
               inputManager = new InputManagerService(context);
         			 //part4
               wm = WindowManagerService.main(context, inputManager,
                       mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
                       !mFirstBoot, mOnlyCore, new PhoneWindowManager());
         			 //part5
               ServiceManager.addService(Context.WINDOW_SERVICE, wm);
               ServiceManager.addService(Context.INPUT_SERVICE, inputManager);
```

- part1部分初始化了WatchDog，WatchDog用来监控系统的一些关键服务运行的情况，如WMS，它每分钟都会对被监控的系统服务进行检查，如果被监控的服务出现了死锁，则会杀死WatchDog所在进程，也就是SystemServer进程。前面提到过，WMS发生异常时必须重启，就是在这里进行监控
- part2部分构造了IMS，IMS负责输入消息，WMS需要持有它的引用
- part3部分通过WindowManagerService.main()构造了一个WMS对象，并传入了构造好的IMS，这样WMS就持有了IMS引用。
- part5中WMS，IMS注册到了ServiceManager中，这样如果某个客户端想使用它们的功能，就可以去ServiceManager查询，进而可以进行进程间通信

WMS是通过它自己的main()构造的，继续看这个方法：

```java
       public static WindowManagerService main() {
           DisplayThread.getHandler().runWithScissors(() ->
                   sInstance = new WindowManagerService(context, im, haveInputMethods, showBootMsgs,
                           onlyCore, policy), 0);
           return sInstance;
       }
```

在DisplayThread线程中创建了WMS，这个线程是一个单例的前台线程，它用来处理需要低延迟显示的相关操作，并只能由WM，DisplayManager和InputManager实时执行快速操作。由于WMS的优先级更高，所以system_server线程需要等待DisplayThread线程执行完创建WMS的操作，才会继续执行，在这之前它会一直等待。

再看WMS的构造方法：

```java
       private WindowManagerService() {
					 //持有IMS引用
           mInputManager = inputManager; // Must be before createDisplayContentLocked.
					 //获取DisplayManager
           mDisplayManager = (DisplayManager)context.getSystemService(Context.DISPLAY_SERVICE);
           //获取Display数组
					 mDisplays = mDisplayManager.getDisplays();
           //遍历Display，将Display封装成DisplayContent，DisplayContent用来描述一块屏幕
           for (Display display : mDisplays) {
               createDisplayContentLocked(display);
           }
   				 //持有AMS引用
           mActivityManager = ActivityManager.getService();
   				 //创建mAnimator，用于管理窗口动画
           mAnimator = new WindowAnimator(this);
           //创建窗口策略管理类PhoneWindowManager
					 initPolicy();
           //将WMS添加到WatchDog中
           Watchdog.getInstance().addMonitor(this);
       }
```

在WMS的构造中进行了一些初始化，持有了AMS，IMS引用，获取了所有管理的屏幕对象DisplayContent，创建了策略管理类PWM,最后将自己添加到WathDog中。再看PWM的初始化，也就是initPolicy():

```java
       private void initPolicy() {
           UiThread.getHandler().runWithScissors(new Runnable() {
               @Override
               public void run() {
                   WindowManagerPolicyThread.set(Thread.currentThread(), Looper.myLooper());
   
                   mPolicy.init(mContext, WindowManagerService.this, WindowManagerService.this);
               }
           }, 0);
       }
   
```

PWM的init()运行在UI线程中，它的优先级高于Display线程，因此DisPlay线程要等待它执行完后才会执行。

**总结：**

- 首先在System_server线程中执行了SystemServer的starOtherService(),在starOtherService()中会调用WMS的main()，这个main()会创建WMS，创建的过程在disPlay线程中实现，因WMS的优先级更高，因此System_server线程要等WMS创建完成后，处于等待状态的System_server线程才会被唤醒从而继续执行下面的代码
- 在WMS的构造中会调用WMS的initPolicy()，在initPolicy()中又会调用PWM的init(),这个init()在UI线程中运行，它的优先级要高于display线程，因此display线程要等到PWM的init()执行完毕后，才会被唤醒执行后面的代码
- PWM的init()执行完毕后，disPlay线程就完成了WMS的创建，等待的system_server线程就会被唤起继续执行main()之后的逻辑

### 3. Window添加过程

window的添加从WMS的addWindow()开始：

```java
       public int addWindow() {
					 //part1权限检查
           int res = mPolicy.checkAddPermission(attrs, appOp);
   
           synchronized(mWindowMap) {
					 //part2寻找displayContent
           final DisplayContent displayContent = mRoot.getDisplayContentOrCreate(displayId);
					 //避免重复添加
           if (mWindowMap.containsKey(client.asBinder())) {
                   return WindowManagerGlobal.ADD_DUPLICATE_ADD;
               }
   				 //part3判断是否子窗口
           if (type >= FIRST_SUB_WINDOW && type <= LAST_SUB_WINDOW) {
  					//寻找父窗口
             parentWindow = windowForClientLocked(null, attrs.token, false);
            }
            //part4获取WindowToken
            WindowToken token = displayContent.getWindowToken(
                       hasParent ? parentWindow.mAttrs.token : attrs.token);
						 //如果有父窗口就将父窗口的type赋值给rootType，如果没有则将当前窗口的type赋值给rootType
             final int rootType = hasParent ? parentWindow.mAttrs.type : type;
             
						if (token == null) {
            //part5如果token为null，则隐式创建token,这说明创建窗口时可以不向WMS提供提供WindowToken。这里的参数false代表token是隐式创建的，不是传进来的
                token = new WindowToken(this, binder, type, false, displayContent,
                           session.mCanAddInternalSystemWindow);
               } else if (rootType >= FIRST_APPLICATION_WINDOW && rootType <= LAST_APPLICATION_WINDOW) {
             //part6如果为应用程序窗口，将windowToken转换为应用程序窗口的AppWindowToken
             atoken = token.asAppWindowToken();
              }
             
             //part7创建windowState，它存有窗口的所有信息，代表一个窗口
             final WindowState win = new WindowState(this, session, client, token, parentWindow, appOp[0], seq, attrs, viewVisibility, session.mUid, session.mCanAddInternalSystemWindow);
              //part8判断请求添加窗口的客户端是否已经死亡
              if (win.mDeathRecipient == null) {
                   return WindowManagerGlobal.ADD_APP_EXITING;
               }
   						//part9判断窗口的DisplayContent是否为null
               if (win.getDisplayContent() == null) {
                   return WindowManagerGlobal.ADD_INVALID_DISPLAY;
               }
   						//part10根据窗口的type对窗口的LayoutParams成员变量进行修改
 	            mPolicy.adjustWindowParamsLw(win.mAttrs);
              win.setShowToOwnerOnlyLocked(mPolicy.checkShowToOwnerOnly(attrs));
   						//part11准备将窗口添加到系统中
              res = mPolicy.prepareAddWindowLw(win, attrs);
      		    win.attach();
              //part12将windowState添加到windowMap中
      		    mWindowMap.put(client.asBinder(), win);

							//part13将windowstate添加到该windowstate对应的windowtoken中，这样windowtoken中就包含了同一个组件的windowstate
              win.mToken.addWindow(win);

   
           return res;
       }
```

- part1处进行了权限检查，如果没有权限则不能添加系统窗口，需要在在manifest中注册权限。没有权限就不会执行后续代码，但是Tost这种系统窗口是用户可以创建的，所以这里需要对权限进行控制。
- part2处通过displayId来获得窗口要添加到哪个DisplayContent上。
- part3处判断了是否是子窗口，如果是子窗口，那么还要找出父窗口；而且父窗口本身不能是其他窗口的子窗口，否则会添加失败
- part4处创建windowToken，如果有父窗口则把父窗口的type赋值给rootType,如果没有则把自身的type赋值给rootType
- part5判断如果token为null，则隐式创建token,这说明创建窗口时可以不向WMS提供提供WindowToken。这里的参数false代表token是隐式创建的，不是传进来的
- part6如果token不为null且为应用程序窗口，将windowToken转换为应用程序窗口的AppWindowToken
-  part7创建windowState，它存有窗口的所有信息，代表一个窗口
- part8判断请求添加窗口的客户端是否已经死亡
- part9判断窗口的DisplayContent是否为null
- part10根据窗口的type对窗口的LayoutParams成员变量进行修改
- part11准备将窗口添加到系统中
- part12将windowState添加到windowMap中
- part13将windowstate添加到该windowstate对应的windowtoken中，这样windowtoken中就包含了同一个组件的windowstate

**总结：addWindow()主要做了一下事情**

- 对所要添加的窗口进行检查，如果窗口不满足一些条件，就不会执行下面的代码。
- WindowToken的相关处理，有的窗口类型需要提供windowToken，没有的话就不会执行下面的逻辑，有的则需要WMS隐式创建windowToken
- WindowState的创建和相关处理，将WindowToken和WindowState相关联
- 创建和配置DisplayContent，完成窗口添加到系统前到准备工作

### 4. Window的删除过程

窗口的删除从WindowManagerGlobal.removeVIew()发起，看代码：

```java
       public void removeView(View view, boolean immediate) {
           synchronized (mLock) {
             	 //获取要删除的view的索引
               int index = findViewLocked(view, true);
               //传入索引
               removeViewLocked(index, immediate);
      }
```

这里获取了要删除的View的索引，然后调用了removeViewLocked():

```java
       private void removeViewLocked(int index, boolean immediate) {
         	 //获取viewRoot
           ViewRootImpl root = mRoots.get(index);
           View view = root.getView();
           if (view != null) {
               //获取InputMethodManager实例
               InputMethodManager imm = InputMethodManager.getInstance();
               if (imm != null) {
                   //结束view的输入法相关逻辑
                   imm.windowDismissed(mViews.get(index).getWindowToken());
               }
           }
           boolean deferred = root.die(immediate);
       }
```

这里调用了ViewRootImpl.die():

```java
       boolean die(boolean immediate) {
         	 //immediate代表是否需要立即执行，及ViewRootImpl是否在执行performTraversals
           if (immediate && !mIsInTraversal) {
               doDie();
               return false;
            }
           mHandler.sendEmptyMessage(MSG_DIE);
           return true;
       }
```

这里判断了是否需要立即执行，以及ViewRootImpl是否在执行performTraversals，当ViewRootImpl在执行performTraversals时mIsInTraversal为被置为true,所以doDie()执行的条件是ViewRootImpl不执行performTraversals()时，再看doDie():

```java
         void doDie() {
         	 //检查线程，判断执行doDie()方法线程是否是创建view的原始线程，如果不是就抛出异常。只有创建view的原始线程才能操作view
           checkThread();
           synchronized (this) {
             		//是否有子view
               if (mAdded) {
  								 //如果有则销毁子view
                   dispatchDetachedFromWindow();
               }
               if (mAdded && !mFirst) {
  								//如果有子view并且不是第一次被添加
                   destroyHardwareRenderer();
               }
           }
	
 					
           WindowManagerGlobal.getInstance().doRemoveView(this);
       }
```

这里做了线程检查，只有创建view的原始线程才能操作view。然后判断是否有子 view有的话则调用dispatchDetachedFromWindow()去销毁，最后调用了WindowManagerGlobal.doRemoveView():

```java
       void doRemoveView(ViewRootImpl root) {
           synchronized (mLock) {
               final int index = mRoots.indexOf(root);
               if (index >= 0) {
                   //
                   mRoots.remove(index);
                   mParams.remove(index);
                   final View view = mViews.remove(index);
                   mDyingViews.remove(view);
               }
           }
       }
```

将要删除的view从WindowManagerGlobal中维护的列表中移除。

再回头去看dispatchDetachedFromWindow():

```java
       void dispatchDetachedFromWindow() {
               mWindowSession.remove(mWindow);
				}

```

这里调用了WindowSession.remove(),那么这里进行了进程间通讯，最终会调用到WMS的removeWindow():

```java
       void removeWindow(Session session, IWindow client) {
           synchronized(mWindowMap) {
             	 //获取WindowState
               WindowState win = windowForClientLocked(session, client, false);
               win.removeIfPossible();
           }
       }
```

这里线获取了WindowState,然后调用了WindowState.removeIfPossible():

```java
       void removeIfPossible() {
           super.removeIfPossible();
           removeIfPossible(false /*keepVisibleDeadWindow*/);
       }
```

继续看removeIfPossible(false):

```java
       private void removeIfPossible(boolean keepVisibleDeadWindow) {
         	//上面省略了一部分代码，主要是进行条件判断，如果满足任何一条则会return,推迟删除操作
           removeImmediately();
       }
```

在removeIfPossible()中会进行一系列判断，如果满足任何一个条件都会推迟操作，如view在进行动画时等。

再看removeImmediately():

```java
       void removeImmediately() {
           super.removeImmediately();
					 //如果当前要删除的是StatusBar或者NavigationBar，将这个window从对应的控制器中删除
           mPolicy.removeWindowLw(this);
   				 //将对应的session删除
           mSession.windowRemovedLocked();
           //调用WMS进行一些清理工作
           mService.postWindowRemoveCleanupLocked(this);
       }
```

最后通过removeImmediately()完成了窗口的删除。

**总结：window的删除可以总结为以下几点**

- 检查删除线程的正确性，如果不正确就抛出异常
- 从ViewRootImpl列表，布局列表和View列表中删除对应的元素
- 判断是否可以执行删除操作，如果不能就推迟删除操作
- 执行删除操作，清理和释放与V相关的一切资源
