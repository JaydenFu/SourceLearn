#   BroadcastReceiver的注册过程

这里分析的是BroadcastReceiver的动态注册过程

1.  ContextWrapper.registerReceiver
```
 @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        //通过ContextImpl完成广播的注册
        return mBase.registerReceiver(receiver, filter);
    }
```

2.  ContextImpl.registerReceiver
```
    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        return registerReceiver(receiver, filter, null, null);
    }

    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
            String broadcastPermission, Handler scheduler) {

        //通过registerReceiverInternal继续注册
        //这里getOutContext返回的是组件，如果是Activity调用注册，则返回的是Activity
        //如果是Service调用注册，则返回的是Service
        //如果是Application调用注册，则返回的是Application
        return registerReceiverInternal(receiver, getUserId(),
                filter, broadcastPermission, scheduler, getOuterContext());
    }
```

3.  ContextImpl.registerReceiverInternal
```
  private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            //正常情况下mPackageInfo和context都不会为null
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {

                    //scheduler是ActivityThread的mh handler
                    scheduler = mMainThread.getHandler();
                }

                //获取用于进程间通信的IIntentReeiver
                rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {

            //通过ActivityManangerProxy注册广播接受者
            final Intent intent = ActivityManagerNative.getDefault().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName,
                    rd, filter, broadcastPermission, userId);
            if (intent != null) {
                intent.setExtrasClassLoader(getClassLoader());
                intent.prepareToEnterProcess();
            }
            return intent;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```

4.  LoadedApk.getReceiverDispatcher
```
public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
            Context context, Handler handler,
            Instrumentation instrumentation, boolean registered) {
        synchronized (mReceivers) {
            LoadedApk.ReceiverDispatcher rd = null;
            ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
            if (registered) {
                //mReceviers是一个ArrayMap<Context,ArrayMap<BoradcastReceiver,LoadedApk.ReceiverDispatcher>>
                //也就是说对于不同的context，BroadcastReceiver被存在不同的map中。
                map = mReceivers.get(context);
                if (map != null) {
                    //从context对应的map中取出BroadcastReceiver对应的ReceiverDispatcher
                    rd = map.get(r);
                }
            }
            if (rd == null) {
                //不存在，则创建新的
                rd = new ReceiverDispatcher(r, context, handler,
                        instrumentation, registered);
                if (registered) {
                    if (map == null) {
                        map = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
                        mReceivers.put(context, map);
                    }
                    //存进相应的map中
                    map.put(r, rd);
                }
            } else {
                rd.validate(context, handler);
            }
            rd.mForgotten = false;

            //返回IIntentReceiver
            return rd.getIIntentReceiver();
        }
    }
```

5.  LoadedApk.ReceiverDispatcher的getIIntentReceiver
```
    //在ReceiverDispatcher构造方法中初始化mIIntentReceiver
    //InnterReceiver 继承自 IIntentService.Stub,可以猜出是aidl，可以进程间通信
    mIIntentReceiver = new InnerReceiver(this, !registered);

    IIntentReceiver getIIntentReceiver() {
        return mIIntentReceiver;
    }
```

6.  ActivityManagerService.registerReceiver
```
public Intent registerReceiver(IApplicationThread caller, String callerPackage,
            IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {

        ArrayList<Intent> stickyIntents = null;
        ProcessRecord callerApp = null;
        int callingUid;
        int callingPid;

        synchronized(this) {
            if (caller != null) {
                callerApp = getRecordForAppLocked(caller);
                ....
                callingUid = callerApp.info.uid;
                callingPid = callerApp.pid;
            } else {
                callerPackage = null;
                callingUid = Binder.getCallingUid();
                callingPid = Binder.getCallingPid();
            }

            userId = mUserController.handleIncomingUser(callingPid, callingUid, userId, true,
                    ALLOW_FULL_ONLY, "registerReceiver", callerPackage);

            //1. 解析广播接收者可以接收的所有action
            Iterator<String> actions = filter.actionsIterator();
                ...

            // Collect stickies of users
            int[] userIds = { UserHandle.USER_ALL, UserHandle.getUserId(callingUid) };
            while (actions.hasNext()) {
                String action = actions.next();
                for (int id : userIds) {
                    //取出已保存的所有粘性广播
                    ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(id);
                    if (stickies != null) {
                        //粘性广播中action可以匹配的广播
                        ArrayList<Intent> intents = stickies.get(action);
                        if (intents != null) {
                            if (stickyIntents == null) {
                                stickyIntents = new ArrayList<Intent>();
                            }
                            //如果存在这样的粘性广播，则添加进stickyIntents集合中
                            stickyIntents.addAll(intents);
                        }
                    }
                }
            }
        }


        ArrayList<Intent> allSticky = null;
        if (stickyIntents != null) {
            final ContentResolver resolver = mContext.getContentResolver();
            for (int i = 0, N = stickyIntents.size(); i < N; i++) {
                Intent intent = stickyIntents.get(i);
                //判断intent是否和IntentFilter匹配，action，schema等
                if (filter.match(resolver, intent, true, TAG) >= 0) {
                    if (allSticky == null) {
                        allSticky = new ArrayList<Intent>();
                    }
                    //匹配则添加到allSticky集合中
                    allSticky.add(intent);
                }
            }
        }

        Intent sticky = allSticky != null ? allSticky.get(0) : null;
        if (receiver == null) {
            return sticky;
        }

        synchronized (this) {
            ...
            //ReceiverList继承自ArrayList
            //mRegisteredReceivers是一个HashMap集合，保存所有注册的广播接受者
            ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
            if (rl == null) {

                //App进程的每个IIntentReceiver，在AMS中都对应各自的一个ReceiverList
                rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                        userId, receiver);

                if (rl.app != null) {
                    //将ReceiverList添加进ProcessRecord的receivers集合中
                    rl.app.receivers.add(rl);
                } else {
                    ...
                }

                //将ReceiverList存进AMS的mRegisteredReceivers中
                mRegisteredReceivers.put(receiver.asBinder(), rl);
            } else if (rl.uid != callingUid) {
                ...
            } else if (rl.pid != callingPid) {
                ...
            } else if (rl.userId != userId) {
                ...
            }

            //创建广播接收者的广播接收过滤器
            BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                    permission, callingUid, userId);

            //将bf添加进rl，rl可以接受多个bf过滤的广播，场景：同一个广播实例注册多次，但是注册时候的IntentFilter不一样
            rl.add(bf);

            //将BroadcastFilter添加进mReceiverResolver中，以后AMS接受到广播，就可以通过
            //mReceiverResolver找到对应的广播接收者了
            mReceiverResolver.addFilter(bf);

            //发送粘性广播
            if (allSticky != null) {
                ArrayList receivers = new ArrayList();
                receivers.add(bf);

                final int stickyCount = allSticky.size();
                for (int i = 0; i < stickyCount; i++) {
                    Intent intent = allSticky.get(i);
                    BroadcastQueue queue = broadcastQueueForIntent(intent);
                    BroadcastRecord r = new BroadcastRecord(queue, intent, null,
                            null, -1, -1, null, null, AppOpsManager.OP_NONE, null, receivers,
                            null, 0, null, null, false, true, true, -1);
                    queue.enqueueParallelBroadcastLocked(r);
                    queue.scheduleBroadcastsLocked();
                }
            }

            return sticky;
        }
    }
```