### 先看RxJava的简单使用及解析：

```
       	//产生事件并返回Single对象,Single和Observable是一样的作用,不一样的地方是Single只回调onSuccess(),不会回调onError()
       	Single<Integer> just = Single.just(1);
				//订阅
        just.subscribe(new SingleObserver<Integer>() {
            @Override
            public void onSubscribe(Disposable d) {
                
            }

            @Override
            public void onSuccess(Integer integer) {
						
            }

            @Override
            public void onError(Throwable e) {

            }
        });
```

以上是Rxjava的一个简单示例，第一步通过Single.just()发送一个事件，第二部调用subscribe()订阅事件。

**先看第一步Single.just():**

```
    @CheckReturnValue
    @SchedulerSupport(SchedulerSupport.NONE)
    public static <T> Single<T> just(final T item) {
        ObjectHelper.requireNonNull(item, "value is null");
      	//构建一个SingleJust返回
        return RxJavaPlugins.onAssembly(new SingleJust<T>(item));
    }

		//钩子方法，如果设置了onSingleAssembly，那么可以通过apply()对数据进行再加工，默认没设置onSingleAssembly，不必关注此方法
    public static <T> Single<T> onAssembly(@NonNull Single<T> source) {
        Function<? super Single, ? extends Single> f = onSingleAssembly;
        if (f != null) {
            return apply(f, source);
        }
        return source;
    }
```

可以看到只是构造了一个SingleJust()对象，传入事件并返回。

那么第一步简单总结一下为：构造新的被观察者SingleJust，并传递事件

```java
public final class SingleJust<T> extends Single<T> {

    final T value;

    public SingleJust(T value) {
        this.value = value;
    }



}
```

可以看到SingleJust在构造里记录了数据，并且它还有一个subscribeActual(),这个方法里回调了onSubscribe()及onSuccess();

**那么简单总结一下，第一步Single.just(1)执行后，构造里一个SingleJust对象，并存储了数据。所以返回的Single其实已经是SingleJust对象了，那么被观察者对象已经切换为SingleJust**

**再看第二步just.subscribe()：**

```
    @SchedulerSupport(SchedulerSupport.NONE)
    @Override
    public final void subscribe(SingleObserver<? super T> observer) {
				//钩子方法，和上面一样，不必关注。
        observer = RxJavaPlugins.onSubscribe(this, observer);
        try {
          	//调用subscribeActual，传入observer
            subscribeActual(observer);
        } catch (Throwable ex) {
            throw npe;
        }
    }
```

第二部是订阅，它实际执行了subscribeActual(),并传入了下游的观察者，由于这里已经是SingleJust，那么执行的就是它的subscrieActual():

```
    @Override
    protected void subscribeActual(SingleObserver<? super T> observer) {
      	//此方法非核心，忽略
        observer.onSubscribe(Disposables.disposed());
      	//执行观察者的onSuccess()
        observer.onSuccess(value);
    }
```

可以看到，最后实际执行了下游观察者的onSuccess()

总结：

以上就是RxJava的一个简单模型过程，当调用一个操作符后，被观察者对象就会改变，同时事件从上往下传递。当产生订阅关系时，下游观察者在上游被观察者的subscribeActual()中获取结果**

为了更好理解复杂的情况，这里再明确一个概念，下游观察者是当前被观察者调用subscribe()时传入的参数。

Rxjava中其他复杂的操作，其实就是操作符的改变及累加，只是在此模型上增加中间过程，如处理数据等。

### 稍复杂的RxJava使用及解析

```
        Single.just("1")
                .map(new Function<String, Integer>() {
                    @Override
                    public Integer apply(String s) throws Exception {
                        return Integer.valueOf(s);
                    }
                }).subscribe(new SingleObserver<Integer>() {
            @Override
            public void onSubscribe(Disposable d) {

            }

            @Override
            public void onSuccess(Integer integer) {

            }

            @Override
            public void onError(Throwable e) {

            }
        });

```

Single.just("1")这一行在上面分析过了，主要是把Single转换为SingleJust，并传递数据。那么再看第二行map():

```
    public final <R> Single<R> map(Function<? super T, ? extends R> mapper) {
        ObjectHelper.requireNonNull(mapper, "mapper is null");
      	//创建了SingleMap对象，并传入了this,和mapper参数，this就是SingleJust对象，mapper处理数据用，用来回调apply()
        return RxJavaPlugins.onAssembly(new SingleMap<T, R>(this, mapper));
    }
```

可以看到，这一步又把SingleJust转换为了一个SingleMap，那么现在被观察者就转换为了SingleMap，那么现在可以把被观察者SingleJust称为上游被观察者，同时把上游被观察者和处理数据的mapper,存入了SingleMap中。

第三行开始订阅，在这里被观察者已经变成了SingleMap，那么会执行它的subscribeActual()：

```
    @Override
    protected void subscribeActual(final SingleObserver<? super R> t) {
      	//这里的source其实是SingleJust，然后订阅了观察者MapSingleObserver，由于观察者是由下方的订阅产生的，这里称观察者t为下游观察者
        source.subscribe(new MapSingleObserver<T, R>(t, mapper));
    }
```

如果这里回调观察者的onSuccess()，那么就和之前总结的没有区别，但是这里调用了source.subscribe(),由上游被观察者再次订阅。也就是说，产生订阅关系后，事件开始从下往上传递，对数据进行处理。

根据之前的代码source.subscribe()，最终会执行source的subscribeActual()，再执行到观察者的onSuccess(),也就是MapSingleObserver的onSuccess()：

```
static final class MapSingleObserver<T, R> implements SingleObserver<T> {
				//观察者
        final SingleObserver<? super R> t;
				//处理数据对象
        final Function<? super T, ? extends R> mapper;
        @Override
        public void onSuccess(T value) {
            try {
              	//这里调用apply()处理数据。
                v = ObjectHelper.requireNonNull(mapper.apply(value), "The mapper function returned a null value.");
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
              //如果出错调用onError()
                onError(e);
                return;
            }
					//处理完成，下游观察者调用onSuccess()
            t.onSuccess(v);
        }
 
}
```

可以看到在onSuccess()中，先对数据进行处理，如果数据没出错，那么再把数据交给最终的观察者。

总结：根据上面的代码再完善一下简单模型，首先当发送一个事件后，事件开始从上游往下传递，传递过程中会由当前调用的操作符暂代被观察者功能，当传递完成后代表设置完成。然后开始订阅事件，发起订阅后那么事件开始从下往上传递，对之前的设置进行处理，最后处理完后成，事件再从上往下传递给最终的观察者。那么最终模型就是：

**从上往下(开始传递事件，初始化操作符)—>从下往下(连接操作符，并进行部分设置)->从上往下(根据操作符对数据进行处理并返回）**

### 根据模型，分析完整示例

```
        Single.just("1")
                .map(new Function<String, Integer>() {
                    @Override
                    public Integer apply(String s) throws Exception {
                        return Integer.valueOf(s);
                    }
                })
                //切换到io线程
                .subscribeOn(Schedulers.io())
          			//切换到主线程
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new SingleObserver<Integer>() {
            @Override
            public void onSubscribe(Disposable d) {

            }

            @Override
            public void onSuccess(Integer integer) {

            }

            @Override
            public void onError(Throwable e) {

            }
        });

```

这里只为分析过程，忽律当前代码并不需要切线程的操作。

**第一步从上往下，初始化操作符:**

map()分析过了，那么直接看subscribeOn():

```
    public final Single<T> subscribeOn(final Scheduler scheduler) {
      	//构建一个SingleSubscribeOn返回，scheduler就是要被设置的参数
        return RxJavaPlugins.onAssembly(new SingleSubscribeOn<T>(this, scheduler));
    }
```

根据之前的分析，那么就是subscribeOn暂时接管了被观察者的职能，subscribeOn完成了设置，继续看observeOn():	

```
    @CheckReturnValue
    @SchedulerSupport(SchedulerSupport.CUSTOM)
    public final Single<T> observeOn(final Scheduler scheduler) {
        //构建一个SingleObserveOn返回，scheduler就是要被设置的参数
        return RxJavaPlugins.onAssembly(new SingleObserveOn<T>(this, scheduler));
    }
```

根据之前的分析，那么就是observeOn暂时接管了被观察者的职能，此时完成了第一步，事件从上到下传递的设置功能。

**第二步从下往上，开始订阅，连接操作符**

调用subscribe()，由于最后一个暂时接管被观察者功能的是observeOn，所以是由它进行的订阅，那么根据之前的代码可知，会执行SingleObserveOn的subscribeActual():

```
    @Override
    protected void subscribeActual(final SingleObserver<? super T> observer) {
      	//调用上游订阅方法，这里的source是subscribeOn
        source.subscribe(new ObserveOnSingleObserver<T>(observer, scheduler));
    }
```

可以看到，调用了上游的subscribe()，那么会再执行上游的subscribeActual(),那么会来到SingleSubscribeOn的subscribeActual():

```
    @Override
    protected void subscribeActual(final SingleObserver<? super T> observer) {
      	//构造一个SubscribeOnObserver
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer, source);
        observer.onSubscribe(parent);
				
        Disposable f = scheduler.scheduleDirect(parent);

        parent.task.replace(f);

    }
```

这里先看一下这个SubscribeOnObserver是什么：

```java
    static final class SubscribeOnObserver<T>
    extends AtomicReference<Disposable>
    implements SingleObserver<T>, Disposable, Runnable
```

可以看到它时一个Runnable对象，那么接上面的第二行：

```
//切换线程，这里具体实现不必深究，只需知道是完成了切换线程操作就行
Disposable f = scheduler.scheduleDirect(parent);
```

可以看到，这是把上面的Runnable对象parent，传了进去。这里面其实使用了Executors进行了线程切换操作

既然是Runnable，那么任务开始执行就会触发它的run()：

```
        @Override
        public void run() {
          //这里的source是SingleMap
            source.subscribe(this);
        }
```

subscribeOn()的实际作用就是切线程， 那么它的设置就在这里生效。另外由于不管下游有什么设置，都会在这里进行切线程操作，然后再进行订阅，那么也就可以得出一个结论：**无论subscribeOn()设置多少次，只有在第一次有效，因为从下往上传播最终都会回到第一次设置的地方进行切线程操作**。

可以看到，继续往上传递，那么会来到SingleMap的subscribeActual():

```
    @Override
    protected void subscribeActual(final SingleObserver<? super R> t) {
      	//这里的source是SingleJust
        source.subscribe(new MapSingleObserver<T, R>(t, mapper));
    }
```

还是继续往上传递，来到SingleJust的subscribeActual():

```
    @Override
    protected void subscribeActual(SingleObserver<? super T> observer) {
        observer.onSubscribe(Disposables.disposed());
      	//这里的observer是MapSingleObserver
        observer.onSuccess(value);
    }
```

最终调用了观察者的onSuccess()，那么这里完成了第二部，从下往上，对数据进行相应的操作处理

**第三步，从上往下，处理数据**：

根据之前的总结:**下游观察者是当前被观察者调用subscribe()时传入的数(之后的推论同理，不再强调)**,那么最后传入的参数的map()操作符中的MapSingleObserver,那么就会执行它的onSuccess():

```
        @Override
        public void onSuccess(T value) {
            R v;
            try {
                v = ObjectHelper.requireNonNull(mapper.apply(value), "The mapper function returned a null value.");
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
                onError(e);
                return;
            }
				//这里的t是MapSingleObserver
            t.onSuccess(v);
        }
```

可以看到，这里首先调用了apply(),让用户对数据进行处理，那么操作符map,已经完全发挥了作用。继续往下，根据之前的总结可得知t是subscribeOn()操作符中的SubscribeOnObserver，再看它的onSuccess：

```
        @Override
        public void onSuccess(T value) {
          //这里的downstream是ObserveOnSingleObserver
            downstream.onSuccess(value);
        }
```

这里没有操作，直接向下传递是因为之前已经分析过，它的作用是切线程，并且已经切过了。上面分析过。那么继续往下，downstream是observeOn()操作符的ObserveOnSingleObserver(),再看它的onSuccess：:

```
        public void onSuccess(T value) {
            this.value = value;
          	//切换线程
            Disposable d = scheduler.scheduleDirect(this);
            DisposableHelper.replace(this, d);
        }
```

observeOn的作用也是切换线程，所以这里也进行了切线程操作，并把this作为参数，那么就是看它自身的run():

```
        @Override
        public void run() {
            Throwable ex = error;
            if (ex != null) {
                downstream.onError(ex);
            } else {
            	 //这里的downstream是最终的观察者
                downstream.onSuccess(value);
            }
        }
```

这里可以看到，切换线程后将结果交给了最终的观察者。完成了整个流程。**observeOn()的线程是多次有效的，从上面的代码可以看出，切完线程后继续将结果传递给下游观察者，假如继续调用observeOn()，那么就会继续切线程，结果也是在下游的run()中执行，所以切线程有效。**

### 总结：

**RxJava总体流程可以概括为以下三步，复杂的操作只是增加中间环节，以及中间环节的各种细化处理**

- 第一步流程-从上往下，初始化操作符，对所有操作符进行初始化
- 第二步流程-从下往上，开始订阅，并连接操作符，对连接的操作符进行设置，此例中主要是异步请求时切换线程，
  - subscribeOn()的设置只有第一次生效，因为在subscribeOn中会切换线程，然后进行订阅。这时的流程时从下往上，最终都会回到第一次的设置。所以之前不管切到哪个线程，最终又会由第一次的设置切回它所设置的线程。
- 第三步流程-从上往下，处理数据，获得数据后由各个操作符对数据进行加工处理，最终传递给最终的观察者。
  - observeOn()每次设置都会起效果，因为observeOn()发挥作用是最后一步从上往下的过程中，所以它每一次切完下次，然后再传递到下一层，下一次同样可以切线程操作。所以如果要对结果再次进行切线程操作，可多次使用observeOn()


