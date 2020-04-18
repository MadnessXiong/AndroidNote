### 1. AMS功能概述

- 组件状态管理：包括四大组件的开启，关闭等一系列操作。如startActivity,startActivityAndWait,activityPaused,startService,stopService,removeContentProvider等
- 组件状态查询：查询组件当前运行等情况。如getCallingActivity,getService等
- Task相关：包括removeTask,removeSubTask,moveTaskBackwards,moveTaskToFront等

AMS是通过ActivityStack及其他数据结构来记录，管理系统中的Activity及其他组件状态的，并提供查询功能的一个系统服务。



### 2. AMS7.0和8.0的区别

- **AMS7.0:**

  ActivityManager(AM)是一个和AMS相关联的类，它主要对运行中的activity进行管理，这些管理工作并不是由ActivityManager来处理的，而是交由AMS来处理的。ActivityManager中的方法会通过ActivityManagerNative(AMN)的getDefault()来得到ActivityManageProxy(AMP)，通过AMP就可以和AMN进行通讯。而AMN是一个抽象类，它将功能交给它的子类AMS来处理，因此AMP就是AMS的代理类。AMS作为系统服务，很多服务是不会暴露给AM的。

  AMS的启动过程会调用Instrumentation.execStartActivities():

  ```java
         public void execStartActivities(Context who, IBinder contextThread,
                 IBinder token, Activity target, Intent[] intents, Bundle options) {
             execStartActivitiesAsUser(who, contextThread, token, target, intents, options,
                     UserHandle.myUserId());
         }
  ```

  这里调用execStartActivitiesAsUser():

  ```java
         public void execStartActivitiesAsUser() {
                 int result = ActivityManagerNative.getDefault()
                     .startActivities();
         }
      
  ```

  通过ActivityManagerNative.getDefault()获取AMP，那么看一下getDefault():

  ```java
         private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
             protected IActivityManager create() {
               	 //通过ServiceManager获取AMS代理对象
                 IBinder b = ServiceManager.getService("activity");
                 IActivityManager am = asInterface(b);
                 return am;
             }
         };
  ```

  这里首先通过ServiceManager获取了AMS的代理对象，然后调用了asInterface():

  ```java
         static public IActivityManager asInterface(IBinder obj) {
             return new ActivityManagerProxy(obj);
         }
  ```

  可以看到在asInterface里，构造了一个ActivityManagerProxy，并传入了AMS代理对象的引用，那么AMP就持有了具有进程间通讯能力的AMS代理对象，那么它也就具备了进程间通讯的能力。那么继续看它的startActivities():

  ```java
         public int startActivity() {
             mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
             reply.readException();
             return result;
         }
  ```

  这里调用了mRemote.transact()，这个mRemote就是传进来的AMS代理对象。调用它的方法实际就是进程间通讯，这时进程就会切换到AMS所在的SystemServer进程。mRemote.transact()经过一系列处理，最终回调到AMS到onTransact():

  ```java
         public boolean onTransact(){
             switch (code) {
             case START_ACTIVITY_TRANSACTION:
             {
               		//这里其实是在父类AMN中，AMS重写了并调用了onTransact()，最后调用AMS的startActivity()
                 int result = startActivity(app, callingPackage, intent, resolvedType,
                   resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
                 return true;
             }
           }
  ```

- **AMS8.0:直接采用了AIDL进行通讯**

  AMS8.0的启动过程同样是Instrumentation.execStartActivities():

  ```java
         public void execStartActivities() {
             execStartActivitiesAsUser();
         }
  ```

  再看execStartActivitiesAsUser():

  ```java
         public void execStartActivitiesAsUser() {
                 int result = ActivityManager.getService()
                     .startActivities();
         }
  ```

  可以看到，这里和7.0的区别是这里调用了ActivityManager.getService()

  ```java
         public static IActivityManager getService() {
             return IActivityManagerSingleton.get();
         }
         private static final Singleton<IActivityManager> IActivityManagerSingleton =
                 new Singleton<IActivityManager>() {
                     @Override
                     protected IActivityManager create() {
                         final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                       	//返回一个代理对象。
                         final IActivityManager am = IActivityManager.Stub.asInterface(b);
                         return am;
                     }
                 };
  ```

  可以看到，这里直接返回了一个AMS代理对象，没有了AMP。

### 3. AMS的启动过程

从[Android系统启动过程简述](https://github.com/MadnessXiong/AndroidNote/blob/master/Frameworks相关/Android系统启动过程简述.md)中可知，AMS在SystemServer的main()中启动。看代码：

```java
       public static void main(String[] args) {
           new SystemServer().run();
       }
```

再看run():

```java
       private void run() {
							 //创建消息Looper
               Looper.prepareMainLooper();
							 //加载动态库
               System.loadLibrary("android_servers");
     
               //创建系统Context
               createSystemContext();
   
               //创建SystemManager
               mSystemServiceManager = new SystemServiceManager(mSystemContext);
							 //启动引导服务
               startBootstrapServices();
         			 //启动核心服务
               startCoreServices();
         			 //启动其他服务
               startOtherServices();
         			 //开启消息循环
               Looper.loop();
           throw new RuntimeException("Main thread loop unexpectedly exited");
       }
```

可以看到，这里创建了SystemManager，然后在startBootstrapServices()中用SystemManager创建了AMS，看代码：

```java
       private void startBootstrapServices() {
         	 //创建AMS
           mActivityManagerService = mSystemServiceManager.startService(
                   ActivityManagerService.Lifecycle.class).getService();
}
```

可以看到这里调用了SystemManager.startService()，并传入了ActivityManagerService.Lifecycle来创建AMS，看代码：

```java
       public <T extends SystemService> T startService(Class<T> serviceClass) {
							  //通过反射的方式创建service，并调用了构造方法
               Constructor<T> constructor = serviceClass.getConstructor(Context.class);
               service = constructor.newInstance(mContext);
						   //调用startService
               startService(service);
               return service;
       }
```

这里首先通过反射的方式创建了service，并调用了它的构造，最后调用()。startService0这个serviceClass是传进来的ActivityManagerService.Lifecycle，看一下这个类：

```java
       public static final class Lifecycle extends SystemService {
           private final ActivityManagerService mService;
   				
           public Lifecycle(Context context) {
               super(context);
             	 //在构造里创建AMS
               mService = new ActivityManagerService(context);
           }
   
           @Override
           public void onStart() {
             	 //调用AMS的start()
               mService.start();
           }
   				 //返回AMS
           public ActivityManagerService getService() {
               return mService;
           }
       }
```

可以看到构造里构造了AMS。那么再看上一步的startService()

```java
       public void startService(@NonNull final SystemService service) {
           //将创建的service保存到ArrayList类型的mServices中，完成注册
           mServices.add(service);
         	 //调用ActivityManagerService.Lifecycle的onStart()
           service.onStart();
      }
```

这里传入的service就是AMS，将它保存到ArrayList类型的mServices中完成注册，最后调用了service.onStart()，从上一步可知，最后调用的是AMS的start()。同样从上一步可知getService()也返回的是构造的AMS。

那么到这里AMS就启动并且返回了。

### 4. AMS与应用程序进程

- 启动应用程序进程时AMS会通过进程名和uid查询这个进程是否存在
- 如果应用程序进程不存在，则AMS就会请求Zygote进程创建需要的应用程序进程

### 5. AMS中重要的数据结构

- ActivityRecord：记录了Activity的所有信息，因此它用来描述一个activity。它是在activity启动时被创建的，具体是在ActivityStarter的startActivity()中被创建的。它存储的信息主要包括以下内容：
  - service：AMS的引用
  - info:ActivityInfo，Activity中代码和AndroidManifest设置的节点信息，如launchMode
  - launcherFromPackage:启动activity的包名
  - taskAffinity:activity希望归属的栈
  - task:TaskRecord，Activity所在的TaskRecord
  - app:ProcessRecord,ActivityRecord所在的应用程序进程
  - state:ActivityState,当前activity的状态
  - icon:Activity的图标资源和标致符
  - theme:Activity的主题资源标识符
- TaskRecord：用来描述一个Activity任务栈
  - taskId:任务栈的唯一标识符
  - affinity:任务栈的倾向性
  - Intent:启动这个activity的intent
  - mActivites:ArrayList<ActivityRecord>,按照历史顺序排列的Activity记录
  - mStack:ActivityStack，当前归属的ActivityStack
  - mService:AMS的引用
- ActivityStack：用来管理系统所有的Activity，内部维护了Activity的所有状态，特殊状态的Activity以及和Activity相关的列表等数据。

### 6. Activity栈管理

Activity是放入Activity任务栈中的，有了任务栈，系统和开发者就能更好地应用和管理Activity，来完成各种业务逻辑

- Activity任务栈模型

  Activity任务栈由多种数据结构共同组合而成。其中ActivityRecord用来记录一个Activity的所有信息。一个TaskRecord包含了一个或多个ActivityRecord，TaskRecord用来表示Activity的任务栈。ActivityStack又包含了一个或者多个TaskRecord，它是TaskRecord的管理者。Activity栈管理是建立在Activity栈模型之上的，有了栈管理，就可以对应用程序进行操作，应用程序可以复用自身应用中以及其他应用的Activity，节省资源。

- Launch Mode:用于设置Activity的启动方式，无论是哪种启动方式，所启动的Activity都会位于Activity的栈顶，主要有4种模式：

  - standerd:默认模式，每次启动Activity都会创建一个新的Activity实例
  - singleTop:如果要启动的Activity已经在栈顶，则不会重新创建Activity，同时该Activity的onNewIntent()会被调用。如果要启动的Activity不在栈顶，则会重新创建该Activity实例
  - singleTask:如果要启动的Activity已经存在与它想要归属的栈中，那么不会创建该Activity的实例，会将栈中所有该Activity上的Activity出栈，同时该Activity的onNewIntent()被调用。如果要启动的Activity不存在于它想要归属的栈中，并且该栈存在，则会创建该Activity实例入栈。如果要启动的Activity想要归属的栈不存在，则新创建一个新栈，然后创建该Activity并入栈。
  - singleInstance:和singeTask基本类似，不同的是启动Activity时，首先要创建一个新栈，然后创建该Activity实例并入栈，新栈只会存在该Activity一个实例。