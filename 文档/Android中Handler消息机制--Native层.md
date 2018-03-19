#   Android中消息机制--Native层

消息机制中Java层和Native层打交道的类是MessageQueue.

1.  MessageQueue中的native方法:
```
    private native static long nativeInit();
    private native static void nativeDestroy(long ptr);
    private native void nativePollOnce(long ptr, int timeoutMillis); /*non-static for callbacks*/
    private native static void nativeWake(long ptr);
    private native static boolean nativeIsPolling(long ptr);
    private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);

    对于主要流程来说.主要就是前面4个方法.
```
2.  MessageQueue的nativeInit方法,该方法在MessageQueue的构造方法中被调用.
```
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();//mPtr代表native层的MessageQueue.
    }
```

3.  android_os_MessageQueue.cpp中: jlong android_os_MessageQueue_nativeInit方法
```
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();//创建Native层的MessageQueue
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }

    nativeMessageQueue->incStrong(env);//强引用加1.将native成的MessageQueue返回给Java层
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```
4.  NativeMessageQueue的初始化:在android_os_MessageQueue.cpp中
```
    NativeMessageQueue::NativeMessageQueue() :
            mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
        mLooper = Looper::getForThread();//获取native层当前线程的Looper对象.
        if (mLooper == NULL) {//如果还没初始化过,则创建一个looper对象,存入当前线程本地变量.
            mLooper = new Looper(false);
            Looper::setForThread(mLooper);
        }
    }
```
5.  Native层的Looper初始化:
```
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mPolling(false), mEpollFd(-1), mEpollRebuildRequired(false),
        mNextRequestSeq(0), mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {

    mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);//够着唤醒事件的文件描述符
    LOG_ALWAYS_FATAL_IF(mWakeEventFd < 0, "Could not make wake event fd: %s",
                        strerror(errno));

    AutoMutex _l(mLock);
    rebuildEpollLocked();
}
```
6.  Looper的rebuildEpollLocked方法:
```
void Looper::rebuildEpollLocked() {
    // Close old epoll instance if we have one.
    if (mEpollFd >= 0) {
#if DEBUG_CALLBACKS
        ALOGD("%p ~ rebuildEpollLocked - rebuilding epoll set", this);
#endif
        close(mEpollFd);//关闭之前的epoll实例
    }

    // Allocate the new epoll instance and register the wake pipe.
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);//EPOLL_SIZE_HINT 为8.表示初始创建的时候最大允许8个文件描述符和当前epoll实例关联.

    LOG_ALWAYS_FATAL_IF(mEpollFd < 0, "Could not create epoll instance: %s", strerror(errno));

    struct epoll_event eventItem;//监听的事件
    memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union

    eventItem.events = EPOLLIN;//监听事件的类型为可读
    eventItem.data.fd = mWakeEventFd;//可读数据的文件描述符为唤醒事件的文件描述符:mWakeEventFd

    //将需要监听的mWakeEventFd描述符添加到epoll句柄.(其实就是事件注册)
    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake event fd to epoll instance: %s",
                        strerror(errno));

    //监听mRequests集合中的其他希望被监听文件描述符以及事件.Looper的addFd方法添加的都会添加到mRequests中.
    //列如:InputEventReceiver.DisplayEventReceiver.对不同的文件描述符进行监听
    for (size_t i = 0; i < mRequests.size(); i++) {
        const Request& request = mRequests.valueAt(i);
        struct epoll_event eventItem;
        request.initEventItem(&eventItem);

        int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, request.fd, & eventItem);
        if (epollResult < 0) {
            ALOGE("Error adding epoll events for fd %d while rebuilding epoll set: %s",
                  request.fd, strerror(errno));
        }
    }
}

```

7.  MessageQueue的nativeDestroy:该方法在MessageQueue的dispose方法中会被调用.
```
    //用来减少强引用.
    static void android_os_MessageQueue_nativeDestroy(JNIEnv* env, jclass clazz, jlong ptr) {
        NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
        nativeMessageQueue->decStrong(env);
    }
```
8.  MessageQueue的nativePollOnce方法:在android_os_MessageQueue.cpp中实现
```
    static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
            jlong ptr, jint timeoutMillis) {
        //这里ptr就是Java层初始化MessageQueue时,执行nativeInit返回的Native层的NativeMessageQueue.所以这里强转回去.
        NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
        //执行NativeMessageQueue的pollOnce方法
        nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
    }
```
9.  NativeMessageQueue的pollOnce方法:
```
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    mLooper->pollOnce(timeoutMillis);//最终调用的Native层Looper的pollOnce方法.
    mPollObj = NULL;
    mPollEnv = NULL;

    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
```
10. Looper的pollOnce方法.在Looper.h中
```
    inline int pollOnce(int timeoutMillis) {
        return pollOnce(timeoutMillis, NULL, NULL, NULL);
    }
```
11. Looper.cpp中的pollOnce4个参数方法:
```
    int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
        int result = 0;
        for (;;) {
            while (mResponseIndex < mResponses.size()) {
                const Response& response = mResponses.itemAt(mResponseIndex++);
                int ident = response.request.ident;
                if (ident >= 0) {
                    int fd = response.request.fd;
                    int events = response.events;
                    void* data = response.request.data;
                    if (outFd != NULL) *outFd = fd;
                    if (outEvents != NULL) *outEvents = events;
                    if (outData != NULL) *outData = data;
                    return ident;
                }
            }

            if (result != 0) {
                if (outFd != NULL) *outFd = 0;
                if (outEvents != NULL) *outEvents = 0;
                if (outData != NULL) *outData = NULL;
                return result;
            }

            //这里在内部轮询
            result = pollInner(timeoutMillis);
        }
    }
```
12. Looper.cpp中的pollInner方法:
```
    int Looper::pollInner(int timeoutMillis) {

        // Adjust the timeout based on when the next message is due.
        //修正超时时长,用超时时间和下一条本地消息执行的时间对比.
        if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
            nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
            int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
            if (messageTimeoutMillis >= 0
                    && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
                timeoutMillis = messageTimeoutMillis;
            }
        }

        // Poll.
        int result = POLL_WAKE;
        //清空事件返回结果集合
        mResponses.clear();
        mResponseIndex = 0;

        // We are about to idle.
        mPolling = true;

        struct epoll_event eventItems[EPOLL_MAX_EVENTS];//EPOLL_MAX_EVENTS为16

        //epoll_wait阻塞当前线程,该函数返回需要处理的事件数目.eventItems存储内核返回的事件,EPOLL_MAX_EVENTS告诉内核,我们给的存储事件的集合多大.
        //timeoutMillis超时时长.当有所监听的文件描述符上事件产生时,该方法会返回.
        int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

        // No longer idling.
        mPolling = false;

        // Acquire lock.
        mLock.lock();

        // Rebuild epoll set if needed.
        if (mEpollRebuildRequired) {
            mEpollRebuildRequired = false;
            rebuildEpollLocked();
            goto Done;
        }

        // Check for poll error.
        if (eventCount < 0) {   //epoll事件个数小于0,发生错误,直接跳转到Done处.
            if (errno == EINTR) {
                goto Done;
            }
            ALOGW("Poll failed with an unexpected error: %s", strerror(errno));
            result = POLL_ERROR;
            goto Done;
        }

        // Check for poll timeout.
        if (eventCount == 0) {//epoll事件个数为0,超时.直接跳转Done处
            result = POLL_TIMEOUT;
            goto Done;
        }

        // Handle all events.
        //循环处理所有返回的事件.
        for (int i = 0; i < eventCount; i++) {
            int fd = eventItems[i].data.fd;
            uint32_t epollEvents = eventItems[i].events;
            if (fd == mWakeEventFd) {
                if (epollEvents & EPOLLIN) {//当有唤醒事件文件描述符以及对应的可读事件时,执行awoken.读取文件描述符所代表文件的数据,其实就是清空数据.
                    awoken();
                } else {
                    ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
                }
            } else {
                ssize_t requestIndex = mRequests.indexOfKey(fd);
                if (requestIndex >= 0) {
                    int events = 0;
                    if (epollEvents & EPOLLIN) events |= EVENT_INPUT;//可读
                    if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;//可写
                    if (epollEvents & EPOLLERR) events |= EVENT_ERROR;//错误
                    if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;//挂起
                    pushResponse(events, mRequests.valueAt(requestIndex));//将事件结果构造成Response存入mResponses集合中.
                } else {
                    ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
                            "no longer registered.", epollEvents, fd);
                }
            }
        }
    Done: ;
        //处理Natice层的Message
        // Invoke pending message callbacks.
        mNextMessageUptime = LLONG_MAX;
        while (mMessageEnvelopes.size() != 0) {
            nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
            const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
            if (messageEnvelope.uptime <= now) {
                // Remove the envelope from the list.
                // We keep a strong reference to the handler until the call to handleMessage
                // finishes.  Then we drop it so that the handler can be deleted *before*
                // we reacquire our lock.
                { // obtain handler
                    sp<MessageHandler> handler = messageEnvelope.handler;
                    Message message = messageEnvelope.message;
                    mMessageEnvelopes.removeAt(0);
                    mSendingMessage = true;
                    mLock.unlock();
                    handler->handleMessage(message);
                } // release handler

                mLock.lock();
                mSendingMessage = false;
                result = POLL_CALLBACK;
            } else {
                // The last message left at the head of the queue determines the next wakeup time.
                mNextMessageUptime = messageEnvelope.uptime;
                break;
            }
        }

        // Release lock.
        mLock.unlock();


        //处理所有epoll返回的事件结果集合
        // Invoke all response callbacks.
        for (size_t i = 0; i < mResponses.size(); i++) {
            Response& response = mResponses.editItemAt(i);

            //如果事件的ident是POLL_CALLBACK类型,则执行callback的handEvent方法.列如:InputEventReceiver和DisplayEventReceiver注册的一样.
            if (response.request.ident == POLL_CALLBACK) {
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;

                // Invoke the callback.  Note that the file descriptor may be closed by
                // the callback (and potentially even reused) before the function returns so
                // we need to be a little careful when removing the file descriptor afterwards.
                int callbackResult = response.request.callback->handleEvent(fd, events, data);

                //根据返回结果,判断是否还需要继续监听.
                if (callbackResult == 0) {
                    removeFd(fd, response.request.seq);
                }

                // Clear the callback reference in the response structure promptly because we
                // will not clear the response vector itself until the next poll.
                response.request.callback.clear();
                result = POLL_CALLBACK;
            }
        }
        return result;
    }
```
13. MessageQueue的nativeWake方法:在android_os_MessageQueue.cpp中.
```
    static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
        NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
        //执行的是NativeMessageQueue的wake方法
        nativeMessageQueue->wake();
    }
```
14. NativeMessageQueue的wake方法:
```
    void NativeMessageQueue::wake() {
        //执行的是Looper的wake方法
        mLooper->wake();
    }
```
15. Native层Looper的wake方法:
```
void Looper::wake() {
    uint64_t inc = 1;
    //手动写入内容.让epoll监听到该唤醒文件描述的可读状态.则epoll_wait方法处就会返回结果.停止阻塞.
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
    if (nWrite != sizeof(uint64_t)) {
        if (errno != EAGAIN) {
            ALOGW("Could not write wake signal: %s", strerror(errno));
        }
    }
}

```

```
总结:其实对于Native层主要就是3块逻辑.nativeInit. nativePollOnce. nativeWake

nativeInit流程:
->  Java 层MessageQueue的构造方法:
->  MessageQueue.nativeInit
->  android_os_MessageQueue.cpp的android_os_MessageQueue_nativeInit
->  创建NativeMessageQueue
->  创建Native层的Looper
->  Looper构造函数中创建mWakeEventFd唤醒文件描述符,并执行rebuildEpollLocked.
->  执行epoll_create创建epoll实例
->  执行epoll_ctl,将mWakeEventFd文件描述符注册监听EPOLLIN(可读)事件.

nativePollOnce流程:
->  Java层MessageQueue的nativePollOnce
->  android_os_MessageQueue.cpp中的android_os_MessageQueue_nativePollOnce
->  NativeMessageQueue.pollOnce
->  Native层Looper.pollOnce
->  Native层Looper.pollInner
->  epoll_wait 可能阻塞,等待监听文件描述符事件 的回调.当有可执行事件的,立即返回,返回值为处理事件的数目.
    阻塞的超时时间由外面调用nativePollOnce时传入的超时时间和native层loop发送的消息执行时间共同决定.
->  有mWakeEventFd的EPOLLIN事件.执行awoken.清空mWakeEventFd文件描述符所代表文件中的内容.
->  将其他文件描述符事件封装成Respouse装到mResponse集合中.
->  处理Native层的消息.
->  处理mResponse中的事件集合.

nativeWake流程:
->  Java层MessageQueue的nativeWake
->  android_os_MessageQueue.cpp 的android_os_MessageQueue_nativeWake
->  NativeMessageQueue.wake
->  Native层Looper.wake
->  向mWakeEventFd文件描述符表示的文件中写入内容.
```
