#   EventBus源码学习

```
重点总结：
1.提供2种方式： 运行时反射方式解析注解，编译时通过注解处理器生成索引类（这种效率会高一些，但是需要自己进行额外配置）
2.ThreadMode5个值，每个值代表订阅方法不同的执行方式
3.3种Poster：HandlerPoster，BackgroundPoster，AsyncPoster
4.一个线程池：Executors.newCachedThreadPool()
5.注册后，在EventBus中保存两个map：1个是事件的Class为key，所有订阅（订阅者，订阅方法二者一定定义一个订阅）为值
  1个是订阅者为key，订阅者的所有订阅方法为值得map
6.注册一个订阅者时，对应的订阅方法通过SubscriberMethodFinder进行查找，这里就是通过2种方式，反射或从编译生成的索引中查找
```

1. 简单使用：
```
public class EventBusTest {

    public void regist(){
        //1. 使用之前需要注册
        EventBus.getDefault().register(this);
    }

    //2. 通过Subscribe注解，标识接受事件的方法，EventBus框架会在编译时，根据Subscirbe注解，将所有接受事件方法
    //进行统一管理
    //指定threadMode：5种值：
    //ThreadMode.Posting: 接受事件的方法在发送事件的线程执行，默认值
    //ThreadMode.Main: 接受方法在主线程被回调，如果发送线程也是主线程，则直接调用，否则通过消息队列，发送到主线程执行
    //ThreadMode.Main_Order: 接受方法在主线程被回调，无论发送线程如何，都通过消息队列，发送到主线程执行
    //ThreadMode.BACKGROUND: 接受方法在后台线程执行，如果发送线程不是主线程，直接在发送线程执行，如果发送线程是主线程，则会用一个后台线程执行，这个后台线程是唯一的，因此为了
                             不阻塞这个后台线程，任务需要尽快完成
    //ThreadMode.ASYNC: 接受方法始终在另外一个线程执行，通过内部线程池实现的
    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onMessage(EventObject eventObject){
        //处理接受到的事件
    }

    public void sendEvent(){
        //2. 发送事件
        //两种方式：
        1.普通发送
        EventBus.getDefault().post(new EventObject());
        2.粘性发送：发送一个粘性事件后，框架会保存这个事件，如果有新的接受者注册了，则将事件发送给接受者
        EventBus.getDefault().postSticky(new EventObject());
    }


    public void destroy(){
        //3. 取消注册，在Activity，Fragment销毁时，一定需要取消注册，否则会出现内存泄漏
        EventBus.getDefault().unregister(this);
    }
}
```

2.  EventBus.getDefault()
```
    //单列模式构建进程唯一的EventBus实例
    static volatile EventBus defaultInstance;

    public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }
```

3. 默认EventBus初始化
```

    private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();

    public EventBus() {
        //通过默认的EventBusBuilder构建EventBus，也就是说参数都是默认的
        this(DEFAULT_BUILDER);
    }

    EventBus(EventBusBuilder builder) {
        logger = builder.getLogger();
        subscriptionsByEventType = new HashMap<>();
        typesBySubscriber = new HashMap<>();
        stickyEvents = new ConcurrentHashMap<>();
        mainThreadSupport = builder.getMainThreadSupport();

        //主线程提交器
        mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
        //后台线程提交器
        backgroundPoster = new BackgroundPoster(this);
        //异步线程提交器
        asyncPoster = new AsyncPoster(this);

        indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;

        //生成订阅方法查找器，如果没有采用编译时，注解处理器，则subscriberInfoIndexes为null，默认是没采用的
        //另外两个参数默认也是false
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);

        .....
        executorService = builder.executorService;
    }
```

4. EventBusBuilder
```
//用于构建EventBus，这里面的参数都是可以自定义的，通常如果
public class EventBusBuilder {
    private final static ExecutorService DEFAULT_EXECUTOR_SERVICE = Executors.newCachedThreadPool();

    boolean logSubscriberExceptions = true;
    boolean logNoSubscriberMessages = true;
    boolean sendSubscriberExceptionEvent = true;
    boolean sendNoSubscriberEvent = true;
    boolean throwSubscriberException;
    boolean eventInheritance = true;

    //是否忽略编译时生成的索引，默认为false
    boolean ignoreGeneratedIndex;
    //验证订阅方法的合法性，默认为false，否则对于不合法的订阅方法会抛异常
    boolean strictMethodVerification;

    //用于执行的线程池，是Executors.newCacheThreadPool()
    ExecutorService executorService = DEFAULT_EXECUTOR_SERVICE;
    List<Class<?>> skipMethodVerificationForClasses;

    //SubscriberInfoIndex为编译时解析注解生成的信息，如果我们没有依赖eventbus-annotation-processor
    //且在build文件中没有配置对应的生成类名，则不会生成对应的index，如果要使用，需要在生成EventBus实例前
    //通过ventBusBuilder.addIndex方法添加我们生成的SubscriberInfoIndex对象。
    List<SubscriberInfoIndex> subscriberInfoIndexes;
    Logger logger;
    MainThreadSupport mainThreadSupport;

     EventBusBuilder() {
     }
    ...
}


//对于配置：2个部分

android {
    compileSdkVersion 26

    defaultConfig {
        ....

        //1. 配置生成的类名
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [eventBusIndex: 'org.greenrobot.eventbusperf.MyEventBusIndex']
            }
        }
    }
}

dependencies {
    //依赖eventbus，同时要声明annotationProcessor
    implementation 'org.greenrobot:eventbus:3.1.1'
    annotationProcessor 'org.greenrobot:eventbus-annotation-processor:3.1.1'
}
```

5.  EventBus.register
```
    public void register(Object subscriber) {
        //1. 获取订阅者的Class
        Class<?> subscriberClass = subscriber.getClass();
        //2.根据Class查找其中的所有订阅方法（有Subscibe注解的方法）
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);


        //这里同步代码块是保证线程安全，避免重复订阅
        synchronized (this) {
            //3. 订阅这些方法
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }

```

6. SubscriberMethodFinder.findSubscriberMethods
```
    private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();

    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {

        //1. 首先从缓存中查找，找到了就直接返回
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

        //2. 是否忽略注解生成的索引，默认为false
        if (ignoreGeneratedIndex) {
            //忽略的情况直接通过反射查找
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            //否则通过编译时生成的索引查找
            subscriberMethods = findUsingInfo(subscriberClass);
        }

        if (subscriberMethods.isEmpty()) {
            //如果当前订阅者及其父类内部没有符合要求的订阅方法，则抛出异常
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {

            //3. 将当前订阅者，以及其内部的订阅方法以key-value方式存入缓存
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }

```

7.  SubscriberMethodFinder.findUsingInfo
```
    //我们先分析findUsingInfo方法，它内部实现首先通过编译时生成的索引查找，如果没找到，则通过反射查找
    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {

        //1. 准备一个查找状态对象，用于保存目标订阅者的Class，以及查找到的该类对应的订阅方法信息
        //查找对象使用了复用池
        FindState findState = prepareFindState();

        //2. 初始化查找状态对象的订阅者
        findState.initForSubscriber(subscriberClass);

        //3. 这里会循环向上也就是父类查找对应的订阅方法
        while (findState.clazz != null) {

            //4. 获取对应的sunscriberInfo，内部通过从编译时生成的索引中查找
            findState.subscriberInfo = getSubscriberInfo(findState);

            //5.如果从编译是生成的索引中查找到了
            if (findState.subscriberInfo != null) {

                //从subscriberInfo中取出多少有订阅方法，添加到findState的subcriberMethods集合中
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                //5.当没有查找到时，就通过反射查找
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }

        //6. 返回订阅方法信息，并且将findState送回复用池
        return getMethodsAndRelease(findState);
    }
```

8. SubscriberMethodFinder.getSubscriberInfo
```
    private SubscriberInfo getSubscriberInfo(FindState findState) {

        //1. 首先判断当前查找状态中subscriberInfo和subscriberInfo的父级是否为null
        //暂时不太清楚是作为什么用的。做了测试，发现superSubscriberInfo始终为null
        //这里的sunscriberInfo实例是SimpleSubscriberInfo实例

        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            if (findState.clazz == superclassInfo.getSubscriberClass()) {
                return superclassInfo;
            }
        }

        //2.上面没找到，则看sunscriberInfoInexes是否为null，默认没配置的时候为null
        if (subscriberInfoIndexes != null) {
            for (SubscriberInfoIndex index : subscriberInfoIndexes) {

                //3.查询index中对应的订阅者类所对应的订阅信息
                SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
                if (info != null) {
                    return info;
                }
            }
        }
        return null;
    }
```

9.  MyEventBusIndex,编译时自动生成的类，也就是上面index
```
/** This class is generated by EventBus, do not edit. */
public class MyEventBusIndex implements SubscriberInfoIndex {
    private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;


    //静态代码块，当类被加载时，执行
    static {

        //初始化订阅者，订阅信息缓存 HashMap
        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();

        //生成订阅信息，存入map中
        //可以看到生成的订阅信息都是SimpleSubcriberInfo，传入的参数是Class，true，以及当前Class中的订阅方法信息
        putIndex(new SimpleSubscriberInfo(com.example.learn.eventbus.EventBusTestFather.class, true,
                new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("helloe", com.example.learn.eventbus.EventObject.class, ThreadMode.MAIN_ORDERED),
        }));

        putIndex(new SimpleSubscriberInfo(com.example.learn.eventbus.EventBusTest.class, true,
                new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onMessage", com.example.learn.eventbus.EventObject.class, ThreadMode.MAIN),
        }));

    }

    private static void putIndex(SubscriberInfo info) {
        SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);
    }

    @Override
    public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {

        //1. 根据订阅者Class从我们编译时生成的订阅关系索引中查找
        SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);
        if (info != null) {
            return info;
        } else {
            return null;
        }
    }
}
```

10. 来看直接通过反射查找
```
    private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {

        //findState和上面一直
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {

            //1. 在当前类中查找
            findUsingReflectionInSingleClass(findState);
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }

    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            //getDeclaredMethods方法返回当前类的public，private，protect，default所有方法，不包过继承父类的方法
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            //getMethods方法返回public方法，但是包含父类，父类的父类的public方法
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }


        for (Method method : methods) {
            int modifiers = method.getModifiers();

            //验证方法合法性，订阅方法必须是public的，并且不能是静态的，也不能是抽象的，只能有一个参数
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];

                        //检查同一个订阅者不能用多个不同的订阅方法订阅同一个事件
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();

                            //将找到的订阅方法添加到findState的订阅方法集合中
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```

11. 在回到第5步中，的第3小步，当找到订阅者的所有订阅方法后，进行注册
```
//EventBus.subscribe

    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;

        //根据订阅者，与订阅方法生成一个Subscribtion，代表一个订阅
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);

        //subscriptionsByEventType是一个HashMap，key中事件的Class，value是一个CopyOnWriteArrayList,该
        //list中保存的是对应的所有订阅
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);


        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }

        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                //根据优先级，添加到订阅列表中
                subscriptions.add(i, newSubscription);
                break;
            }
        }

        //typesBySubscriber也是一个HashMap，用于存的是一个订阅者，订阅了哪些事件
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        //将订阅的事件eventType的Class存入subscribedEvents集合中
        subscribedEvents.add(eventType);


        //如果是粘性方法，就会发送粘性事件给它
        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }

```

12. 注册流程看完了，再来看取消注册
```
    public synchronized void unregister(Object subscriber) {
        //1.首先获取这个订阅者订阅了那些事件
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
        if (subscribedTypes != null) {
            //2. 其次，从事件为key的map中移除当前订阅者对应的订阅
            for (Class<?> eventType : subscribedTypes) {
                unsubscribeByEventType(subscriber, eventType);
            }
            //3. 最后再从订阅者订阅了那些事件的map中移除当前订阅者
            typesBySubscriber.remove(subscriber);
        } else {
            logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }
```

13. 发送消息
```
    final static class PostingThreadState {
        final List<Object> eventQueue = new ArrayList<>();
        boolean isPosting;
        boolean isMainThread;
        Subscription subscription;
        Object event;
        boolean canceled;
    }


public void post(Object event) {

        //PostingThreadState表示一个当前发送线程的状态，currentPostingTrheadState是TheadLocal的
        PostingThreadState postingState = currentPostingThreadState.get();

        //首先将当前发送的事件添加到当前线程的事件队列中
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);

        //当前线程如果不在执行post，则立刻进入post，否则不做处理处理，当前面的post完成之后，会继续执行post
        if (!postingState.isPosting) {

            //确认当前线程是否是主线程
            postingState.isMainThread = isMainThread();
            //标识当前正在投递状态
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                //当投递队列不为空时，则循环投递
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                //当队列中所有消息投递完成，重新设置投递状态为false
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
```

14. EventBus.postSingleEvent
```
 private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;

        //1.事件是否可继承，对与这种情况，也要查看事件的子类的订阅，默认true
        if (eventInheritance) {
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {

            //进行投递事件
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }

        //如果没有找到事件对应的订阅，进行额外的处理
        if (!subscriptionFound) {

            //打印日志，默认情况是ture
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }

            //如果配置了发送NoSubcriberEvent，就发送一个NoSubscriberEvent事件，默认情况下是true
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }

     private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
            CopyOnWriteArrayList<Subscription> subscriptions;

            //1. 获取当前事件对应的所有订阅
            synchronized (this) {
                subscriptions = subscriptionsByEventType.get(eventClass);
            }

            //2. 遍历订阅集合，对每个订阅发送消息
            if (subscriptions != null && !subscriptions.isEmpty()) {
                for (Subscription subscription : subscriptions) {

                    //将事件赋值给，订阅赋值给postingStat
                    postingState.event = event;
                    postingState.subscription = subscription;
                    boolean aborted = false;
                    try {
                        //进行最后的投递
                        postToSubscription(subscription, event, postingState.isMainThread);
                        aborted = postingState.canceled;
                    } finally {
                        //投递完成后，复位postState
                        postingState.event = null;
                        postingState.subscription = null;
                        postingState.canceled = false;
                    }
                    if (aborted) {
                        break;
                    }
                }
                return true;
            }
            return false;
        }



        private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {

            //根据订阅方法所指定的线程信息，选择如果投递
             switch (subscription.subscriberMethod.threadMode) {
                 case POSTING:
                     //invokeSubscriber通过反射执行方法
                     invokeSubscriber(subscription, event);
                     break;
                 case MAIN:
                     if (isMainThread) {//当前在主线程，直接执行
                         invokeSubscriber(subscription, event);
                     } else {
                        //当前不在主线线，加入mainThreadPoster的队列
                         mainThreadPoster.enqueue(subscription, event);
                     }
                     break;
                 case MAIN_ORDERED:
                    //无论当前是在什么线程，都加入mainThreadPoster的队列
                     if (mainThreadPoster != null) {
                         mainThreadPoster.enqueue(subscription, event);
                     } else {
                         // temporary: technically not correct as poster not decoupled from subscriber
                         invokeSubscriber(subscription, event);
                     }
                     break;
                 case BACKGROUND:
                     if (isMainThread) {
                         //如果是在主线程，加入backgroundPoster队列
                         backgroundPoster.enqueue(subscription, event);
                     } else {
                        //如果当前不在主线程，直接执行
                         invokeSubscriber(subscription, event);
                     }
                     break;
                 case ASYNC:
                      //无论当前线程， 都加入到asyncPoster的队列中
                     asyncPoster.enqueue(subscription, event);
                     break;
                 default:
                     throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
             }
         }
```

15. mainThreadPoster： 其实是一个HandlerPoster
```
    protected HandlerPoster(EventBus eventBus, Looper looper, int maxMillisInsideHandleMessage) {
         super(looper);
         this.eventBus = eventBus;
         this.maxMillisInsideHandleMessage = maxMillisInsideHandleMessage;
         //PengidngPostQueue是一个链表结构，里面含有head和tail连个节点
         queue = new PendingPostQueue();
     }

 public void enqueue(Subscription subscription, Object event) {
        //1. 首先获取一个PendingPost，表示一个等待投递事件
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            //2. 进入队列，
            queue.enqueue(pendingPost);
            //3.如果当前没有在执行，就通过发消息，切换到主线程执行
            if (!handlerActive) {
                handlerActive = true;
                if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }

    @Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            //循环从queue中取出要执行的消息
            while (true) {
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        // Check again, this time in synchronized
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }

                //如果不为空，就通过反射执行方法
                eventBus.invokeSubscriber(pendingPost);

                //这里为了不让主线程执行消息的时间太长，当超过10ms时，就又重新发消息等待下次执行。
                long timeInMethod = SystemClock.uptimeMillis() - started;
                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
```

16. backgroundPoster： BackgroundPoster
```
 BackgroundPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            //添加到队列
            queue.enqueue(pendingPost);
            //如果当前正在执行，就不做任何处理，当前一次执行完了，会自动自行后续的，所以指定ThereadMode.BACKGROUND
            //是会一个一个执行，不会同时并发执行消息
            if (!executorRunning) {
                executorRunning = true;
                //如果当前没有执行，才通过线程池进行处理
                eventBus.getExecutorService().execute(this);
            }
        }
    }

    @Override
    public void run() {
        try {
            try {
                //循环取消息
                while (true) {
                    //在这方法内部，如果没有消息，就wait阻塞线程，但是这里设置了1000ms的超时
                    PendingPost pendingPost = queue.poll(1000);

                    if (pendingPost == null) {
                        synchronized (this) {
                            // Check again, this time in synchronized
                            pendingPost = queue.poll();
                            if (pendingPost == null) {
                                executorRunning = false;
                                return;
                            }
                        }
                    }

                    //通过放射执行相应的订阅方法
                    eventBus.invokeSubscriber(pendingPost);
                }
            } catch (InterruptedException e) {
                eventBus.getLogger().log(Level.WARNING, Thread.currentThread().getName() + " was interruppted", e);
            }
        } finally {
            //当超时了，还没有消息时，则会当前执行线程会停掉，将executorRunning设置为false
            executorRunning = false;
        }
    }
```

17. asyncPoster ： AsyncPoster
```
  AsyncPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        //AsncPoster处理很简单，就是直接放进queue中，然后通过线程池执行
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        queue.enqueue(pendingPost);
        eventBus.getExecutorService().execute(this);
    }

    @Override
    public void run() {
        //同队列中取出下一个消息
        PendingPost pendingPost = queue.poll();
        if(pendingPost == null) {
            throw new IllegalStateException("No pending post available");
        }

        //通过反射执行订阅方法
        eventBus.invokeSubscriber(pendingPost);
    }

```