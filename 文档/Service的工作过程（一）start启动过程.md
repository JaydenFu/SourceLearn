#   Service的工作过程---启动过程

在Service启动过程中涉及几个比较重要的类：

1.  ActivityManagerProxy：负责与系统进程的ActivityManagerService通信
2.  ActivityManagerService： 持有负责管理Service生命周期的ActiveServices
3.  ActiveServices: 管理所有Service.
4.  ApplicationThreadProxy: ActiveService在系统进程通过该类与Service所在进程的ApplicationThread通信。
5.  ActivityThread: 调度Service的创建、启动、停止、绑定、解绑

启动流程：
1.  Context.startService
```
Context的该方法是个抽象方法。其子类ContextWrapper实现。
```
2.  ContextWrapper.startService
```
  @Override
    public ComponentName startService(Intent service) {
        //这里mBase其实是ContextImpl的实例
        return mBase.startService(service);
    }
```
3.  ContextImpl.startService
```
    @Override
    public ComponentName startService(Intent service) {
        warnIfCallingFromSystemProcess();
        return startServiceCommon(service, mUser);
    }
```
4.  ContextImpl.startServiceCommon
```

    private ComponentName startServiceCommon(Intent service, UserHandle user) {
        try {
            validateServiceIntent(service);
            service.prepareToLeaveProcess(this);

            //通过ActivityManagerProxy与AMS通信
            ComponentName cn = ActivityManagerNative.getDefault().startService(
                mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                            getContentResolver()), getOpPackageName(), user.getIdentifier());
            .....
            return cn;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
5.  AMS.startService
```
 @Override
    public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, String callingPackage, int userId)
            throws TransactionTooLargeException {
        .....

        synchronized(this) {
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();

            //调用ActiveServics的startServiceLocked方法。
            ComponentName res = mServices.startServiceLocked(caller, service,
                    resolvedType, callingPid, callingUid, callingPackage, userId);
            Binder.restoreCallingIdentity(origId);
            return res;
        }
    }
```
6.  ActiveServices.startServiceLocked
```
 ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
            int callingPid, int callingUid, String callingPackage, final int userId)
            throws TransactionTooLargeException {

        final boolean callerFg;
        if (caller != null) {
            final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
            if (callerApp == null) {
                throw new SecurityException(
                        "Unable to find app for caller " + caller
                        + " (pid=" + Binder.getCallingPid()
                        + ") when starting service " + service);
            }
            callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
        } else {
            callerFg = true;
        }


        //查找和Intent对应的ServiceRecord
        ServiceLookupResult res =
            retrieveServiceLocked(service, resolvedType, callingPackage,
                    callingPid, callingUid, userId, true, callerFg, false);

        if (res == null) {
            return null;
        }
        if (res.record == null) {
            return new ComponentName("!", res.permission != null
                    ? res.permission : "private to package");
        }

        ServiceRecord r = res.record;

        ....


        NeededUriGrants neededGrants = mAm.checkGrantUriPermissionFromIntentLocked(
                callingUid, r.packageName, service, service.getFlags(), null, r.userId);
        ....

        //设置一些和启动相关的参数
        r.lastActivity = SystemClock.uptimeMillis();
        r.startRequested = true;
        r.delayedStop = false;

        //r.makeNextStartId()会将Service的id每次+1.id最小为1.当stopSerivce时需要id能对上才能正确stop。
        r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                service, neededGrants));

        final ServiceMap smap = getServiceMap(r.userId);

        boolean addToStarting = false;
        ....

        //继续启动service
        return startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
    }
```
7.  ActiveServices.retrieveServiceLocked
```
private ServiceLookupResult retrieveServiceLocked(Intent service,
            String resolvedType, String callingPackage, int callingPid, int callingUid, int userId,
            boolean createIfNeeded, boolean callingFromFg, boolean isBindExternal) {
        ServiceRecord r = null;

        userId = mAm.mUserController.handleIncomingUser(callingPid, callingUid, userId, false,
                ActivityManagerService.ALLOW_NON_FULL_IN_PROFILE, "service", null);

        //在ActiveServices内部，管理着一个mServiceMap的ArrayMap集合，key为callingUserId，value为ServiceMap
        //而在ServiceMap中又管着4个集合:
        //ArrayMap<ComponentName,ServiceRecord>:mServicesByName
        //ArrayMap<Intent.FilterComparison,ServiceRecord>:mServicesByIntent
        //ArrayList<ServiceRecord>:mDelayedStartList
        //ArrayList<ServiceRecord>:mStartingBackground

        ServiceMap smap = getServiceMap(userId);

        //通过Intent中的name信息或则action、data等过滤信息查找对应的service
        final ComponentName comp = service.getComponent();
        if (comp != null) {
            r = smap.mServicesByName.get(comp);
        }
        if (r == null && !isBindExternal) {
            Intent.FilterComparison filter = new Intent.FilterComparison(service);
            r = smap.mServicesByIntent.get(filter);
        }
        if (r != null && (r.serviceInfo.flags & ServiceInfo.FLAG_EXTERNAL_SERVICE) != 0
                && !callingPackage.equals(r.packageName)) {
            r = null;
        }


        if (r == null) {
            try {
                //通过PackageManager查找对应的Service
                ResolveInfo rInfo = AppGlobals.getPackageManager().resolveService(service,
                        resolvedType, ActivityManagerService.STOCK_PM_FLAGS
                                | PackageManager.MATCH_DEBUG_TRIAGED_MISSING,
                        userId);
                ServiceInfo sInfo =
                    rInfo != null ? rInfo.serviceInfo : null;
                if (sInfo == null) {
                    return null;
                }
                ComponentName name = new ComponentName(
                        sInfo.applicationInfo.packageName, sInfo.name);

                ......

                if (userId > 0) {
                    if (mAm.isSingleton(sInfo.processName, sInfo.applicationInfo,
                            sInfo.name, sInfo.flags)
                            && mAm.isValidSingletonCall(callingUid, sInfo.applicationInfo.uid)) {
                        userId = 0;
                        smap = getServiceMap(0);
                    }
                    sInfo = new ServiceInfo(sInfo);
                    sInfo.applicationInfo = mAm.getAppInfoForUser(sInfo.applicationInfo, userId);
                }

                r = smap.mServicesByName.get(name);

                //如果ServiceRecord不存在，则创建新的
                if (r == null && createIfNeeded) {
                    Intent.FilterComparison filter
                            = new Intent.FilterComparison(service.cloneFilter());
                    ServiceRestarter res = new ServiceRestarter();
                    BatteryStatsImpl.Uid.Pkg.Serv ss = null;
                    BatteryStatsImpl stats = mAm.mBatteryStatsService.getActiveStatistics();
                    synchronized (stats) {
                        ss = stats.getServiceStatsLocked(
                                sInfo.applicationInfo.uid, sInfo.packageName,
                                sInfo.name);
                    }

                    //创建新的ServiceRecord
                    r = new ServiceRecord(mAm, ss, name, filter, sInfo, callingFromFg, res);
                    res.setService(r);

                    //将新创建的ServiceRecord添加到mServicesByName和mServicesByIntent中。
                    smap.mServicesByName.put(name, r);
                    smap.mServicesByIntent.put(filter, r);

                    // Make sure this component isn't in the pending list.
                    for (int i=mPendingServices.size()-1; i>=0; i--) {
                        ServiceRecord pr = mPendingServices.get(i);
                        if (pr.serviceInfo.applicationInfo.uid == sInfo.applicationInfo.uid
                                && pr.name.equals(name)) {
                            mPendingServices.remove(i);
                        }
                    }
                }
            } catch (RemoteException ex) {
                // pm is in same process, this will never happen.
            }
        }


        if (r != null) {

            //检测权限
            if (mAm.checkComponentPermission(r.permission,
                    callingPid, callingUid, r.appInfo.uid, r.exported)
                    != PackageManager.PERMISSION_GRANTED) {
                if (!r.exported) {
                    return new ServiceLookupResult(null, "not exported from uid "
                            + r.appInfo.uid);
                }
                return new ServiceLookupResult(null, r.permission);
            } else if (r.permission != null && callingPackage != null) {
                final int opCode = AppOpsManager.permissionToOpCode(r.permission);
                if (opCode != AppOpsManager.OP_NONE && mAm.mAppOpsService.noteOperation(
                        opCode, callingUid, callingPackage) != AppOpsManager.MODE_ALLOWED) {
                    return null;
                }
            }

            if (!mAm.mIntentFirewall.checkService(r.name, service, callingUid, callingPid,
                    resolvedType, r.appInfo)) {
                return null;
            }

            //当没有任何问题时候，从这里返回查找结果。
            return new ServiceLookupResult(r, null);
        }
        return null;
    }
```
8.  ActiveServices.startServiceInnerLocked
```
    ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
            boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
        ....
        r.callStart = false;
        synchronized (r.stats.getBatteryStats()) {
            r.stats.startRunningLocked();
        }

        //通过bringUpServiceLocked方法继续启动Service
        String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
        if (error != null) {
            return new ComponentName("!!", error);
        }

        ....

        return r.name;
    }
```
9.  ActiveServices.bringUpServiceLocked
```
 private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {

        //对于新创建的ServiceRecord而言,r.app=null,对于已经启动Service而言，
        //r.app和r.app.thread都不会为null,因此只有没有启动过的Serivice才会执行后面的
        //realStartServiceLocked方法。而对于已经启动过的service，则在这里执行sendServiceArgsLocked方法
        //通知app进程执行service的onStartCommand方法。
        if (r.app != null && r.app.thread != null) {
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }

        ....

        final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
        final String procName = r.processName;
        ProcessRecord app;

        if (!isolated) {
            //获取service所在的进程
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
            if (app != null && app.thread != null) {
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);

                    //通过realStartServiceLocked继续启动service
                    realStartServiceLocked(r, app, execInFg);
                    return null;
                } catch (TransactionTooLargeException e) {
                    throw e;
                } catch (RemoteException e) {
                    ...
                }

            }
        } else {
           ...
        }

        if (app == null && !permissionsReviewRequired) {
                //如果Sevice所在进程还没有启动，则通过AMS启动进程
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

            //进程启动成功，将ServiceRecord添加到mPendingServices中，当AMP接收到attachApplication时，
            //会通过ActiveServices将mPendingServices中的所有service通过realStartServiceLocked启动
            if (!mPendingServices.contains(r)) {
                mPendingServices.add(r);
            }
        ...

        return null;
    }
```
10.  ActiveServices.realStartServiceLocked
```
 private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        if (app.thread == null) {
            throw new RemoteException();
        }

        //将Services所在进程赋值给ServiecRecord.app
        r.app = app;
        r.restartTime = r.lastActivity = SystemClock.uptimeMillis();

        final boolean newService = app.services.add(r);

        //这里这个是用于控制ANR的
        bumpServiceExecutingLocked(r, execInFg, "create");
        mAm.updateLruProcessLocked(app, false, null);
        mAm.updateOomAdjLocked();

        boolean created = false;
        try {
            synchronized (r.stats.getBatteryStats()) {
                r.stats.startLaunchedLocked();
            }
            mAm.notifyPackageUse(r.serviceInfo.packageName,
                                 PackageManager.NOTIFY_PACKAGE_USE_SERVICE);
            app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);

            //通知service所在进程创建service，并执行其onCreate方法
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            r.postNotification();
            created = true;
        } catch (DeadObjectException e) {
            mAm.appDiedLocked(app);
            throw e;
        } finally {
            if (!created) {
                ...
            }
        }

        if (r.whitelistManager) {
            app.whitelistManager = true;
        }

        requestServiceBindingsLocked(r, execInFg);

        updateServiceClientActivitiesLocked(app, null, true);


        if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
            r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                    null, null));
        }

        //通过该方法调度Service执行其onStartCommand方法。
        sendServiceArgsLocked(r, execInFg, true);

        if (r.delayed) {
            getServiceMap(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        if (r.delayedStop) {
            r.delayedStop = false;
            if (r.startRequested) {
                stopServiceLocked(r);
            }
        }
    }
```
11. ActiveServices.sendServiceArgsLocked
```
 private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
            boolean oomAdjusted) throws TransactionTooLargeException {
        final int N = r.pendingStarts.size();
        if (N == 0) {
            return;
        }

        while (r.pendingStarts.size() > 0) {
            Exception caughtException = null;
            ServiceRecord.StartItem si = null;
            try {
                si = r.pendingStarts.remove(0);
                if (si.intent == null && N > 1) {
                    continue;
                }
                si.deliveredTime = SystemClock.uptimeMillis();
                r.deliveredStarts.add(si);
                si.deliveryCount++;
                if (si.neededGrants != null) {
                    mAm.grantUriPermissionUncheckedFromIntentLocked(si.neededGrants,
                            si.getUriPermissionsLocked());
                }
                bumpServiceExecutingLocked(r, execInFg, "start");
                if (!oomAdjusted) {
                    oomAdjusted = true;
                    mAm.updateOomAdjLocked(r.app);
                }
                int flags = 0;
                if (si.deliveryCount > 1) {
                    flags |= Service.START_FLAG_RETRY;
                }
                if (si.doneExecutingCount > 0) {
                    flags |= Service.START_FLAG_REDELIVERY;
                }

                //调度Service执行其onStartCommand方法
                r.app.thread.scheduleServiceArgs(r, si.taskRemoved, si.id, flags, si.intent);
            } catch (TransactionTooLargeException e) {
                caughtException = e;
            } catch (RemoteException e) {
                caughtException = e;
            } catch (Exception e) {
                caughtException = e;
            }

           ....
        }
    }
```

12. ActivityThread.handleCreateService
```
 private void handleCreateService(CreateServiceData data) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();

        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            java.lang.ClassLoader cl = packageInfo.getClassLoader();

            //通过反射加载Service实例
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }

        try {
            //Service继承自ContextWrapper，这里创建的context会作为ContextWrapper的mBase。
            //Activity继承自ContextThemeWrapper,ContextThemeWrapper继承自ContextWrapper
            //二者的mBase区别：Activity的有IApplicationToken.Stub类的appToken（是一个window manager token），Service没有。

            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);

            Application app = packageInfo.makeApplication(false, mInstrumentation);

            //执行attach初始化
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
            //执行onCreate方法
            service.onCreate();
            //将service放进ActivityThread管理Service的集合mService（ArrayMap）中。
            mServices.put(data.token, service);
            try {
                ActivityManagerNative.getDefault().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to create service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }
    }
```
13. ActivityThread.handleServiceArgs
```
private void handleServiceArgs(ServiceArgsData data) {
        Service s = mServices.get(data.token);
        if (s != null) {
            try {
                if (data.args != null) {
                    data.args.setExtrasClassLoader(s.getClassLoader());
                    data.args.prepareToEnterProcess();
                }
                int res;
                if (!data.taskRemoved) {

                    //执行service的onStartCommand方法
                    res = s.onStartCommand(data.args, data.flags, data.startId);
                } else {
                    s.onTaskRemoved(data.args);
                    res = Service.START_TASK_REMOVED_COMPLETE;
                }

                QueuedWork.waitToFinish();

                try {
                    ActivityManagerNative.getDefault().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
                ensureJitEnabled();
            } catch (Exception e) {
                ...
            }
        }
    }
```