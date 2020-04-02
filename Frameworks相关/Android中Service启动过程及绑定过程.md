先了解以下2篇，可以对Service的启动过程有更好的理解
[Android应用程序进程启动过程](https://github.com/MadnessXiong/AndroidNote/blob/master/Frameworks相关/Android应用程序进程启动过程.md)
[Binder深入理解及与AIDL的使用和理解](https://github.com/MadnessXiong/AndroidNote/blob/master/Frameworks相关/Binder深入理解及与AIDL的使用和理解.md)
### Service启动过程

**1. 先看一下startService()的使用示例**

- 首先创建一个Service：

  ```java
  public class ClientService extends Service {
  		//startService()方式启动Service不会回调此方法
      @Override
      public IBinder onBind(Intent intent) {
          return  null;
      }
  		//当Service被创建时调用
      @Override
      public void onCreate() {
          super.onCreate();
      }
  		//当Service运行时调用
      @Override
      public int onStartCommand(Intent intent, int flags, int startId) {
          return super.onStartCommand(intent, flags, startId);
      }
  		//当Service被销毁时调用
      @Override
      public void onDestroy() {
          super.onDestroy();
      }
  }
  ```

- 启动Service:

  ```java
          Intent intent = new Intent(this, ClientService.class);
          startService(intent);
  ```

  以上就是startService()方式启动Service的示例

> 生命周期：onCreate()->onStartCommand()->onDestory()。多次调用startService()时onCreate()只会执行一次，但是onStartCommand()会多次执行。多次调用stopService()，onDestory()只会执行一次。
>
> 通过startService()方式启动Service，只要不调用stopService(),就会一直运行，不会与调用者产生关联。它的onBind()没有返回Binder对象，所以调用者无法与Service产生交互。

**2. Service的启动过程：**

startService()其实是由ContextImpl类型的mBase成员变量调用的，这个变量在ActivityThread的performLaunchActivity()中被创建，并通过attach()与Activity产生关联。

startService()的启动分为3个部分

> ContextImpl到AMS到过程
>
> AMS到ActivityThread过程
>
> ActivityThread启动Service过程

- **ContextImpl到AMS到过程：**

  那么看一下ContextImpl的startService():

  ```java
         @Override
         public boolean stopService(Intent service) {
             return stopServiceCommon(service, mUser);
         }
  ```

  直接调用了stopServiceCommon():

  ```java
         private ComponentName startServiceCommon(Intent service, boolean requireForeground,
                 UserHandle user) {\
           				//调用AMS的startService
                 ComponentName cn = ActivityManager.getService().startService(
                     mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                                 getContentResolver()), requireForeground,
                                 getOpPackageName(), user.getIdentifier());
         }
  ```

  这里通过ActivityManager.getService()获取了AMS，然后调用了它的startService(),而AMS是运行在SystemServer进程的，所以这里进程了进程间通讯。

  **总结：可以看到，ContextImpl到AMS到过程就是简单的通知了AMS开始工作**

- **AMS到ActivityThread过程：**

  这里继续看AMS的startService():

  ```java
         public ComponentName startService(IApplicationThread caller, Intent service,
                 String resolvedType, boolean requireForeground, String callingPackage, int userId){
           					//mServices是ActiveServices类型
                     res = mServices.startServiceLocked(caller, service,
                             resolvedType, callingPid, callingUid,
                             requireForeground, callingPackage, userId);
                 return res;
         }
  ```

  这里调用了ActiveServices的startServiceLocked(),看代码：

  ```java
         ArrayMap<ComponentName, ServiceRecord> getServicesLocked(int callingUser) {
             return getServiceMapLocked(callingUser).mServicesByName;
         }
  					 //查找是否有与参数service对应的ServiceRecord，如果没找到就会调用PackageManagerService去获取参数service对应的Service信息，并封装到ServiceRecord中。最后将ServiceRecord封装为ServiceLookupResult返回，ServiceRecord用于描述一个Service
             ServiceLookupResult res =
                 retrieveServiceLocked(service, resolvedType, callingPackage,
                         callingPid, callingUid, userId, true, callerFg, false);
  						//获取ServiceRecord
             ServiceRecord r = res.record;
  						//传入ServiceRecord
             ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
             return cmp;
         }
  ```

  可以看到，这一步其实是获取了ServiceRecord，它用来描述一个service,然后调用了startServiceInnerLocked()，看代码：

  ```java
  526      ComponentName startServiceInnerLocked()  {
  
             String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
             return r.name;
         }
  ```

  这里调用了bringUpServiceLocked(),继续看代码：

  ```java
         private String bringUpServiceLocked() {
           	 //获取Service想要在哪个进程中运行，默认为当前进程
             final String procName = r.processName;
           	 //ProcessRecord用来描述一个应用程序进程
             ProcessRecord app;
             if (!isolated) {
               	 //mAm为AMS，通过AMS查询是否存在一个与Service对应的ProcessRecord(进程)类型对象的app
                 app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
  						 	 //判断用来运行Service的应用程序进程是否存在
                 if (app != null && app.thread != null) {
                   			 //应用程序进程如果存在就启动service
                         realStartServiceLocked(r, app, execInFg);
                         return null;
  
                 }
             } 
  
             if (app == null && !permissionsReviewRequired) {
                 //用来运行Service的应用程序进程如果不存在就去启动应用程序进程
                 if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                         hostingType, r.name, false, isolated, false)) == null) {
                     return msg;
                 }
  
             }
             return null;
         }
  ```

  可以看到这一步主要是判断应用程序是否存在，如果不存在就去启动应用程序进程。如果存在则继续调用realStartServiceLocked(),并把进程信息ProcessRecord传入，代码：

  ```java
         private final void realStartServiceLocked(ServiceRecord r,
                 ProcessRecord app, boolean execInFg) throws RemoteException {
  							 //这里这个app.thread指的是ActivityThread的内部类ApplicationThread。
                 app.thread.scheduleCreateService(r, r.serviceInfo,
                         mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                         app.repProcState);
  
         }
  ```

  可以看到这里调用了app.thread.scheduleCreateService(),这里这个app.thread指的是ActivityThread的内部类ApplicationThread。这里进行了进程间通讯，切换到了应用程序进程。参考：Android中根Activity的启动过程。

  **总结：可以看到，AMS到ActivityThread过程其实就是获取要启动的Service的信息，并判断要启动service的应用程序进程是否存在，如果不存在，就去启动应用程序进程。如果存在，就进行进程间通讯，通知应用程序进程开始工作。**

- **ActivityThread启动Service过程：**

  ```java
          public final void scheduleCreateService(IBinder token,
                  ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
              updateProcessState(processState, false);
              CreateServiceData s = new CreateServiceData();
              s.token = token;
              s.info = info;
              s.compatInfo = compatInfo;
  
              sendMessage(H.CREATE_SERVICE, s);
          }
  ```

  再看这个sendMessage():

  ```java
      private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
          Message msg = Message.obtain();
          msg.what = what;
          msg.obj = obj;
          msg.arg1 = arg1;
          msg.arg2 = arg2;
          if (async) {
              msg.setAsynchronous(true);
          }
        	//mH是H,继承自Handler
          mH.sendMessage(msg);
      }
  ```

  这里的mH是H，继承自Handler。因为ApplicationThread负责进程间通讯，它运行在Binder线程池中，这里需要利用Handler切换到主线程。是Handler，那么就看它的handlerMessage():

  ```java
          public void handleMessage(Message msg) {
              switch (msg.what) {
                  case CREATE_SERVICE:
                      handleCreateService((CreateServiceData)msg.obj);
                      break;
                      }
  ```

  调用了handleCreateService(),继续看代码：

  ```java
         private void handleCreateService(CreateServiceData data) {
           	 //获取要启动的Service的应用程序的LoadedApk,LoadedApk为APK文件的描述类
             LoadedApk packageInfo = getPackageInfoNoCheck(
                     data.info.applicationInfo, data.compatInfo);
                 Service service = null;
           			 //获取类加载器
                 java.lang.ClassLoader cl = packageInfo.getClassLoader();
                 //使用类加载器创建service
                 service = (Service) cl.loadClass(data.info.name).newInstance();
  							 //创建service的上下文环境ContextImpl对象
                 ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
                 context.setOuterContext(service);
  
                 Application app = packageInfo.makeApplication(false, mInstrumentation);
           			 //初始化Service
                 service.attach(context, this, data.info.name, data.token, app,
                         ActivityManager.getService());
           			 //调用onCreate()
                 service.onCreate();
           			 //将service添加到ActivityThread的ArrayMap类型的成员变量mServices中
                 mServices.put(data.token, service);
         }
  ```

  这里可以看到，通过类加载器完成了service的创建，创建了context。然后通过attach()完成了service内部成员变量的初始化。最后调用了onCreate()。完成了startService()的启动流程。

### Service的绑定过程

**1. 先看一下bindService()的使用示例：**

- 首先创建一个Service

  ```java
  public class ClientService extends Service {
      private ServiceBinder serviceBinder;
  		////当Service绑定时调用，并返回binder对象
      @Override
      public IBinder onBind(Intent intent) {
          return  serviceBinder;
      }
  		//当Service被创建时调用
      @Override
      public void onCreate() {
          super.onCreate();
          serviceBinder = new ServiceBinder();
      }
  	 //当Service被解绑调用
      @Override
      public boolean onUnbind(Intent intent) {
          return super.onUnbind(intent);
      }
    	//当Service执行unbindService()解绑后,再次调用bindService()后调用
      @Override
      public void onRebind(Intent intent) {
          super.onRebind(intent);
      }
     //当Service销毁时调用
      @Override
      public void onDestroy() {
          super.onDestroy();
      }
    	//Binder类
      class ServiceBinder extends Binder {
  
      }
  
  }
  ```

- 创建一个Connection:

  ```java
  public class ClientConnection implements ServiceConnection {
  		//当Service连接时调用，这里的ibinder就是Service里的onBinder()返回的binder
      @Override
      public void onServiceConnected(ComponentName name, IBinder ibinder) {
      }
  }
  ```

- 绑定service:

  ```java
          Intent intent = new Intent(this, ClientService.class);
          clientConnection = new ClientConnection();
          bindService(intent, clientConnection, Context.BIND_AUTO_CREATE);
  ```

  以上就是bindService()的示例。

  > 生命周期：onCreate()->onBind()->onUnbind()->onDestory()
  >
  > 通过bindService()方式启动Service会调用onCreate->onBinde(),多次调用bindService()，onCreate->onBinde()都只会执行一次，但是Connection的onServiceConnected()会执行多次。当执行unbindService()后再次bindService()则会执行onRebind()。
  >
  > 调用onBind()之后会在ClientConnection里返回onBinde()里的Binder对象，这个Binder对象是Service的成员变量，所以外界就可以访问Service里。
  >
  > 通过bindService()方式启动Service会与Activity产生关联，所以必须在activity销毁时执行onUnbind()，所以通过bindService()方式Service不能独立运行。

**2. Service的绑定过程**

bindService()其实是由ContextImpl类型的mBase成员变量调用的，这个变量在ActivityThread的performLaunchActivity()中被创建，并通过attach()与Activity产生关联。

bindService()的绑定过程分为3个部分：

> ContextImpl到AMS到过程
>
> Service的绑定过程

- **ContextImpl到AMS到过程：**

  那么看一下ContextImpl的bindService():

  ```java
         @Override
         public boolean bindService(Intent service, ServiceConnection conn,
                 int flags) {
           		//调用bindServiceCommon(),并传入了ServiceConnection
             return bindServiceCommon(service, conn, flags, mMainThread.getHandler(),
                     Process.myUserHandle());
         }
  ```

  可以看到，这里调用了bindServiceCommon(),继续看:

  ```java
         private boolean bindServiceCommon() {
          IServiceConnection sd;
          //这里的mPackageInfo是LoadAPK类型
          sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
          //获取AMS，调用AMS的bindService(),并传入了IServiceConnection，这个IServiceConnection是一个本地代理对象，具备进程间通讯能力
          int res = ActivityManager.getService().bindService(
            mMainThread.getApplicationThread(), getActivityToken(), service,
            service.resolveTypeIfNeeded(getContentResolver()),
            sd, flags, getOpPackageName(), user.getIdentifier());
         }
  ```

  这里先看一下LoadAPK的getServiceDispatcher():

  ```java
         public final IServiceConnection getServiceDispatcher() {
                 LoadedApk.ServiceDispatcher sd = null;
           			 //sd是一个ServiceDispatcher对象
                 return sd.getIServiceConnection();
             }
         }
  ```

  继续看getIServiceConnection():

  ```java
           ServiceDispatcher.InnerConnection mIServiceConnection;
  					IServiceConnection getIServiceConnection() {
                 return mIServiceConnection;
             }
  				private static class InnerConnection extends IServiceConnection.Stub {
  
             }
  ```

  可以看到，最终返回的是一个InnerConnection对象，它继承自IServiceConnection.Stub，意味着实现Binder机制，让Service具备了进程间通讯的能力。

- **Service的绑定过程：**

  这里继续看AMS的bindService():

  ```java
         public int bindService() {
             synchronized(this) {
               		//mServices是ActiveServices类型
                 return mServices.bindServiceLocked(caller, token, service,
                         resolvedType, connection, flags, callingPackage, userId);
             }
         }
  ```

  这里调用了ActiveServices.bindServiceLocked()，继续看代码：

  ```java
         int bindServiceLocked() {
           			 //获取AppBindRecord
                 AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
           			 //获取ConnectionRecord
                 ConnectionRecord c = new ConnectionRecord(b, activity,
                         connection, flags, clientLabel, clientIntent);
  							     //启动Service，这里启动service和startService()里的启动部分是一样的，会调用到同一个方法
                     if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                             permissionsReviewRequired) != null) {
                         return 0;
                     }
  
                 if (s.app != null && b.intent.received) {
                 //s.app != null表示Service已经运行，b.intent.received表示当前应用程序进程已经收到绑定Service时返回到Binder，如果是第一次调用那么onBind()还没调用，所以这里暂时不会执行。在重新绑定时才会执行。
  											 //这里的c.conn是InnerConnection，这里最终会执行onServiceConnected(),所以多次绑定会多次执行onServiceConnected();
                         c.conn.connected(s.name, b.intent.binder, false);
                     if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                       	//解绑后，再次绑定时。也就是执行了unbindService()后再次执行bindService()
                         requestServiceBindingLocked(s, b.intent, callerFg, true);
                     }
                 } else if (!b.intent.requested) {
                    //第一次绑定时，也就是执行bindService()时
                     requestServiceBindingLocked(s, b.intent, callerFg, false);
                 }
  
            return 1;
         }
  ```

  > AppBindRecord:应用程序进程通过Intent绑定Service时，会通过AppBindRecord来维护Service与应用程序进程之间的关联。其内部存储了谁绑定的Service(ProcessRecord)，被绑定的Service(AppBindRecord),绑定Service的Intent(IntentBindRecord)和所有绑定通信记录的信息(ArraySet<ConnectionRecord>)。
  >
  > ServiceRecord:用于描述一个Service
  >
  > ConnectionRecord：用于描述应用程序进程和Service建立的一次通信
  >
  > IntentBindRecord：用于描述绑定Service的Intent。

  这里可以看到，先是获取了Service的相关信息，然后启动了Service。最后调用了requestServiceBindingLocked(s, b.intent, callerFg, false):

  ```java
         private final boolean requestServiceBindingLocked() {
           		//r.app.thread是ActivityThread的内部类ApplicationThread类型
              r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                             r.app.repProcState);
             return true;
         }
  ```

  这里执行了BInder代理对象ActivityThread的scheduleBindService(),那么会切换到应用程序进程，执行ActivityThread的scheduleBindService(),看代码：

  ```java
             public final void scheduleBindService(IBinder token, Intent intent,
                     boolean rebind, int processState) {
                 BindServiceData s = new BindServiceData();
                 s.token = token;
                 s.intent = intent;
                 s.rebind = rebind;
                 sendMessage(H.BIND_SERVICE, s);
             }
  ```

  这里会继续执行到H类的HandleMessage():

  ```java
             public void handleMessage(Message msg) {
                     case BIND_SERVICE:
                         handleBindService((BindServiceData)msg.obj);
                   }
  ```

  继续看handleBindService():

  ```java
         private void handleBindService(BindServiceData data) {
           	 //取出Service
             Service s = mServices.get(data.token);
                     try {
                         if (!data.rebind) {
                           	 //调用onBind()
                             IBinder binder = s.onBind(data.intent);
                           	 //调用AMS的publishService()
                             ActivityManager.getService().publishService(
                                     data.token, data.intent, binder);
                         } else {
                           	//调用Rebind()
                             s.onRebind(data.intent);
                             ActivityManager.getService().serviceDoneExecuting(
                                     data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                         }
                     } 
         }
  ```

  前面分析过，Service建立后会存在Service中，这里再取出来。然后调用onBind(),这时onBind()生命周期就被执行了。

  这里不关注reBind()的情况。

  然后这里又再次调用了AMS的publishService(),又进行了进程间通讯，又来到了AMS所在的SystemServer进程，看publishService()代码：

  ```java
         public void publishService(IBinder token, Intent intent, IBinder service) {
  							 //mServices是ActiveServices类型
                 mServices.publishServiceLocked((ServiceRecord)token, intent, service);
         }
  ```

  再看ActiveServices.publishServiceLocked():

  ```java
         void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
           		 //这里的c.conn是InnerConnection,InnerConnection
  						 c.conn.connected(r.name, service, false);
         }
  ```

  可以看到，这里调用了InnerConnection类型的c.conn的connected(),在前面分析过InnerConnection是LoadApk内部类ServiceDispatcher的内部类，是一个Binder，这里是它的代理Binder对象。

  那么这里就是通过代理Binder对象进行进程间通信，会执行应用程序端的InnerConnection的connected()。看代码：

  ```java
                 public void connected(ComponentName name, IBinder service, boolean dead)
                         throws RemoteException {
                     LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                     if (sd != null) {
                         sd.connected(name, service, dead);
                     }
                 }
  ```

  可以看到执行了ServiceDispatcher的connected():

  ```java
             public void connected(ComponentName name, IBinder service, boolean dead) {
                 if (mActivityThread != null) {
                     //将消息发送到主线程
                     mActivityThread.post(new RunConnection(name, service, 0, dead));
                 } else {
                     doConnected(name, service, dead);
                 }
             }
  ```

  现在还处于binder线程池中，所以要切换到主线程，RunConnection是一个Runnable，同时也是ServiceDispatcher的内部类，那么看它的run():

  ```java
                 public void run() {
                     if (mCommand == 0) {
                         doConnected(mName, mService, mDead);
                     } else if (mCommand == 1) {
                         doDeath(mName, mService);
                     }
                 }
  ```

  这里执行了ServiceDispatcher的doConnected():

  ```java
             public void doConnected(ComponentName name, IBinder service, boolean dead) {
             				 //执行自定义的Connection的onServiceConnected()
                     mConnection.onServiceConnected(name, service);
             }
  ```

  到这时，就会执行自定义的Connection的onServiceConnected()，完成整个bindService()流程。

### 附：

- startService()方式能长时间运行在主线程，但是Activity无法操作Service。bindService()方式可以让调用端获取binder对象，但是无法长期运行。所以可以结合使用。先startService()再bindService()，这样使用时如果要销毁Service的话必须stopService()和uBindService()一起使用(顺序无所谓)。