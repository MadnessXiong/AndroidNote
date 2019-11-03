# Retrofit的使用（获取天气数据）
## 1.  定义接口
> 只要了解Http的基础知识，理解请求时请求方法，请求头的作用与区别，参数是写@Query，@Field，还是@Path等就很好分辨。具体参考：[Http知识点梳理](https://github.com/MadnessXiong/AndroidNote/blob/master/Network/Http知识点梳理.md)
```
public interface ApiInterface {

    @GET("weather/index")
    Call<WeatherBean> getWeatherData(@Query("cityname") String cityName,@Query("key") String key);

}
```

## 2. 配置Retrofit并构建ApiInterface
```
  Retrofit build = new Retrofit.Builder()
                //配置URL
                .baseUrl("http://v.juhe.cn/")
                //添加Gson转换
                .addConverterFactory(GsonConverterFactory.create(new Gson()))
                .build();
    //通过create()构建ApiInterface对象，这里是整个Retrofit的核心，后面会分析
   ApiInterface apiInterface = build.create(ApiInterface.class);
```
## 3. 通过ApiInterface对象构建Call对象然后执行请求方法，获取结果
```
   //构建Call对象
   Call<WeatherBean> weatherData =apiInterface.getWeatherData("1", "64ca790d87353bb6b2e56e8d0bbaab08");
   //执行请求方法获取结果（这里是执行的异步方法）
   weatherData.enqueue(new Callback<WeatherBean>() {
            @Override
            public void onResponse(Call<WeatherBean> call, Response<WeatherBean> response) {

            }

            @Override
            public void onFailure(Call<WeatherBean> call, Throwable t) {

            }
        });
```
以上就是Retrofit的基本使用，Retrofit的网络请求实际是由okHttp发起的，并且Retrofit内置了okHttp，默认就是okHttp请求。至于如何添加Rxjava等支持以及是如何支持的，看完流程分析就明白了。

# Retrofit的调用流程源码分析

##  Retrofit的核心create()方法
```
public <T> T create(final Class<T> service) {
        Utils.validateServiceInterface(service);
        if (this.validateEagerly) {
            this.eagerlyValidateMethods(service);
        }

        return Proxy.newProxyInstance(service.getClassLoader(), new Class[]{service}, new InvocationHandler() {
            private final Platform platform = Platform.get();

            public Object invoke(Object proxy, Method method, @Nullable Object[] args) throws Throwable {
                //如果这个方法是Object声明的方法则直接调用
                if (method.getDeclaringClass() == Object.class) {
                    return method.invoke(this, args);
                //如果这个方法是默认方法的处理（java8中开始支持接口有默认方法，在这里没有意义）
                } else if (this.platform.isDefaultMethod(method)) {
                    return this.platform.invokeDefaultMethod(method, service, proxy, args);
                } else {
                    ServiceMethod<Object, Object> serviceMethod = Retrofit.this.loadServiceMethod(method);
                    OkHttpCall<Object> okHttpCall = new OkHttpCall(serviceMethod, args);
                    return serviceMethod.callAdapter.adapt(okHttpCall);
                }
            }
        });
    }
```
> create()使用了动态代理模式（具体参考-[设计模式之代理模式](https://github.com/MadnessXiong/AndroidNote/blob/master/DesignPattern/代理模式.md)）,然后再看看create()方法具体都做了些什么。
### 1. 代码检测 
- 1. Utils.validateServiceInterface(service)这行判断传进来的Class是否是一个接口，是接口的话是不是原生的，不能是继承其他接口的接口，不是的话抛出异常，所以我们之前构造了ApiInterface,在里面定义了请求方法。
```
 static <T> void validateServiceInterface(Class<T> service) {
        if (!service.isInterface()) {
            throw new IllegalArgumentException("API declarations must be interfaces.");
        } else if (service.getInterfaces().length > 0) {
            throw new IllegalArgumentException("API interfaces must not extend other interfaces.");
        }
    }
```
- 2. 是否对声明的参数等合法性等进行检测（默认不开启）
```
 if (this.validateEagerly) {
            this.eagerlyValidateMethods(service);
        }
```
### 2. 核心功能
```
  ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
```
> 之前的代码都是一系列检测，这3行才是create真正做的事情

####  第一行loadServiceMethod(method)看看它做了什么。
```
    ServiceMethod<?, ?> loadServiceMethod(Method method) {
        ServiceMethod<?, ?> result = (ServiceMethod)this.serviceMethodCache.get(method);
        if (result != null) {
            return result;
        } else {
            synchronized(this.serviceMethodCache) {
                result = (ServiceMethod)this.serviceMethodCache.get(method);
                if (result == null) {
                    result = (new retrofit2.ServiceMethod.Builder(this, method)).build();
                    this.serviceMethodCache.put(method, result);
                }

                return result;
            }
        }
    }
```

> 可以看到，先尝试从serviceMethodCache取一个ServiceMethod，如果取不到就通过ServiceMethod.Builder(this, method)).build()创建一个。然后存入serviceMethodCache并返回，总之就是创建了一个ServiceMethod。那么ServiceMethod又是什么？做了什么事情？
```
        Builder(Retrofit retrofit, Method method) {
            this.retrofit = retrofit;
            this.method = method;
            //获取方法的注解
            this.methodAnnotations = method.getAnnotations();
           //获取方法的参数类型
            this.parameterTypes = method.getGenericParameterTypes();
          //获取每一个参数的每一个注解
            this.parameterAnnotationsArray = method.getParameterAnnotations();
        }
```
1.  通过Retrofit.Builder（）的构造方法来看，它传入了retrofit和method，然后获取了方法的注解，方法的参数类型，方法的每一个参数的每一个注解。然后赋值到Builder的成员变量，记录下来。

2.  再看build()做了什么
```
 public ServiceMethod build() {
            //仅保留核心代码，省略其他。
            this.callAdapter = this.createCallAdapter();
            this.responseConverter = this.createResponseConverter();
            this.parseMethodAnnotation(annotation);
            parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
        }
```
 - 1. createCallAdapter(),创建了一个callAdapter,继续看
```
 public CallAdapter<?, ?> nextCallAdapter(@Nullable retrofit2.CallAdapter.Factory skipPast, Type returnType, Annotation[] annotations) {
      // //仅保留核心代码，省略其他。
        for(i = this.adapterFactories.size(); i < i; ++i) {
            CallAdapter<?, ?> adapter = ((retrofit2.CallAdapter.Factory)this.adapterFactories.get(i)).get(returnType, annotations, this);
            if (adapter != null) {
                return adapter;
            }
        }   
```
 上面可以看到从CallAdapter.Factory里get了一个CallAdapter并返回，那么这个CallAdapter.Factory又是什么？在哪里创建的？继续找
```
        public Retrofit build() {
            if (this.baseUrl == null) {
                throw new IllegalStateException("Base URL required.");
            } else {
                Factory callFactory = this.callFactory;
                if (callFactory == null) {
                    callFactory = new OkHttpClient();
                }

                Executor callbackExecutor = this.callbackExecutor;
                if (callbackExecutor == null) {
                    callbackExecutor = this.platform.defaultCallbackExecutor();
                }
               //在这里创建了adapterFactories
                List<retrofit2.CallAdapter.Factory> adapterFactories = new ArrayList(this.adapterFactories);
                adapterFactories.add(this.platform.defaultCallAdapterFactory(callbackExecutor));
            
                List<retrofit2.Converter.Factory> converterFactories = new ArrayList(this.converterFactories);
                return new Retrofit((Factory)callFactory, this.baseUrl, converterFactories, adapterFactories, callbackExecutor, this.validateEagerly);
            }
        }
```

这里可以看到在Retrofit的build()方法里创建了adapterFactories，并添加了platform.defaultCallAdapterFactory(callbackExecutor))，实际是添加了ExecutorCallAdapterFactory，由于最终是要得到CallAdapter，那么再来看ExecutorCallAdapterFactory的get()方法
```
@Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public Call<Object> adapt(Call<Object> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }
```
在get()方法中，直接new CallAdapter（）返回，并实现了它的抽象方法，其中adapt()方法重写后返回了new ExecutorCallbackCall(callbackExecutor, call)，
这里传入了2个参数call是调用adapt()方法时传入的，callbackExecutor其实是在上面的build方法里创建的
```
 if (callbackExecutor == null) {

     callbackExecutor = this.platform.defaultCallbackExecutor();

  }
```
点进去发现Android继承Platform并实现了这个方法
```
static class Android extends Platform {
    @Override public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }

    @Override CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }

    static class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override public void execute(Runnable r) {
        handler.post(r);
      }
    }
  }
```
这个方法返回了new MainThreadExecutor()，MainThreadExecutor中的execute（）使用handler把当前任务发送到了主线程。

**也就是说，如果这个callbackExecutor执行了execute（）就会把任务切换到主线程！！！**

OK，这时再来看ExecutorCallbackCall这个类
```
static final class ExecutorCallbackCall<T> implements Call<T> {
   // 仅保留核心代码，省略其他
    final Executor callbackExecutor;
    final Call<T> delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
      this.callbackExecutor = callbackExecutor;
      this.delegate = delegate;
    }
    //callB的enqueue
    @Override public void enqueue(final Callback<T> callback) {
      if (callback == null) throw new NullPointerException("callback == null");
        //callA的enqueue
      delegate.enqueue(new Callback<T>() {
        @Override public void onResponse(Call<T> call, final Response<T> response) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              if (delegate.isCanceled()) {
                // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
              } else {
                callback.onResponse(ExecutorCallbackCall.this, response);
              }
            }
          });
        }
  }
```
可以看到只要执行enqueue（）方法，那么就会执行传进来那么call的enqueue（）方法然后在它的onResponse中使用callbackExecutor.execute()切换到主线程。

**那么总结一下，当一个callAdapter执行它的adapt（Call call）方法后，会把传入的callA转换为另外一个callB返回，同时当callB执行它的enqueue（）方法时实际会执行CallA的enqueue(),同时做一些附加操作，如这里进行了线程切换的操作。**
> adapt的作用是对call进行转换，那么意味着可以返回我们任何想要的格式，比如要返回RxJava的Flowable，只需要RxJava去实现自己的CallAdapter并重写adapt方法,然后在build的时候添加Rxjava的CallAdapter就行了。同理，也可以自定义自己需要的CallAdapter实现特定的需求。

- 2. createResponseConverter()负责把服务器返回的数据转化为你设定的类型。流程和CallAdapter差不多，这里不再赘述。

- 3. parseMethodAnnotation(annotation)
 ```
private void parseMethodAnnotation(Annotation annotation) {
      if (annotation instanceof DELETE) {
        parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
      } else if (annotation instanceof GET) {
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
      } else if (annotation instanceof HEAD) {
        parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
        if (!Void.class.equals(responseType)) {
          throw methodError("HEAD method must use Void as response type.");
        }
      } else if (annotation instanceof PATCH) {
        parseHttpMethodAndPath("PATCH", ((PATCH) annotation).value(), true);
      } else if (annotation instanceof POST) {
        parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
      } else if (annotation instanceof PUT) {
        parseHttpMethodAndPath("PUT", ((PUT) annotation).value(), true);
      } else if (annotation instanceof OPTIONS) {
        parseHttpMethodAndPath("OPTIONS", ((OPTIONS) annotation).value(), false);
      } else if (annotation instanceof HTTP) {
        HTTP http = (HTTP) annotation;
        parseHttpMethodAndPath(http.method(), http.path(), http.hasBody());
      } else if (annotation instanceof retrofit2.http.Headers) {
        String[] headersToParse = ((retrofit2.http.Headers) annotation).value();
        if (headersToParse.length == 0) {
          throw methodError("@Headers annotation is empty.");
        }
        headers = parseHeaders(headersToParse);
      } else if (annotation instanceof Multipart) {
        if (isFormEncoded) {
          throw methodError("Only one encoding annotation is allowed.");
        }
        isMultipart = true;
      } else if (annotation instanceof FormUrlEncoded) {
        if (isMultipart) {
          throw methodError("Only one encoding annotation is allowed.");
        }
        isFormEncoded = true;
      }
    }
```
可以看到这个方法是解读方法的每一个注解也就是Get，PUt这些信息的，同时做书写检查，如果书写错误，类型不匹配则报错。

- 3. parseParameter(p, parameterType, parameterAnnotations)，同样这个方法是解读参数的注解，然后赋值给ParameterHandler保存，同时做书写检查，如果书写错误，类型不匹配则报错。（这里不再贴代码）；

以上就是ServiceMethod的build做的事情，总结一下
>1: 创建了一个CallAdapter。
  2: createResponseConverter()把服务器返回的数据转换成自己想要的类型
  3: parseMethodAnnotation(annotation)对方法的注解进行解析
  4: parseParameter(p, parameterType, parameterAnnotations)对参数的注解进行解析

再回头来看loadServiceMethod(method)，它构造了一个ServiceMethod，在构造过程中首先创建了CallAdapter,然后完成了对我们定义的方法的信息的采集，存储了我们所有声明的东西
 
####  第二行OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
```
  OkHttpCall(ServiceMethod<T, ?> serviceMethod, Object[] args) {
    this.serviceMethod = serviceMethod;
    this.args = args;
  }
```
可以看到第二行只是传入了采集了所有信息的serviceMethod以及方法参数args，然后都交给okHttpCall，并构造了okhttpCall
>从这里可以看出Retrofit的网络请求是由okhttp完成的，毕竟所有网络请求需要的东西都交给了okhttpCall。

#### 第三行 return serviceMethod.callAdapter.adapt(okHttpCall);
上面分析过adapt（），这里就是把okHttpCall传入，同时返回我们需要的格式（本例中是返回了Call<WeatherBean>

### 3. create()方法总结
- 1. 进行代码检测
- 2. 创建CallAdapter,默认的CallAdapter主要是对线程进行了切换
- 3. 通过ServiceMethod采集请求网络需要的信息，并检查代码合法性
- 4. 构建okHttpCall,并把ServiceMethod的信息交给okHttp
- 5. CallAdapter的adapt()方法把okHttpCall转换为使用者需要的Call

# 从源码解读Retrofit的配置与使用
- 首先回顾下Retrofit的配置
```
 Retrofit build = new Retrofit.Builder()
                //配置URL
                .baseUrl("http://v.juhe.cn/")
                .client(okHttpClient)
                //添加Gson转换
                .addConverterFactory(GsonConverterFactory.create(new Gson()))
                //添加RxJava支持
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .build();
```
> 1. Retrofit的网络请求默认是由okHttp发起的，默认不用设置，但是如果要对okHttpClient进行配置，则需要在配置里添加 client(okHttpClient)
> 2. 网络请求默认返回的是ResponseBody，前面分析过如果要把数据转换为自己设定的类型需要添加特定的ConverterFactory，所以这里.addConverterFactory(GsonConverterFactory.create(new Gson()))。
> 3. Retrofit默认返回一个Call，前面分析过如果要支持RxJava，返回RxJava的Flowable，需要添加RxJava自己的CallAdapter,所以这里
.addCallAdapterFactory(RxJava2CallAdapterFactory.create())

- 再回顾下是如何使用的
```
   //通过create()构建ApiInterface对象
   ApiInterface apiInterface = build.create(ApiInterface.class);
   //构建Call对象
   Call<WeatherBean> weatherData =apiInterface.getWeatherData("1", "64ca790d87353bb6b2e56e8d0bbaab08");
   //执行请求方法获取结果（这里是执行的异步方法）
   weatherData.enqueue(new Callback<WeatherBean>() {
            @Override
            public void onResponse(Call<WeatherBean> call, Response<WeatherBean> response) {

            }

            @Override
            public void onFailure(Call<WeatherBean> call, Throwable t) {

            }
        });
```
> 1. 首先通过create()方法构建了ApiInterface的代理对象apiInterface，这个时候Proxy.newProxyInstance的invoke（）方法还并没有执行。
> 2. 调用apiInterface调用getWeatherData（）方法返回了Call对象，这里实际执行了Proxy.newProxyInstance的invoke（）方法。前面分析过这里采集了getWeatherData（）方法要访问网络所设置的所有信息，然后交给了okHttpCall,然后把okHttpCall转换为自己需要的Call返回。
> 3. 拿到返回的Call执行enqueue（）方法。前面分析过，我们的Call执行enqueue（）后实际的操作会转交给okHttpCall，okHttpCall会执行它的enqueue（）方法发起网络请求。返回结果后再由Retrofit默认实现的CallAdapter里的callbackExecutor通过handler完成线程的切换。然后通过回调的方式把结果返回。(注：RxJava的切换线程由RxJava自己的CallAdapter实现)





