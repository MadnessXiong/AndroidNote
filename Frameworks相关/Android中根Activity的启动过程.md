Activity的启动过程分为2种，一种是根Activity的启动过程，另一种是普通Activity的启动过程。根Activity指的是应用程序启动的第一个Activity。因此根Activity的启动过程一般情况下，也可以理解为应用程序的启动过程。普通Activity指的是除应用程序启动的第一个Activity之外的其他Activity。

**1. 根Activity的启动过程**

根Activity的启动过程分为3个部分：

> 1. Launcher请求AMS过程
> 2. AMS到ApplicationThread的调用过程
> 3. ActivityThread启动Activity过程

- **Launcher请求AMS过程：**

  Launcher启动后，会将已安装应用程序的快捷图标显示到桌面上，这些应用程序的快捷图标就是启动根Activity的入口，当点击某个应用程序的快捷图标时，就会通过Launcher请求AMS来启动该应用程序，调用的第一个方法为Launcher的startActivitySafely(),看代码：

  ```java
         public boolean startActivitySafely(View v, Intent intent, ItemInfo item) {
             if (mIsSafeModeEnabled && !Utilities.isSystemApp(this, intent)) {
              //设置FLAG
             intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
              //调用Activity的startActivity()
             startActivity(intent, optsBundle);
             return false;
         }
  ```

  可以看到，先是设置了FLAG,因为这是第一个启动的Activity，所以要给它一个新的任务栈。

  然后调用了Activity的startActivity()，继续看代码：

  ```java
         @Override
         public void startActivity(Intent intent, @Nullable Bundle options) {
           	//参数-1代表Launcher不需要知道启动结果
             if (options != null) {
                 startActivityForResult(intent, -1, options);
             } else {
                 startActivityForResult(intent, -1);
             }
         }
  ```

  这里调用了startActivityForResult():

  ```java
         public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
                 @Nullable Bundle options) {
             if (mParent == null) {
                 Instrumentation.ActivityResult ar =
                  //调用Instrumentation的execStartActivity()
                     mInstrumentation.execStartActivity(
                         this, mMainThread.getApplicationThread(), mToken, this,
                         intent, requestCode, options);
             }
         }
  ```

  由于这是第一个Activity，所以父类mParent肯定是null,所以这里调用了Instrumentation.execStartActivity():

  ```java
         public ActivityResult execStartActivity(
             Context who, IBinder contextThread, IBinder token, String target,
             Intent intent, int requestCode, Bundle options) {
  							 //获取AMS代理对象
                 int result = ActivityManager.getService()
                     .startActivity(whoThread, who.getBasePackageName(), intent,
                             intent.resolveTypeIfNeeded(who.getContentResolver()),
                             token, target, requestCode, 0, null, options);
                 checkStartActivityResult(result, intent);
  
             return null;
         }
  ```

  首先获取了AMS的代理对象，然后调用了它的startActivity()。这里采用了AIDL的方式进行进程间通讯，关于AIDL参考：[Binder深入理解及与AIDL的使用和理解](https://github.com/MadnessXiong/AndroidNote/blob/master/Frameworks相关/Binder深入理解及与AIDL的使用和理解.md)

  **总结：可以看到，第一步主要是通知AMS，代码从这里就进入了AMS里。**

- **AMS到ApplicationThread的调用过程：**

  接上面，来到AMS的startActivity():

  ```java
         @Override
         public final int startActivity(IApplicationThread caller, String callingPackage,
                 Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                 int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
           		//调用startActivityAsUser，这里多传了一个getCallingUserId()
             return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                     resultWho, requestCode, startFlags, profilerInfo, bOptions,
                     UserHandle.getCallingUserId());
         }
  ```

  再看startActivityAsUser():

  ```java
         @Override
         public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
                 Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                 int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    				 //判断调用者进程是否被隔离
             enforceNotIsolatedCaller("startActivity");
    				 //根据传入的UserID来确定调用者权限
             userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
                     userId, false, ALLOW_FULL_ONLY, "startActivity", null);
             return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
                     resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                     profilerInfo, null, null, bOptions, false, userId, null, null,
                     "startActivityAsUser");
         }
  ```

  这里确定了调用者权限，然后调用了ActivityStarter.startActivityMayWait(),其中倒数第二个参数为TaskRecord，最后一个参数代表启动的理由，继续看代码：

  ```java
         final int startActivityMayWait() {
                 int res = startActivityLocked();
                 return res;
             }
         }
  ```

  ActivityStarter主要是对Intent和Flags进行处理，然后又调用了startActivityLocked():

  ```java
         int startActivityLocked() { 
             if (TextUtils.isEmpty(reason)) {
                 throw new IllegalArgumentException("Need to specify a reason.");
             }
     
             mLastStartActivityResult = startActivity();
             return mLastStartActivityResult;
         }
  ```

  这里先判断了启动理由，如果不为空，则继续调用ActivityStarter的startActivity():

  ```java
         private int startActivity() {
  					//ProcessRecord用于描述一个应用程序进程
        	 ProcessRecord callerApp = null;
                    	//判断caller是否为null，这个caller实际是ApplicationThread对象，是从startActivityForResult()里传递过来的
             if (caller != null) {
               	//调用AMS的getRecordForAppLocked(),获取Launcher进程
                 callerApp = mService.getRecordForAppLocked(caller);
                 if (callerApp != null) {
                   //获取Launcher的pid和uid
                     callingPid = callerApp.pid;
                     callingUid = callerApp.info.uid;
                 } 
             }
  					//创建即将要启动的activity的描述类ActivityRecord，ActivityRecord记录了一个activity的所有信息
             ActivityRecord r = new ActivityRecord());
             if (outActivity != null) {
                 outActivity[0] = r;
             }
     		 return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,true,
                     options, inTask, outActivity);
         }
  ```

  可以看到，这里通过AMS获取了Launcher进程，同时创建了ActivityRecord,然后将ActivityRecord赋值给了outActivity，然后将outActivity作为参数，继续调用startActivity():

  ```java
         private int startActivity() {
                 result = startActivityUnchecked()
             return result;
         }
  ```

  这里又调用了startActivityUnchecked():

  ```java
         private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
                 IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                 int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                 ActivityRecord[] outActivity) {
  							//创建新的TaskRecord
                 result = setTaskFromReuseOrCreateNewTask(
                         taskToAffiliate, preferredLaunchStackId, topStack);
  								
                     mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                             mOptions);
             return START_SUCCESS;
         }
  ```

  前面说过，这时第一个Activity，所以要创建一个新的栈，所以这里创建了新的TaskRecord，然后进入了ActivityStackSupervisor.resumeFocusedStackTopActivityLocked():

  ```java
         boolean resumeFocusedStackTopActivityLocked(
                 ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
  							//获取要启动的Activity所在的栈的栈顶不是处于停止状态的ActivityRecord
             final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
             if (r == null || r.state != RESUMED) {
                 mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
             } 
             return false;
         }
  ```

  这里会调用ctivityStack.resumeTopActivityUncheckedLocked():

  ```java
         boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
               result = resumeTopActivityInnerLocked(prev, options);
             return result;
         }
  ```

  调用了resumeTopActivityInnerLocked():

  ```java
         private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
  				 mStackSupervisor.startSpecificActivityLocked(next, true, false);
           return true;
         }
  ```

  这里调用了ActivityStackSupervisor.startSpecificActivityLocked():

  ```java
         void startSpecificActivityLocked(ActivityRecord r,
                 boolean andResume, boolean checkConfig) {
  						//获取即将启动的Activity的所在的应用程序进程
             ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                     r.info.applicationInfo.uid, true);
     				  //判断应用程序进程是否已经运行
             if (app != null && app.thread != null) {
              //应用程序进程已经运行		
              realStartActivityLocked(r, app, andResume, checkConfig);
  
             }
         }
  ```

  这里可以看到，先去获取了应用程序进程ProcessRecord，如果不为null，则代表应用程序进程已经启动，这里是分析Activity的启动过程，默认应用程序进程已经启动，所以会来到realStartActivityLocked():

  ```java
         final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
                 boolean andResume, boolean checkConfig) throws RemoteException {
  							//调用ApplicationThread的scheduleLaunchActivity启动Activity
                 app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                         System.identityHashCode(r), r.info,
                         mergedConfiguration.getGlobalConfiguration(),
                         mergedConfiguration.getOverrideConfiguration(), r.compat,
                         r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                         r.persistentState, results, newIntents, !andResume,
                         mService.isNextTransitionForward(), profilerInfo);
     
             return true;
         }
  ```

  这里的app是要启动的activity所在应用程序的进程，app.thread其实是ApplicationThread，它是ActivityThread的内部类，它继承了IApplication.Stub。那么这个ApplicationThread就具备了进程间通讯的能力。参考[Binder深入理解及与AIDL的使用和理解](https://github.com/MadnessXiong/AndroidNote/blob/master/Frameworks相关/Binder深入理解及与AIDL的使用和理解.md)，可知这个app.thread其实就是ApplicationThread的代理对象，负责AMS进程(SystemServer进程)与应用程序进程间的通讯。

  **总结：可以看到，第二步主要是对要启动的Activity的进程一些权限检查，创建ActivityRecord，它里面记录了Activity所有信息以及创建堆栈信息等操作，最后判断应用程序进程是否已经运行，如果没运行则去创建应用程序进程，如果已运行则通过Binder代理对象，通知ApplicationThread，执行它的scheduleLaunchActivity()**

- **ActivityThread启动Activity过程：**

  接上面，代码来到了ApplicationThread.scheduleLaunchActivity():

  ```java
             public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                     ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                     CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                     int procState, Bundle state, PersistableBundle persistentState,
                     List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                     boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
   							//创建一个ActivityClientRecord，用于封装启动Activity的参数
                 ActivityClientRecord r = new ActivityClientRecord();
     
                 r.token = token;
                 r.ident = ident;
                 r.intent = intent;
                 r.referrer = referrer;
                 r.voiceInteractor = voiceInteractor;
                 r.activityInfo = info;
                 r.compatInfo = compatInfo;
                 r.state = state;
                 r.persistentState = persistentState;
                 r.pendingResults = pendingResults;
                 r.pendingIntents = pendingNewIntents;  
                 r.startsNotResumed = notResumed;
                 r.isForward = isForward; 
                 r.profilerInfo = profilerInfo;  
                 r.overrideConfig = overrideConfig;
                 updatePendingConfiguration(curConfig);  
               	//发送消息
                 sendMessage(H.LAUNCH_ACTIVITY, r);
             }
  ```

  这里将Activity启动的参数封装成了一个ActivityClientRecord，然后通过sendMessage()，发送了出去，再看代码：

  ```java
         private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
             if (DEBUG_MESSAGES) Slog.v(
                 TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
                 + ": " + arg1 + " / " + obj);
             Message msg = Message.obtain();
             msg.what = what;
             msg.obj = obj;
             msg.arg1 = arg1;
             msg.arg2 = arg2;
             if (async) {
                 msg.setAsynchronous(true);
   }
           	//发送消息
             mH.sendMessage(msg);
         }
  ```

  这里调用mH.sendMessage(msg)发送消息，这个H是ActivityThread的内部类并继承自Handler，是应用程序进程中主线程的消息管理类，因为ApplicationThread是一个Binder，它的调用逻辑运行在Binder线程池中，所以这里需要用Handler将代码逻辑切换到主线程中。

  > 在[Android应用程序进程启动过程中](https://github.com/MadnessXiong/AndroidNote/blob/master/Frameworks相关/Android应用程序进程启动过程.md)，最后建立了一个消息循环。就是这个Handler。

  Handler处理消息会来到它的HandleMessage(),看代码：

  ```java
             public void handleMessage(Message msg) {
                 if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
                 switch (msg.what) {
                     case LAUNCH_ACTIVITY: {
                         final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
     										//获取LoadedAPK类型对象，并存入r.packageInfo中。
                         r.packageInfo = getPackageInfoNoCheck(
                                 r.activityInfo.applicationInfo, r.compatInfo);
                         handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                     } break;
             }
  ```

  这里先获得了LoadedAPK，然后存入ActivityClientRecord.packageInfo中。

  应用程序进程要启动Activity时需要将该Activity所属的APK加载进来，LoadedAPK就是描述已加载的APK文件的。

  最后调用handleLaunchActivity(),看代码:

  ```java
         private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
  						//启动Activity
             Activity a = performLaunchActivity(r, customIntent);
           	  //将Activity的状态设置为resume
     				 handleResumeActivity(r.token, false, r.isForward,
                         !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
  
         }
  ```

  可以看到，这里调用了performLaunchActivity()启动Activity，再看代码：

  ```java
         private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
               	//这获取ActivityInfo,用于存储代码以及AndroidManifest设置的activity和receiver节点信息，如activity的theme和launchMode。
             ActivityInfo aInfo = r.activityInfo;
           		//获取描述文件LoadAPK
             if (r.packageInfo == null) {
                 r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                         Context.CONTEXT_INCLUDE_CODE);
             }
           	 //获取要启动的Activity的ComponentName，它保存了该Activity的包名和类名
          	 ComponentName component = r.intent.getComponent();
     				 //创建要启动activity的上下文环境
             ContextImpl appContext = createBaseContextForActivity(r);
             Activity activity = null;
    						
                 java.lang.ClassLoader cl = appContext.getClassLoader();
           			//在newActivity()中用类加载器创建activity的实例，
                 activity = mInstrumentation.newActivity(
                         cl, component.getClassName(), r.intent);
  							//创建Application
                 Application app = r.packageInfo.makeApplication(false, mInstrumentation);
  							//初始化activity
                     activity.attach(appContext, this, getInstrumentation(), r.token,
                             r.ident, app, r.intent, r.activityInfo, title, r.parent,
                             r.embeddedID, r.lastNonConfigurationInstances, config,
                             r.referrer, r.voiceInteractor, window, r.configCallback);
  						 //调用onCreate()
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
  
             return activity;
         }
  ```

  可以看到，这里首先获取了各种信息，然后在Instrumentation.newActivity()里通过classLoader创建了Activity实例，然后又创建了Application，然后调用activity.attach(),初始化activity，最后调用Instrumentation.callActivityOnCreate()去调用Activity的onCreate,这里先看attach()：

  ```java
         final void attach() {、
           	 //初始化PhoneWindow
             mWindow = new PhoneWindow(this, window, activityConfigCallback);
             //将PhoneWindow和当前activity关联
             mWindow.setCallback(this);
    				 //初始化UI线程
             mUiThread = Thread.currentThread();
             //初始化主线程
             mMainThread = aThread;
  					//设置WindowManager
             mWindow.setWindowManager(
                     (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                     mToken, mComponent.flattenToString(),
                     (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
  
         }
  ```

  可以看到在attach()里初始化了PhoneWindow，并和activity关联了起来，然后初始化了Ui线程和主线程，这2个线程在这里基本是一回事，除非执行attach()时当前线程不是主线程。最后设置了WindowManager。

  然后再来看Instrumentation.callActivityOnCreate()：

  ```java
         public void callActivityOnCreate(Activity activity, Bundle icicle) {
             activity.performCreate(icicle);
         }
  ```

  可以看到，这里调用了Activity的performCreate()，继续看代码:

  ```java
         final void performCreate(Bundle icicle) {
             onCreate(icicle);
         }
  ```

  可以看到，最终调用了Activity的Create()。到这里，根Activity就被启动了。

**2. 根Activity启动过程中涉及到的进程**

Launcher进程首先向AMS请求创建根Activity，AMS会判断根Activity所需要的应用程序进程是否存在并启动，如果不存在就会请求Zygote进程创建应用程序进程(这一步是通过Socket通信)。如果存在或者应用程序进程启动后，AMS会请求创建根Activity。

根Activity启动过程中涉及以下几个进程：Launcher进程，AMS所在的SystemServer进程，Zygote进程，应用程序进程

普通Activity启动过程中设计2个进程：AMS所在的SystemServer进程，应用程序进程。


