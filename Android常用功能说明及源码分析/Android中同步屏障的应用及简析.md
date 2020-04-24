### 同步屏障的应用

```java
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true
            //通过postSyncBarrier()设置同步屏障,
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
          	//handler发送消息，有了同步屏障mTraversalRunnable就会被优先执行。
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        }
    }
```

ViewRootImpl.scheduleTraversals()里发送了Handler消息，最终会执行TraversalRunnable的run(),在这个run()中会执行doTraversal()，最终会触发View的绘制流程：measure(),layout(),draw()。为了让绘制流程尽快被执行，用到了同步屏障技术。

### 源码分析

开启同步屏障是通过MessageQueue.postSyncBarrier():

```java
    private int postSyncBarrier(long when) {
        synchronized (this) {
            final int token = mNextBarrierToken++;
          	//构造一个Message，没有对msg.target赋值,target是Handler
            final Message msg = Message.obtain();
          	
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
           //将该message插入表头
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
```

这里构造了一个Message，并且没有指定msg.target,最后将这个Message插入表头。这个Message就构成了一个内存屏障。

再看它是如何构成内存屏障的。根据[Handler的使用及调用流程源码分析](https://github.com/MadnessXiong/AndroidNote/blob/master/Android常用功能说明及源码分析/Handler的使用及调用流程源码分析.md)可知，取消息调用的是MessageQueue.next():

```java
    Message next() {
        for (;;) {
            synchronized (this) {
                Message prevMsg = null;
                Message msg = mMessages;
              	//根据msg.target是否为null判断内存屏障是否存在。
                if (msg != null && msg.target == null) {
                 
                    do {
                      //找出异步消息
                        prevMsg = msg;
                        msg = msg.next;
                     //根据isAsynchronous判断是否是异步消息，是异步消息则跳出循环
                    } while (msg != null && !msg.isAsynchronous());
                }
										//将消息返回
                    return msg;
                } 

    }
```

在next()中如果设置了同步屏障，那么就会通过do..while()循环优秀去找消息列表中的异步消息，找到后返回。

所以所有的异步消息都处理完后，才会处理同步消息。同步屏障就是添加了一个标识，这个标识是一个没有target的Message。如果有这个标识，就先去处理异步消息。

再看异步消息如何发送，发送消息会调到Handler.enqueueMessage()：

```java
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
      	//mAsynchronous只能通过Handler构造设置
        if (mAsynchronous) {
          	//标记为异步消息
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

根据mAsynchronous判断是否需要标记为异步消息，这个mAsynchronous变量可以在Handler的构造中设置，一旦设置了以后，该Handler发送的所有消息都是异步消息，不能修改。

如果需要同步异步消息都发送，可以通过构造普通Handler，然后发送消息时设置msg.setAsynchronous(true)将消息标记为异步消息。