**LeakCanary原理概述：** 在Java中，一个对象不再被任何GC roots引用，那么虚拟机在gc时就会被回收。 如果一个对象已经完成了它的任务，但是还是被其他引用所引用，对象所占用的内存得不到释放，那么就会造成内存泄露。

Java中提供了WeakReference(弱引用)，当一个对象只有被弱引用引用时，那么在gc发生时就会被回收()，弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收器回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。

LeakCanary的原理就是这样，判断一个对象是否内存泄漏，先把这个对象添加到一个WeakReference集合中，然后调用gc,把gc后的对象放入ReferenceQueue中,通过ReferenceQueue中是否存在这个引用，来判断对象是否被回收，如果没被回收，则代表发生了内存泄漏。

**LeakCanary源码分析：**

先看代码：

```
	  if (LeakCanary.isInAnalyzerProcess(this)){
            return;
        }
        LeakCanary.install(this);
		//备注：LeakCanary2.0以后已经不需要自己去设置以上2行代码了
```
LeakCanary的使用只需要在Application里执行以上代码，那么看一下install()的内部实现：

```
  public static @NonNull RefWatcher install(@NonNull Application application) {
    			//第一行主要是添加内存泄漏时，顶部的通知栏逻辑，不重点关注
    return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
      		//第二行主要是过滤一些无法处理的内存泄漏，比如android本身的内存泄漏，已知的第三方库的内存泄漏等
        .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
      	//创建并执行LeakCanary
        .buildAndInstall();
  }
```

那么重点看一下第三行：

```
 public @NonNull RefWatcher buildAndInstall() {
    //创建一个watcher，不必重点关注这里的实现
    RefWatcher refWatcher = build();
    	//是否监听Activity
      if (watchActivities) {
        ActivityRefWatcher.install(context, refWatcher);
      }
     //是否监听Activity
      if (watchFragments) {
        FragmentRefWatcher.Helper.install(context, refWatcher);
      }
    }
    LeakCanaryInternals.installedRefWatcher = refWatcher;
    return refWatcher;
  }
```
这里以如何监听Activity的内存泄漏为例,看一下ActivityRefWatcher的install()：

```
 public static void install(@NonNull Context context, @NonNull RefWatcher refWatcher) {
    Application application = (Application) context.getApplicationContext();
    //对watcher进行包装
    ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);
    //注册ActivityLifecycleCallbacks
    application.registerActivityLifecycleCallbacks(activityRefWatcher.lifecycleCallbacks);
  }
```

可以看到这里先是把RefWatcher包装成了一个ActivityRefWatcher，然后注册了ActivityLifecycleCallbacks,那么当activity的onDestory()执行时，也就是activity销毁时，就可以通过传入的lifecycleCallbacks收到它的回调。那么看一下代码：

> ActivityLifecycleCallbacks一经注册，就会自动监听整个APP 中所有Activity 的生命周期方法的执行（会在对应的父类Activity的方法被执行之后自动触发）。

```
  private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
      new ActivityLifecycleCallbacksAdapter() {
        @Override public void onActivityDestroyed(Activity activity) {
          //watch当前activity对象
          refWatcher.watch(activity);
        }
      };
```
这里调用了RefWatcher的watch()，并把需要watch的对象activity传了进来：

```
 public void watch(Object watchedReference) {
    watch(watchedReference, "");
  }
  public void watch(Object watchedReference, String referenceName) {
    if (this == DISABLED) {
      return;
    }
    //生成一个唯一ID
    String key = UUID.randomUUID().toString();
    //将唯一ID添加到一个Set集合中，只要代表对象的key还存在这个集合中，那么代表对象还没被回收
    retainedKeys.add(key);
    //KeyedWeakReference是一个弱引用，这里把key和watchedReference以及queue(ReferenceQueue)包装成一个弱引用对象
    final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);
		//核心逻辑
    ensureGoneAsync(watchStartNanoTime, reference);
  }
```
可以看到，首先记录了开始watch的时间，然后生成了一个唯一ID用作key,用来和被watch的对象绑定。然后把被watch对象包装成一个弱引用对象。再看ensureGoneAsync():

```
 private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    //这里的watchExecutor实际是AndroidWatchExecutor
    watchExecutor.execute(new Retryable() {
      @Override public Retryable.Result run() {
        //这里执行在子线程中
        return ensureGone(reference, watchStartNanoTime);
      }
    });
  }
```

可以看到，实际是执行了AndroidWatchExecutor的execute():

```
	@Override public void execute(@NonNull Retryable retryable) {
    if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
      waitForIdle(retryable, 0);
    } else {
      postWaitForIdle(retryable, 0);
    }
  }
  private void waitForIdle(final Retryable retryable, final int failedAttempts) {
    // This needs to be called from the main thread.
    //通过addIdleHandler的方式，让主线程在空闲时才去执行postToBackgroundWithDelay()
    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
      @Override public boolean queueIdle() {
        postToBackgroundWithDelay(retryable, failedAttempts);
        return false;
      }
    });
  }

  private void postToBackgroundWithDelay(final Retryable retryable, final int failedAttempts) {
    long exponentialBackoffFactor = (long) Math.min(Math.pow(2, failedAttempts), maxBackoffFactor);
    long delayMillis = initialDelayMillis * exponentialBackoffFactor;
    //backgroundHandler绑定的是一个HandlerThread的looper,所以这里将线程切换到了子线程
    backgroundHandler.postDelayed(new Runnable() {
      @Override public void run() {
        Retryable.Result result = retryable.run();
        if (result == RETRY) {
          postWaitForIdle(retryable, failedAttempts + 1);
        }
      }
    }, delayMillis);
  }
```

可以看到在execute()中首先首先在主线程空闲时才去执行postToBackgroundWithDelay()，这是因为主线程空闲了，那么被watch的对象，也就是activity才真正被释放了。

最后再把线程切到子线程执行run(),最终会执行刚才的ensureGone():

> activity的onDestory执行时，不会从activity集合中删除，执行完后才会再去执行删除的代码。所以为了保证activity不再被引用，必须等到主线程空闲时，也就是activity被删除引用的代码也被执行了，而不是onDestory执行完。详见ActivityThread类的performDestroyActivity()。

```
  Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
		//将已经gc掉的对象代表的key移除
    removeWeaklyReachableReferences();
	
    if (gone(reference)) {
      	//如果对象已被回收，那么代表不存在内存泄漏，直接返回
      return DONE;
    }
    //主动进行gc
    gcTrigger.runGc();
    ////将已经gc掉的对象代表的key移除
    removeWeaklyReachableReferences();
    //判断被观察到对象是否被回收
    if (!gone(reference)) {
      //如果未回收，则代表已内存泄漏，进行分析堆文件等操作，这里不再细致展开。
      long startDumpHeap = System.nanoTime();
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

      File heapDumpFile = heapDumper.dumpHeap();
      if (heapDumpFile == RETRY_LATER) {
        // Could not dump the heap.
        return RETRY;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);

      HeapDump heapDump = heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key)
          .referenceName(reference.name)
          .watchDurationMs(watchDurationMs)
          .gcDurationMs(gcDurationMs)
          .heapDumpDurationMs(heapDumpDurationMs)
          .build();

      heapdumpListener.analyze(heapDump);
    }
    return DONE;
  }
	//移除已经被gc掉的对象的key
  private void removeWeaklyReachableReferences() {
		//如果queue中存在这个对象，那么代表已经被回收，那么从保存的set集合retainedKeys中移除代表它的key
    KeyedWeakReference ref;
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
      retainedKeys.remove(ref.key);
    }
  }
	//判断是代表集合的key是否还存在，代表对象是否已经被回收，true代表已回收，false代表未回收
  private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
  }
```
可以看到，到这里就可以判断出是否发生了内存泄漏。

### 总结

- 首先注册了ActivityLifecycleCallbacks，那么当activity被销毁时，也就是它的onDestory()被执行时，LeakCanary可以收到回调
- 收到回调后调用watch()，对当前activity进行观察，并生成一个唯一ID-key，保存到Set集合retainedKeys中。retainedKeys中存储了被观察且未被回收的对象的key。
- 将key和被观察到对象包装成一个KeyedWeakReference(弱引用)对象，并传入一个ReferenceQueue对象queue，那么当对象被回收后，就会放入这个queue中。
- 然后执行了AndroidWatchExecutor的execute(),这个方法中主要做了2件事，1是确保在主线程空闲时开始执行任务，因为当Activity执行onDestory时，它还没有被移除，需要等它执行完后才会被移除。那么主线程空闲代表已经执行完了方法。2是将线程切换到了自线程。
- 线程切换到子线程后执行了ensureGone(),在这个方法中主动调用gc,gc后所有该被回收到对象已被回收，这时调用removeWeaklyReachableReferences()，这个方法中会从之前的ReferenceQueue对象queue中取对象，前面说过如果对象被回收，就会放入这个queue中，那么如果从这个queue中取出了对象，那么就把这些对象对应的key从retainedKeys中移除。
- 再调用gone()方法，判断当前activity对应的key是否还在retainedKeys中，如果不在，代表已经被回收。如果还在，代表发生了内存泄漏。
- 如果发生了内存泄漏，那么会进一步分析堆文件，找出泄漏的对象，发送通知等操作。(这里不是重点，不再细致展开)
