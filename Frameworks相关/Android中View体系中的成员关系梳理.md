- Activity：用于UI界面的显示和处理

- Window：

  - Window的意思是窗口。Window是一个抽象的基类，根据不同的产品可以生成不同的窗口。在Activity中默认实现也是唯一实现是PhoneWindow。

    Window是面向Activity的，表示UI界面的框架，而框架里的内容是由实现类PhoneWindow来规划的。但无论最终生成的窗口怎样，Activity是不需要修改的。

  - Window的另外一层含义是与WMS进行通信，但它并没有直接在自身实现。原因是，一个程序中可能存在多个window，如果它们都单独与WMS进行通信，那么既浪费资源又会造成管理混乱。那么它们需要统一的进行管理，所以就有了WindowManager。

- WindowManager：WindowManager是一个接口类，它的实现是WindowManagerImpl,WindowManagerImpl也是整个应用程序中所有window的实际管理者,一个Activity对应一个WindowManagerImpl

- ViewRootImpl：

  - ViewRootImpl是View树的管理者,它实现了接口ViewParent，它既非View的子类，也非View的父类。它有一个成员变量mView，指向它所管理的View树的根，既布局中最外层的View
  - ViewRootImpl的核心任务是与WMS进行通信,ViewRootImpl内部持有一个IWindowSession对象，它是由WindowManagerGlobal.getWindowSession()获得的，负责与WMS之间的通信，一个进程只有一个。
  - ViewRootImpl还负责触发View的测量，布局和绘制。以及输入事件的中转，管理Surface。

  

- WindowManagerGlobal：是View的全局管理者，内部的mRoots记录了各Activity的ViewRootImpl，mViews记录了View树的根节点，mParams记录了window的属性。WindowManagerGlobal对象是一个单例，一个进程只有一个。

- DecorView：Decor的原义是装饰。它包含了Activity中实际想要显示的内容。产生DecorView的过程是由PhoneWindow中的setContentView发起的。setContentView()同时还会生成一个变量mContentParent,用于容纳ContentView。这个ContentView就是用户要添加的内容，和系统的状态栏等共同组成了DecorView。

