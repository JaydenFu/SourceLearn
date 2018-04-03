#   Service的stop过程

通过AMP和系统进程的AMS通信：
1.  AMS将stopService的操作分配给ActiveService
2.  ActiveService通过retrieveServiceLocked方法查找对应的ServiceRecord
3.  ActiveService在bringDownServiceIfNeededLocked中通过个条件判断是否需要stop掉service
    *   Service是否又被显示的start了
    *   Service是否被Bind过，且ConnectionRecord设置了Context.BIND_AUTO_CREATE的标识

    如果存在以上两种情况，则不继续执行stop的操作。
4.  ActiveService的bringDownServiceLocked中：
    *   从mServicesByName和mServicesByIntent两个集合中移除该ServiceRecord
5.  通过ApplicationThreadProxy和Service所在进程的ApplicationThread通信，使service执行onDestroy，
    并且从ActivityThread的mService集合中移除service