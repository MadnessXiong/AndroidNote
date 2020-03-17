### Lifecycle的使用
```
       	  Lifecycle lifecycle = getLifecycle();
					lifecycle.addObserver(new DefaultLifecycleObserver() {
            @Override
            public void onCreate(@NonNull LifecycleOwner owner) {

            }

            @Override
            public void onStart(@NonNull LifecycleOwner owner) {

            }

            @Override
            public void onResume(@NonNull LifecycleOwner owner) {

            }

            @Override
            public void onPause(@NonNull LifecycleOwner owner) {

            }

            @Override
            public void onStop(@NonNull LifecycleOwner owner) {

            }

            @Override
            public void onDestroy(@NonNull LifecycleOwner owner) {

            }
        });

```
直接在Activity中添加以上代码，那么当Activity的状态改变时，相应的回调就会执行

### Lifecycle源码简析

可以看到这是一个观察者模式，Activity是最终的被观察者，当它的状态发生改变时，通知观察者。

那么只需要了解这个观察者机制是如何运转的就可以了，也就是说观察者是如何注册的，状态发生时是怎么获取的(因为Activity并不是直接的观察者，但又要获取它的状态)，以及状态发生时是是如何分发消息的。

**1. 注册观察者：**

可以看到在Activity中通过getLifecycle()获取一个Lifecycle对象，然后再执行addObserver()添加一个观察者,这个观察者是一个LifecycleObserver对象，Android提供了默认实现接口DefaultLifecycleObserver。

首先看getLifecycle()：
```
public class SupportActivity extends Activity implements LifecycleOwner, Component {
    private SimpleArrayMap<Class<? extends SupportActivity.ExtraData>, 
    private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
      	//这里随后分析
        ReportFragment.injectIfNeededIn(this);
    }

    @CallSuper
    protected void onSaveInstanceState(Bundle outState) {
        this.mLifecycleRegistry.markState(State.CREATED);
        super.onSaveInstanceState(outState);
    }

    public Lifecycle getLifecycle() {
        return this.mLifecycleRegistry;
    }
}
```
可以看到最终调用了SupportActivity的getLifecycle()，返回了一个LifecycleRegistry对象。所以当执行addObserver()时，执行的是Lifecycle的实现类LifecycleRegistry的addObserver()：

```
    public void addObserver(@NonNull LifecycleObserver observer) {
      	//对观察者observer进行包装
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
      	//存储观察者
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);
    }

    @Override
    public V putIfAbsent(@NonNull K key, @NonNull V v) {
        Entry<K, V> current = get(key);
        if (current != null) {
            return current.mValue;
        }
      //存储对象
        mHashMap.put(key, put(key, v));
        return null;
    }
		//把对象包装成Entry
    protected Entry<K, V> put(@NonNull K key, @NonNull V v) {
      	//省略代码细节
        return newEntry;

    }
```
可以看到addObserver()中先对观察者observer进行包装，方便后期的分发事件，然后把观察者observer存入了mObserverMap中，mObserverMap是一个FastSafeIterableMap对象，内部把observer包装成了一个Entry，并通过HashMap的方式存储。

**2. 获取状态：**

在SupportActivity有这样一段代码：

```
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ReportFragment.injectIfNeededIn(this);
    }
```

再看一下这个injectIfNeededIn()：

```
    public static void injectIfNeededIn(Activity activity) {
        // ProcessLifecycleOwner should always correctly work and some activities may not extend
        // FragmentActivity from support lib, so we use framework fragments for activities
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            // Hopefully, we are the first to make a transaction.
            manager.executePendingTransactions();
        }
    }
```

可以看到，添加了一个ReportFragment到当前activity，在Android中当activity生命周期变化时，依附它的fragment也会发生对应的变化。那么只要在fragment的状态发生时通知观察者就可以了。如下：

```
    @Override
    public void onResume() {
        super.onResume();
        dispatch(Lifecycle.Event.ON_RESUME);
    }

    private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }
```
最终会调用到LifecycleRegistry的handleLifecycleEvent(),并把当前状态传进去。

**3. 分发消息：**

消息分发会从LifecycleRegistry的handleLifecycleEvent()开始，看下代码：

```
  public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        moveToState(next);
    }
    private void moveToState(State next) {
        if (mState == next) {
            return;
        }
        sync();
    }
    private void sync() {
        while (!isSynced()) {
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                backwardPass(lifecycleOwner);
            }
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
  
    }
		//backwardPass()和forwardPass()代码差不多，这里只展示forwardPass()
    private void forwardPass(LifecycleOwner lifecycleOwner) {
      	//获取迭代器
        Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
                mObserverMap.iteratorWithAdditions();
        while (ascendingIterator.hasNext() && !mNewEventOccurred) {
          	//获取观察者，前面说过观察者observer被包装成了一个Entry对象存储，在这里取出
            Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                pushParentState(observer.mState);
                observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
                popParentState();
            }
        }
    }
```
通过一系列调研，最终会通过mObserverMap，取出之前存的观察者observer，调用它的dispatchEvent()进行分发。(前面说过对observer进行了包装，所以它具有了分发的能力)，看下代码：

```
        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
          //里面还有相对比较复杂的逻辑，这里不再展开
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
```
最后通过onStateChanged()进行回调(内部逻辑不再展开)。