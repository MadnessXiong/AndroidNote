### 插件化简介

Android中热修复主要用来修复bug,插件化则主要用来增加功能，将一些独立的功能打包为单独的dex作为插件。在需要的时候再动态加载。

Android中的插件化以Hook方式为主流。

### Activity插件化

Activity的插件化必然涉及Activity的启动过程。从[Android中根Activity的启动过程](https://github.com/MadnessXiong/AndroidNote/blob/master/Frameworks相关/Android中根Activity的启动过程.md)可知，Activity的启动分为3个阶段

> Launcher或Activity请求AMS过程(第一次启动应用程序或应用程序打开新的Activity)
>
> AMS到ApplicationThread的调用过程
>
> ActivityThread启动Activity过程

因为Activity启动时必须先在Manifest中注册。而插件中的Activity是无法提前注册的。再根据Activity的启动过程，那么Activity的插件化可以分为以下3步:

> 占坑：使用一个占坑Activity-LocalActivity在Manifest中注册，避免插件Activity没有注册导致崩溃的情况
>
> 绕过AMS验证：启动插件Activity-PlugActivity时，将PlugActivity替换为LocalActivity，进行AMS的验证，生命周期及栈管理。
>
> 还原Activity：AMS调用ActivityThread启动Activity时，将LocalActivity再替换为PlugActivity

那么再看每一步的具体实现:

- **创建占坑Activity-LocalActivity**

  ```java
  //一个空实现的Activity
  public class LocalActivity extends AppCompatActivity {
  
      @Override
      protected void onCreate(Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
          setContentView(R.layout.activity_local);
      }
  }
  //在manifest中注册
  <activity android:name=".LocalActivity"></activity>
  ```

- **绕过AMS验证：**

  要绕过AMS验证，必须让在Manifest中注册的LocalActivity代替要启动的PlugActivity去验证，那么就要在启动时将PlugActivity替换为LocalActivity。

  在[Android中Hook简析](https://github.com/MadnessXiong/AndroidNote/blob/master/Frameworks相关/Android中Hook简析.md)中分析过，Activity启动时调用了Instrumentation的execStartActivity()，那么可以通过Hook的方式在这里进行替换，先看一下之前Hook的代码：

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
        		 //将要启动的Activity改为LocalActivity
        		intent.setClassName(who,"com.madnessxiong.client.LocalActivity");
        	//通过反射获取Instrumentation的class对象
          Class<? extends Instrumentation> aClass = Instrumentation.class;
          try{
            //通过class对象获取真正启动Activity的execStartActivities()
  			 method = aClass.getMethod("execStartActivity",                Context.class,IBinder.class,IBinder.class,Activity.class,Intent.class,int.class,Bundle.class);
           //通过反射的方式执行真正启动Activity的execStartActivities()
          ActivityResult invoke = (ActivityResult) method.invoke(instrumentation, who, contextThread, token, target, intent, options);
              return invoke;
          }catch (Exception e){
          }
          return null;
      }
  
  
  }
  ```

  在这里修改了intent，将要启动的Activity改为了LocalActivity。完成了替换工作。这时启动Activity的就是LocalActivity。由于LocalActivity在Manifest中注册过，所以可以通过AMS的校验。

- **还原Activity**

  AMS最终会调用到ActivityThread.performLaunchActivity():

  ```java
         private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
           			//在newActivity()中用类加载器创建activity的实例，
                 activity = mInstrumentation.newActivity(
                         cl, component.getClassName(), r.intent);
             return activity;
         }
  ```

  然后通过mInstrumentation.newActivity()创建一个Activity，那么这里可以作为一个hook点。代理Instrumentation的newActivity()。

  根据上面的分析，那么通过Hook的方式将ActivityThread的mInstrumentation替换为InstrumentationProxy：

  ```java
      public void HookActivityThread(){
          try {
            	//获取ContextImpl.class
              Class<?> contextImplClass = Class.forName("android.app.ContextImpl");
            	//获取mMainThread对应的成员变量
              Field mMainThreadFiled = contextImplClass.getDeclaredField("mMainThread");
              mMainThreadFiled.setAccessible(true);
            	//成员变量activityThread
              Object activityThread = mMainThreadFiled.get(getBaseContext());
  						//获取ActivityThread.class
              Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
            	//获取Instrumentation属性
              Field field = activityThreadClass.getDeclaredField("mInstrumentation");
              field.setAccessible(true);
  
              Instrumentation mInstrumentation = (Instrumentation)field.get(activityThread);
              InstrumentationProxy instrumentationProxy = new InstrumentationProxy(mInstrumentation);
              field.set(activityThread,instrumentationProxy);
          }catch (Exception e){
            
          }
  
      }
  ```

  由于ActivityThread时@hide无法直接获取它的对象，而在进程启动时已经将一个mMainThread存入了ContextImpl中。所以这里先通过反射的方式在ContextImpl中获取了ActivityThread对象，最后用InstrumentationProxy替换mInstrumentation。那么调用newActivity()时，就会来到InstrumentationProxy.newActivity():

  ```java
      public Activity newActivity(ClassLoader cl, String className,
                                  Intent intent)
              throws InstantiationException, IllegalAccessException,
              ClassNotFoundException {
          return super.newActivity(cl,"com.madnessxiong.client.PlugActivity",intent);
      }
  ```

  在InstrumentationProxy.newActivity()中，直接调用父类，也就是Instrumentation自身的newActivity()，将类名改为实际要启动的PlugActivity。就完成了Activity的还原。而整个替换的过程只是对intent中的类名进行了替换，而在AMS中是通过token对Activity进行验证的，所以并不影响它的生命周期。