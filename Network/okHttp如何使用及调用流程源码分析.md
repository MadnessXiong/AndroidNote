### 1. okHttp的使用

- 1.1 构造并配置okHttpClient

`OkHttpClient okHttpClient = new OkHttpClient();`

然后看下配置项：

```
  final Dispatcher dispatcher;//调度器,线程管理
  final @Nullable Proxy proxy;//代理
  final List<Protocol> protocols;//协议（如Http1.1，Http2）
  final List<ConnectionSpec> connectionSpecs;//传输层版本和连接协议
  final List<Interceptor> interceptors;//拦截器
  final List<Interceptor> networkInterceptors;//网络拦截器
  final EventListener.Factory eventListenerFactory;
  final ProxySelector proxySelector;//代理选择器
  final CookieJar cookieJar;//cookie
  final @Nullable Cache cache;//缓存
  final @Nullable InternalCache internalCache;//内部缓存
  final SocketFactory socketFactory;//socket工厂
  final @Nullable SSLSocketFactory sslSocketFactory;//安全套层socket工厂，用于https
  final @Nullable CertificateChainCleaner certificateChainCleaner;//验证确认响应证书
  final HostnameVerifier hostnameVerifier;//主机名确认
  final CertificatePinner certificatePinner;//用于自签名证书
  final Authenticator proxyAuthenticator;//代理身份验证
  final Authenticator authenticator;//本地身份验证
  final ConnectionPool connectionPool;/连接池
  final Dns dns;//dns
  final boolean followSslRedirects;/安全套接层重定向
  final boolean followRedirects;//本地重定向(http和https互相跳转时，默认为true)
  final boolean retryOnConnectionFailure;//连接失败重试
  final int connectTimeout;//连接超时时间
  final int readTimeout;//读取超时时间
  final int writeTimeout;//写入超时时间
  final int pingInterval;//ping的间隔
```

其实构建OkHttpClient时okHttp已经帮我们实现了默认配置，我们可以根据需要进行修改。

```
 OkHttpClient.Builder okHttpBuilder = new OkHttpClient.Builder();
        okHttpBuilder.addInterceptor(...);
        okHttpBuilder.cache(...);
        OkHttpClient okHttpClient = okHttpBuilder.build();
```



- 1.2 构造并配置Request

  ```
  Request.Builder builder = new Request.Builder(); 
  
  builder.url("http://v.juhe.cn/"); 
  
  Request request = builder.build(); 
  ```

  这里只配置了url,再看下其他的配置项：

  ```
  public Builder() {
        this.method = "GET";
        this.headers = new Headers.Builder();
      }
  
      Builder(Request request) {
        this.url = request.url;
        this.method = request.method;
        this.body = request.body;
        this.tag = request.tag;
        this.headers = request.headers.newBuilder();
      }
  ```

  可以看到如果不设置的话，默认是GET请求，同时还可以设置body,tag,headers等信息。

 - 1.3 构造Call并执行网络请求

    ```
     Call call = okHttpClient.newCall(request);
            call.enqueue(new Callback() {
                @Override
                public void onFailure(Call call, IOException e) {
    
                }
    
                @Override
                public void onResponse(Call call, Response response) throws IOException {
                  
                }
            });
    ```

    以上就是okHttp等基本使用（一般结合Retrofit，效果更佳）。

### 2.  okHttp的调用流程源码分析

- 2.1 从使用中可以看到首先通过newCall()方法构造了一个Call,那么看下newCall():

  ```
   @Override public Call newCall(Request request) {
      return new RealCall(this, request, false);
    }
  ```

  newCall()实际返回了一个RealCall,那么执行的enqueue()就是RealCall的enqueue()：

  ```
  @Override public void enqueue(Callback responseCallback) {
      synchronized (this) {
        if (executed) throw new IllegalStateException("Already Executed");
        executed = true;
      }
      captureCallStackTrace();
      client.dispatcher().enqueue(new AsyncCall(responseCallback));
    }
  ```

  可以看到在RealCall的enqueue()中实际执行了dispatcher的enqueue(AsyncCall(responseCallback))，并传入了AsyncCall，dispatcher其实是一个调度器，用来调度请求网络的线程。看一下它的enqueue()：

  ```
  synchronized void enqueue(AsyncCall call) {
      if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
        runningAsyncCalls.add(call);
        executorService().execute(call);
      } else {
        readyAsyncCalls.add(call);
      }
    }
  ```

  从上面代码可以看到，先判断正在执行的任务是否超过最大请求数，如果超了就先存进去。，如果没超则使用ExecutorService直接执行，再看执行的这个call,它是一个AsyncCall,AsyncCall继承了NamedRunnable，NamedRunnable则继承了Runnable，所以AsyncCall是一个Runnable，同时NamedRunnable的run()执行了execute()，AsyncCall重写了execute()，所以最后会执行到AsyncCall的execute()里：

  ```
      @Override protected void execute() {
        boolean signalledCallback = false;
        try {
          Response response = getResponseWithInterceptorChain();
          if (retryAndFollowUpInterceptor.isCanceled()) {
            signalledCallback = true;
            responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
          } else {
            signalledCallback = true;
            responseCallback.onResponse(RealCall.this, response);
          }
        } catch (IOException e) {
          if (signalledCallback) {
            // Do not signal the callback twice!
            Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
          } else {
            responseCallback.onFailure(RealCall.this, e);
          }
        } finally {
          client.dispatcher().finished(this);
        }
      }
    }
  ```

  可以看到getResponseWithInterceptorChain()返回了请求结果，然后就通过回调返回给调用端了，这个就是大概的调用流程。

  那么这个getResponseWithInterceptorChain()其实就完成了请求前的数据处理，建立连接，对返回数据进行处理等操作。那么这个getResponseWithInterceptorChain()其实就是整个okHttp的重点。

  

  - 2.2 看一下getResponseWithInterceptorChain()代码：

    ```
      Response getResponseWithInterceptorChain() throws IOException {
        // Build a full stack of interceptors.
        List<Interceptor> interceptors = new ArrayList<>();
        interceptors.addAll(client.interceptors());
        interceptors.add(retryAndFollowUpInterceptor);
        interceptors.add(new BridgeInterceptor(client.cookieJar()));
        interceptors.add(new CacheInterceptor(client.internalCache()));
        interceptors.add(new ConnectInterceptor(client));
        if (!forWebSocket) {
          interceptors.addAll(client.networkInterceptors());
        }
        interceptors.add(new CallServerInterceptor(forWebSocket));
    
        Interceptor.Chain chain = new RealInterceptorChain(
            interceptors, null, null, null, 0, originalRequest);
        return chain.proceed(originalRequest);
      }
    ```

    可以看到这个方法里添加了一系列的Interceptor，然后把Interceptor集合传递给RealInterceptorChain，构建了一个Chain。最后执行了chain.proceed()，这里使用了责任链模式，它的执行流程大概是这样的：

    > Interceptor最重要的就是它的intercept()方法，所有核心逻辑都在这里。

    
![image-20191113010116100.png](/work/learn/note/AndroidNote/Network/Res/image-20191113010116100.png)



先执行第一个Interceptor的前置逻辑，然后执行chain.proceed()，把任务转交给下一个Interceptor，下一个Interceptor同样执行它的前置逻辑，完成后通过chain.proceed()把任务继续传递下去，直到最后一个Interceptor。最后一个Interceptor获取到结果后返回到上一个Interceptor，执行它的后置逻辑，执行完后再返回给上一次，最后返回结果给getResponseWithInterceptorChain()。

**总结一下：当我们发起一个请求后，当来到getResponseWithInterceptorChain（）时，它添加了一系列Interceptor，构成一个责任链。每一层的Interceptor通过前置逻辑对请求的原始数据（request）进行加工，然后再通过chain.proceed()把任务传递给下一层，当传递到最后一个Interceptor时会返回请求的结果(response)，然后再逐层返回，通过每一层的后置逻辑对结果(response)进行处理，最后返回给请求端。**

> 所以我们可以通过添加自定义Interceptor的方式，在请求前通过前置逻辑对请求request进行修改，如添加header等。等返回结果后，通过后置逻辑对返回数据response进行处理。OkHttp通过OkHttpClient.Builder的addInterceptor()和addNetworkInterceptor()支持了添加自定义Interceptor，后面会说到。

 - 2.3 那么再看一下每个Interceptor具体都做了什么

    - 2.3.1 添加自定义的Interceptor
      ```
      interceptors.addAll(client.interceptors());
      ```

      可以看到，第一步其实是添加了我们自定义的所有的Interceptor，一般来说可以添加一些信息如：

      ```
       builder.addInterceptor(new Interceptor() {
                      @Override
                      public Response intercept(Chain chain) throws IOException {
      										//前置逻辑
                          Request request = chain.request().newBuilder()
                                  .addHeader("Content-Type", ...)
                                  .addHeader("Date", ...)
                                  .addHeader("User-Agent", ...)
                                  .addHeader("uid",...)
                                  .build();
                        	//传递任务给下一层
                          Response response = chain.proceed(request);
                          //后置逻辑，这里已经拿到了请求结果，可以根据需要对结果直接进行处理
                          return response;
                      }
                  });
      ```

    - 2.3.2 添加RetryAndFollowUpInterceptor

      ```
      interceptors.add(retryAndFollowUpInterceptor);
      ```

      从名字可以看出这是一个重试的Interceptor,再看它的intercept()：

      ```
      //篇幅原因，省略了很多代码
      @Override public Response intercept(Chain chain) throws IOException {
          Request request = chain.request();
          //前置逻辑，初始化连接对象
          streamAllocation = new StreamAllocation(
              client.connectionPool(), createAddress(request.url()), callStackTrace);
      
          while (true) {
          //是否已取消了连接
            if (canceled) {
              streamAllocation.release();
              throw new IOException("Canceled");
            }
      
            try {
            //移交任务，等待结果
              response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);
              releaseConnection = false;
            } catch (RouteException e) {
              //路由出错，判断是否进行重试
              if (!recover(e.getLastConnectException(), false, request)) {
                throw e.getLastConnectException();
              }
            } 
      
            Request followUp = followUpRequest(response);
             //如果followUp返回空，则代表请求成功，直接结束循环，如果请求失败，那么生成新的request在条件符合的时候去重试
            if (followUp == null) {
              if (!forWebSocket) {
                streamAllocation.release();
              }
              return response;
            }
        }
      
      ```

      > 请注意，这里省略了很多代码，判断条件不是全部。之后的Interceptor同样会省略，不再提示

      首先RetryAndFollowUpInterceptor的前置逻辑是初始化了一个连接对象StreamAllocation，然后判断是否取消了请求，请求过程中是否路由出错，取消或出错的话就直接抛出异常，否则执行proceed()传递请求，然后等待请求结果。请求结果返回后，执行后置逻辑，根据设置的条件对结果进行判断，看是否需要去重试，如果需要则再次进入while()循环进行请求，是否需要进行重定向，都不需要则直接返回结果。（注意！这里省略了一些代码）

    - 2.3.3 添加BridgeInterceptor

      ```
      interceptors.add(new BridgeInterceptor(client.cookieJar()));
      ```

      可以看到，这一步添加了BridgeInterceptor，并且传入了CookieJar，那么意味着，可以自定义自己的cookieJar：

      ```
        OkHttpClient okHttpClient = okHttpBuilder
                     .cookieJar(new CookieJar() {
                         @Override
                         public void saveFromResponse(HttpUrl url, List<Cookie> cookies) {
                             //保存cookie
                         }
      
                         @Override
                         public List<Cookie> loadForRequest(HttpUrl url) {
                        		 //加载cookie
                             return null;
                         }
                     })
                      .build();
      ```

      再看它的intercept()：

      ```
      @Override public Response intercept(Chain chain) throws IOException {
          Request userRequest = chain.request();
          Request.Builder requestBuilder = userRequest.newBuilder();
      
          RequestBody body = userRequest.body();
          if (body != null) {
            MediaType contentType = body.contentType();
            if (contentType != null) {
              requestBuilder.header("Content-Type", contentType.toString());
            }
      
            long contentLength = body.contentLength();
            if (contentLength != -1) {
              requestBuilder.header("Content-Length", Long.toString(contentLength));
              requestBuilder.removeHeader("Transfer-Encoding");
            } else {
              requestBuilder.header("Transfer-Encoding", "chunked");
              requestBuilder.removeHeader("Content-Length");
            }
          }
      
          if (userRequest.header("Host") == null) {
            requestBuilder.header("Host", hostHeader(userRequest.url(), false));
          }
      
          if (userRequest.header("Connection") == null) {
            requestBuilder.header("Connection", "Keep-Alive");
          }
      
          // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
          // the transfer stream.
          boolean transparentGzip = false;
          if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
            transparentGzip = true;
            //添加gzip压缩格式
            requestBuilder.header("Accept-Encoding", "gzip");
          }
      
          List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
          if (!cookies.isEmpty()) {
            requestBuilder.header("Cookie", cookieHeader(cookies));
          }
      
          if (userRequest.header("User-Agent") == null) {
            requestBuilder.header("User-Agent", Version.userAgent());
          }
      
          Response networkResponse = chain.proceed(requestBuilder.build());
      
          HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());
      
          Response.Builder responseBuilder = networkResponse.newBuilder()
              .request(userRequest);
      
          if (transparentGzip
              && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
              && HttpHeaders.hasBody(networkResponse)) {
            GzipSource responseBody = new GzipSource(networkResponse.body().source());
            Headers strippedHeaders = networkResponse.headers().newBuilder()
                .removeAll("Content-Encoding")
                .removeAll("Content-Length")
                .build();
            responseBuilder.headers(strippedHeaders);
            responseBuilder.body(new RealResponseBody(strippedHeaders, Okio.buffer(responseBody)));
          }
      
          return responseBuilder.build();
        }
      ```

      首先BridgeInterceptor的前置逻辑对请求的header进行了设置，调用者设置过的就直接使用，没设置过的则设置一个默认的。然后还添加了gzip压缩的支持。然后执行proceed()传递请求，等待请求结果。请求结果返回后，执行后置逻辑，这里的后置逻辑主要就是对数据进行了解压缩后返回。

    - 2.3.4 添加CacheInterceptor

      ```
      interceptors.add(new CacheInterceptor(client.internalCache()));
      ```

      从名字可以看出它是一个缓存的Interceptor，并且传入了Cache,那么意味着可以自定义Cache：

      ```
       File cacheFile = new File(...);
       Cache cache = new Cache(cacheFile, ...);
       OkHttpClient okHttpClient = okHttpBuilder
                      .cache(cache)
                      .build();
      ```

      再看它的intercept()：

      ```
      //篇幅原因，省略了很多代码
      @Override public Response intercept(Chain chain) throws IOException {
      		//创建缓存策略
          CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
          
          // If we're forbidden from using the network and the cache is insufficient, fail.
          //如果网络有问题，同时没有缓存，就构建并返回一个code是504的response
          if (networkRequest == null && cacheResponse == null) {
            return new Response.Builder()
                .request(chain.request())
                .protocol(Protocol.HTTP_1_1)
                .code(504)
                .message("Unsatisfiable Request (only-if-cached)")
                .body(Util.EMPTY_RESPONSE)
                .sentRequestAtMillis(-1L)
                .receivedResponseAtMillis(System.currentTimeMillis())
                .build();
          }
      
          // If we don't need the network, we're done.
          //如果网络有问题，则使用缓存构建并返回一个response
          if (networkRequest == null) {
            return cacheResponse.newBuilder()
                .cacheResponse(stripBody(cacheResponse))
                .build();
          }
      
          Response networkResponse =networkResponse = chain.proceed(networkRequest);
      
          if (cacheResponse != null) {
          //服务器返回304，代表内容没有更新，继续使用缓存
            if (networkResponse.code() == HTTP_NOT_MODIFIED) {
              Response response = cacheResponse.newBuilder()
                  .headers(combine(cacheResponse.headers(), networkResponse.headers()))
                  .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
                  .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
                  .cacheResponse(stripBody(cacheResponse))
                  .networkResponse(stripBody(networkResponse))
                  .build();
              networkResponse.body().close();
              return response;
            } else {
              closeQuietly(cacheResponse.body());
            }
          }
          //读取网路结果
          Response response = networkResponse.newBuilder()
              .cacheResponse(stripBody(cacheResponse))
              .networkResponse(stripBody(networkResponse))
              .build();
          //进行缓存
          if (cache != null) {
              CacheRequest cacheRequest = cache.put(response);
              return cacheWritingResponse(cacheRequest, response);
            }
          }
          //返回结果
          return response;
        }
      ```

      首先CacheInterceptor的前置逻辑创建了一个缓存策略，然后判断网络是否有问题，有问题的话是否有缓存，有缓存直接返回缓存，没有缓存的话则则直接返回一个带错误信息的Response,不再往下执行。如果有网络也没缓存，那么执行proceed()传递请求，等待结果。返回结果后，在后置逻辑中，如果结果返回304则代表服务器没有更新数据，直接返回缓存。如果正常返回数据则读取结果后对结果进行缓存后返回。

    - 2.3.5 添加ConnectInterceptor

      ```
      interceptors.add(new ConnectInterceptor(client));
      ```

      从名字可以看出这是一个连接的Interceptor，再看它的intercept()：

      ```
      @Override public Response intercept(Chain chain) throws IOException {
          RealInterceptorChain realChain = (RealInterceptorChain) chain;
          Request request = realChain.request();
        	//获取连接对象
          StreamAllocation streamAllocation = realChain.streamAllocation();
      
          // We need the network to satisfy this request. Possibly for validating a conditional GET.
          boolean doExtensiveHealthChecks = !request.method().equals("GET");
          //建立连接，并返回编码解码器
          HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
          //获取连接
          RealConnection connection = streamAllocation.connection();
      
          return realChain.proceed(request, streamAllocation, httpCodec, connection);
        }
      ```

      ConnectInterceptor的intercept()中首先获取了之前构建的连接对象StreamAllocation，然后建立了连接，生成了编码解码器，获取了连接后执行 realChain.proceed(request, streamAllocation, httpCodec, connection)，传递请求给下一层，同时直接返回结果，没有后置逻辑。

    - 2.3.6 添加networkInterceptors

      ```
      interceptors.addAll(client.networkInterceptors());
      ```

      可以看到，这个networkInterceptors位置是在建立连接之后，写数据以及获取数据之前，所以它一般是对发出的请求做最后的处理以及拿到数据后最初的处理，如使用Stetho抓包时可以添加：

      ```
      builder.addNetworkInterceptor(new StethoInterceptor());
      ```

    - 2.3.7 添加CallServerInterceptor

      ```
      interceptors.add(new CallServerInterceptor(forWebSocket));
      ```

      看一下它的intercept():

      ```
      @Override public Response intercept(Chain chain) throws IOException {
          RealInterceptorChain realChain = (RealInterceptorChain) chain;
          HttpCodec httpCodec = realChain.httpStream();
          StreamAllocation streamAllocation = realChain.streamAllocation();
          RealConnection connection = (RealConnection) realChain.connection();
          Request request = realChain.request();
      
          long sentRequestMillis = System.currentTimeMillis();
          //写头信息
          httpCodec.writeRequestHeaders(request);
      
          Response.Builder responseBuilder = null;
          if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
            // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
            // Continue" response before transmitting the request body. If we don't get that, return what
            // we did get (such as a 4xx response) without ever transmitting the request body.
            if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
              httpCodec.flushRequest();
              responseBuilder = httpCodec.readResponseHeaders(true);
            }
      
            if (responseBuilder == null) {
              // Write the request body if the "Expect: 100-continue" expectation was met.
              Sink requestBodyOut = httpCodec.createRequestBody(request, request.body().contentLength());
              BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
              request.body().writeTo(bufferedRequestBody);
              bufferedRequestBody.close();
            } else if (!connection.isMultiplexed()) {
              // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection from
              // being reused. Otherwise we're still obligated to transmit the request body to leave the
              // connection in a consistent state.
              streamAllocation.noNewStreams();
            }
          }
      
          httpCodec.finishRequest();
            
          if (responseBuilder == null) {
            responseBuilder = httpCodec.readResponseHeaders(false);
          }
          //构建response 
          Response response = responseBuilder
              .request(request)
              .handshake(streamAllocation.connection().handshake())
              .sentRequestAtMillis(sentRequestMillis)
              .receivedResponseAtMillis(System.currentTimeMillis())
              .build();
      
          int code = response.code();
          if (forWebSocket && code == 101) {
            // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
            response = response.newBuilder()
                .body(Util.EMPTY_RESPONSE)
                .build();
          } else {
              //获取body信息
            response = response.newBuilder()
                .body(httpCodec.openResponseBody(response))
                .build();
          }
      
          if ("close".equalsIgnoreCase(response.request().header("Connection"))
              || "close".equalsIgnoreCase(response.header("Connection"))) {
            streamAllocation.noNewStreams();
          }
      
          if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
            throw new ProtocolException(
                "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
          }
      
          return response;
        }
      ```

      这个Interceptor是最后一个，连接之前的ConnectInterceptor已经建立，这里就只需要写数据和读数据就行了，可以看到writeRequestHeaders()先写了头信息，之后再写body等。然后等数据返回后读取，然后返回给上一层。由于它是最后一个Interceptor所以它不需要再传递请求，直接返回就OK了。

    

### 3. okHttp调用流程总结

1. 构建okHttpClient，并添加需要的配置。如添加自定义的Interceptor，是否重连，设置超时时间，Cookie，Cache等。

2. 配置Request，如请求方法，请求头，请求体等。

3. 通过okHttpClient.newCall(request)生成Call对象，这个Call实际是RealCall。

4. 执行它的enqueue(),最后实际会执行到AsyncCall的execute()中，然后执行它的getResponseWithInterceptorChain()等待返回结果。

   > 在getResponseWithInterceptorChain()中使用了责任链模式，每一个节点是一个interceptor，在哦它的interceptors()中：添加前置逻辑，然后调用proceed方法，传递请求到下一层，等待结果返回，直达最后网络请求完成返回数据，又原路返回执行每一个interceptors的后置逻辑。

5. getResponseWithInterceptorChain()首先执行了自定义的Interceptor的前置逻辑，它获取了用户设置的自定义配置并添加到请求中，然后转交给下一层，并等待结果返回。

6. 第二个执行了RetryAndFollowUpInterceptor的前置逻辑，它初始化了连接对象，如果没有取消请求，则继续转交请求给下一层，并等待结果返回。

7. 第三个执行了BridgeInterceptor的前置逻辑，它处理了header信息，并添加了gzip支持。然后转交请求给下一层，并等待结果返回。

8. 第四个执行了CacheInterceptor的前置逻辑，它创建了缓存策略，如果可以返回缓存，或网络有问题这直接返回。否则转交请求给下一层，并等待结果返回。

9. 第五个执行了ConnectInterceptor的前置逻辑，它构建并建立了连接，然后转交请求给下一层，并等待结果返回。

10. 第五个执行了networkInterceptors的前置逻辑，这里如果设置了，那么可以在请求发出前做最后的处理，然后转交请求给下一层，并等待结果返回。

11. 第五个执行了CallServerInterceptor的前置逻辑，这是最后一个Interceptor，它已经拿到了客户端设置的请求的所有配置，并且连接已经建立。在这里正式开始往网络写请求，并等待返回请求结果。返回结果进行一定的判断处理后直接返回给上一层。

12. 如果设置了networkInterceptors，那么结果先在networkInterceptors的后置逻辑处理后再返回给ConnectInterceptor

13. ConnectInterceptor没有后置逻辑，直接返回给CacheInterceptor

14. CacheInterceptor的后置逻辑对返回结果进行缓存后，然后返回给BridgeInterceptor

15. BridgeInterceptor的后置逻辑对返回结果进行解压缩，然后返回给RetryAndFollowUpInterceptor

16. RetryAndFollowUpInterceptor的后置逻辑对返回结果进行判断，是否需要重定向，是否需要执行重试逻辑，不需要则返回请求结果

17. 如果设置了自定义Interceptor，那么会返回到这里，根据需求对结果进行处理，然后最终返回给调用端。