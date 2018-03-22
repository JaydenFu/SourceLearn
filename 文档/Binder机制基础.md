#   Binder机制基础
参考链接:   http://gityuan.com/2015/11/28/binder-summary/

1.  Binder概述
```
    1.  从进程间通信的角度来看,它是一种Android中跨进程通信的方式,是Android中独有的.
        该通信方法在Linux并没有.
    2.  从我们日常开发角度来看.Binder是客户端和服务端(普通服务和基于AIDL的服务)进行通信的媒介.当bindService的
        时候,服务端会返回一个包含了服务端业务调用的Binder对象.通过这个Binder对象,客户
        端就可以获取服务端提供的服务或者数据.
    3.  从Android Framework层看,Binder还是各种Manager和对应ManagerService的桥梁.
    4.  从Native层看.Binder是创建Service Manager以及BpBinder/BBinder模型,搭建与
        binder驱动的桥梁.
    5.  从驱动层看:Binder还可以理解为一种序列的物理设备.设备驱动是/dev/binder
```
2.  Android为什么要采用Binder来实现IPC?
```
    主要从3个方面考虑:
    1.性能:Linux中除了共享内存,其他通信方式都需要2次数据拷贝.而Binder只需要一次.
    2.稳定:Binder是基于C/S架构的.Client端和Server端相对独立,Client端有需求,直接发送
           给Server端去完成.而共享内存实现方式复杂,还需要别的通信机制来实现同步的问题,
           相对来说Binder方式稳定性比较好.
    3.安全性: 传统Linux IPC的接受方无法获得对方进程的可靠UID/PID.因此无法鉴别对方身份.
           Android为每个安装的应用都分配了自己的UID.因此进程的UID是鉴别进程身份的重要
           标志.Server端会更具权限控制策略.判断UID/PID是否满足访问权限.
```
3.  Framework层Binder机制涉及到的类
```
    1.  IInterface 接口:
        方法:public IBinder asBinder()
        对于想要通过Binder机制提供远程服务能力的类,必须实现该接口.

    2.  IBinder 接口:
        描述了和远程对象交互的抽象协议.
        通常不会直接使用该接口.而是使用它的实现类Binder

    3.  Binder 类:
        实现了IBinder接口.
        方法:public IInterface queryLocalInterface(String descriptor)被实现.返回mOwner.
        成员:
             private long mObject;持有的是JavaBBinderHolder对象的地址.在Natiive层被赋值.
             private IInterface mOwner;//持有的是提供远程服务能力的类本身.
             private String mDescriptor;//描述符.通常是提供业务接口的全名.
                                        //名称采用AIDL会自动生成.在类初始化的时候,会赋值给mDescriptor.
        对于要提供远程服务能力的类.必须继承该类.实际开发中.我们一般使用AIDL,当我们
        定义了接口aidl.会帮我们自动生成实现了IInterface接口的业务接口.并且生成继承
        Binder类并且实现了前述的业务接口的抽象静态内部Stub类.

    4.  BinderProxy 类:Binder的内部类.
        实现了IBinder接口.
        方法:public IInterface queryLocalInterface(String descriptor)返回null.
        成员:
            final private WeakReference mSelf;//记录的是BinderProxy的弱引用.
            private long mObject;//记录的是Native层的BpBinder对象.
            private long mOrgue;//记录的是死亡通知对象.

        对于我们平时同bindService而言,如果是进程间的,那么ServiceConnection回调回来的
        就是一个BinderProxy对象.通过将BinderProxy执行asInterface过程.返回的其实一个实现了
        远程业务接口的代理类.对于业务调用过程.最终还是通过BinderProxy的
        transact方法(内部通过调用native层)完成.
```

4.  Native层Binder机制涉及到的类
```
    1.  IBinder类:
        方法:
            BBinder* IBinder::localBinder()
            {
                return NULL;
            }

            BpBinder* IBinder::remoteBinder()
            {
                return NULL;
            }

            sp<IInterface>  IBinder::queryLocalInterface(const String16& /*descriptor*/)
            {
                return NULL;
            }

    2.  BpBinder类.继承了IBinder类:
        会作为远程binder返回给服务调用者.在java层中被BinderProxy的mObject成员变量持有
        调用方通过BpBinder和Binder驱动进行通信.
        方法:
            BpBinder* BpBinder::remoteBinder()
            {
                return this;
            }

    3.  BBinder类.继承了IBinder类.
        BBinder在native层与Java层提供服务者Binder相关联.Binder驱动通过BBinder与
        服务提供者进行通信.
        方法:
            BBinder* BBinder::localBinder()
            {
                return this;
            }
```
