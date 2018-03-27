#   ANR产生原理(二)--BroadcastReceiver中的ANR产生

BroadCast TimeOut: 在普通Intent启动的广播中60s内未执行完成, 前台广播则为10s(Intent设置了FLAG_RECEIVER_FOREGROUND flag,默认情况时没有设置的).
*   对于通过代码动态注册的广播而言.接收无序广播,在onReceive方法是不直接导致ANR的(但是可能会间接导致.因为onReceiver占用主线程时间.
如果还有另一个接收同样广播的静态注册的广播接收者,那么这个静态注册的广播接收者的onReceive方法就不能在超时时间内完成.同样会引起ANR).
接收有序广播.超时会直接导致ANR.

*   通过Manifest静态注册的广播无论接收的是否是有序广播都存在超时一说导致ANR.

*   对于接收同一个广播的所有广播接收者响应完的总时间也有要求:总时间<2\*广播接受者个数\*TIME_OUT;//对于普通的来说.TIME_OUT是60s.前台广播来说:TIME_OUT是10s.
    否则,同样会产生ANR.


我们知道BroadcastReceiver接受到广播后,执行的是onReceive方法.先来看这个方法是怎么被调用执行的.
我们知道AMS负责管理四大组件.所以先从AMS中看看和广播相关的方法:

1.  ActivityManagerService的broadcastIntentLocked方法:这个方法很长.
```
    //这里处理找到了动态注册的广播接收器,
    int NR = registeredReceivers != null ? registeredReceivers.size() : 0;

    //这里要无序广播且对应的动态注册的广播接收者数量大于0才执行
    if (!ordered && NR > 0) {
        if (isCallerSystem) {
            checkBroadcastFromSystem(intent, callerApp, callerPackage, callingUid,
                    isProtectedBroadcast, registeredReceivers);
        }
        final BroadcastQueue queue = broadcastQueueForIntent(intent);
        BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                callerPackage, callingPid, callingUid, resolvedType, requiredPermissions,
                appOp, brOptions, registeredReceivers, resultTo, resultCode, resultData,
                resultExtras, ordered, sticky, false, userId);
        final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);
        if (!replaced) {
            //对于动态注册的广播接收器而言,这里会放进mParallelBroadcasts列表.后面在想这里面的广播接收器传递广播的时候,是没有超时检测的.
            queue.enqueueParallelBroadcastLocked(r);
            queue.scheduleBroadcastsLocked();
        }
        registeredReceivers = null;
        NR = 0;
    }
    ....//这里省略了对有序广播而言.将动态注册的广播接收器放入receivers的逻辑.

    //这里是通过Manifest静态注册的广播接收器.和有序广播对应的动态注册的广播接收者.
   if ((receivers != null && receivers.size() > 0)|| resultTo != null) {

       //根据intent获取对应的BroadcastQueue
       BroadcastQueue queue = broadcastQueueForIntent(intent);

       //创建一条广播记录.
       BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
               callerPackage, callingPid, callingUid, resolvedType,
               requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
               resultData, resultExtras, ordered, sticky, false, userId);
       //是否从队列中替换了广播.
       boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r);

       if (!replaced) {
            //将BroadcastRecord加入mOrderedBroadcasts队列.这个队列里面的处理.系统会做超时检测.
           queue.enqueueOrderedBroadcastLocked(r);
           //调度执行.
           queue.scheduleBroadcastsLocked();
       }
   }
```

2.  ActivityManagerService的broadcastQueueForIntent方法:
```
    mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
            "foreground", BROADCAST_FG_TIMEOUT, false);//BROADCAST_FG_TIMEOUT为10s
    mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
            "background", BROADCAST_BG_TIMEOUT, true);//BROADCAST_BG_TIMEOUT为60s.

    BroadcastQueue broadcastQueueForIntent(Intent intent) {
        //对于我们通过context.sendBroadcast而言.默认情况下.Intent是没有设置FLAG_RECEIVER_FOREGROUND标志的.
        //除非手动对发送的Intent设置该值.
        final boolean isFg = (intent.getFlags() & Intent.FLAG_RECEIVER_FOREGROUND) != 0;
        //所以默认情况下,这里返回的是mBgBroadcastQueue
        return (isFg) ? mFgBroadcastQueue : mBgBroadcastQueue;
    }
```
3.  BroadcastReceiver的enqueueOrderedBroadcastLocked方法:
```
    final ArrayList<BroadcastRecord> mParallelBroadcasts = new ArrayList<>();
    public void enqueueParallelBroadcastLocked(BroadcastRecord r) {
        //添加进列表中
        mParallelBroadcasts.add(r);
        r.enqueueClockTime = System.currentTimeMillis();
    }
```
4.  BroadcastReceiver的scheduleBroadcastsLocked方法:
```
    public void scheduleBroadcastsLocked() {
        //mBroadcastsScheduled为true表明正在处理,不用再通知处理了.
        if (mBroadcastsScheduled) {
            return;
        }
        //通过发送消息执行广播的调度,mHandler是BroadcastHandler的实例
        mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
        mBroadcastsScheduled = true;
    }

    private final class BroadcastHandler extends Handler {
        public BroadcastHandler(Looper looper) {
            super(looper, null, true);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case BROADCAST_INTENT_MSG: {
                    //所以对于BROADCAST_INTENT_MSG消息.执行的是BroadcastReceiver的processNextBroadcast方法.
                    processNextBroadcast(true);
                } break;
                case BROADCAST_TIMEOUT_MSG: {
                    synchronized (mService) {
                        broadcastTimeoutLocked(true);
                    }
                } break;
                case SCHEDULE_TEMP_WHITELIST_MSG: {
                ....
                } break;
            }
        }
    }
```
5.  BroadcastQueue的processNextBroadcast方法:
```
final void processNextBroadcast(boolean fromMsg) {
        synchronized(mService) {
            BroadcastRecord r;
            mService.updateCpuStats();

            if (fromMsg) {
                mBroadcastsScheduled = false;
            }

            //循环处理动态注册的广播接收者,可以接受的广播.处理这部分的时候,没有超时检测.所以不会直接导致ANR的产生.
            while (mParallelBroadcasts.size() > 0) {
                r = mParallelBroadcasts.remove(0);
                r.dispatchTime = SystemClock.uptimeMillis();
                r.dispatchClockTime = System.currentTimeMillis();
                final int N = r.receivers.size();
                for (int i=0; i<N; i++) {
                    Object target = r.receivers.get(i);
                    //通过该方法传递广播
                    deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false, i);
                }
                addBroadcastToHistoryLocked(r);
            }

            if (mPendingBroadcast != null) {
                boolean isDead;
                synchronized (mService.mPidsSelfLocked) {
                    ProcessRecord proc = mService.mPidsSelfLocked.get(mPendingBroadcast.curApp.pid);
                    isDead = proc == null || proc.crashing;
                }
                if (!isDead) {
                    return;
                } else {
                    mPendingBroadcast.state = BroadcastRecord.IDLE;
                    mPendingBroadcast.nextReceiver = mPendingBroadcastRecvIndex;
                    mPendingBroadcast = null;
                }
            }

            boolean looped = false;

            do {
                //对于传递给静态注册的广播接受者的广播个数不为0才会可能产生ANR.
                if (mOrderedBroadcasts.size() == 0) {
                    mService.scheduleAppGcsLocked();
                    if (looped) {
                        mService.updateOomAdjLocked();
                    }
                    return;
                }

                //从列表中取出第一个广播.
                r = mOrderedBroadcasts.get(0);
                boolean forceReceive = false;
                //可接受该广播的广播接收者数量
                int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;

                //每个广播在被第一次传递之前,dispatchTime是为0的
                if (mService.mProcessesReady && r.dispatchTime > 0) {
                    long now = SystemClock.uptimeMillis();
                    if ((numReceivers > 0) &&
                            (now > r.dispatchTime + (2*mTimeoutPeriod*numReceivers))) {
                        //当前广播已经超时,强制结束该广播
                        broadcastTimeoutLocked(false); // forcibly finish this broadcast
                        forceReceive = true;
                        r.state = BroadcastRecord.IDLE;
                    }
                }

                if (r.state != BroadcastRecord.IDLE) {
                    return;
                }

                //r.nextReceiver >= numReceivers意味着当所有该广播所有可处理的广播接收者都处理完了.
                if (r.receivers == null || r.nextReceiver >= numReceivers
                        || r.resultAbort || forceReceive) {
                    ....
                    //移除超时检测.
                    cancelBroadcastTimeoutLocked();

                    addBroadcastToHistoryLocked(r);
                    ....
                    //移除该广播.继续处理下一个广播
                    mOrderedBroadcasts.remove(0);
                    r = null;
                    looped = true;
                    continue;
                }
            } while (r == null);

            //取当前广播的下一个广播接受者
            int recIdx = r.nextReceiver++;
            //receiverTime广播每次被开始传递的时间.
            r.receiverTime = SystemClock.uptimeMillis();
            if (recIdx == 0) {
                //dispatchTime第一次传递该广播时间.
                r.dispatchTime = r.receiverTime;
                r.dispatchClockTime = System.currentTimeMillis();
            }

            //mPendingBroadcastTimeoutMessage默认是false.
            if (! mPendingBroadcastTimeoutMessage) {
                long timeoutTime = r.receiverTime + mTimeoutPeriod;

                //发送超时检测消息.查看第6步.
                setBroadcastTimeoutLocked(timeoutTime);
            }

            final BroadcastOptions brOptions = r.options;
            final Object nextReceiver = r.receivers.get(recIdx);

            //当广播接收者类型为BroadcastFilter时.执行这里.也就是说这里处理的是有序广播且对应的广播接收者是动态注册的.
            if (nextReceiver instanceof BroadcastFilter) {
                BroadcastFilter filter = (BroadcastFilter)nextReceiver;
                //这里传递广播
                deliverToRegisteredReceiverLocked(r, filter, r.ordered, recIdx);
                if (r.receiver == null || !r.ordered) {
                    // The receiver has already finished, so schedule to
                    // process the next one.
                    r.state = BroadcastRecord.IDLE;
                    scheduleBroadcastsLocked();
                } else {
                   .....
                }
                return;
            }

            //当广播接收者类型为ResolveInfo时.说明这里处理的广播接收者都是动态Manifest静态注册的.
            ResolveInfo info =
                (ResolveInfo)nextReceiver;
            ComponentName component = new ComponentName(
                    info.activityInfo.applicationInfo.packageName,
                    info.activityInfo.name);
            .......

            //当广播接收者所在进程已经启动.
            if (app != null && app.thread != null) {
                try {
                    app.addPackage(info.activityInfo.packageName,
                            info.activityInfo.applicationInfo.versionCode, mService.mProcessStats);
                    //处理当前广播
                    processCurBroadcastLocked(r, app);
                    return;
                } catch (RemoteException e) {
                } catch (RuntimeException e) {
                    logBroadcastReceiverDiscardLocked(r);
                    finishReceiverLocked(r, r.resultCode, r.resultData,
                            r.resultExtras, r.resultAbort, false);
                    scheduleBroadcastsLocked();
                    r.state = BroadcastRecord.IDLE;
                    return;
                }
            }

            //如果处理广播的广播接收者所在进程还没启动.启动进程先.
            if ((r.curApp=mService.startProcessLocked(targetProcess,
                    info.activityInfo.applicationInfo, true,
                    r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
                    "broadcast", r.curComponent,
                    (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, false, false))
                            == null) {
                logBroadcastReceiverDiscardLocked(r);
                finishReceiverLocked(r, r.resultCode, r.resultData,
                        r.resultExtras, r.resultAbort, false);
                scheduleBroadcastsLocked();
                r.state = BroadcastRecord.IDLE;
                return;
            }
            mPendingBroadcast = r;
            mPendingBroadcastRecvIndex = recIdx;
        }
    }
```

6.  BroadcastReceiver的setBroadcastTimeoutLocked方法:
```
    final void setBroadcastTimeoutLocked(long timeoutTime) {
        if (! mPendingBroadcastTimeoutMessage) {
            //发送检测广播超时的消息.实际会调用broadcastTimeoutLocked方法:
            Message msg = mHandler.obtainMessage(BROADCAST_TIMEOUT_MSG, this);
            mHandler.sendMessageAtTime(msg, timeoutTime);
            mPendingBroadcastTimeoutMessage = true;
        }
    }
```

7.  BroadcastReceiver的broadcastTimeoutLocked方法:
```
 final void broadcastTimeoutLocked(boolean fromMsg) {
        if (fromMsg) {
            mPendingBroadcastTimeoutMessage = false;
        }

        //如果待处理的广播列表中没有广播了.则直接返回.
        if (mOrderedBroadcasts.size() == 0) {
            return;
        }

        long now = SystemClock.uptimeMillis();
        BroadcastRecord r = mOrderedBroadcasts.get(0);
        if (fromMsg) {
            .....
            //通过广播的传递事件+超时时间.与当前事件对比.如果大于当前时间.则还未超时,否则就超时了.
            long timeoutTime = r.receiverTime + mTimeoutPeriod;
            if (timeoutTime > now) {
                //当前还未超时,再次发送超时检测消息.
                setBroadcastTimeoutLocked(timeoutTime);
                return;
            }
        }

        ......
        //执行到这里说明已经超时啦.
        .....
        if (app != null) {
            anrMessage = "Broadcast of " + r.intent.toString();
        }
        .....

        //产生ANR
        if (anrMessage != null) {
            mHandler.post(new AppNotResponding(app, anrMessage));
        }
    }
```
8.  当一个广播处理完成后,会回调AMS的finishReceiver方法.在该方法内部会调用BroadcastQueue的processNextBroadcast方法.这样就循环处理的消息.