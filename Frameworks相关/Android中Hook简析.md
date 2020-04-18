### Hook简介

Android中的系统调用是无法直接修改的，如启动PlugActivity时，动态地将它改为启动LocalActivity。这就要用到Hook技术。

Hook技术是指当A和B交互时，可以通过劫持A或的B的对象或方法，通过代理的方式对它进行功能的扩展或删除等操作，最后再完成交互，

被Hook的对象或方法称为Hook点，为了保证Hook点稳定性，Hook点一般选择容易找到且不易变化的对象，静态变量和单例是一般常用的Hook点。

### Hook实现示例

如Activity启动另一个Activity时会在Activity里调用到mInstrumentation.execStartActivity()。这个mInstrumentation是一个单例对象，那么它可以很好地作为一个hook点。

找好了Hook点，然后把mInstrumentation替换为代理对象。然后启动时就是执行的代理对象的execStartActivity()，那么就可以在自定义的代理类中先进行扩展功能或修改的操作，这里就是将要启动的LocalActivity替换为PlugActivity。修改完后再执行最初的mInstrumentation.execStartActivity()。

那么就看具体实现的代码，第一步先定义一个代理类：

```java
public class InstrumentationProxy extends Instrumentation {
    Instrumentation instrumentation;
    private Method method;
		//持有真正启动Activity的Instrumentation引用
    public InstrumentationProxy(Instrumentation instrumentation) {
        this.instrumentation = instrumentation;
    }
  	//重写启动Activity的execStartActivity()
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
      		 //将要启动的PlugActivity修改为LocalActivity
      		 intent.setClassName(who,"com.madnessxiong.client.LocalActivity");
      	//通过反射获取Instrumentation的class对象
        Class<? extends Instrumentation> aClass = Instrumentation.class;
        try{
          //通过class对象获取真正启动Activity的execStartActivities()
          method = aClass.getMethod("execStartActivities");
         //通过反射的方式执行真正启动Activity的execStartActivities()
        ActivityResult invoke = (ActivityResult) method.invoke(instrumentation, who, contextThread, token, target, intent, options);
            return invoke;

        }catch (Exception e){

        }
        return null;
    }


}
```

因为最终要执行原方法，所以代理类必须持有原对象的引用，这里通过构造将原对象instrumentation传入。

重写要代理的execStartActivity()，然后修改intent，将要启动的Activity替换为LocalActivity。最后通过反射的方式执行真正启动Activity的execStartActivity()。

代理类以及设置完成，然后就是用代理对象，替换掉原对象：

```java
    public void replace(Activity activity){
				//通过反射获取Activity的class对象
        Class<? extends Activity> aClass = activity.getClass();
        try {
          	//获取Activity的mInstrumentation属性
            Field field= aClass.getDeclaredField("mInstrumentation");
          	//取消权限检查
            field.setAccessible(true);
          	//获取传入的activity的mInstrumentation字段
            mInstrumentation = (Instrumentation)field.get(activity);
          	//构造InstrumentationProxy，并将获取到的Activity字段传入
            InstrumentationProxy instrumentationProxy = new InstrumentationProxy(mInstrumentation);
            //用构造InstrumentationProxy替换掉当前Activity的mInstrumentation。
            field.set(activity,instrumentationProxy);

        }catch (Exception e){

        }

    }
```

### Hook Activity总结：

1. 由于Activity的启动执行了mInstrumentation.execStartActivity(),且它是一个单例对象，所以以mInstrumentation为Hook点

2. 定义一个代理类，代理类持有Instrumentation引用。并重写execStartActivity()。在execStartActivity()中进行修改工作。最后通过持有的Instrumentation引用，执行原execStartActivity()。

3. 定义一个replace(),通过反射的方式获取Activity的mInstrumentation，将这个mInstrumentation传入InstrumentationProxy中，同时用InstrumentationProxy对象替换Activity中的mInstrumentation。这时当发起一个启动Activity的请求时，执行的就是InstrumentationProxy.execStartActivity()