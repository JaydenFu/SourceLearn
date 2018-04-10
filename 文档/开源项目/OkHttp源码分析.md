#   OkHttp源码分析

```
重要知识点：
1. Dispatcher请求分发
2. 通过RealInterceptorChain完成拦截器的顺序调用
3. RetryAndFollowUpInterceptor处理重定向，以及创建StreamAllocation
4. CacheInterceptor处理缓存，缓存通过DiskLruCache实现
5. ConnectionInterceptor：建立连接（可能是用连接池中找到，也可能是新创建）
6. CallServerInterceptor：写入请求，读取响应
7. ConnectionPool：连接池管理连接,内部通过ArrayDeque队列管理,内部有个cleanRunnable进行清理工作
8. RouteDataBase管理失败过的Route
```

1.  OkHttp默认配置：
```
    //超时时间：单位ms
    connectTimeout = 10_000;
    readTimeout = 10_000;
    writeTimeout = 10_000;

    //默认的连接池
    public ConnectionPool() {
        //最大空闲连接5，keep-alive时长5分钟
        this(5, 5, TimeUnit.MINUTES);
    }

    public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
        this.maxIdleConnections = maxIdleConnections;
        this.keepAliveDurationNs = timeUnit.toNanos(keepAliveDuration);
        ...
    }

    //默认Dispatcher
    private int maxRequests = 64;//最大并发请求数
    private int maxRequestsPerHost = 5;//每个Host最大并发请求数
    //默认线程池，核心线程数为0，最大线程数为Integer.MAX_VALUE,线程空闲时间60s，才有SynchornousQueue阻塞队列
    executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));

```

2.  构建Request
```
        //通过Request.Builder构建Request,可以配置url，请求方式，请求头，请求体，缓存控制
        Request request = new Request.Builder()
                .url("http://www.baidu.com")
                .get()
                .cacheControl(CacheControl.FORCE_CACHE)
                .header("host","www.baidu.com")
                .build();

        //CacheControl通过CacheContol.Builder构造
        CacheControl cacheControl = new CacheControl.Builder()
                        .maxAge(1000, TimeUnit.DAYS)
                        .noCache()
                        .build();
```

3.  通过Request构建Call
```
    Call call = okHttpClient.newCall(request);

    //OkHttpClient的newCall方法
    @Override
    public Call newCall(Request request) {
        //通过RealCall.newRealCall方法构建一个新的RealCall实例
        return RealCall.newRealCall(this, request, false /* for web socket */);
    }

    //RealCall的静态方法newRealCall
    static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {

        //创建RealCall实例，这里client是我们外面传入的OkHttpClient实例，originalRequest是
        //最开始构建的Request实例，对应普通的请求而言，forWebSocket为false
        RealCall call = new RealCall(client, originalRequest, forWebSocket);
        call.eventListener = client.eventListenerFactory().create(call);
        return call;
    }

```

4.  通过RealCall发起异步请求
```
    //Callback是一个请求完成后的回调，这里callback运行在子线程中
    call.enqueue(new Callback() {
                @Override
                public void onFailure(Call call, IOException e) {
                    //请求失败,通常是发生了IO异常
                }

                @Override
                public void onResponse(Call call, Response response) throws IOException {
                    //根据response的状态吗判断是否请求成功
                }
            });
```

5.  RealCall的enqueue方法
```
    @Override
    public void enqueue(Callback responseCallback) {
        synchronized (this) {
           //如果一个RealCall被enqueue过，则不能再次enqueue，否则会抛异常
          if (executed) throw new IllegalStateException("Already Executed");
          executed = true;
        }
        captureCallStackTrace();
        eventListener.callStart(this);

        //通过OkHttpCilent中的Dispatcher执行实际的enqueue，这里Dispatcher在我们
        //够着OkHttpCilent实例时，可以配置我们自己的Dispatcher，否则将使用默认的配置的
        //Dispatcher
        client.dispatcher().enqueue(new AsyncCall(responseCallback));
    }

    AsyncCall是RealCall的一个内部类，所以它持有外部对象RealCall的引用
    
```

6. Dispatcher.enqueue
```
  //readyAsyncCalls和runningAsyncCalls是两个队列，分别用于保存正在执行的AsyncCall以及等待执行的AsyncCall
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  //这里采用同步方法的方式，保证线程安全
  synchronized void enqueue(AsyncCall call) {

    //1. 判断当正在运行的请求数小于maxRequests（默认64，可以设置）时，并且对单给Host的请求数量小于maxRequestPerHost
    //（默认5，可以设置）时，就执行提交到线程池运行，同时将AsyncCall添加到runningAsncCalls队列中
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      //2. 当1的情况不满足时，则将当前AsyncCall添加到等待执行的队列当中
      readyAsyncCalls.add(call);
    }
  }

  对于Dispatcher.executorServices():
    //该线程池核心线程数量为0，Keep-Alive时长为60s，因此当空闲的时候，该线程池会释放掉所有线程，
    //因为核心线程数为0，所以当有任务提交时，首先会将提交offer到SynchronousQueue队列，这个时候线程池中如果有空
    //闲线程，则会相应的执行SynchronousQueue的poll方法，那么任务就给空闲线程执行，如果没有空闲线程，则offer会提交
    //失败，因为offer操作是有超时的，超时时间是0，所以是立即返回的。
    //所以为什么需要在Dispatcher的enqueue方法中限制最大并发数，不然这个线程池，只要有任务提交，且没有空闲线程可执行，就会创建线程
    public synchronized ExecutorService executorService() {
        if (executorService == null) {
        executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
            new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
        }
        return executorService;
    }
```

7.  AsyncCall继承NamedRunnable，NameRunnable实现了Runnable，所以提交到线程池后，后执行AsyncCall的run方法
    在run方法中又调用了AsyncCall的execute方法
```
@Override protected void execute() {

      //该标志用于避免重复回调
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();

        //当RealCall执行cancle方法时，会使retryAndFollowUpIntrceptor.isCanceled返回true
        //表示任务取消
        if (retryAndFollowUpInterceptor.isCanceled()) {

          signalledCallback = true;
          //任务取消了，回调onFailure方法，异常描述是canceled
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;

          //任务没有取消，回到onResponse方法，表示请求被响应
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {

        //异常情况
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {

          //如果异常抛出前，还没有回调过，则回调onFailure，通知请求失败
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {

        //最后通过Dispatcher执行finished方法，该方法内部会将当前AysncCall从正在执行的队列中移除，并且尝试将等待请求的任务提交到线程池执行
        client.dispatcher().finished(this);
      }
    }
  }
```

8.  Dispatcher.finished(AsyncCall call)
```
    void finished(AsyncCall call) {
        finished(runningAsyncCalls, call, true);
    }

    private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
        int runningCallsCount;
        Runnable idleCallback;

        //因为finished方法是在线程池的线程中调用，所以这里使用的同步代码块带保证线程安全
        synchronized (this) {
            //1. 首先从calls中移除call
            if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");

            //2. 当promoteCalls为true时，执行promoteCalls，对于完成异步请求而言，promotCalls是ture，同步请求为false
            if (promoteCalls) promoteCalls();

            //3.计算正在执行请求数量，包括AsyncCall和SyncCall
            runningCallsCount = runningCallsCount();
            idleCallback = this.idleCallback;
        }

        //当没有任务执行时，且idleCallback不为null时，执行idleCallback的run方法
        //对于默认的Dispatcher而言，没有设置idleCallback
        if (runningCallsCount == 0 && idleCallback != null) {
            idleCallback.run();
        }
    }

    //promoteCalls方法的目的是将准备执行的AsyncCall提交到线程池执行
    private void promoteCalls() {
        //1.如果当前异步运行的任务数量大于maxRequests，则不作任何操作
        if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.

        //2. 当准备执行的队列为空时，也不作任何操作
        if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.


        for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
            AsyncCall call = i.next();

            //循环准备执行的队列，判断call的host对应的当前执行的数量，如果小于maxRequestPerHost，则提交到线程池执行,
            //并且AsyncCall从等待执行队列中移除，将其加入正在执行的队列中
            if (runningCallsForHost(call) < maxRequestsPerHost) {
                i.remove();
                runningAsyncCalls.add(call);
                executorService().execute(call);
            }

            //直到异步运行的数量大于最大限制时，结束循环
            if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
        }
    }
```

9. AsyncCall.getResponseWithInterceptorChain()
```
    Response getResponseWithInterceptorChain() throws IOException {

        //1. 创建一个拦截器集合，用于存所有拦截器
        List<Interceptor> interceptors = new ArrayList<>();
        //2. 首先添加client中设置的拦截器
        interceptors.addAll(client.interceptors());
        //3. 其次添加每个RealCall自己内部的retryAndFollowUpInterceptor，这个拦截器在RealCall的构造方法中初始化
        //负责失败重试，重定向，StreamAllocation在其内部创建
        interceptors.add(retryAndFollowUpInterceptor);

        //4. 然后在添加BridgeInterceptor拦截器，该拦截器主要是处理请求头，响应头，Gzip解压操作
        interceptors.add(new BridgeInterceptor(client.cookieJar()));

        //5. CacheIntceptor根据HTTP请求头，响应头中的一些缓存相关的字段处理缓存
        interceptors.add(new CacheInterceptor(client.internalCache()));

        //6. ConnectInterceptor 负责和服务器建立连接
        interceptors.add(new ConnectInterceptor(client));

        //添加OkHttpClient中配置的用于在产生网络访问时的拦截器
        if (!forWebSocket) {
            nterceptors.addAll(client.networkInterceptors());

        //7. CallServerInterceptor 向服务器发送数据，从服务器读取响应数据
        interceptors.add(new CallServerInterceptor(forWebSocket));


        //构造RealInterceptorChain，就是由该类来控制拦截器的按序调用,初始构造时，传入的StreamAllocation,HttpCodec,
        //RealConnection都为null，index初始为0
        Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
            originalRequest, this, eventListener, client.connectTimeoutMillis(),
            client.readTimeoutMillis(), client.writeTimeoutMillis());

        return chain.proceed(originalRequest);
    }
```

10. RealInterceptorChain.proceed
```
    @Override
    public Response proceed(Request request) throws IOException {
        return proceed(request, streamAllocation, httpCodec, connection);
    }

    public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
          RealConnection connection) throws IOException {

        //index用于控制拦截器的顺序访问
        if (index >= interceptors.size()) throw new AssertionError();

        calls++;

        // If we already have a stream, confirm that the incoming request will use it.
        if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
          throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
              + " must retain the same host and port");
        }

        // If we already have a stream, confirm that this is the only call to chain.proceed().
        if (this.httpCodec != null && calls > 1) {
          throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
              + " must call proceed() exactly once");
        }


        //再次创建一个RealInterceptorChain，对index+1,这里有多少个拦截器，在调用过程中就会创建多少个实例
        //当执行最后一个拦截器CallServerInterceptor的intercept方法时，虽然传入了next，但是实际并不会在执行其proceed方法，
        //而对于之前的拦截器，如果没有直接返回（比如CacheInterceptor中找到了可使用的缓存），则都会调用其proceed方法，从而继续
        //向后面的拦截器传递
        //也就是说在整个过程中，完全是通过index控制调用的哪一个拦截器
        RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
            connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
            writeTimeout);

        //取出拦截器列表中第index位置的拦截器，执行其interceptor方法
        Interceptor interceptor = interceptors.get(index);
        Response response = interceptor.intercept(next);

        //当所有的拦截器执行完成后，返回到这里继续执行

        // Confirm that the next interceptor made its required call to chain.proceed().
        if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
          throw new IllegalStateException("network interceptor " + interceptor
              + " must call proceed() exactly once");
        }

        // Confirm that the intercepted response isn't null.
        if (response == null) {
          throw new NullPointerException("interceptor " + interceptor + " returned null");
        }

        if (response.body() == null) {
          throw new IllegalStateException(
              "interceptor " + interceptor + " returned a response with no body");
        }
        return response;
    }
```

11. RetryAndFollowUpInterceptor.intercept
```
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        RealInterceptorChain realChain = (RealInterceptorChain) chain;
        Call call = realChain.call();
        EventListener eventListener = realChain.eventListener();


        //1.创建新的StreamAllocation实例，并复制给成员变量streamAllocation
        //这里主要传入了3个参数，ConnectionPool，Address，以及Call
        //ConnectionPool传入的是OkHttpClient中配置的ConnectionPool
        //Address是通过createAddress方法创建的实例
        //Call 其实就是我们代表当前请求任务的AsyncCall
        StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
            createAddress(request.url()), call, eventListener, callStackTrace);
        this.streamAllocation = streamAllocation;

        int followUpCount = 0;
        Response priorResponse = null;

        //循环处理，当重定向或者超时或其他某些异常时，会重新发起请求，直到获取到响应或者重新请求超过一定次数
        while (true) {
            //如果在请求中，通过RealCall.cancel方法取消了，则释放straemAllocation，然后抛出异常
            if (canceled) {
                streamAllocation.release();
                throw new IOException("Canceled");
            }

            Response response;
            boolean releaseConnection = true;

            try {
                response = realChain.proceed(request, streamAllocation, null, null);
                releaseConnection = false;
            } catch (RouteException e) {
                // The attempt to connect via a route failed. The request will not have been sent.
                //当catch到RouteException时，不可恢复就抛出异常
                if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
                  throw e.getLastConnectException();
                }
                releaseConnection = false;
                continue;
            } catch (IOException e) {
                // An attempt to communicate with a server failed. The request may have been sent.
                //IO异常时，同样根据请求是否可以被恢复重新发送，决定是否重新请求或抛异常
                boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
                if (!recover(e, streamAllocation, requestSendStarted, request)) throw e;
                releaseConnection = false;
                continue;
            } finally {
                // We're throwing an unchecked exception. Release any resources.
                if (releaseConnection) {
                  streamAllocation.streamFailed(null);
                  streamAllocation.release();
                }
            }

            // Attach the prior response if it exists. Such responses never have a body.
            if (priorResponse != null) {
                response = response.newBuilder()
                    .priorResponse(priorResponse.newBuilder()
                            .body(null)
                            .build())
                    .build();
            }

            //检测是否需要重定向或者请求失败了
            //401，407：需要认证，默认没配置，返回null，如果配置了会相应的根据配置的情况返回重新请求的request
            //307，308：如果不是get请求，也不是head请求，则返回null
            //300，301，302，303：根据OkHttpClient是否配置允许重定向，返回相应值，默认允许重定向，
            //                   这个时候根据响应头中的location重新构造request
            // 408： 根据OkHttpClient配置的是否重试策略，尝试重新请求，当前面一次已经是408，这次又是408时不再重新请求
            Request followUp = followUpRequest(response, streamAllocation.route());

            if (followUp == null) {
                if (!forWebSocket) {
                  streamAllocation.release();
                }
                return response;
            }

            closeQuietly(response.body());

            if (++followUpCount > MAX_FOLLOW_UPS) {
                streamAllocation.release();
                throw new ProtocolException("Too many follow-up requests: " + followUpCount);
            }

            if (followUp.body() instanceof UnrepeatableRequestBody) {
                streamAllocation.release();
                throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
            }

            //如果请求地址发生改变，则需要重新创建新的StreamAllocation实例
            if (!sameConnection(response, followUp.url())) {
                streamAllocation.release();
                streamAllocation = new StreamAllocation(client.connectionPool(),
                    createAddress(followUp.url()), call, eventListener, callStackTrace);
                this.streamAllocation = streamAllocation;
            } else if (streamAllocation.codec() != null) {
                throw new IllegalStateException("Closing the body of " + response
                    + " didn't close its backing stream. Bad interceptor?");
            }

            request = followUp;
            priorResponse = response;
        }
    }


    private Address createAddress(HttpUrl url) {
        SSLSocketFactory sslSocketFactory = null;
        HostnameVerifier hostnameVerifier = null;
        CertificatePinner certificatePinner = null;

        //如果当前是https请求，获取SSLSocketFactory以及HostnameVerfier域名验证器，CertificatePinner证书
        if (url.isHttps()) {
            sslSocketFactory = client.sslSocketFactory();
            hostnameVerifier = client.hostnameVerifier();
            certificatePinner = client.certificatePinner();
        }

        //可以看到Address表示一个地址包装类，封装了主机号，端口号等相关信息
        return new Address(url.host(), url.port(), client.dns(), client.socketFactory(),
            sslSocketFactory, hostnameVerifier, certificatePinner, client.proxyAuthenticator(),
            client.proxy(), client.protocols(), client.connectionSpecs(), client.proxySelector());
    }
```

12. ConnectInterceptor.interceptor
```
    @Override
    public Response intercept(Chain chain) throws IOException {
        RealInterceptorChain realChain = (RealInterceptorChain) chain;
        Request request = realChain.request();
        StreamAllocation streamAllocation = realChain.streamAllocation();

        // We need the network to satisfy this request. Possibly for validating a conditional GET.
        //对于非get请求，会对连接作额外的检测
        boolean doExtensiveHealthChecks = !request.method().equals("GET");

        //通过streamAllocation创建HttpCodec实例，HttpCodec代表一个流
        HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
        //通过streamAllocation获取一个RealConnection，其代表一个连接
        RealConnection connection = streamAllocation.connection();

        //将streamAllocation，httpCodec，connection作为参数继续在拦截器链路上传递
        return realChain.proceed(request, streamAllocation, httpCodec, connection);
    }
```

13. StreamAllocation.newStream过程
```
    public HttpCodec newStream(
          OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {

        //获取配置的连接超时时长 默认10s
        int connectTimeout = chain.connectTimeoutMillis();
        //获取读取超时时长 默认10s
        int readTimeout = chain.readTimeoutMillis();
        //获取写超时市场 默认10s
        int writeTimeout = chain.writeTimeoutMillis();
        //获取ping间隔  默认0
        int pingIntervalMillis = client.pingIntervalMillis();
        //连接失败是否重试， 默认true
        boolean connectionRetryEnabled = client.retryOnConnectionFailure();

        try {
          //找到一条可用的连接
          RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
              writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);

          //HttpCodec可以理解为解码器，作为网络读写的管理类
          //写入请求头，请求体，读取响应头，响应体
          HttpCodec resultCodec = resultConnection.newCodec(client, chain, this);

          synchronized (connectionPool) {
            codec = resultCodec;
            return resultCodec;
          }
        } catch (IOException e) {
          throw new RouteException(e);
        }
    }
```

14. StreamAllocation.findHealthyConnection
```
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
      boolean doExtensiveHealthChecks) throws IOException {

    while (true) {

      //1.获取到一个链接
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          pingIntervalMillis, connectionRetryEnabled);

      // If this is a brand new connection, we can skip the extensive health checks.
      //如果这是一个新的连接，则直接返回
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
          return candidate;
        }
      }

      //检测这个链接还是否可用（socket是否closed，输入，输出流是否被关闭），不可用，则放弃这个链接，从新获取链接，只有非get请求，doExtensiveHealthChecks才为true
      //
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        //noNewStreams会禁止再在当前链接上创建流
        noNewStreams();
        continue;
      }

      //可用，则返回该链接
      return candidate;
    }
  }


    public void noNewStreams() {
      Socket socket;
      Connection releasedConnection;
      synchronized (connectionPool) {
        releasedConnection = connection;
        //在deallocate中可能会将连接从连接池中移除
        socket = deallocate(true, false, false);
        if (connection != null) releasedConnection = null;
      }
      closeQuietly(socket);
      if (releasedConnection != null) {
        eventListener.connectionReleased(call, releasedConnection);
      }
    }
```

15. StreamAllocation.findConnection
```
   //1. 尝试从连接池中获取链接，找到了直接返回连接
   //2. 如果没有找到可用的连接，则创建新的连接，并将其放入连接池中
  private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    RealConnection result = null;
    Route selectedRoute = null;
    Connection releasedConnection;
    Socket toClose;


    synchronized (connectionPool) {
      if (released) throw new IllegalStateException("released");
      if (codec != null) throw new IllegalStateException("codec != null");
      if (canceled) throw new IOException("Canceled");

      // Attempt to use an already-allocated connection. We need to be careful here because our
      // already-allocated connection may have been restricted from creating new streams.
      releasedConnection = this.connection;
      toClose = releaseIfNoNewStreams();

      //如果当前StreamAllocation的connection为不null，则将connection赋值给result
      if (this.connection != null) {
        result = this.connection;
        releasedConnection = null;
      }

      if (!reportedAcquired) {
        // If the connection was never reported acquired, don't report it as released!
        releasedConnection = null;
      }

      //对于每个请求的第一次findConnection，result一定为null
      if (result == null) {

        //试图从ConnectionPool从获取一个可用的链接
        Internal.instance.get(connectionPool, address, this, null);

        //如果从链接池中找到了可用的链接，则不需要再通过路由创建新的链接
        if (connection != null) {
          foundPooledConnection = true;
          result = connection;
        } else {
          selectedRoute = route;
        }
      }
    }
    closeQuietly(toClose);

    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
    }
    if (result != null) {
      // If we found an already-allocated or pooled connection, we're done.
      return result;
    }

    // If we need a route selection, make one. This is a blocking operation.
    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }

    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");

      if (newRouteSelection) {
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        List<Route> routes = routeSelection.getAll();
        for (int i = 0, size = routes.size(); i < size; i++) {
          Route route = routes.get(i);
          Internal.instance.get(connectionPool, address, this, route);
          if (connection != null) {
            foundPooledConnection = true;
            result = connection;
            this.route = route;
            break;
          }
        }
      }

      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }

        // Create a connection and assign it to this allocation immediately. This makes it possible
        // for an asynchronous cancel() to interrupt the handshake we're about to do.
        route = selectedRoute;
        refusedStreamCount = 0;
        //创建新的链接
        result = new RealConnection(connectionPool, selectedRoute);
        acquire(result, false);
      }
    }

    // If we found a pooled connection on the 2nd time around, we're done.
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }

    // Do TCP + TLS handshakes. This is a blocking operation.
    //开始连接
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
        connectionRetryEnabled, call, eventListener);
    routeDatabase().connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      reportedAcquired = true;

      // Pool the connection.
      //将新创建的连接放进连接池
      Internal.instance.put(connectionPool, result);

      // If another multiplexed connection to the same address was created concurrently, then
      // release this connection and acquire that one.
      if (result.isMultiplexed()) {
        socket = Internal.instance.deduplicate(connectionPool, address, this);
        result = connection;
      }
    }
    closeQuietly(socket);

    eventListener.connectionAcquired(call, result);
    return result;
  }
```

16. RealConnection.newCodec
```
  public HttpCodec newCodec(OkHttpClient client, Interceptor.Chain chain,
      StreamAllocation streamAllocation) throws SocketException {
    if (http2Connection != null) {
      return new Http2Codec(client, chain, streamAllocation, http2Connection);
    } else {

      //设置超时时长
      socket.setSoTimeout(chain.readTimeoutMillis());
      source.timeout().timeout(chain.readTimeoutMillis(), MILLISECONDS);
      sink.timeout().timeout(chain.writeTimeoutMillis(), MILLISECONDS);

      //创建网络输入输出的管理类：用于写请求头，请求行，读取响应头，响应行
      return new Http1Codec(client, streamAllocation, source, sink);
    }
  }
```

17. CallServerInterceptor.interceptor
```
 @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    HttpCodec httpCodec = realChain.httpStream();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();

    long sentRequestMillis = System.currentTimeMillis();

    realChain.eventListener().requestHeadersStart(realChain.call());

    //1.写入请求头
    httpCodec.writeRequestHeaders(request);
    realChain.eventListener().requestHeadersEnd(realChain.call(), request);

    Response.Builder responseBuilder = null;
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
      // Continue" response before transmitting the request body. If we don't get that, return
      // what we did get (such as a 4xx response) without ever transmitting the request body.
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        httpCodec.flushRequest();
        realChain.eventListener().responseHeadersStart(realChain.call());
        responseBuilder = httpCodec.readResponseHeaders(true);
      }

      if (responseBuilder == null) {
        // Write the request body if the "Expect: 100-continue" expectation was met.
        realChain.eventListener().requestBodyStart(realChain.call());

        //2. 写入请求体
        long contentLength = request.body().contentLength();
        CountingSink requestBodyOut =
            new CountingSink(httpCodec.createRequestBody(request, contentLength));

        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
        realChain.eventListener()
            .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
      } else if (!connection.isMultiplexed()) {
        streamAllocation.noNewStreams();
      }
    }

    httpCodec.finishRequest();

    if (responseBuilder == null) {
      realChain.eventListener().responseHeadersStart(realChain.call());

      //3.读取响应头
      responseBuilder = httpCodec.readResponseHeaders(false);
    }

    Response response = responseBuilder
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    int code = response.code();
    if (code == 100) {
      // server sent a 100-continue even though we did not request one.
      // try again to read the actual response
      responseBuilder = httpCodec.readResponseHeaders(false);

      response = responseBuilder
              .request(request)
              .handshake(streamAllocation.connection().handshake())
              .sentRequestAtMillis(sentRequestMillis)
              .receivedResponseAtMillis(System.currentTimeMillis())
              .build();

      code = response.code();
    }

    realChain.eventListener()
            .responseHeadersEnd(realChain.call(), response);

    if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {

      //4.将响应体封装RealResponseBody中存到Response的body中
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
