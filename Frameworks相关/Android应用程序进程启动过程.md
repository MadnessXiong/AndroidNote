### 1. **应用程序进程简介：**

要想启动一个应用程序，首先要保证这个应用程序所需要的应用程序进程已经启动。AMS在启动应用程序时会检查这个应用程序需要的应用程序进程是否存在，不存在就会请求Zygote进程启动所需要的应用程序进程。Zygote的Java层会创建一个Server端的Socket，这个Socket用来等待AMS请求Zygote来创建新的应用程序进程。Zygote进程通过fork自身创建应用程序进程，这样应用程序进程就会获得zygote进程在启动时创建的虚拟机实例，同时还创建了Binder线程池和消息循环。这样运行在应用程序进程中的应用程序就可以方便地使用Binder进程进程间通讯以及处理消息了。

### **2. 应用程序启动过程**

从简介里可以看出，应用程序启动主要包括以下几个过程：

> 第一步：AMS启动向Zygote发起启动应用程序进程请求
>
> 第二步：Zygote进程fork自创建一个应用程序进程，同时创建BInder线程池，让应用程序具备进程间通讯能力
>
> 第三步：启动消息循环，让应用程序进程中的应用程序可以使用消息处理机制

那么从根据以上几点，细化的看一下每个步骤的细节

- 第一步：AMS启动向Zygote发起启动应用程序进程请求

  ```java
    private final void startProcessLocked(ProcessRecord app, String hostingType,
                String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
  
     					//获取要创建的应用程序进程id
                 int uid = app.uid;
                 int[] gids = null;
  						//对用户组ID-gid进行创建和赋值
             if (ArrayUtils.isEmpty(permGids)) {
                  gids = new int[3];
              } else {
                  gids = new int[permGids.length + 3];
                  System.arraycopy(permGids, 0, gids, 3, permGids.length);
              }
              gids[0] = UserHandle.getSharedAppGid(UserHandle.getAppId(uid));
              gids[1] = UserHandle.getCacheAppGid(UserHandle.getAppId(uid));
              gids[2] = UserHandle.getUserGid(UserHandle.getUserId(uid));
  						//启动应用程序进程
              startResult = Process.start(entryPoint,
                     app.processName, uid, uid, gids, debugFlags, mountExternal,
                     app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                     app.info.dataDir, invokeWith, entryPointArgs);
  
         }
  ```

  可以看到，这里先获取了应用程序进程id,然后对用户组ID-gid,进行创建和赋值，最后调用了Process.start()启动应用程序进程，那么再看这个方法:

  ```java
         public static final ProcessStartResult start(final String processClass,
                                       final String niceName,
                                       int uid, int gid, int[] gids,
                                       int debugFlags, int mountExternal,
                                       int targetSdkVersion,
                                       String seInfo,
                                       String abi,
                                       String instructionSet,
                                       String appDataDir,
                                       String invokeWith,
                                       String[] zygoteArgs) {
             return zygoteProcess.start(processClass, niceName, uid, gid, gids,
                         debugFlags, mountExternal, targetSdkVersion, seInfo,
                         abi, instructionSet, appDataDir, invokeWith, zygoteArgs);
         }
  ```

  这里调用了ZygoteProcess的start(),继续看它的代码：

  ```java
         public final Process.ProcessStartResult start(final String processClass,
                                                       final String niceName,
                                                       int uid, int gid, int[] gids,
                                                       int debugFlags, int mountExternal,
                                                       int targetSdkVersion,
                                                       String seInfo,
                                                       String abi,
                                                       String instructionSet,
                                                       String appDataDir,
                                                       String invokeWith,
                                                       String[] zygoteArgs) {
             try {
                 return startViaZygote(processClass, niceName, uid, gid, gids,
                         debugFlags, mountExternal, targetSdkVersion, seInfo,
                         abi, instructionSet, appDataDir, invokeWith, zygoteArgs);
             } catch (ZygoteStartFailedEx ex) {
                 Log.e(LOG_TAG,
                         "Starting VM process through Zygote failed");
                 throw new RuntimeException(
                         "Starting VM process through Zygote failed", ex);
             }
         }
  ```

  这里直接调用了startViaZygote(),继续看：

  ```java
         private Process.ProcessStartResult startViaZygote(final String processClass,
                                                           final String niceName,
                                                           final int uid, final int gid,
                                                           final int[] gids,
                                                           int debugFlags, int mountExternal,
                                                           int targetSdkVersion,
                                                           String seInfo,
                                                           String abi,
                                                           String instructionSet,
                                                           String appDataDir,
                                                           String invokeWith,
                                                           String[] extraArgs)
                                                           throws ZygoteStartFailedEx {
           	//创建一个argsForZygote
             ArrayList<String> argsForZygote = new ArrayList<String>();
  					//应用程序的启动参数，省略部分参数添加代码
             argsForZygote.add("--runtime-args");
             argsForZygote.add("--setuid=" + uid);
             argsForZygote.add("--setgid=" + gid);
  
     
             synchronized(mLock) {
                 return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
             }
         }
  ```

  这一步主要是创建了一个argsForZygote,并把应用程序的启动参数保存在其中。然后再调用了zygoteSendArgsAndGetResult(),并把openZygoteSocketIfNeeded()的返回值传入，先看openZygoteSocketIfNeeded():

  ```java
         @GuardedBy("mLock")
         private ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
             Preconditions.checkState(Thread.holdsLock(mLock), "ZygoteProcess lock not held");
  						//与zygote进程建立socket连接
            	primaryZygoteState = ZygoteState.connect(mSocket);
     			  	//尝试连接主模式
             if (primaryZygoteState.matches(abi)) {
                 return primaryZygoteState;
             }
             //主模式不匹配，则尝试辅模式
  					  secondaryZygoteState = ZygoteState.connect(mSecondarySocket);
  
             }
         }
  ```

  在这里与zygote进程建立了连接，并返回了一个ZygoteState，这个ZygoteState是负责连接的东西，内部封装了socket。再看zygoteSendArgsAndGetResult():

  ```java
         @GuardedBy("mLock")
         private static Process.ProcessStartResult zygoteSendArgsAndGetResult(
                 ZygoteState zygoteState, ArrayList<String> args)
                 throws ZygoteStartFailedEx {
             try {
                 // Throw early if any of the arguments are malformed. This means we can
                 // avoid writing a partial response to the zygote.
                 int sz = args.size();
                 for (int i = 0; i < sz; i++) {
                     if (args.get(i).indexOf('\n') >= 0) {
                         throw new ZygoteStartFailedEx("embedded newlines not allowed");
                     }
                 }
     
                 final BufferedWriter writer = zygoteState.writer;
                 final DataInputStream inputStream = zygoteState.inputStream;
     
                 writer.write(Integer.toString(args.size()));
                 writer.newLine();
     
                 for (int i = 0; i < sz; i++) {
                     String arg = args.get(i);
                     writer.write(arg);
                     writer.newLine();
                 }
     
                 writer.flush();
     
                 // Should there be a timeout on this?
                 Process.ProcessStartResult result = new Process.ProcessStartResult();
     
                 // Always read the entire result from the input stream to avoid leaving
                 // bytes in the stream for future process starts to accidentally stumble
                 // upon.
                 result.pid = inputStream.readInt();
                 result.usingWrapper = inputStream.readBoolean();
     
                 if (result.pid < 0) {
                     throw new ZygoteStartFailedEx("fork() failed");
                 }
                 return result;
             } catch (IOException ex) {
                 zygoteState.close();
                 throw new ZygoteStartFailedEx(ex);
             }
         }
  ```

  可以看到，这里主要进行了写操作，将之前启动参argsForZygote写入了ZygoteState中，那么就是启动应用程序进程所需要的参数发送给了Zygote进程，然后等待返回。

  **总结：第一步的操作就是将启用应用程序所需的参数进行封装->建立socket连接->将参数写入socket里发送建立应用程序进程的请求，然后等待结果的返回**

- 第二步：在第二步开始之前，先看一下ZygoteInit的main():

  ```java
  public static void main(String argv[]) {
                ZygoteServer zygoteServer = new ZygoteServer();
  							//创建一个Server端的Socket，socketName为zygote
                zygoteServer.registerServerSocket(socketName);
  							
                 if (!enableLazyPreload) {
                   //预加载类和资源
                     preload(bootTimingsTraceLog);
                 } else {
                     Zygote.resetNicePriority();
                 }
    
                 if (startSystemServer) {
                 //启动SystemServer进程
                 startSystemServer(abiList, socketName, zygoteServer);
                 }
     						 //等待AMS请求
                 zygoteServer.runSelectLoop(abiList);
           
        }
  ```

  可以看到，Server端建立了一个Socket连接，然后调用了ZygoteServer的runSelectLoop()等待AMS发起请求

  在第一步的分析中AMS已经发起了一个请求，那么第二步就来到了runSelectLoop()，看代码：

  ```java
        void runSelectLoop(String abiList) throws Zygote.MethodAndArgsCaller {
             ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
  						boolean done = peers.get(i).runOnce(this);
  
         }
  
  ```

  可以看到，每当有请求时，会调用ZygoteConnection的runOnce(),继续看代码：

  ```java
         boolean runOnce(ZygoteServer zygoteServer) throws Zygote.MethodAndArgsCaller {
    							//获取应用程序进程的启动参数
                 args = readArgumentList();
  								//将启动参数封装到Arguments中
                 parsedArgs = new Arguments(args);
  								//创建应用程序进程
                 pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, 
  
    							//如果当前代码在子进程中运行，pid=0就代表在新建的子进程中
                 if (pid == 0) {
                     // in child
                     zygoteServer.closeServerSocket();
                     IoUtils.closeQuietly(serverPipeFd);
                     serverPipeFd = null;
    									//处理应用程序进程
                     handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);
         }
  
  ```

  在这里首先获取了AMS传递过来的参数，并把它封装到Arguments中，然后根据参数调用Zygote.forkAndSpecialize()，通过fork的方式，创建了请求的应用程序进程。最后调用handleChildProc()对新创建的应用程序进程进行处理

  ```java
         private void handleChildProc(Arguments parsedArgs,
                 FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)
                 throws Zygote.MethodAndArgsCaller {
                 ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion,
                         parsedArgs.remainingArgs, null /* classLoader */);
         }
  ```

  这个方法中主要是调用了ZygoteInit.zygoteInit(),继续看代码：

  ```java
         public static final void zygoteInit(int targetSdkVersion, String[] argv,
                 ClassLoader classLoader) throws Zygote.MethodAndArgsCaller {
           		//创建Binder线程池
             ZygoteInit.nativeZygoteInit();
             RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
         }
  ```

  可以看到，首先为当前进程创建Binder线程池，那么当前进程就具备了进程间通讯的能力，然后调用RuntimeInit.applicationInit()，继续看代码：

  ```java
         protected static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
                 throws Zygote.MethodAndArgsCaller {
  
             invokeStaticMain(args.startClass, args.startArgs, classLoader);
         }
  ```

  这里主要是调用了invokeStaticMain(),继续看代码：

  ```java
         private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
                 throws Zygote.MethodAndArgsCaller {
             Class<?> cl;
           			//通过反射，获得ActivityThread类
                 cl = Class.forName(className, true, classLoader);
  
             Method m;
           				//通过反射获得ActivityTread的main()
                 m = cl.getMethod("main", new Class[] { String[].class });
           		//通过抛异常的方式，进入Zygote的Main()
             throw new Zygote.MethodAndArgsCaller(m, argv);
         }
  ```

  可以看到，在这里通过反射到方式获得了ActivityThread类，然后获得了它的main()，然后再通过抛异常的方式，将方法传递出去，调用链的起点是ZygoteInit的main(),那么异常就会被它捕获，那么看一下代码：

  ```java
         public static void main(String argv[]) {
             try {
                 }
             } catch (Zygote.MethodAndArgsCaller caller) {
                 caller.run();
  				}
  ```

  当异常被捕获时，调用了MethodAndArgsCaller的run(),继续看代码：

  ```java
            public void run() {
              		//调用ActivityThread的main()
                    mMethod.invoke(null, new Object[] { mArgs });
            }
  ```

  可以看到这里调用了mMethod.invoke()，而这个mMethod前面说过，是传进来的ActivityThread的main(),那么这时ActivityThread的main()就被调用，这样应用程序进程就创建完成了，并运行了主线程的管理类ActivityThread。

  **总结：第二步的操作就是，Zygote收到创建应用程序进程的请求后，通过fork自身的方式，创建了一个应用程序进程，然后创建了Binder线程池，让应用程序具备了进程间通讯的能力。最后通过反射的方式获得ActivityThread的main()，并最终执行这个main(),到此完成了应用程序的创建，并具备了进程间通讯能力，然后进入了ActivityThread的main()当中。**

- **第三步：启动消息循环，让应用程序进程中的应用程序可以使用消息处理机制**

  ```java
   	      public static void main(String[] args) {
  					 //创建主线程Looper
             Looper.prepareMainLooper();
    				//
             ActivityThread thread = new ActivityThread();
     
             if (sMainThreadHandler == null) {
               	//创建主线程Handler
                 sMainThreadHandler = thread.getHandler();
             }
     				//启动Looper
             Looper.loop();
         }
  ```

  可以看到，在ActivityThread中建立了消息循环机制，让应用程序具备了消息处理能力。

  **应用程序进程的启动过程就此完成。**

  