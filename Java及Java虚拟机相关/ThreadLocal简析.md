### 1. ThreadLocal简介

顾名思义，ThreadLocal的功能就是为每一个使用局部变量的线程都提供一个变量值的副本，使每一个线程都可以独立地改变自己的副本，而不会和其他的线程副本冲突。从线程的角度讲，就好像每一个线程都完全拥有该变量。

### 2. ThreadLocal源码简析

#### 2.1 ThreadLocal如何维护变量

既然ThreadLocal让每一个线程都拥有自己的变量，那么就需要维护此变量，在ThreadLocal类中，有一个内部类ThreadLocal.ThreadLocalMap来负责此工作：

```java
static class ThreadLocalMap {	
				/**
				* 定义一个弱引用Entry类，用来保存ThreadLocal和value之间的关系
				* 之所以用弱引用，是为了解决线程与ThreadLocal之间的强绑定关系，如果线程没有被回收，GC也可以回收这个对			  * 象
				*/
        static class Entry extends WeakReference<ThreadLocal<?>> {
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
    }
```

ThreadLocalMap内部通过一个Entry来保存ThreadLocal和value之间的关系，Entry是一个弱引用，用来防止内存泄漏

#### 2.2 ThreadLocal的set():

```java
    public void set(T value) {
      	//获取当前线程
        Thread t = Thread.currentThread();
      	//根据当前线程获取ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        if (map != null)
          	//存储
            map.set(this, value);
        else
            createMap(t, value);
    }
		//创建一个ThreadLocalMap
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
		//获取ThreadLocalMap
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

首先看createMap(),在这个方法中，构造了一个ThreadLocalMap，将ThreadLocal本身和要绑定的变量传入，然后将构造好的ThreadLocalMap赋值给了当前线程t,也就是说当前线程t持有了ThreadLocalMap对象。而这个ThreadLocalMap对象绑定了当前ThreadLocal和变量，形成了Thread->ThreadLocal->变量之间的绑定关系。

再看set()本身，首先获取当前线程t,根据当前线程t去获取当前的ThreadLocalMap，上面说过，当前线程t持有了当前ThreadLocalMap。所以假如ThreadLocalMap已创建，是可以获取到的。如果没创建就去创建。如果创建了就调用ThreadLocalMap.set()将当前ThreadLocal和变量绑定。

### 2.3 ThreadLocal的get():

```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

get()中首先获取了当前线程t,然后通过getMap()，获取了当前线程的ThreadLocalMap，然后再通过map.getEntry(this)，传入当前ThreadLocal取出与之绑定的变量。

