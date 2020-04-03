### 简介：

> Android中四大组件都会涉及到Context。Context意为上下文，是一个应用程序环境信息的接口。Context是一个抽象类，它的内部定义了很多方法及静态常量。
>
> 它的使用场景分为2类：
>
> - 使用Context调用方法，如启动Activity，访问资源，调用系统服务等。
> - 调用方法时传入Context，比如弹出Toast，创建Dialog等

### Context的关联类：

Context关联类的关系如下图：
![image-20200403192950842.png](Frameworks相关/Android中Context理解及创建过程.md)

从上图可知，一个应用程序进程的Context数量等于Activity和Service的总个数加一个Application。

ContexImpl和ContextWapper都继承自Context。其中ContextWapper内部包含了一个ContextImpl类型的mBase。ContextImpl提供了很多功能，但是外界需要使用并拓展ContextImpl的功能，因此设计上使用了装饰模式，ContextW apper是装饰类，它对ContextImpl进行包装。ContextWapper主要起了方法传递的作用，ContextWapper中几乎所有的方法都是调用ContextImpl的相应方法来实现的。

ContextThemeWapper，Service和Application都继承自ContextWapper。这样它们都可以使用mBase使用Context的方法。同时它们也是装饰类，在ContextWapper的基础上又增加了不同的功能。ContextThemeWapper中包含和主题相关的方法(如getTheme()),因此需要主题的Activity继承自ContextThemeWapper，而不需要主题的Service和Application都继承自ContextWapper。

Context的关联类采用装饰模式主要有以下优点：

- 使用者能够方便地使用Context
- 如果ContextImpl发生了变化，它的装饰类ContextWapper不需要做任何修改
- ContextImpl的实现不会暴露给使用者，使用者也不必关心ContextImpl的实现
- 通过组合而非继承的方式，拓展ContextImpl的功能，在运行时选择不同的装饰类，实现不同的功能

### Context的创建过程

- **Application Context的创建过程：**

  在一个应用程序启动完成后，应用程序就可以通过getApplicationContext()获取全局的Application Context。应用程序的启动过程也是根Activity的启动过程，最后会调用performLaunchActivity()，看代码：

  ```java
       private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
  					 //创建Application，这里的r.packageInfo类型为LoadApk
             Application app = r.packageInfo.makeApplication(false, mInstrumentation);
             return activity;
         }
  ```

  可以看到，调用了LoadApk.makeApplication()创建了Application对象，那么看下代码:

  ```java
         public Application makeApplication(boolean forceDefaultAppClass,
                 Instrumentation instrumentation) {
           			 //创建ContextImpl
                 ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
                 //创建Application，并传入ContextImpl类型的appContext
           			 app = mActivityThread.mInstrumentation.newApplication(
                         cl, appClass, appContext);
           			 //将appLication赋值给ContextImpl
                 appContext.setOuterContext(app); 
           		   //将appLication赋值给LoadApk的成员变量mApplication
                 mApplication = app;
               return app;
         }
  ```

  可以看到，这里首先创建了ContextImpl对象，然后创建了Application对象。再将appLication赋值给ContextImpl。

  这样ContextImpl就持有了Application的引用。最后将appLication赋值给LoadApk的成员变量mApplication,那么LoadApk就持有了Application Context。

  这里主要关注Application的创建，所以看一下Instrumentation.newApplication():

  ```java
         static public Application newApplication(Class<?> clazz, Context context)
                 throws InstantiationException, IllegalAccessException,
                 ClassNotFoundException {
             //创建Application
             Application app = (Application)clazz.newInstance();
             //将Application和ContextImpl关联
             app.attach(context);
             return app;
         }
  ```

  这里可以看到，通过反射创建了Application，然后调用了Application.attach(),再看代码：

  ```java
        	final void attach(Context context) {
            //调用父类ContextWapper的attachBaseContext，传入ContextImpl
             attachBaseContext(context);
         }
  ```

  这里attachBaseContext()是在父类ContextWapper里实现的，看代码：

  ```java
         protected void attachBaseContext(Context base) {
             mBase = base;
         }
  ```

  到这里就可以看到，将创建的ContextImpl对象赋值给了ContextWapper的mBase。这样ContextWapper以及它的子类就可以通过mBase调用ContextImpl的方法了。

- **Application Context的获取过程**

  获取Application Context的获取是通过ContextWapper.getApplicationContext()获取的，那么看代码：

  ```java
         @Override
         public Context getApplicationContext() {
             return mBase.getApplicationContext();
         }
  ```

  上面已经分析出了mBase是ContextImpl，那么继续看它的getApplicationContext()：

  ```java
         public Context getApplicationContext() {
             return (mPackageInfo != null) ?
               			//调用loadApk的getApplication()
                     mPackageInfo.getApplication() : mMainThread.getApplication();
         }
  ```

  这里调用了loadApk.getApplication()，看代码：

  ```java
         Application getApplication() {
             return mApplication;
         }
  ```

  前面分析过创建Application后就直接赋值给了LoadApk的成员变量mApplication，所以这里直接返回。

  **总结：Application Context在应用程序启动后，创建第一个Activity时创建，创建后赋值给了LoadApk时的成员变量mApplication，同时也创建了ContextImpl,并且将ContextImpl对象赋值给了自身的mBase。在获取Application Context时也是通过获取调用mBase.getApplicationContext()，里面又调用了LoadAPk的getApplication()，获取了之前保存的mApplication。**

- **Activity Context的创建过程：**

  Activity的Context再Activity的创建过程中被创建，那么还是看performLaunchActivity():

  ```java
         private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
  					 //创建ContextImpl
             ContextImpl appContext = createBaseContextForActivity(r);
  					 //创建activity
             activity = mInstrumentation.newActivity(
                         cl, component.getClassName(), r.intent);
  					 //将activity和ContextImpl关联
             activity.attach(appContext, this, getInstrumentation(), r.token,
                             r.ident, app, r.intent, r.activityInfo, title, r.parent,
                             r.embeddedID, r.lastNonConfigurationInstances, config,
                             r.referrer, r.voiceInteractor, window, r.configCallback);
             return activity;
         }
  ```

  可以看到，先创建了ContextImpl，然后创建了Activity，最后通过attach()关联，看一下attach():

  ```java
         final void attach() {
           	 //关联ContextImpl
             attachBaseContext(context);
  
         }
  ```

  attachBaseContext()过程和Application里一样，最后还是将ContextImpl赋值给mBase。这样Activity就可以使用ContextImpl的方法了。

- **Service Context的创建过程:**

  Service的创建过程也是一样，在创建后调用了attachBaseContext(),将ContextImpl赋值给mBase。这里不再展开。



