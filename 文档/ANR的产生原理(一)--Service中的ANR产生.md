#   ANR的产生原理(一)--Service中的ANR产生

ANR:Application Not Responding. 应用程序无响应.Android系统对于一些事件的处理需要在一定的时间范围内
完成,如果超过了预定时间而未得到有效响应或者响应时间过长,则系统会提示ANR.

通常产生ANR的场景:

   *    Service TimeOut:在正常启动的服务中20s未执行完成  (目前知道在执行onDestroy和unBind过程中,可能还有其他情况)后台服务则为200s
   *    BroadCast TimeOut: 在正常启动的广播中10s内未执行完成,  (还不清楚啥时候触发该模式)后台广播则为60s
   *    ContentProvider TimeOut:    内容提供者,在publish过超时10s.
   *    InputDispatching    TimeOut:    输入事件(按键/触摸)分发超过5s.

一.  Service中产生ANR

1.  当通过startService启动service实际会通过进程间通信调用ActivityManagerService的startService方法
最终启动通过ActiveServices的realStartServiceLocked方法启动service.
```
private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {

        //在这里面发送SERVICE_TIMEOUT_MSG延迟消息
        //在前台进程操作情况下execInFg是true
        bumpServiceExecutingLocked(r, execInFg, "create");
        ......
        try {
            .....
            //通过进程间通信,让ApplicationThread启动service执行其onCreate方法
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            r.postNotification();
            created = true;
        } catch (DeadObjectException e) {
            mAm.appDiedLocked(app);
            throw e;
        } finally {
           .....
        }
        ......
    }
```
2.  ActiveServices的bumpServiceExecutingLocked方法:
```
    private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
        long now = SystemClock.uptimeMillis();
        if (r.executeNesting == 0) {
            r.executeFg = fg;
            ServiceState stracker = r.getTracker();
            if (stracker != null) {
                stracker.setExecuting(true, mAm.mProcessStats.getMemFactorLocked(), now);
            }
            if (r.app != null) {
                //将serviceRecord添加进正在执行的serviec集合中.
                r.app.executingServices.add(r);
                r.app.execServicesFg |= fg;
                if (r.app.executingServices.size() == 1) {
                    //通过scheduleServiceTimeoutLocked发送消息
                    scheduleServiceTimeoutLocked(r.app);
                }
            }
        } else if (r.app != null && fg && !r.app.execServicesFg) {
            r.app.execServicesFg = true;
            //通过scheduleServiceTimeoutLocked发送消息
            scheduleServiceTimeoutLocked(r.app);
        }
        r.executeFg |= fg;
        r.executeNesting++;
        //设置service启动时刻的时间.后面会通过该时间对比.检测ANR.
        r.executingStart = now;
    }

```
3.  ActiveServices的scheduleServiceTimeoutLocked方法
```
    void scheduleServiceTimeoutLocked(ProcessRecord proc) {
        if (proc.executingServices.size() == 0 || proc.thread == null) {
            return;
        }
        long now = SystemClock.uptimeMillis();
        Message msg = mAm.mHandler.obtainMessage(
                ActivityManagerService.SERVICE_TIMEOUT_MSG);
        msg.obj = proc;
        //对于前台进程中执行,超时时间是20s.后台进程.超时时间是200s
        //mAm是ActivityManagerSerivce.   mHander是AMS中的MainHandler
        mAm.mHandler.sendMessageAtTime(msg,
                proc.execServicesFg ? (now+SERVICE_TIMEOUT) : (now+ SERVICE_BACKGROUND_TIMEOUT));
    }

```
4.  AMS中MainHandler的handMessage的处理SERVICE_TIMEOUT_MSG部分:
```
 case SERVICE_TIMEOUT_MSG: {
                .....
                mServices.serviceTimeout((ProcessRecord)msg.obj);
            } break;
```
5.  ActiveServices的serviceTimeout方法:
```
void serviceTimeout(ProcessRecord proc) {
        String anrMessage = null;

        synchronized(mAm) {
            if (proc.executingServices.size() == 0 || proc.thread == null) {
                return;
            }
            final long now = SystemClock.uptimeMillis();
            //通过当前时间减去超时时间获得如果不超时的情况下,Service最早执行时间
            final long maxTime =  now -
                    (proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
            ServiceRecord timeout = null;
            long nextTime = 0;

            //遍历所有在执行中的service.如果发现有service的执行时间再maxTime之前.
            //则说明超时了.
            for (int i=proc.executingServices.size()-1; i>=0; i--) {
                ServiceRecord sr = proc.executingServices.valueAt(i);
                if (sr.executingStart < maxTime) {
                    timeout = sr;
                    break;
                }
                if (sr.executingStart > nextTime) {
                    nextTime = sr.executingStart;
                }
            }
            if (timeout != null && mAm.mLruProcesses.contains(proc)) {
                //生成ANR日志
                StringWriter sw = new StringWriter();
                PrintWriter pw = new FastPrintWriter(sw, false, 1024);
                pw.println(timeout);
                timeout.dump(pw, "    ");
                pw.close();
                mLastAnrDump = sw.toString();
                mAm.mHandler.removeCallbacks(mLastAnrDumpClearer);
                mAm.mHandler.postDelayed(mLastAnrDumpClearer, LAST_ANR_LIFETIME_DURATION_MSECS);
                anrMessage = "executing service " + timeout.shortName;
            } else {
                //如果没有发现有超时service.则再发送检测超时的消息.
                Message msg = mAm.mHandler.obtainMessage(
                        ActivityManagerService.SERVICE_TIMEOUT_MSG);
                msg.obj = proc;
                mAm.mHandler.sendMessageAtTime(msg, proc.execServicesFg
                        ? (nextTime+SERVICE_TIMEOUT) : (nextTime + SERVICE_BACKGROUND_TIMEOUT));
            }
        }

        //ANR 日志不为空,通过AMS的AppErrors报告ANR.
        if (anrMessage != null) {
            mAm.mAppErrors.appNotResponding(proc, null, null, false, anrMessage);
        }
    }
```
6.  上面分析了Service如何产生ANR.接下来分析系统在未超时完成的情况下是如何没有引发ANR的.
在前面第一步的事件分析了.在通过ApplicationThreadProxy执行scheduleCreateService之前发送的超时检测
消息.那么在scheduleCreateService之后一定也有相应的措施来保证检测没问题.
```
    //scheduleCreateService最终会调用执行ActivityThread的handCreateService方法.
    private void handleCreateService(CreateServiceData data) {
             ....
            LoadedApk packageInfo = getPackageInfoNoCheck(
                    data.info.applicationInfo, data.compatInfo);
            Service service = null;
            try {
                //通过反射创建Service实例
                java.lang.ClassLoader cl = packageInfo.getClassLoader();
                service = (Service) cl.loadClass(data.info.name).newInstance();
            } catch (Exception e) {
                ....
            }

            try {
                ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
                context.setOuterContext(service);
                Application app = packageInfo.makeApplication(false, mInstrumentation);
                //执行service的attach方法.初始化一些service的成员信息
                service.attach(context, this, data.info.name, data.token, app,
                        ActivityManagerNative.getDefault());

                 //执行Service的生命周期onCreate方法.
                service.onCreate();
                mServices.put(data.token, service);
                try {
                    //然后通过ActivityManagerProxy通知AMS我们的service执行完onCreate方法了.
                    ActivityManagerNative.getDefault().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            } catch (Exception e) {
                .....
            }
        }
```
7.  ActivityManagerService的serviceDoneExecuting方法:
```
    public void serviceDoneExecuting(IBinder token, int type, int startId, int res) {
        synchronized(this) {
            if (!(token instanceof ServiceRecord)) {
                throw new IllegalArgumentException("Invalid service token");
            }
            //调用serviceDoneExecutingLocked方法.对用onCreate执行完后,回调而言,这里type是
            //SERVICE_DONE_EXECUTING_ANON
            mServices.serviceDoneExecutingLocked((ServiceRecord)token, type, startId, res);
        }
    }
```
8.  ActiveServices的serviceDoneExecutingLocked方法:
```
 void serviceDoneExecutingLocked(ServiceRecord r, int type, int startId, int res) {
        boolean inDestroying = mDestroyingServices.contains(r);
        if (r != null) {
            对于onCreate执行完后回调而言,type为SERVICE_DONE_EXECUTING_ANON
            所以if 和else if里面的代码都不会执行.
            if (type == ActivityThread.SERVICE_DONE_EXECUTING_START) {
                .....
            } else if (type == ActivityThread.SERVICE_DONE_EXECUTING_STOP) {
               .....
            }
            final long origId = Binder.clearCallingIdentity();
            //正常情况下,service刚执行完onCreate,isDetroying为false.
            serviceDoneExecutingLocked(r, inDestroying, inDestroying);
            Binder.restoreCallingIdentity(origId);
        } else {
            ....
        }
    }
```
9.  ActiveServices的serviceDoneExecutingLocked方法:
```
private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
            boolean finishing) {
        r.executeNesting--;
        if (r.executeNesting <= 0) {
            if (r.app != null) {
                r.app.execServicesFg = false;
                //从正在执行的service中移除我们已经执行完onCreate的service
                //从上面第5步可以知道.检测超时的时候,正是从executingServices集合中取service来检测.
                //所里当service先在这里被移除了.则在检测的时候就不会再检测到超时了.
                r.app.executingServices.remove(r);

                //如果这个时间没有正在启动的service了.就没有必要在执行SERVICE_TIMEOUT检测了.,
                //所以会从消息队列中移除该检测消息.否则还是需要继续其他的Service检测.
                if (r.app.executingServices.size() == 0) {
                    //移除我们发送的SERVICE_TIMEOUT_MSG消息.
                    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
                } else if (r.executeFg) {
                    for (int i=r.app.executingServices.size()-1; i>=0; i--) {
                        if (r.app.executingServices.valueAt(i).executeFg) {
                            r.app.execServicesFg = true;
                            break;
                        }
                    }
                }
                if (inDestroying) {
                    mDestroyingServices.remove(r);
                    r.bindings.clear();
                }
                mAm.updateOomAdjLocked(r.app);
            }
            r.executeFg = false;
            ....
            if (finishing) {
                if (r.app != null && !r.app.persistent) {
                    r.app.services.remove(r);
                    if (r.whitelistManager) {
                        updateWhitelistManagerLocked(r.app);
                    }
                }
                r.app = null;
            }
        }
    }
```
10. 回到ActiveServices的realStartServiceLocked方法中:
```
 private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        ....
        boolean created = false;
        try {
            ....
            //调度Service执行其onCreate方法,这里是异步执行Binder的.ONEWAY方式
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            r.postNotification();
            created = true;
        } catch (DeadObjectException e) {
            mAm.appDiedLocked(app);
            throw e;
        } finally {
            if (!created) {//对于启动正常而言,不会执行这里
                ....
            }
        }

        if (r.whitelistManager) {
            app.whitelistManager = true;
        }
        .......

        //将当前service添加到即将start的集合里面
        if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
            r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                    null, null));
        }
        //在该方法里会去调度执行Service的onStartCommand方法.
        sendServiceArgsLocked(r, execInFg, true);
        ......
    }
```
11. ActiveServices的sendServiceArgsLocked方法:
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

                //在调度执行service的onStartCommand方法之前.再次通过bumpServiceExecutingLocked方法
                //根据情况是否发送超时检测.如果之前的onCreate时的超时检测没有完成.那么这里不会重新发送消息.
                //只是会调整ServiceRecord的executingStart时间值,已经增加一次executeNesting值.这种情况下,之前
                //超时检测达到时,会根据最新的executingStart时间检测.
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
                //调度Service执行onStartCommand方法.
                r.app.thread.scheduleServiceArgs(r, si.taskRemoved, si.id, flags, si.intent);
            } catch (TransactionTooLargeException e) {
                caughtException = e;
            } catch (RemoteException e) {
                caughtException = e;
            } catch (Exception e) {
                caughtException = e;
            }

            if (caughtException != null) {
                ....
                break;
            }
        }
    }
```
12. 当service的onStartCommand方法执行后.同样会回调会ActivityManagerService的serviceDoneExecuting方法.且type为SERVICE_DONE_EXECUTING_START
这个时候步骤又是重复第7.8.9步骤.

```
总结:
    对于Service的任何一个什么周期方法而言.都会在执行前.添加超时检测.执行后移除.如果任何一个生命周期方法中超时,则都会ANR.
    service的onDestory和unBind方法中,超时时间是200s.(通过源码知道.实际实验验证也确实如此)
    其他什么周期则是10s.
```

