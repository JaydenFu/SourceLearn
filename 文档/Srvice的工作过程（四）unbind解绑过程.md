#   Service的unbind过程
1.  AMS.unbindService
```
    public boolean unbindService(IServiceConnection connection) {
        synchronized (this) {
            //1.通过ActiveServices解绑
            return mServices.unbindServiceLocked(connection);
        }
    }
```

2. ActiveServices.unbindServiceLocked
```
 boolean unbindServiceLocked(IServiceConnection connection) {
        IBinder binder = connection.asBinder();

        //1. 获取当前binder对应的全部ConnectionRecord列表
        ArrayList<ConnectionRecord> clist = mServiceConnections.get(binder);
        if (clist == null) {
            return false;
        }

        final long origId = Binder.clearCallingIdentity();
        try {
            while (clist.size() > 0) {
                ConnectionRecord r = clist.get(0);
                //2. 循环移除Connection
                removeConnectionLocked(r, null, null);
                if (clist.size() > 0 && clist.get(0) == r) {
                    clist.remove(0);
                }

                ....
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }

        return true;
    }
```

2.  ActiveServices.removeConnectionLocked
```
void removeConnectionLocked(
        ConnectionRecord c, ProcessRecord skipApp, ActivityRecord skipAct) {
        IBinder binder = c.conn.asBinder();
        AppBindRecord b = c.binding;
        ServiceRecord s = b.service;

        //1.下面主要都是移除和binder相关的ConnectionRecord
        //几个地方：ServiceRecord的connections集合，AppBindRecord的connections集合
        //ProcessRecord的connectins集合，ActiityRecord的connections集合
        //ActiveService的mServiceConnections集合
        ArrayList<ConnectionRecord> clist = s.connections.get(binder);
        if (clist != null) {
            clist.remove(c);
            if (clist.size() == 0) {
                s.connections.remove(binder);
            }
        }
        b.connections.remove(c);
        if (c.activity != null && c.activity != skipAct) {
            if (c.activity.connections != null) {
                c.activity.connections.remove(c);
            }
        }
        if (b.client != skipApp) {
            b.client.connections.remove(c);
            ...
        }
        clist = mServiceConnections.get(binder);
        if (clist != null) {
            clist.remove(c);
            if (clist.size() == 0) {
                mServiceConnections.remove(binder);
            }
        }

        mAm.stopAssociationLocked(b.client.uid, b.client.processName, s.appInfo.uid, s.name);

        if (b.connections.size() == 0) {
            b.intent.apps.remove(b.client);
        }

        if (!c.serviceDead) {
            //只有当所有bindService的所有进程都执行了unbind操作，service才会真正的unbind
            if (s.app != null && s.app.thread != null && b.intent.apps.size() == 0
                    && b.intent.hasBound) {
                try {
                    ...
                    b.intent.hasBound = false;
                    b.intent.doRebind = false;

                    //调度service执行unbind
                    s.app.thread.scheduleUnbindService(s, b.intent.intent.getIntent());
                } catch (Exception e) {
                    serviceProcessGoneLocked(s);
                }
            }

            mPendingServices.remove(s);

            if ((c.flags&Context.BIND_AUTO_CREATE) != 0) {
                boolean hasAutoCreate = s.hasAutoCreateConnections();
                ...
                //如果service不在需要了，调度service执行onDestroy。和stop过程一致。
                bringDownServiceIfNeededLocked(s, true, hasAutoCreate);
            }
        }
    }
```