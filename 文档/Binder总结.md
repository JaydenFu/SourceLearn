#   Binder总结

http://gityuan.com/2015/11/28/binder-summary/

```
Binder是Android提供给我们的一种进程间通信的方式.平时我们开发中使用AIDL来进行进程间通信时,其实底层采用的就是Binder机制.
它应该也是Andorid系统中最核心的基础功能.像我们平时调用startActivity.startService.bindService等.其实底层都用到了Binder机制.

完整Bidner架构,贯穿了应用层.Native层.内核层.

内核层: Binder驱动程序.    Binder实际对象, Binder引用对象

    进程创建的时候会初始化ProcesState,初始化过程中会调用open系统调用打开binder驱动.获取到对应的文件描述符.同时会通过mmap系统调用将用户空间的虚拟地址
    内核空间的虚拟地址映射到同一块物理内存.然后会创建Binder线程池(加上主Binder线程,Binder线程池最多16个线程).
    通过Binder引用对象可以找到对应的Binder实际对象.然后会向目标进程的Binder线程的todo列表中,插入一条通信事务.
    而在目标Binder所在进程的Binder线程中会处理该事物.
    在传输过程中.如果通信内容包含binder对象.会在驱动中创建相应的Binder实际对象binder_node.以及Binder引用对象binder_ref.


Native层:IPCThreadState,JavaBBinder(继承BBinder)(Binder本地对象).  BpBinder(Binder代理对象,持有驱动层Binder引用对象的句柄).

    IPCThreadState是和Binder驱动通信的桥梁.不管是Client端还是Server端都是通过该类完成
    JavaBBinder持有Java层服务提供者.Binder驱动通过BBinder向Server端发起通信.
    BpBinder被Java层BinderProxy持有.Client端通过BpBidner向Binder驱动发起通信.
    BpBinder持有驱动层Binder引用对象的句柄.在transact过程中,会将该句柄已经听信内容通过io_ctl传递给驱动层.
    驱动层根据该句柄找到Binder实际对象.然后完成信息的传输.

应用层:Binder.BinderProxy.

    Binder: Server端提供服务的类继承该类.同时实现IInterface接口.
    BinderProxy: Client端持有.Client端通过BinderProxy向Server端发起通信.

Client端发起通信:
->  BinderProxy.transact()  java
->  BpBinder.transact() native
->  IPCThreadState.transact()   native
->  io_ctl和binder驱动层通信  native

Server端响应消息
->  IPCThreadState.executeCommand() native
->  JavaBBinder.transact()      native
->  JavaBBinder.onTransact()    native
->  Binder.executeTransact  java
->  Binder.onTransact()     java
```
