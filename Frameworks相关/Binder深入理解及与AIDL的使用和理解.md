

### Binder深入理解

1. **背景知识**

   - 进程隔离：进程隔离是为保护操作系统中进程互不干扰而设计的一组不同硬件和软件的技术。这个技术是为了避免进程A写入进程B的情况发生。进程隔离的实现，使用了虚拟机地址空间。进程A的虚拟地址和进程B的虚拟地址不同，这样就防止进程A将数据写入进程B。

     操作系统不同进程之间，数据不共享。因此一个进程要与另外一个进程通信，需要某种机制才能完成。

   - 用户空间/内核空间：Linux内核是操作系统的核心，所以为了它的安全，不可能允许其他的应用程序随便调用或访问。所以它提供了一个保护机制，这个保护机制把内核和上层应用程序抽象的隔离开，分别为内核空间和用户空间。

   - 内核态/用户态：用户空间访问内核空间的唯一方式是系统调用，这是让用户空间对内核空间的访问通过一个统一的入口，这样所有的资源(如访问文件)访问都在内核的控制下，以免用户程序对系统资源的越权访问，从而保障了系统的安全和稳定。当一个任务(进程)执行系统调用而陷入内核代码中执行时，就称之为内核态。当进程在执行用户自己的代码时，则称之为用户态。

   - 内核模块/驱动：通过系统调用用户空间可以访问内核空间，但是用户空间之间的相互通信还是不行的，那么就需要系统内核的支持，如Socket,管道等都是内核支持的。但是Binder并不是Linux内核的一部分。需要Linux的动态可加载内核模块来解决这个问题。用户进程之前通过这个模块作为桥梁，就可以相互通信了。在Android系统中，这个运行在内核空间，负责各个用户进程通过Binder通信的模块叫做Binder驱动。

     > 动态可加载内核是指模块可以在系统运行期间加载到内核或从内核卸载。模块是具有独立功能的程序，它可以被单独编译，但不能独立运行。它在运行时被链接到内核作为内核的一部分在内核空间中运行。模块通常由一组函数和数据结构组成。

2. **使用Binder的原因：**

   - 稳定性：Binder采用C/S的通信模式，客户端有什么需求直接交给服务端去完成，架构清晰，职责明确又相互独立，自然稳定性更好。在Linux通信机制中，目前只有socket支持C/S的通信模式，但是Socket有其劣势
   - 高效性：Socket是一个通用接口，导致其传输效率低，开销大；管道和消息队列因为采用存储转发方式，所以至少需要拷贝2次数据，效率低；共享内存虽然传输时没有拷贝数据，不是C/S模式，控制机制复杂。Binder只拷贝一次
   - 安全性：Linux的IPC机制在本身的实现中，并没有安全措施，传统的进程通讯方式对与通信双方的身份并没有严格的验证，只有在上层协议上进行架设，比如Socket通信IP地址是客户端手动填入的，可以进行伪造。而Binder机制的UID/PID是由Binder机制本身在内核空间添加身份标识，因而大大提高了安全性。

3. **Binder的通信模型：**

   Binder采用C/S通信模式。那么通信的2端可以分为Server进程，Client进程。由于进程隔离，它们之间没法直接进行通信。需要一定的机制帮助他们进行通信。

   假如将2端之间的通讯看成A给B打电话。那么首先A要知道B的电话号码，其次拨号以后需要基站将信息传播出去。那么将有4个角色，分别是发起通话的Client，通讯录ServerManager(SM),基站Binder驱动驱动，接电话的Server。那么整个通讯过程如下：

   - 建立通讯录ServerManager。首先有一个进程向驱动提出申请为SM，驱动同意后，SM进程负责管理Service。这时还没有任何Server进行注册，所以通讯录还是空的
   - 各个Server向SM进行注册。每个Server端启动后，在SM进行注册，SM会建立一张表，并把这些Server的名字和地址存入其中。
   - Client要与某个Server进行通讯，那么会去SM里查询这个Server的地址，然后开始通讯。

   以上就是进程间通讯的简单过程，Server端将自己注册到SM中，Client要与哪个Server进行通讯，就去SM里找对应的地址，这时打电话的双方号码都有了，就差一个基站帮忙发送信息了。这个基站就是Binder驱动，它的原理是整个跨进程通讯的核心。

4. **Binder机制跨进程原理**

   上面分析了Binder的通讯过程，建立联系后通讯由Binder驱动来完成，那么再来看Binder驱动完成通讯的过程。

   假如Client进程要调用Server进程的Object对象的一个方法get(),那么具体过程如下：

   - Server进程向SM进行注册，告诉SM它的地址，及它有一个Object对象，可以执行get()操作

   - Client向SM进行查询Server的Object对象。这时Binder驱动并不会返回一个真正的Object对象，而是返回一个代理对象ObjectProxy。这时连接已经建立，然后准备开始通讯。Client进程拿到ObjectProxy后调用它的get(),这时Binder驱动收到这个调用请求，发现ObjectProxy是Server的Object代理，于是Binder驱动通知Server进程调用它的get(),Server收到消息后调用自己的get()并把结果返回给Binder驱动，Binder驱动再把结果返回给Client。

     可以看出Binder跨进程实际并不是真正把一个对象传输到了另外一个进程。Client只是对代理对象的操作，然后Binder驱动又把对代理对象的操作告诉Server，让Server执行真正的操作，并把结果返回给Binder驱动，Binder驱动再通过代理对象把结果返回给Client。

> SM和Server通常也不在一个进程，所以Server注册的过程其实也是跨进程通讯。SM内部已经有一个默认地址，Server通过这个默认地址查找到SM。Server存在SM里的对象其实也是一个代理对象。Server的本地对象只有一个，其他都是代理对象。

5. **总结**
   - Binder是一种通讯机制，ALDL使用的就是Binder进行通讯
   - 对于Server来说Binder指的是Binder本地对象
   - 对于Client来说，Binder指的是Binder代理对象，它只是Binder本地对象的一个远程代理。对于这个代理对象的操作，最终会通过Binder驱动转发到Binder本地对象上完成。
   - 对于传输过程来说，Binder驱动会对代理对象和本地对象进行自动转换处理
   - 要完成本地Binder和代理BInder的转换，那么Binder驱动里必然保存了每一个跨进程Binder对象的相关信息，在Binder驱动中，Binder本地对象的代表是binder_node的数据结构，Binder代理对象是用binder_ref代表的

### AIDL

Java为了方便进行进程进程间通讯，对Binder的通讯过程进行了封装，这就是AIDL，

**AIDL的使用**：

1. Server端：

   1. 定义Server端的aidl接口，此接口主要定义可以对外提供的服务

      ```java
      // IServer.aidl
      package com.madnessxiong.server;
      interface IServer {
          void get();
      }
      ```

   2. 编译生成aidl接口实现文件，这一步主要是生成Binder对象。

      ```java
      /*
       * This file is auto-generated.  DO NOT MODIFY.
       * Original file: /work/project/Server/app/src/main/aidl/com/madnessxiong/server/IServer.aidl
       */
      package com.madnessxiong.server;
      // Declare any non-default types here with import statements
      
      public interface IServer extends android.os.IInterface
      {
      /** Local-side IPC implementation stub class. */
      public static abstract class Stub extends android.os.Binder implements com.madnessxiong.server.IServer
      {
      private static final java.lang.String DESCRIPTOR = "com.madnessxiong.server.IServer";
      /** Construct the stub at attach it to the interface. */
      public Stub()
      {
      this.attachInterface(this, DESCRIPTOR);
      }
      /**
       * Cast an IBinder object into an com.madnessxiong.server.IServer interface,
       * generating a proxy if needed.
       */
      public static com.madnessxiong.server.IServer asInterface(android.os.IBinder obj)
      {
      if ((obj==null)) {
      return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin!=null)&&(iin instanceof com.madnessxiong.server.IServer))) {
      return ((com.madnessxiong.server.IServer)iin);
      }
      return new com.madnessxiong.server.IServer.Stub.Proxy(obj);
      }
      @Override public android.os.IBinder asBinder()
      {
      return this;
      }
      @Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
      {
      java.lang.String descriptor = DESCRIPTOR;
      switch (code)
      {
      case INTERFACE_TRANSACTION:
      {
      reply.writeString(descriptor);
      return true;
      }
      case TRANSACTION_get:
      {
      data.enforceInterface(descriptor);
      this.get();
      reply.writeNoException();
      return true;
      }
      default:
      {
      return super.onTransact(code, data, reply, flags);
      }
      }
      }
      private static class Proxy implements com.madnessxiong.server.IServer
      {
      private android.os.IBinder mRemote;
      Proxy(android.os.IBinder remote)
      {
      mRemote = remote;
      }
      @Override public android.os.IBinder asBinder()
      {
      return mRemote;
      }
      public java.lang.String getInterfaceDescriptor()
      {
      return DESCRIPTOR;
      }
      @Override public void get() throws android.os.RemoteException
      {
      android.os.Parcel _data = android.os.Parcel.obtain();
      android.os.Parcel _reply = android.os.Parcel.obtain();
      try {
      _data.writeInterfaceToken(DESCRIPTOR);
      mRemote.transact(Stub.TRANSACTION_get, _data, _reply, 0);
      _reply.readException();
      }
      finally {
      _reply.recycle();
      _data.recycle();
      }
      }
      }
      static final int TRANSACTION_get = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
      }
      public void get() throws android.os.RemoteException;
      }
      
      
      ```

   3. 创建Service,在Service中返回IBinder对象，这个IBinder是系统生成的IServer.Stub()。

      ```java
      public class ClientService extends Service {
      
          @Override
          public IBinder onBind(Intent intent) {
              // TODO: Return the communication channel to the service.
              return  iBinder;
          }
      
         IBinder iBinder= new IServer.Stub(){
             @Override
             public void get() throws RemoteException {
      
             }
         };
      }
      
      //在manifests注册service
              <service
                  android:name=".ClientService"
                  android:enabled="true"
                  android:exported="true">
                  <intent-filter>
                      <action android:name="com.madnessxiong.server.server" />
                  </intent-filter>
              </service>
      ```

   4. 启动Service提供服务

      ```
          Intent intent = new Intent(this, ClientService.class);
          startService(intent);
      ```

2. Client端：

   1. 将Server端的aidl文件复制到Client端,包名要和Server端保持一致

      ```java
      interface IServer {
          void get();
      }
      ```

   2. 编译，生成aidl实现接口文件

      ```java
      /*
       * This file is auto-generated.  DO NOT MODIFY.
       * Original file: /work/project/Client/app/src/main/aidl/com/madnessxiong/server/IServer.aidl
       */
      package com.madnessxiong.server;
      // Declare any non-default types here with import statements
      
      public interface IServer extends android.os.IInterface
      {
      /** Local-side IPC implementation stub class. */
      public static abstract class Stub extends android.os.Binder implements com.madnessxiong.server.IServer
      {
      private static final java.lang.String DESCRIPTOR = "com.madnessxiong.server.IServer";
      /** Construct the stub at attach it to the interface. */
      public Stub()
      {
      this.attachInterface(this, DESCRIPTOR);
      }
      /**
       * Cast an IBinder object into an com.madnessxiong.server.IServer interface,
       * generating a proxy if needed.
       */
      public static com.madnessxiong.server.IServer asInterface(android.os.IBinder obj)
      {
      if ((obj==null)) {
      return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin!=null)&&(iin instanceof com.madnessxiong.server.IServer))) {
      return ((com.madnessxiong.server.IServer)iin);
      }
      return new com.madnessxiong.server.IServer.Stub.Proxy(obj);
      }
      @Override public android.os.IBinder asBinder()
      {
      return this;
      }
      @Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
      {
      java.lang.String descriptor = DESCRIPTOR;
      switch (code)
      {
      case INTERFACE_TRANSACTION:
      {
      reply.writeString(descriptor);
      return true;
      }
      case TRANSACTION_get:
      {
      data.enforceInterface(descriptor);
      this.get();
      reply.writeNoException();
      return true;
      }
      default:
      {
      return super.onTransact(code, data, reply, flags);
      }
      }
      }
      private static class Proxy implements com.madnessxiong.server.IServer
      {
      private android.os.IBinder mRemote;
      Proxy(android.os.IBinder remote)
      {
      mRemote = remote;
      }
      @Override public android.os.IBinder asBinder()
      {
      return mRemote;
      }
      public java.lang.String getInterfaceDescriptor()
      {
      return DESCRIPTOR;
      }
      @Override public void get() throws android.os.RemoteException
      {
      android.os.Parcel _data = android.os.Parcel.obtain();
      android.os.Parcel _reply = android.os.Parcel.obtain();
      try {
      _data.writeInterfaceToken(DESCRIPTOR);
      mRemote.transact(Stub.TRANSACTION_get, _data, _reply, 0);
      _reply.readException();
      }
      finally {
      _reply.recycle();
      _data.recycle();
      }
      }
      }
      static final int TRANSACTION_get = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
      }
      public void get() throws android.os.RemoteException;
      }
      
      ```

   3. 建立ServiceConnection

      ```java
      public class ClientConnection implements ServiceConnection {
      
          private IServer iServer;
      
          @Override
          public void onServiceConnected(ComponentName name, IBinder service) {
              iServer = IServer.Stub.asInterface(service);
          }
      
          @Override
          public void onServiceDisconnected(ComponentName name) {
      
          }
      
          public IServer getiServer() {
              return iServer;
          }
      }
      ```

      在onServiceConnected使用 IServer.Stub.asInterface返回IBinder代理对象。

   4. 绑定服务，建立连接

      ```java
      public void bindService() {
              Intent intent = new Intent();
              intent.setAction("com.madnessxiong.server.server");
              intent.setPackage("com.madnessxiong.server");
              clientConnection = new ClientConnection();
              bindService(intent, clientConnection, Context.BIND_AUTO_CREATE);
          }
      ```

   5. 开始跨进程通讯

      ```java
      clientConnection.getiServer().get();
      ```

   以上就是简单的AIDL的整个通信过程。

**AIDL原理分析：**

根据Binder原理分析，需要4个角色，Server端，本地Binder对象，Client端，代理Binder对象。

1. Server端是提供服务的进程，那么再看下一它的本地Binder对象

   ```java
      IBinder iBinder= new IServer.Stub(){
          @Override
          public void get() throws RemoteException {
          }
      };
   ```

   可以看到是通过IServer.Stub返回的，这个是编译后自动生成的，看一下它的实现：

   ```java
   public static abstract class Stub extends android.os.Binder implements com.madnessxiong.server.IServer {
       }
   ```

   可以看到Stub继承自Binder，意味着Stub是一个本地Binder对象。也就是它具有进程间通讯的能力。同时实现了IServer接口，这个接口代表能为客户端提供的服务。

2. Client是调用服务的进程，那么看一下它的代理Binder对象

   ```java
   iServer = IServer.Stub.asInterface(service);
   ```

   可以看到是通过asInterface()获得本地代理Binder对象的，看一下它的实现

   ```java
           public static com.madnessxiong.server.IServer asInterface(android.os.IBinder obj) {
               if ((obj == null)) {
                   return null;
               }
             	//尝试查找本地代理对象
               android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
             	//根据本地代理对象是否存在，判断是否是当前进程，如果是当前进程则返回本地Binder对象
               if (((iin != null) && (iin instanceof com.madnessxiong.server.IServer))) {
                   return ((com.madnessxiong.server.IServer) iin);
               }
             	//如果不是当前进程，那么代表进程间通讯，则返回代理Binder对象
               return new com.madnessxiong.server.IServer.Stub.Proxy(obj);
           }
   ```

3. 获得代理对象后，那么开始通信：

   ```java
   clientConnection.getiServer().get();
   ```

   可以看到调用了代理对象的get(),看一下代码:

   ```java
               @Override
               public void get() throws android.os.RemoteException {
                   android.os.Parcel _data = android.os.Parcel.obtain();
                   android.os.Parcel _reply = android.os.Parcel.obtain();
                   try {
                       _data.writeInterfaceToken(DESCRIPTOR);
                       mRemote.transact(Stub.TRANSACTION_get, _data, _reply, 0);
                       _reply.readException();
                   } finally {
                       _reply.recycle();
                       _data.recycle();
                   }
               }
   ```

   在这里首先对数据进行了序列化，然后调用了transact(),这是一个本地方法，最终会调用到talkWithDriver方法，看名字就知道通讯过程交给了驱动完成，这个方法最后通过ioctl系统调用，Client进程进入内核态，Client调用get()的线程挂起等待返回。驱动然后去唤醒Server进程，调用Server本地Binder对象的onTransact()(实际上由Server端线程池完成),那么再看本地Binder对象的onTransact():

   ```java
           @Override
           public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
               java.lang.String descriptor = DESCRIPTOR;
               switch (code) {
                   case INTERFACE_TRANSACTION: {
                       reply.writeString(descriptor);
                       return true;
                   }
                   case TRANSACTION_get: {
                       data.enforceInterface(descriptor);
                       this.get();
                       reply.writeNoException();
                       return true;
                   }
                   default: {
                       return super.onTransact(code, data, reply, flags);
                   }
               }
           }
   ```

   在这里根据调用号(每个AIDL方法都有一个编号，在跨进程的时候，不会传递方法，而是传递编号指明调用哪个方法)，这里将调用本地的get()，然后返回给驱动。驱动唤醒挂起的Client线程，将结果返回。这样就完成了一次进程间通讯。

**总结：**

- 角色：以上就是AIDL的跨进程通讯过程，可以看到和Binder分析里的基本一致，Client及代理Binder对象，Server及本地Binder对象，Stub继承Binder接口进行进程间通讯功能。

- 过程：代理Binder对象通过Binder驱动调用方法，Binder驱动通知本地Binder调用方法返回结果，Binder驱动将结果返回给代理Binder，这样就完成了一次进程间通讯。