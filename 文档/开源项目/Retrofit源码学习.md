#   Retrofit源码学习

```
1. 动态代理创建请求接口类
2. 反射获取注解信息构造ServiceMethod，单例设计模式，解析ServiceMethod
3. 适配器模式，根据不同的返回值，为不同的ServiceMethod创建不同的CallAdapter和ConvertAdapter
4. 对于Rxjava方式，根据返回值得参数化类型，策略模式生产不同的Observable
5. 对于返回Call方式，采用装饰模式ExecutorCallAdapter包装了OkHttpCall，在里面增加了切换回主线程的逻辑
```

1. Retrofit.create
```
 public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);

    //1. 可以配置，提前把接口中的方法解析出来保存在serviceMethodCache中
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }

    //2. 动态代理创建service对应的代理类
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }

            //3. 根据method获取对应的ServiceMethod
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);

            //4. 创建OkHttpCall实例，传入该方法对应的ServiceMethod以及对应的参数值
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);

            //5. 对okHttpCall适配，返回Call或者Observable等
            return serviceMethod.adapt(okHttpCall);
          }
        });
  }
```

2. Retrofit.loadServiceMethod
```
  ServiceMethod<?, ?> loadServiceMethod(Method method) {

    //1. 先从缓存中查找
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    //2. 缓存中没找到，同步代码块保证线程安全，执行解析
    synchronized (serviceMethodCache) {
      //再次查找，类似于单例
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder<>(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```

3.  Service.Builder.build
```
    public ServiceMethod build() {

      //1. 根据返回值获取对应的callAdapter，如果是Call，则返回ExecutorCallAdapter
      //如果是Observable等Rxjava相关，则返回RxJava2CallAdapter
      callAdapter = createCallAdapter();
      responseType = callAdapter.responseType();
      if (responseType == Response.class || responseType == okhttp3.Response.class) {
        throw methodError("'"
            + Utils.getRawType(responseType).getName()
            + "' is not a valid response body type. Did you mean ResponseBody?");
      }

      //2.根据返回值以及注解获取responseConverter
      responseConverter = createResponseConverter();

      for (Annotation annotation : methodAnnotations) {
        //3. 解析方法上的注解，
        parseMethodAnnotation(annotation);
      }

      if (httpMethod == null) {
        throw methodError("HTTP method annotation is required (e.g., @GET, @POST, etc.).");
      }

      if (!hasBody) {
        if (isMultipart) {
          throw methodError(
              "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
        }
        if (isFormEncoded) {
          throw methodError("FormUrlEncoded can only be specified on HTTP methods with "
              + "request body (e.g., @POST).");
        }
      }

      //4.解析参数上的注解
      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {
        Type parameterType = parameterTypes[p];
        if (Utils.hasUnresolvableType(parameterType)) {
          throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
              parameterType);
        }

        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        if (parameterAnnotations == null) {
          throw parameterError(p, "No Retrofit annotation found.");
        }

        //5. 将参数上的注解都生产对应的ParamterHandler存在parametrHandler数组中
        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
      }


      if (relativeUrl == null && !gotUrl) {
        throw methodError("Missing either @%s URL or @Url parameter.", httpMethod);
      }
      if (!isFormEncoded && !isMultipart && !hasBody && gotBody) {
        throw methodError("Non-body HTTP method cannot contain @Body.");
      }
      if (isFormEncoded && !gotField) {
        throw methodError("Form-encoded method must contain at least one @Field.");
      }
      if (isMultipart && !gotPart) {
        throw methodError("Multipart method must contain at least one @Part.");
      }

      return new ServiceMethod<>(this);
    }
```

4. ServiceMethod.adapt
```
  T adapt(Call<R> call) {
    //这里callAdapter是由ServiceMethod在建造过程中根据返回类型创建
    return callAdapter.adapt(call);
  }

  对于返回类型为Call：callAdapt为：
    CallAdapter<Object, Call<?>>() {
          @Override public Type responseType() {
            return responseType;
          }

          @Override public Call<Object> adapt(Call<Object> call) {

            //对于外界而言，持有的Call就是ExecutorCallbackCall，装饰模式，装饰OkHttpCall对象
            return new ExecutorCallbackCall<>(callbackExecutor, call);
          }
        };

  对于返回类型为Observable等：RxJava2CallAdapter

   @Override public Object adapt(Call<R> call) {

      //通常都是采用异步请求，使用OkHttp的线程池完成请求，
      //所以，这里实际用的是CallEnqueueObservable，不过默认的是同步，需要在配置的时候设置为异步
      Observable<Response<R>> responseObservable = isAsync
          ? new CallEnqueueObservable<>(call)
          : new CallExecuteObservable<>(call);


      //策略模式返回不同的Observable对象

      Observable<?> observable;
      if (isResult) {
        observable = new ResultObservable<>(responseObservable);
      } else if (isBody) {
        observable = new BodyObservable<>(responseObservable);
      } else {

        //对于普通情况，返回给外界的就是这个
        observable = responseObservable;
      }

      if (scheduler != null) {
        observable = observable.subscribeOn(scheduler);
      }

      if (isFlowable) {
        return observable.toFlowable(BackpressureStrategy.LATEST);
      }
      if (isSingle) {
        return observable.singleOrError();
      }
      if (isMaybe) {
        return observable.singleElement();
      }
      if (isCompletable) {
        return observable.ignoreElements();
      }
      return observable;
    }

```

5.  ExecutorCallbackCall.enqueue
```
 @Override public void enqueue(final Callback<T> callback) {
      checkNotNull(callback, "callback == null");

      //通过包装的OkHttpCall完成enqueue

      delegate.enqueue(new Callback<T>() {
        @Override public void onResponse(Call<T> call, final Response<T> response) {

          //这里通过callbackExecutor线程切换主线程
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

        @Override public void onFailure(Call<T> call, final Throwable t) {

          //这里通过callbackExecutor线程切换回主线程
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              callback.onFailure(ExecutorCallbackCall.this, t);
            }
          });
        }
      });
    }
```

6.  CallEnqueueObservable.subscribeActual
```
  @Override protected void subscribeActual(Observer<? super Response<T>> observer) {
    // Since Call is a one-shot type, clone it for each new observer.
    Call<T> call = originalCall.clone();
    CallCallback<T> callback = new CallCallback<>(call, observer);
    observer.onSubscribe(callback);

    //这里的call就是OkHttpCall
    call.enqueue(callback);
  }

   private static final class CallCallback<T> implements Disposable, Callback<T> {
      private final Call<?> call;
      private final Observer<? super Response<T>> observer;
      private volatile boolean disposed;
      boolean terminated = false;

      CallCallback(Call<?> call, Observer<? super Response<T>> observer) {
        this.call = call;
        this.observer = observer;
      }

      @Override public void onResponse(Call<T> call, Response<T> response) {
        if (disposed) return;

        try {
          //通知外界数据，该回调在OkHttp线程池中的线程中
          observer.onNext(response);

          if (!disposed) {
            terminated = true;
            observer.onComplete();
          }
        } catch (Throwable t) {
          if (terminated) {
            RxJavaPlugins.onError(t);
          } else if (!disposed) {
            try {
              observer.onError(t);
            } catch (Throwable inner) {
              Exceptions.throwIfFatal(inner);
              RxJavaPlugins.onError(new CompositeException(t, inner));
            }
          }
        }
      }

      @Override public void onFailure(Call<T> call, Throwable t) {
        if (call.isCanceled()) return;

        //通知外界错误，也在子线程中
        try {
          observer.onError(t);
        } catch (Throwable inner) {
          Exceptions.throwIfFatal(inner);
          RxJavaPlugins.onError(new CompositeException(t, inner));
        }
      }

      @Override public void dispose() {
        disposed = true;
        call.cancel();
      }

      @Override public boolean isDisposed() {
        return disposed;
      }
    }
```

7. OkHttpCall.enqueue
```
 @Override public void enqueue(final Callback<T> callback) {
    checkNotNull(callback, "callback == null");

    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      call = rawCall;
      failure = creationFailure;
      if (call == null && failure == null) {
        try {

          //1. 构造OkHttp中的Call，RealCall
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          throwIfFatal(t);
          failure = creationFailure = t;
        }
      }
    }

    if (failure != null) {
      callback.onFailure(this, failure);
      return;
    }

    if (canceled) {
      call.cancel();
    }


    //执行OkHttp的逻辑
    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
        Response<T> response;
        try {

          //解析数据
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }

        try {

           //回调数据
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        callFailure(e);
      }

      private void callFailure(Throwable e) {
        try {
          //回调请求失败
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
    });
  }


    private okhttp3.Call createRawCall() throws IOException {
      //通过serviceMethod以及实际参数值构造对应的RealCall
      okhttp3.Call call = serviceMethod.toCall(args);
      if (call == null) {
        throw new NullPointerException("Call.Factory returned null.");
      }
      return call;
    }

    //ServiceMethod.toCall方法

    /** Builds an HTTP request from method arguments. */
    okhttp3.Call toCall(@Nullable Object... args) throws IOException {

      //1.根据之前解析出的请求方法，请求地址，请求头等信息构造RequestBuilder
      RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
          contentType, hasBody, isFormEncoded, isMultipart);

      @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
      ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

      int argumentCount = args != null ? args.length : 0;
      if (argumentCount != handlers.length) {
        throw new IllegalArgumentException("Argument count (" + argumentCount
            + ") doesn't match expected count (" + handlers.length + ")");
      }

      //将参数值设置到对应的参数上，并添加进requestBuiler
      for (int p = 0; p < argumentCount; p++) {
        handlers[p].apply(requestBuilder, args[p]);
      }

      //默认情况下callFactory都是OkHttpClient，返回RealCall
      return callFactory.newCall(requestBuilder.build());
    }
```

8. OkHttpCall.parseResponse
```
 Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {

    //1. 取出响应体
    ResponseBody rawBody = rawResponse.body();

    // Remove the body's source (the only stateful object) so we can pass the response along.
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();

    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
      try {
        // Buffer the entire body to avoid future I/O.
        ResponseBody bufferedBody = Utils.buffer(rawBody);
        return Response.error(bufferedBody, rawResponse);
      } finally {
        rawBody.close();
      }
    }

    if (code == 204 || code == 205) {//204 205 表示没有content
      rawBody.close();
      return Response.success(null, rawResponse);
    }

    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {

      //2. 通过serviceMethod去解析响应体
      T body = serviceMethod.toResponse(catchingBody);

      //3. 将解析后的数据放在Response的body字段返回
      return Response.success(body, rawResponse);

    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }
```

9. ServiceMethod.toResponse
```
  R toResponse(ResponseBody body) throws IOException {
    //这里responseConverter同样是在ServiceMethod在被构建的时候根据返回值创建
    //如果是ResponseBody类型，则会通过BuiltInConverters返回对应的convert
    //如果是其他，则会返回GsonResponseBodyConverter。对于GsonConvertFactroy也是需要配置的

    return responseConverter.convert(body);
  }

   //BuiltInConverters，

    public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
        Retrofit retrofit) {

      //当返回类型是ResponseBody时，如果有Streaming注解，则不会将流读入内存，否则会读入内存
      if (type == ResponseBody.class) {
        return Utils.isAnnotationPresent(annotations, Streaming.class)
            ? StreamingResponseBodyConverter.INSTANCE
            : BufferingResponseBodyConverter.INSTANCE;
      }
      if (type == Void.class) {
        return VoidResponseBodyConverter.INSTANCE;
      }
      return null;
    }

      static final class StreamingResponseBodyConverter
          implements Converter<ResponseBody, ResponseBody> {
        static final StreamingResponseBodyConverter INSTANCE = new StreamingResponseBodyConverter();

        @Override public ResponseBody convert(ResponseBody value) {
          //直接返回，这个时候需要使用者使用完了后自己关闭流
          return value;
        }
      }

      static final class BufferingResponseBodyConverter
          implements Converter<ResponseBody, ResponseBody> {
        static final BufferingResponseBodyConverter INSTANCE = new BufferingResponseBodyConverter();

        @Override public ResponseBody convert(ResponseBody value) throws IOException {
          try {
            // Buffer the entire body to avoid future I/O.
            //读取到内存缓存
            return Utils.buffer(value);
          } finally {
            //关闭流
            value.close();
          }
        }
      }

    static final class VoidResponseBodyConverter implements Converter<ResponseBody, Void> {
      static final VoidResponseBodyConverter INSTANCE = new VoidResponseBodyConverter();

      @Override public Void convert(ResponseBody value) {
        //直接关闭流
        value.close();
        return null;
      }
    }

    //GsonResponseBodyConverter

      @Override public T convert(ResponseBody value) throws IOException {
        JsonReader jsonReader = gson.newJsonReader(value.charStream());
        try {
          T result = adapter.read(jsonReader);
          if (jsonReader.peek() != JsonToken.END_DOCUMENT) {
            throw new JsonIOException("JSON document was not fully consumed.");
          }
          return result;
        } finally {
            //关闭流
          value.close();
        }
      }

```