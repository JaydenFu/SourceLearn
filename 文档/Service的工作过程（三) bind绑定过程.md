#   Service的bind过程

1.  ContextImpl.bindService
```
 @Override
    public boolean bindService(Intent service, ServiceConnection conn,
            int flags) {
        warnIfCallingFromSystemProcess();

        //这里mMainThread.getHandler（）获取的ActivityThread中的mH handler.
        return bindServiceCommon(service, conn, flags, mMainThread.getHandler(),
                Process.myUserHandle());
    }

```
2.  ContextImpl.bindServiceCommon
```
 private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler
            handler, UserHandle user) {
        IServiceConnection sd;
        if (conn == null) {
            throw new IllegalArgumentException("connection is null");
        }
        if (mPackageInfo != null) {
            //1. 获取和当前ServiceConnection以及调用bindService的context相关的IserviceConnection
            //当bind执行完成后，系统进程通过进程间通信与该sd通信，是ServiceConnection执行onServiceConnected.

            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
        } else {
            throw new RuntimeException("Not supported in system context");
        }

        //2.检验service的合法性,对于5.0以上的系统，必须是显示启动service
        validateServiceIntent(service);
        try {
            //3.获取当前的activityToken。如果当前ContextImpl属于activity，则token不为null，否则token为null.
            //AMS通过该token可以找到对应的ActivityRecord.并将ConnectionRecord加入ActivityRecord的connections集合中，
            //当activity销毁时，AMS同样会unbind掉通过activity执行bind的service。
            IBinder token = getActivityToken();
            ...

            //4.进程间通信，执行AMS的bindService方法
            int res = ActivityManagerNative.getDefault().bindService(
                mMainThread.getApplicationThread(), getActivityToken(), service,
                service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, getOpPackageName(), user.getIdentifier());
            ...
            return res != 0;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```

3.  LoadedApk.getServiceDispatcher
```
    public final IServiceConnection getServiceDispatcher(ServiceConnection c,
            Context context, Handler handler, int flags) {
        synchronized (mServices) {
            LoadedApk.ServiceDispatcher sd = null;

            //mService是一个ArrayMap<Context,ArrayMap<ServiceConnection,LoadedApk.ServiceDispatcher>>
            //的集合，管理着所有context对应的serviceConnection
            ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = mServices.get(context);


            if (map != null) {
                //1.获取ServiceConnection对应的ServiceDispatcher
                sd = map.get(c);
            }
            if (sd == null) {
                //2. 如果还没有当前ServiceConnection对应的就创建ServiceDispatcher
                sd = new ServiceDispatcher(c, context, handler, flags);
                if (map == null) {
                    map = new ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>();
                    mServices.put(context, map);
                }
                //3.添加到map中
                map.put(c, sd);
            } else {
                sd.validate(context, handler);
            }
            return sd.getIServiceConnection();
        }
    }
```

4.  LoadedApk.ServiceDispatcher的getIServiceConnection()
```
    //在ServiceDispatcher初始化时，会创建mIServiceConnection。
    mIServiceConnection = new InnerConnection(this);

    IServiceConnection getIServiceConnection() {
        return mIServiceConnection;
    }


    //可以看出InnterConnection应该是继承Binder的，用于进程间通信。
    private static class InnerConnection extends IServiceConnection.Stub {
        final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

        InnerConnection(LoadedApk.ServiceDispatcher sd) {
            mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
        }

        public void connected(ComponentName name, IBinder service) throws RemoteException {
            LoadedApk.ServiceDispatcher sd = mDispatcher.get();
            if (sd != null) {
                sd.connected(name, service);
            }
        }
    }
```

5.  ActivityManagerService.bindService方法
```
    public int bindService(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, IServiceConnection connection, int flags, String callingPackage,
            int userId) throws TransactionTooLargeException {

        ....
        synchronized(this) {
            //mServices是ActiveServices,在AMS的构造方法中被初始化
            return mServices.bindServiceLocked(caller, token, service,
                    resolvedType, connection, flags, callingPackage, userId);
        }
    }
```

6.  ActiveServices.bindServiceLocked
```
int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, final IServiceConnection connection, int flags,
            String callingPackage, final int userId) throws TransactionTooLargeException {
        final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
        ...

        ActivityRecord activity = null;
        if (token != null) {
            activity = ActivityRecord.isInStackLocked(token);
            if (activity == null) {
                return 0;
            }
        }

        int clientLabel = 0;
        PendingIntent clientIntent = null;
        final boolean isCallerSystem = callerApp.info.uid == Process.SYSTEM_UID;

        ...

        final boolean callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
        final boolean isBindExternal = (flags & Context.BIND_EXTERNAL_SERVICE) != 0;

        //1. 查找intent对应的ServiceRecord，如果不存在就创建新的
        ServiceLookupResult res =
            retrieveServiceLocked(service, resolvedType, callingPackage, Binder.getCallingPid(),
                    Binder.getCallingUid(), userId, true, callerFg, isBindExternal);


        if (res == null) {
            return 0;
        }
        if (res.record == null) {
            return -1;
        }
        ServiceRecord s = res.record;

        boolean permissionsReviewRequired = false;

        ....

        final long origId = Binder.clearCallingIdentity();

        try {
            ...

            //2. 在ServiceRecord中查找intent对应的AppBindRecord，不存在就创建
            AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
            //3. 创建一个新的ConnectionRecord，表示一个bind连接记录
            ConnectionRecord c = new ConnectionRecord(b, activity,
                    connection, flags, clientLabel, clientIntent);

            //这里这个connection就是InnerConnection
            IBinder binder = connection.asBinder();

            //4.从ServiceRecord的connections中取出binder对应的ConnectionRecord集合
            ArrayList<ConnectionRecord> clist = s.connections.get(binder);
            if (clist == null) {
                clist = new ArrayList<ConnectionRecord>();
                s.connections.put(binder, clist);
            }
            clist.add(c);
            b.connections.add(c);

            //5.如果bind操作是由activity发起，则将ConnectionRecord同样加入到ActiivtyRecord的connections集合中
            //用于Activity销毁时，解绑Service
            if (activity != null) {
                if (activity.connections == null) {
                    activity.connections = new HashSet<ConnectionRecord>();
                }
                activity.connections.add(c);
            }

            b.client.connections.add(c);
            ...

            clist = mServiceConnections.get(binder);

            if (clist == null) {
                clist = new ArrayList<ConnectionRecord>();
                mServiceConnections.put(binder, clist);
            }
            clist.add(c);

            if ((flags&Context.BIND_AUTO_CREATE) != 0) {
                s.lastActivity = SystemClock.uptimeMillis();

                //6. 通过bringUpServiceLocked启动service,并绑定,在内部如果Service绑定过，则不会再绑定
                if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                        permissionsReviewRequired) != null) {
                    return 0;
                }
            }

            ...

            //7.如果已经bind过了，则直接回调.当重复bind时，就是执行这里
            if (s.app != null && b.intent.received) {
                try {
                    c.conn.connected(s.name, b.intent.binder);
                } catch (Exception e) {
                }

                if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                    requestServiceBindingLocked(s, b.intent, callerFg, true);
                }
            } else if (!b.intent.requested) {
                //当还没有bind过，则去bind
                requestServiceBindingLocked(s, b.intent, callerFg, false);
            }

            ...
        } finally {
            Binder.restoreCallingIdentity(origId);
        }

        return 1;
    }
```

7.  ActiveServices.bringUpServiceLocked
```
  private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {

        //当Service被启动了才会执行里面
        if (r.app != null && r.app.thread != null) {
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }

        ...

        final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
        final String procName = r.processName;
        ProcessRecord app;

        if (!isolated) {
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);

            //1.Service所在进程已经启动
            if (app != null && app.thread != null) {
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);

                    //2.启动Service 并绑定service
                    realStartServiceLocked(r, app, execInFg);
                    return null;
                } catch (TransactionTooLargeException e) {
                    throw e;
                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when starting service " + r.shortName, e);
                }

            }
        } else {
            app = r.isolatedProc;
        }

        //3.service所在进程还没启动，则先启动进程
        if (app == null && !permissionsReviewRequired) {
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    "service", r.name, false, isolated, false)) == null) {
                String msg = "Unable to launch app "
                        + r.appInfo.packageName + "/"
                        + r.appInfo.uid + " for service "
                        + r.intent.getIntent() + ": process is bad";
                bringDownServiceLocked(r);
                return msg;
            }
            if (isolated) {
                r.isolatedProc = app;
            }
        }

        //将ServiceRecord添加到mPendingServices中，当进程启动完成，并且AMS的attachApplication执行时，会继续该集合里的service的绑定
        if (!mPendingServices.contains(r)) {
            mPendingServices.add(r);
        }

        ...
        return null;
    }
```

8.  ActiveServices.realStartServiceLocked：
```
private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        if (app.thread == null) {
            throw new RemoteException();
        }
        //1.将进程ProcessRecord赋值给SrviceRecord的app
        r.app = app;
        ...
        boolean created = false;
        try {
            ...
            //2.通过ApplicationThreadProxy与ApplicationThread进程间通信，完成Service的创建于onCreate方法回调
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            created = true;
        } catch (DeadObjectException e) {
            mAm.appDiedLocked(app);
            throw e;
        } finally {
            if (!created) {
            }
        }
        ...
        //2.执行bind操作
        requestServiceBindingsLocked(r, execInFg);

        ...
        sendServiceArgsLocked(r, execInFg, true);
        ...
    }

```

9.  ActiveServices.requestServiceBindingLocked
```
     private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
               boolean execInFg, boolean rebind) throws TransactionTooLargeException {
           if (r.app == null || r.app.thread == null) {
               return false;
           }
           if ((!i.requested || rebind) && i.apps.size() > 0) {

                   //1.进程间通信执行bind操作
                   r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                           r.app.repProcState);
                   if (!rebind) {
                       i.requested = true;
                   }
                   i.hasBound = true;
                   i.doRebind = false;
               } catch (TransactionTooLargeException e) {
                ...
                   throw e;
               } catch (RemoteException e) {
                ...
                   return false;
               }
           }
           return true;
       }
```

10. ActivityThread.handleBindService
```
private void handleBindService(BindServiceData data) {
        Service s = mServices.get(data.token);
        if (s != null) {
            try {
                data.intent.setExtrasClassLoader(s.getClassLoader());
                data.intent.prepareToEnterProcess();
                try {
                    if (!data.rebind) {
                        //1.执行Service的onBind操作，并且回调publishService将binder传递给AMS，
                        //通过AMS将binder传递给ServiceConnection
                        IBinder binder = s.onBind(data.intent);
                        ActivityManagerNative.getDefault().publishService(
                                data.token, data.intent, binder);
                    } else {
                        ...
                    }
                    ensureJitEnabled();
                } catch (RemoteException ex) {
                    throw ex.rethrowFromSystemServer();
                }
            } catch (Exception e) {
                ...
            }
        }
    }
```

11. AMS.publishService
```
    public void publishService(IBinder token, Intent intent, IBinder service) {
        // Refuse possible leaked file descriptors
        if (intent != null && intent.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        synchronized(this) {
            if (!(token instanceof ServiceRecord)) {
                throw new IllegalArgumentException("Invalid service token");
            }

            //1. 还是通过ActiveServices完成binder的回传工作
            mServices.publishServiceLocked((ServiceRecord)token, intent, service);
        }
    }
```

12. ActiveServices.publishServiceLocked
```
    void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
        final long origId = Binder.clearCallingIdentity();
        try {
            if (r != null) {
                Intent.FilterComparison filter
                        = new Intent.FilterComparison(intent);


                IntentBindRecord b = r.bindings.get(filter);
                if (b != null && !b.received) {
                    b.binder = service;
                    b.requested = true;
                    b.received = true;
                    for (int conni=r.connections.size()-1; conni>=0; conni--) {
                        ArrayList<ConnectionRecord> clist = r.connections.valueAt(conni);
                        for (int i=0; i<clist.size(); i++) {
                            ConnectionRecord c = clist.get(i);
                            if (!filter.equals(c.binding.intent.intent)) {
                                continue;
                            }
                            try {
                                //1.与LoadedApk.ServiceDispatcher.InnerConnection进程间通信
                                c.conn.connected(r.name, service);
                            } catch (Exception e) {
                            }
                        }
                    }
                }

                serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```

13. LoadedApk.ServiceDispatcher.InnerConnection.connected
```
 private static class InnerConnection extends IServiceConnection.Stub {
            final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

            InnerConnection(LoadedApk.ServiceDispatcher sd) {
                mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
            }

            public void connected(ComponentName name, IBinder service) throws RemoteException {
                LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                if (sd != null) {
                    //执行ServiceDispatcher的connected操作
                    sd.connected(name, service);
                }
            }
        }

```

14. LoadedApk.ServiceDispatcher.connected
```
    public void connected(ComponentName name, IBinder service) {
        //这里mActivityThread是ActivityThread的mH handler
        if (mActivityThread != null) {
            //所以会执行这里，在主线程中执行RunConnection
            mActivityThread.post(new RunConnection(name, service, 0));
        } else {
            doConnected(name, service);
        }
    }
```

15. RunConnection.run
```
    public void run() {
        if (mCommand == 0) {
            //1.bind操作，执行doConnected
            doConnected(mName, mService);
        } else if (mCommand == 1) {
            //2.解绑操作，执行doDeath
            doDeath(mName, mService);
        }
    }

```

16. LoadedApk.ServiceDispatcher.doConnected
```
public void doConnected(ComponentName name, IBinder service) {
            ServiceDispatcher.ConnectionInfo old;
            ServiceDispatcher.ConnectionInfo info;

            synchronized (this) {
                if (mForgotten) {
                    return;
                }
                old = mActiveConnections.get(name);

                //1.如果已经bind过，并且binder是同一个，则不再回调ServiceConnection的onServiceConnectd方法
                if (old != null && old.binder == service) {
                    return;
                }

                if (service != null) {
                    // A new service is being connected... set it all up.
                    info = new ConnectionInfo();
                    info.binder = service;
                    info.deathMonitor = new DeathMonitor(name, service);
                    try {
                        //2.对binder设置一个死亡代理，当binder死掉的时候，会通知deathMonitor
                        service.linkToDeath(info.deathMonitor, 0);
                        //3.将ConnectonInfo添加到mActiveConnections集合中
                        mActiveConnections.put(name, info);
                    } catch (RemoteException e) {
                        mActiveConnections.remove(name);
                        return;
                    }

                } else {
                    mActiveConnections.remove(name);
                }

                if (old != null) {
                    old.binder.unlinkToDeath(old.deathMonitor, 0);
                }
            }

            // If there was an old service, it is now disconnected.
            if (old != null) {
                mConnection.onServiceDisconnected(name);
            }

            //执行bind成功回调
            if (service != null) {
                mConnection.onServiceConnected(name, service);
            }
        }
```