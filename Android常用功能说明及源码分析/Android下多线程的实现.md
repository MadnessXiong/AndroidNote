### Android下多线程相关

**线程安全相关问题参考：**[java内存模型与线程](https://github.com/MadnessXiong/AndroidNote/blob/master/Java及Java虚拟机相关/Java内存模型与线程.md)

**android下与多线程有关的主要有以下几个类：**

* Handler：主要用于线程间通信（常用）参考：[Handler的使用及调用流程源码分析](https://github.com/MadnessXiong/AndroidNote/blob/master/Android常用功能说明及源码分析/Handler的使用及调用流程源码分析.md)

* HandlerThread：继承了Thread，实际上就是封装了Handler的Thread。（基本不用）

* AsyncTask：轻量级的异步任务类，它可以在线程池中执行后台任务，然后把执行的进度和最终结果传递到主线程（基本废弃）

* IntentService：是一种特殊的Service,它执行完成后就会自行销毁。可以用来执行耗时任务，由于是Service，所以优先级较高，不会轻易被系统杀死。它的内部其实封装了Handler和HandlerThread。（常用）

* Executor：线程池，多线程的主要使用方式

由于多线程操作目前主要由线程池方式实现，所以只重点关注Executors

### 线程池

**1. 线程池的优势：**

* 重用线程池中的线程，避免因为线程的创建和销毁所带来的性能开销

* 有效控制线程池的最大并发数，避免大量的线程之前因相互之间抢占系统资源而导致阻塞现象

* 能够对线程池进行简单管理，并提供定时执行以及指定间隔循环执行等功能

**2. 线程池的实现**

**ThreadPoolExecutor是线程池的真正实现。看一下它的主要参数：**

* corePoolSize：线程池的核心线程数，默认情况下，核心线程会在线程池中一直存活，即使他们处于闲置状态。如果将ThreadPoolExecutor的allowCoreThreadTimeOut属性设置为true,那么闲置的核心线程在等待新任务到来时会有超时策略，这个时间由keepAliveTime决定，超过keepAliveTime指定的时间后，核心线程就会被终止。

* maximumPoolSize：线程池能容容纳的最大线程数，当活动线程达到这个数值后，后续任务将会被阻塞

* keepAliveTime：非核心线程闲置时的超时时长，如果超过这个时长，非核心就会被回收。当allowCoreThreadTimeOut为true时，也会作用于核心线程。

* unit：用于指定keepAliveTime参数的时间单位

* workQueue：线程池中的任务队列，通过线程池的execute方法提交的Runnable对象会存储在这个参数中

* threadFactory：线程工厂，为线程池提供创建新线程的功能。ThreadFactory是一个接口，只有一个方法newThread(Runnable r)。

**ThreadPoolExecutor执行时大致遵循如下规则：**

* 如果线程池中的线程数量未达到核心线程数量，那么会直接启动一个核心线程来执行任务。

* 如果线程池中的线程数量已经达到或者超过核心线程的数量，那么任务就会被插入到任务队列中排队等待执行

* 如果任务队列已满，这时如果线程数量未达到线程池规定的最大数量，那么会立即启动一个非核心线程来执行任务

* 如果以达到线程数量已达到线程池的最大值，那么就拒绝执行任务。会通过RejectedExecutionException来通知调用者。

**3. 线程池的分类**

**Java默认实现了4种线程池，它们都是通过配置ThreadPoolExecutor实现的。**

* Executors.newFixedThreadPool()：一种数量固定的线程池。当线程处于空闲状态时，它们并不会被回收，除非线程池关闭了。当所有的线程都处于活动状态时，新任务会处于等待状态，直到有线程空闲出来。由于只有核心线程并且这些核心线程不会被回收，意味着它能更快响应外界的请求。

* Executors.newCachedThreadPool()：数量不定的线程池，并且最大线程数量为Integer.MAX_VALUE，基本意味着无限大。当线程池中的线程池都处于活动状态时，线程池会创建新的线程来处理新任务，否则就会利用空闲线程来处理任务。线程池的空闲线程都有超时机制，这个超时时长为60秒，因为超时就会被回收，所以比较适合执行大量耗时较少的任务。

* Executors.newScheduledThreadPool()：这个线程池的核心线程数量是固定的，而非核心线程是没有限制的。并且当非核心线程闲置时会被立即回收。它主要用于执行定时任务或具有固定周期的重复任务。

* Executors.newSingleThreadExecutor()：这个线程池内部只有一个核心线程，它确保所有任务都在一个线程池中按顺序执行。它的意义在于统一所有外界的任务到一个线程中，使得这些任务之间不需要处理线程同步的问题。

### 附：Thread与Runnable的区别

* Thread是类，Runnable是接口。Thread继承自Runnable

* Runnable接口可以避免java单继承带来的局限

* Runnable适合多个相同的程序代码的线程去处理同一个资源

* Runnable可以增加程序的健壮性，代码可以被多个线程共享，代码和数据独立。如线程池就是接收一个Runnable参数




