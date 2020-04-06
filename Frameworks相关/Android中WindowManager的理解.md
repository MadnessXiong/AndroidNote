### WindowManager简介

WindowManager(WM)是一个接口类，继承自接口ViewManager，它是用来管理Window的。它的实现类为WindowManagerImpl。对窗口的添加，更新和删除操作，由WM负责。它会将具体的工作交由WMS来处理，WM和WMS通过Binder来进行进程间通信，WMS作为系统服务有很多API是不会暴露给WM的

### WindowManager的关联类

- WindowManager继承自ViewManager，ViewManager中定义了3个方法，分别用来添加，更新和删除View

  ```java
  public interface ViewManager{
    	//添加view
      public void addView(View view, ViewGroup.LayoutParams params);
    	//更新view
      public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    	//删除view
      public void removeView(View view);
  }
  ```

  WindowManager在继承了ViewManager的同时，又加入了很多功能，包括Window的类型和层级相关的常量，内部类以及一些方法，其中有2个方法是根据Window的特性加入的：

  ```java
   		//这个方法能够得知这个windowManager实例将window添加到哪个屏幕上了，也就是得到WindowManager所管理的屏幕
  		public Display getDefaultDisplay();
  		//规定在这个方法返回前要立即执行View.onDetachedFromWindow()，来完成传入的View的相关的销毁工作
        public void removeViewImmediate(View view);
  ```

- Window是一个抽象类，它的具体实现类为PhoneWindow，PhoneWindow在Activity启动时最终会在它的attach()中被创建。

  ```java
      final void attach() {
        	//创建PhoneWindow
          mWindow = new PhoneWindow(this, window, activityConfigCallback);
  				//设置windowManager
          mWindow.setWindowManager(
                  (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                  mToken, mComponent.flattenToString(),
                  (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
  			  //对持有的WindowManager引用赋值
          mWindowManager = mWindow.getWindowManager();
      }
  ```

  可以看到，这首先构造了PhoneWidow，然后设置了WindowManager，WindowManager通过getSystemService()方式获得，看setWindowManager()的代码：

  ```java
         public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
                 boolean hardwareAccelerated) {
             if (wm == null) {
                 wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
             }
             mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
         }
  ```

  如果传入的WM为null，那么就通过ContextImpl.getSystemService()再获取一次，那么看一下获取的方法：

  ```java
         @Override
         public Object getSystemService(String name) {
             return SystemServiceRegistry.getSystemService(this, name);
         }
  ```

  接着调用了SystemServiceRegistry.getSystemService(),继续看：

  ```java
         public static Object getSystemService(ContextImpl ctx, String name) {
             ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
             return fetcher != null ? fetcher.getService(ctx) : null;
         }
  ```

  这里从SYSTEM_SERVICE_FETCHERS取出一个ServiceFetcher，最终取出需要的Service，那么看一下这个SYSTEM_SERVICE_FETCHERS：

  ```java
        //将Service注册
  			private static <T> void registerService(String serviceName, Class<T> serviceClass,
                 ServiceFetcher<T> serviceFetcher) {
             SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
             SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
         }
  			 //在静态代码块中注册具体的Service
  			registerService(Context.WINDOW_SERVICE, WindowManager.class,
                     new CachedServiceFetcher<WindowManager>() {
                 @Override
                 public WindowManager createService(ContextImpl ctx) {
                     return new WindowManagerImpl(ctx);
                 }});
  ```

  SystemServiceRegistry的静态代码块中通过registerService()注册了各种Service，这里可以看到Context.WINDOW_SERVICE对应的是WindowManagerImpl，那么最终getSystemService()得到的就是WindowManagerImpl对象。

  继续看setWindowManager():

  ```java
         public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
                 boolean hardwareAccelerated) {
             if (wm == null) {
                 wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
             }
             mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
         }
  ```

  前面的分析已经知道wm实际是WindowManagerImpl，这里又调用了createLocalWindowManager(),那么看代码：

  ```java
         public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
             return new WindowManagerImpl(mContext, parentWindow);
         }
  
  ```

  可以看到，这里同样也是创建了WindowManagerImpl，不过这次将window传了进去，这样WindowManagerImpl就持有了window的引用。

  WindowManagerImpl持有了window的引用以后，就可以对window进行操作，如addVIew(),看代码：

  ```java
      public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
          applyDefaultToken(params);
        	//mGlobal是WindowManagerGlobal类型
          mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
      }
  
  ```

  这里可以看到，addView()的功能又转交给了WindowManagerGlobal

  再看WindowManagerGlobal：

  ```java
  public final class WindowManagerGlobal {
  
      private static IWindowManager sWindowManagerService;
    	//负责与WMS进行通信
      private static IWindowSession sWindowSession;
  
  		//mView记录了所有View树的根节点(每个布局最外层的view)
      private final ArrayList<View> mViews = new ArrayList<View>();
    	//mRoots记录了所有Activity的ViewRootImpl
      private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
      //mParams记录了所有window的属性
      private final ArrayList<WindowManager.LayoutParams> mParams =
              new ArrayList<WindowManager.LayoutParams>();
      private final ArraySet<View> mDyingViews = new ArraySet<View>();
  
  		//单例对象
      public static WindowManagerGlobal getInstance() {
          synchronized (WindowManagerGlobal.class) {
              if (sDefaultWindowManager == null) {
                  sDefaultWindowManager = new WindowManagerGlobal();
              }
              return sDefaultWindowManager;
          }
      }
  
  ```

  可以看到WindowManagerGlobal中定义了sWindowSession，负责与WMS的通信，mViews记录了所有view树的根节点，mRoots记录了所有Activity的ViewRootlmpl,mParams记录了所有window的属性。

  这里的WindowManagerGlobal是一个单例对象，一个进程只有一个。而WindowManagerImpl从它的创建来看，是有多个实例的

  **总结：PhoneWindow继承自WIndow，Window通过setWindowManager()与WindowManager产生关联。WindowManager继承自接口ViewManager，WindowManagerImpl是WIndowManager接口的实现类，但是具体功能会委托给WindowManagerGlobal来实现。**

### Window的属性

Window为了更好地与WMS通讯，它们之间制定了一些协议，这个协议就是Window的属性。这些属性定义在WindowManager的LayoutParams中。Window的属性有很多种，与开发关系密切的有3种，它们分别是Type(window的类型)，Flag(Window的标志)和SoftInputMode(软键盘相关模式)

- Window的类型和显示次序

  - Window的类型分为三大类型：Application Window(应用程序窗口)，Sub Window(子窗口)，System Window(系统窗口)。每个大类型中又包含了很多种类型，它们都定义在了WindowManager的静态内部类LayoutParams中
    - 应用程序窗口：应用程序窗口的Type值的范围为1-99
    - 子窗口：它不能独立存在，需要附着在其他窗口上才可以。如PopupWindow。它的Type值范围为1000-1999
    - 系统窗口：Toast，输入法窗口，系统音量条窗口，系统错误窗口都属于系统窗口。系统窗口的Type值范围为2000-2999

  - 窗口的显示次序：当一个进程向WMS申请一个窗口时，WMS会为窗口确定显示次序。为了方便窗口显示次序管理，手机屏幕可以虚拟地用X，Y，Z轴来表示，其中Z轴垂直与屏幕，从屏幕内向屏幕外，这样确定窗口显示次序也就是确定窗口在Z轴上的次序，这个次序称为Z_Oder。Type值是Z_oder排序的依据，Type值越大则Z-Oder排序越靠前，就越靠近用户。

- Window的标志

  window的标志也就是flag，用于控制window的显示，同样被定义在windowmanager的内部类LayoutParams中。

  设置Window的flag有3种方法：

  ```java
  //第一种，内部也是调用setFlags()
  mWindow.addFlags()
  //第二种
  mWindow.setFlags()
  //第三种
  WindowManager.LayoutParams layoutParams = new WindowManager.LayoutParams();
  layoutParams.flags=WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON;
  TextView textView = new TextView(this);
  windowManager.addView(textView,layoutParams);
  
  ```

### Window的操作

对Window的操作，最终都会交由WMS来进行处理。窗口的处理分为2部分，一部分是WindowManager处理部分，一部分是WMS处理部分。对于不同的窗口类型(应用程序窗口，子窗口，系统窗口)添加过程有所不同，但是对于WMS处理的过程，基本上是一样的。WMS对这三大窗口基本是一视同仁的。

- 系统窗口的添加过程

  这里看一下StatusBar的添加过程

  ```java
         private void addStatusBarWindow() {
             makeStatusBarView();
             mStatusBarWindowManager = Dependency.get(StatusBarWindowManager.class);
             mRemoteInputController = new RemoteInputController(mHeadsUpManager);
           
             mStatusBarWindowManager.add(mStatusBarWindow, getStatusBarHeight());
         }
  
  ```

  继续再看这个add():

  ```java
         public void add(View statusBarView, int barHeight) {
  					 //通过WindowManager.LayoutParams配置属性
             mLp = new WindowManager.LayoutParams(
                     ViewGroup.LayoutParams.MATCH_PARENT,
                     barHeight,
                     WindowManager.LayoutParams.TYPE_STATUS_BAR,
                     WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                             | WindowManager.LayoutParams.FLAG_TOUCHABLE_WHEN_WAKING
                             | WindowManager.LayoutParams.FLAG_SPLIT_TOUCH
                             | WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH
                             | WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS,
                     PixelFormat.TRANSLUCENT);
             //调用windowManager的addView()
             mWindowManager.addView(mStatusBarView, mLp);
             mLpChanged = new WindowManager.LayoutParams();
         }
  
  ```

  这里调用了mWindowManager.addView()添加status，前面说过，具体是由WindowMangerImpl实现的，那么看一下它的addView():

  ```java
      public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
          applyDefaultToken(params);
          mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
      }
  
  ```

  这里WindowManagerGlobal.addView()，看代码：

  ```java
         public void addView(View view, ViewGroup.LayoutParams params,
                 Display display, Window parentWindow) {
  
             ViewRootImpl root;
             View panelParentView = null;
             //创建ViewRootImpl
             root = new ViewRootImpl(view.getContext(), display);
     
             view.setLayoutParams(wparams);
     				 //添加view
             mViews.add(view);
             //添加root
             mRoots.add(root);
             //添加窗口参数
             mParams.add(wparams);
             //将窗口及窗口参数设置进ViewRootImpl中
             root.setView(view, wparams, panelParentView);
  
         }
  
  ```

  这里可以看到添加窗口的操作是通过ViewRootImpl进行的，看一下它的setView():

  ```java
         public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
           		//进行进程间通信
  						res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                    getHostVisibility(), mDisplay.getDisplayId(),
                    mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                     mAttachInfo.mOutsets, mInputChannel);
         }
  
  ```

  这里最终调用了mWindowSession.addToDisplay()进行进程间通信，最终会调用到WMS到addView(),并将自身也就是Session传进去。每个应用程序都会对应一个Session，WMS会用ArrayList来保存这些Session。剩下的工作就会交由WMS处理，在WMS中会为这个添加的窗口分配Surface，并确定窗口显示次序。

- Activity的添加过程

  无论哪种窗口，它的添加过程在WMS处理部分中基本是类似的，只不过会在权限和窗口显示次序等方面会又些不同。但是在WindowManager处理部分会有所不同。

  当Activity启动时会调用ActivityThread的handleResumeActivity():

  ```java
      @Override
      public void handleResumeActivity() {
  
  				//这里会调用到Activity的attach()，会初始化WindowManager
          final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
  						//获取初始化好的WindowManager
              ViewManager wm = a.getWindowManager();
        			//wm添加decorView
              wm.addView(decor, l);
      }
  
  ```

  可以看到，最后这里将decorView添加到了Activity里，之后的流程和status一样。

- Window到更新过程

  Window的更新方法是调用upDateViewLayout(),同样是看WindowManagerGlobal.upDateViewLayout(),看代码：

  ```java
         public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
  
             final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
     				 //将更新的参数设置进view中
             view.setLayoutParams(wparams);
     
             synchronized (mLock) {
                	//找到要更新的窗口的索引
                 int index = findViewLocked(view, true);
                 //根据所有的到它的ViewRootImpl
                 ViewRootImpl root = mRoots.get(index);
               	 //更新布局列表参数
                 mParams.remove(index);
                 mParams.add(index, wparams);
                 //将更新好的参数设置到ViewRootImpl中
                 root.setLayoutParams(wparams, false);
             }
         }
  
  ```

  可以看到，这里首先找到要更新的窗口的索引，然后根据索引找到它的ViewRootImpl，最后更新参数后通过ViewRootImpl.setLayoutParams()进行设置，看代码：

  ```java
         void setLayoutParams() {
                 scheduleTraversals();
         }
  
  ```

  这里调用了scheduleTraversals(),看代码：

  ```java
         void scheduleTraversals() {
         	//发起一个回调
          mChoreographer.postCallback(
                         Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
  
         }
  
  ```

  这里通过Choreographer发起一个回调，这个回调在下一帧被渲染时执行它的run(),那么再看它的run():

  ```java
         final class TraversalRunnable implements Runnable {
             @Override
             public void run() {
                 doTraversal();
             }
         }
  ```

  这里执行了doTraversal(),继续看：

  ```java
         void doTraversal() {
          //开始绘制流程
          performTraversals();
        }
  ```

  这里可以看到，调用performTraversals(),这个方法最终会调用performMeasure(),performLayout(),performDraw()完成整个绘制流程。