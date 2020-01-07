### 1.  Handler的使用

- 1.1 构建Handler

  ```
   	       //方式一
   	      Handler handler = new Handler();
   				
  	      //方式二
          Handler handler=new Handler(new Handler.Callback() {
              @Override
              public boolean handleMessage(Message msg) {
                  return false;
              }
          });
          
          Looper.prepare();

  	      //方式三
          Handler handler= new Handler(Looper.myLooper());
          
          //方式四
          Handler handler=new Handler(Looper.myLooper(), new Handler.Callback() {
              @Override
              public boolean handleMessage(Message msg) {
                  return false;
              }
          });
  ```

  可以看到，构建Handler有2类，第一种不传入Looper，第二种传入Looper，至于区别是什么，及应用场景，待源码分析完后，自然明了。

  1.2 构建Message

  ```
          Message message = Message.obtain();
         //消息内容
  	      message.arg1=1;
          message.obj="Handler";
         //消息标识
          message.what=0;
  ```

  Message一般通过obtain()创建，可以通过arg1,arg2发送int类型的消息，也可以通过obj发送对象或其他类型的消息。还可以通过what对消息进行标识。

  1.3 发送消息

  ```
          //省略部分延伸方法及部分简单的参数
          handler.sendMessage(message);
          handler.sendEmptyMessage(0);
  
          handler.post(new Runnable() {
              @Override
              public void run() {
                  
              }
          });
          handler.postAtTime(...);
          handler.postDelayed(...);
          handler.postAtFrontOfQueue(...);
  ```

  可以看到Handler主要通过sendMessage()和post()，2种方式发送消息，sendMessage()直接发送了消息或者发送了空消息。post则传入了一个Runnable，并没有传递消息。至于区别是什么，及应用场景，待源码分析完后，自然明了。

  1.4  处理消息

  ```
         private Handler handler =new Handler(){
           //这里这个handleMessage()和构造传入那个CallBack的handleMessage()不一样
          @Override
          public void handleMessage(Message msg) {
              super.handleMessage(msg);
          }
      };
  ```

  最后直接在handleMessage()里处理消息。这就是Handler的简单使用。

  > 一般Handler用于线程间通讯。

### 2. Handler调用流程源码分析

- 2.1使用Handler是从构造一个Handler开始的，它有2种构造，第一种不传Looper，第二种传Looper，看一下第一种

  ```
   //构造方法1，不传Looper
  public Handler(Handler.Callback callback, boolean async) {
          mLooper = Looper.myLooper();
          if (mLooper == null) {
              throw new RuntimeException(
                      "Can't create handler inside thread " + Thread.currentThread()
                              + " that has not called Looper.prepare()");
          }
          mQueue = mLooper.mQueue;
          mCallback = callback;
          mAsynchronous = async;
      }    
  ```

  首先先通过myLooper()获取一个Looper(),在这里获取了Looper，并赋值给Handler的Looper（也就是对Handler和Looper进行了绑定），然后进行了mQueue和mCallback的赋值，先看一下myLooper():

  ```
   public static @Nullable Looper myLooper() {
          return sThreadLocal.get();
      }
  ```

  myLooper()里直接通过ThreadLocal的get()直接获取了一个Looper，Looper是由ThreadLocal保存的。那么这个Looper是怎么来的？找一下它的set():

  ```
      private static void prepare(boolean quitAllowed) {
          if (sThreadLocal.get() != null) {
              throw new RuntimeException("Only one Looper may be created per thread");
          }
          sThreadLocal.set(new Looper(quitAllowed));
      }
  ```

  这里可以看到,在Looper的prepare()里通过new Looper()的方式往ThreadLocal存了一个Looper对象。那么它是如何存的？再看ThreadLocal的set()，以及Looper的构造具体实现：

  ```
      	//ThreadLocal的set()
  		public void set(T value) {
          Thread t = Thread.currentThread();
          ThreadLocalMap map = getMap(t);
          if (map != null)
              map.set(this, value);
          else
              createMap(t, value);
      }
  		//Looper的构造
      private Looper(boolean quitAllowed) {
          mQueue = new MessageQueue(quitAllowed);
          mThread = Thread.currentThread();
      }
  ```

  可以看到，在ThreadLocal的set()中先获取当前的了线程t,然后以当前线程t为键，以Looper为值存入一个ThreadLocalMap中。也就是说Looper和当前线程进行了绑定。然后Looper的构造里初始化了一个MessageQueue对象存了起来。

  

  **小结一下：**

  **1:当构造一个Handler时，会先获取Looper并进行绑定，所以首先必须有一个Looper，不然会抛出异常。所以使用Handler时必须先通过Looper.prepare()初始化Looper。**

  **2:这个Looper创建时通过ThreadLocalMap和当前线程进行了绑定。同时初始化了MessageQueue对象，这个MessageQueue又在构建Handler时赋值给了Handler。所以Handler和Looper共享一个MessageQueue，同时每一个Looper绑定了当前的线程。**

  > 一般主线程使用Handler时，系统已经在ActivityThread的main()中调用了Looper.prepareMainLooper()，所以主线程不必再自己主动调用Looper.prepare()，但是在其他线程使用时就必须主动调用Looper.prepare()，不然会直接抛出异常。第二种构造就是让我们构造一个Looper传入，其他没区别，这里不再展开

  

- 2.2 再看Message

  ```
      public static Message obtain() {
          synchronized (sPoolSync) {
              if (sPool != null) {
                  Message m = sPool;
                  sPool = m.next;
                  m.next = null;
                  m.flags = 0; // clear in-use flag
                  sPoolSize--;
                  return m;
              }
          }
          return new Message();
      }
  
  ```

  sPool是缓存池，可以看到通过obtain()的方式创建Message，如果有可用对象，则可以循环利用，节约资源。如果没有再去创建。

- 2.3 发送消息之前说过有2种，sendMessage()和post()。那么分别先看一下他们的调用链

  sendMessage():

  ```
  //在这里对MessageQueue进行赋值，并判断
  public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
          MessageQueue queue = mQueue;
          if (queue == null) {
              RuntimeException e = new RuntimeException(
                      this + " sendMessageAtTime() called with no mQueue");
              Log.w("Looper", e.getMessage(), e);
              return false;
          }
          return enqueueMessage(queue, msg, uptimeMillis);
      }
  		//最终来到enqueueMessage
      private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
          msg.target = this;
          if (mAsynchronous) {
              msg.setAsynchronous(true);
          }
          return queue.enqueueMessage(msg, uptimeMillis);
      }
  
  ```

  post():

  ```
     //在这里把Runnable传给getPostMessage()
     public final boolean post(Runnable r){
         return  sendMessageDelayed(getPostMessage(r), 0);
      }
  		//在这里创建了一个message,同时把Runnable赋值给Message的callback
      private static Message getPostMessage(Runnable r) {
          Message m = Message.obtain();
          m.callback = r;
          return m;
      }
  		//在这里对MessageQueue进行赋值，并判断
      public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
          MessageQueue queue = mQueue;
          if (queue == null) {
              RuntimeException e = new RuntimeException(
                      this + " sendMessageAtTime() called with no mQueue");
              Log.w("Looper", e.getMessage(), e);
              return false;
          }
          return enqueueMessage(queue, msg, uptimeMillis);
      }
  		//最终来到enqueueMessage
      private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
          msg.target = this;
          if (mAsynchronous) {
              msg.setAsynchronous(true);
          }
          return queue.enqueueMessage(msg, uptimeMillis);
      }
  
  
  ```

  可以看到，2种方式的区别只是sendMessage直接发送了Message对象（也可以通过sendEmptyMessage发送空消息），而post则是传入了一个Runnable，然后在内部构建了一个新的Message,再把Runnable赋值给了Message的callback。也就是说pos和sendMessage的区别是它实际没有发送任何消息，只是给Message的callback进行赋值,然后最终也走到enqueueMessage()中。至于这个callback什么时候发挥作用，之后再说。

  再看enqueueMessage()中，**首先把当前handler赋值给了msg.target，也就是说，在这里把Handler和Message绑定起来了。**

  再看MessageQueue的enqueueMessage();

  ```
      boolean enqueueMessage(Message msg, long when) {
          //msg必须绑定handler
          if (msg.target == null) {
              throw new IllegalArgumentException("Message must have a target.");
          }
         //msg不能是使用状态
          if (msg.isInUse()) {
              throw new IllegalStateException(msg + " This message is already in use.");
          }
  
          synchronized (this) {
              if (mQuitting) {
                  IllegalStateException e = new IllegalStateException(
                          msg.target + " sending message to a Handler on a dead thread");
                  Log.w(TAG, e.getMessage(), e);
                  msg.recycle();
                  return false;
              }
  
              msg.markInUse();
              msg.when = when;
              Message p = mMessages;
              boolean needWake;
              if (p == null || when == 0 || when < p.when) {
                  // New head, wake up the event queue if blocked.
                  msg.next = p;
                  mMessages = msg;
                  needWake = mBlocked;
              } else {
                  // Inserted within the middle of the queue.  Usually we don't have to wake
                  // up the event queue unless there is a barrier at the head of the queue
                  // and the message is the earliest asynchronous message in the queue.
                  needWake = mBlocked && p.target == null && msg.isAsynchronous();
                  Message prev;
                  for (;;) {
                      prev = p;
                      p = p.next;
                      if (p == null || when < p.when) {
                          break;
                      }
                      if (needWake && p.isAsynchronous()) {
                          needWake = false;
                      }
                  }
                  msg.next = p; // invariant: p == prev.next
                  prev.next = msg;
              }
  
              // We can assume mPtr != 0 because mQuitting is false.
              if (needWake) {
                  nativeWake(mPtr);
              }
          }
          return true;
      }
  
  ```

  可以看到MessageQueue使用链表结构在enqueueMessage()中把消息存储起来。既然存，那么肯定得取，再看一下它取的方法:

  ```
      Message next() {
          // Return here if the message loop has already quit and been disposed.
          // This can happen if the application tries to restart a looper after quit
          // which is not supported.
          final long ptr = mPtr;
          if (ptr == 0) {
              return null;
          }
  
          int pendingIdleHandlerCount = -1; // -1 only during first iteration
          int nextPollTimeoutMillis = 0;
          for (;;) {
              if (nextPollTimeoutMillis != 0) {
                  Binder.flushPendingCommands();
              }
  
              nativePollOnce(ptr, nextPollTimeoutMillis);
  
              synchronized (this) {
                  // Try to retrieve the next message.  Return if found.
                  final long now = SystemClock.uptimeMillis();
                  Message prevMsg = null;
                  Message msg = mMessages;
                  if (msg != null && msg.target == null) {
                      // Stalled by a barrier.  Find the next asynchronous message in the queue.
                      do {
                          prevMsg = msg;
                          msg = msg.next;
                      } while (msg != null && !msg.isAsynchronous());
                  }
                  if (msg != null) {
                      if (now < msg.when) {
                          // Next message is not ready.  Set a timeout to wake up when it is ready.
                          nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                      } else {
                          // Got a message.
                          mBlocked = false;
                          if (prevMsg != null) {
                              prevMsg.next = msg.next;
                          } else {
                              mMessages = msg.next;
                          }
                          msg.next = null;
                          if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                          msg.markInUse();
                          return msg;
                      }
                  } else {
                      // No more messages.
                      nextPollTimeoutMillis = -1;
                  }
  
                  // Process the quit message now that all pending messages have been handled.
                  if (mQuitting) {
                      dispose();
                      return null;
                  }
  
                  // If first time idle, then get the number of idlers to run.
                  // Idle handles only run if the queue is empty or if the first message
                  // in the queue (possibly a barrier) is due to be handled in the future.
                  if (pendingIdleHandlerCount < 0
                          && (mMessages == null || now < mMessages.when)) {
                      pendingIdleHandlerCount = mIdleHandlers.size();
                  }
                  if (pendingIdleHandlerCount <= 0) {
                      // No idle handlers to run.  Loop and wait some more.
                      mBlocked = true;
                      continue;
                  }
  
                  if (mPendingIdleHandlers == null) {
                      mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                  }
                  mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
              }
  
              // Run the idle handlers.
              // We only ever reach this code block during the first iteration.
              for (int i = 0; i < pendingIdleHandlerCount; i++) {
                  final IdleHandler idler = mPendingIdleHandlers[i];
                  mPendingIdleHandlers[i] = null; // release the reference to the handler
  
                  boolean keep = false;
                  try {
                      keep = idler.queueIdle();
                  } catch (Throwable t) {
                      Log.wtf(TAG, "IdleHandler threw exception", t);
                  }
  
                  if (!keep) {
                      synchronized (this) {
                          mIdleHandlers.remove(idler);
                      }
                  }
              }
  
              // Reset the idle handler count to 0 so we do not run them again.
              pendingIdleHandlerCount = 0;
  
              // While calling an idle handler, a new message could have been delivered
              // so go back and look again for a pending message without waiting.
              nextPollTimeoutMillis = 0;
          }
      }
  
  ```

  这里可以看到next()里是一个死循环，有消息就取消息，没有消息就阻塞等待被唤醒。

  >  这里的死循环并不会消耗太多cpu资源，如果有消息则取，如果没有消息则是一个睡眠等待状态，直到有消息唤醒，才会继续工作，并不会消耗太多cpu资源。

  **小结一下：sendMessage()和post()的区别主要是post()没发送实际消息内容，并传递了Runnable。2个方法最终都会调用MessageQueue的enqueueMessage()将消息入队，取消息的话则是next()。**

- 2.4 处理消息，前面分析完了Hander的构建，消息的构建，然后通过enqueueMessage()让消息入队，通过next()取消息，但是整个消息机制还并没有运转起来的，让消息机制运转起来，需要调用next取消息并把消息传递给调用端，需要看Looper的loop()：

  ```
  public static void loop() {
    			//通过ThreadLocal取出当前线程的looper
          final Looper me = myLooper();
          if (me == null) {
              throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
          }
    			//通过looper获取消息队列
          final MessageQueue queue = me.mQueue;
  
  				//死循环
          for (;;) {
            	//取消息
              Message msg = queue.next(); // might block
              if (msg == null) {
                  // No message indicates that the message queue is quitting.
                  return;
              }
            
              try {
                //分发消息
                  msg.target.dispatchMessage(msg);
                  dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
              } finally {
                  if (traceTag != 0) {
                      Trace.traceEnd(traceTag);
                  }
              }
  						//回收消息，以便重复利用
              msg.recycleUnchecked();
          }
      }
  
  ```

  可以看到loop()首先获取了当前的Looper对象，然后通过Looper对象拿到消息队列MessageQueue,然后通过MessageQueue的next()获取消息，获取到消息后通过之前Message绑定的handler，调用它的dispatchMessage()来分发消息，最后回收消息以重复利用。那么看一下Handler的dispatchMessage()是如何分发的：

  ```
      public void dispatchMessage(Message msg) {
          if (msg.callback != null) {
              handleCallback(msg);
          } else {
              if (mCallback != null) {
                  if (mCallback.handleMessage(msg)) {
                      return;
                  }
              }
              handleMessage(msg);
          }
      }
  
  ```

  可以看到首先判断了Message的callback(是一个Runnable)，如果不为null则调用handleCallback(),然后直接结束，那么看一下这个方法:

  ```
      private static void handleCallback(Message message) {
          message.callback.run();
      }
  
  ```

  可以看到这个方法执行了message的callback的run()，也就是说，如果使用post()发送消息，最后会直接回调它传递的Runnable的run(),并且不会再执行Handler的handleMessage()。

  如果Message的callback是null，则会先判断mCallback是否为null，这个mCallback实际是Handler的构建方式二中的那个CallBack:

  ```java
        Handler handler=new Handler(new Handler.Callback() {
              @Override
              public boolean handleMessage(Message msg) {
                  return false;
              }
          });
  
  ```

  如果这个Callback为null则直接调用Handler的handleMessage(),如果不为null，则回调CallBack的handleMessage()，并且这个CallBack的handleMessage()是有返回值的，默认返回false,也就是默认执行了Callback的handleMessage()还会继续回调Handler的handleMessage()。如果不希望再执行Handler的handleMessage(),这里返回true就可以了。

  **小结一下：要开启消息轮训必须调用Looper.loop()方法，在这个方法中，通过Looper获取了消息队列MessageQueue，然后在一个死循环中通过MessageQueue的next()方法获取消息，获取到后就通过dispatchMessage()分发，分发时先看Message中的Runnable字段是否被赋值，如果被赋值（post传入Runnable）则直接调用Runnable的run(),分发完成，如果没有赋值，则继续看是否传入了Handler.Callback，如果没传则直接执行Handler的handleMessage()。如果传入了，则执行Handler.Callback的handMessage(),Handler.Callback的handMessage()是由返回值的，如果返回true，则直接结束分发，如果返回false则继续执行handler的handMessage()。**

  > 同样，在ActivityThread类中系统已经调用过了Looper.loop()，所以在主线程中不需要自己再调用，如果在子线程使用则需要主动调用此方法。

### 3. Handler使用流程总结

1. 构造Handler时，它的构造方法会先获取一个Looper和当前Handler进行绑定。通过Looper.prepare()可以创建一个Looper。一般在主线程使用时系统在ActivityThread里已经调用过这个方法，如果是子线程，则必须自己调用Looper.prepare()；

2. 在调用Looper.prepare()创建Looper时会创建一个MessageQueue和当前Looper绑定。同时会获取当前线程通过ThreadLocalMap和Looper也进行绑定。所以一个线程对应了一个Looper和一个MessageQueue。

3. 构造Handler时有2类方法，一类传入Looper，一类不传。一般主线程已经创建好了Looper，所以只有在子线程时才需要自己创建Looper。

4. 构造Handler时会把Looper的MessageQueue赋值给Handler的MessageQueue。所以Handler和Looper共享同一个消息队列。

5. 通过obtain()的方式构建Message。它通过缓存池的方式对Message进行重复利用，避免资源浪费。

6. 发送消息时有2类方法sendMessage()和post(),他们的主要区别是post()没有传递实际消息，只是发送了一个Runnable,消息回调时会在这个Runnable的run()里执行。

7. 发送消息最终都会先来到Handler的enqueueMessage()，在这里吧当前Handler赋值给了Message的target字段，完成了Handler和Message的绑定。

8. 然后会来到MessageQueue的enqueueMessage(),对消息进行入队操作。

9. 要处理消息必须开启循环取消息，则必须调用Looper.loop()。loop()先获取了当前线程的looper对象，再通过looper对象获取到它绑定的MessageQueue。然后再开启一个死循环，重复调用MessageQueue的next()获取消息。获取到消息后交给handler的dispatchMessage()进行分发。

   > 同样，在ActivityThread类中系统已经调用过了Looper.loop()，所以在主线程中不需要自己再调用，如果在子线程使用则需要主动调用此方法。
   >
   > next()获取消息时也会开启一个死循环获取消息，如果没有消息则进入阻塞等待状态。如果有消息则直接返回后继续获取下一条。

10. dispatchMessage()分发消息时会优先分发给Message的Runnable，如果有则直接分发并结束。如果没有则再看有没有Handler.Callback对象，如果没有则直接执行Handler的handleMessage()，如果有则先执行Handler.Callback的handleMessage(),并且看它的返回值，如果返回false,则继续执行Handler的handleMessage()，如果返回true则结束分发。