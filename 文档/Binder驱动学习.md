#   Binder驱动学习
参考链接:
*   http://gityuan.com/2015/11/01/binder-driver/
*   http://gityuan.com/2015/11/02/binder-driver-2/

1.  binder_open打开驱动设备过程
```
在Binder驱动打开过程中:
    1.创建binder_proc并在内核空间为其分配内存.
    2.将当前线程的task保存到binder_proc的tsk中.
    3.初始化binder_proc中的todo列表与wait队列.
    4.将binder_proc的proc_node节点添加到binder_procs为表头的队列中.binder_procs是个静态的变量.
    5.将当前线程所在的进程id赋值给binder_proc的pid.
    6.把代表binder驱动文件的file文件指针的private_data变量指向binder_proc数据.
```

2. binder_mmap:
```
    1.在内核虚拟地址空间申请一块与用户虚拟内存相同大小的内存.
    2.再申请1个page大小的物理内存.再将一块物理内存分别映射到内核虚拟地址空间和
      用户虚拟内存空间,从而实现用户空间的Buffer和内核空间的Buffer同步操作的功能.

    在这个过程会通过加锁的方式保证一次只有一个进程分配内存.保证多进程间的并发访问.

```
3.  binder_ioctl:
```
    binder_ioctl()函数负责在两个进程间收发IPC数据和IPC reply数据.
```
4.  IPC层与驱动层的之间通信协议类型包含在IPC数据中:binder_transaction_data结构
````
    IPC层->Binder驱动层 的协议:
    通过IPCThreadState中的writeTransactionData写入.
    在Binder驱动层的binder_thread_write中被识别

    BC_TRANSACTION
    BC_REPLY
    BC_REGISTER_LOOPER
    BC_ENTER_LOOPER
    BC_CLEAR_DEATH_NOTIFICATION

    驱动层Binder驱动层->IPC层协议:
    在Binder驱动层的binder_thread_read中被写入.
    在IPCTreadState的waitForResonse和executeCommand中被读取识别.

    BR_TRANSACTION
    BR_REPLY
    BR_TRANSACTION_COMPLETE
    BR_SPAWN_LOOPER
    BR_DEAD_BINDER
````

5.  binder_transaction_data结构
```
    struct binder_transaction_data {
        union {
            __u32    handle;       //binder_ref（即handle）
            binder_uintptr_t ptr;     //Binder_node的内存地址
        } target;  //RPC目标
        binder_uintptr_t    cookie;    //BBinder指针
        __u32        code;        //RPC代码，代表Client与Server双方约定的命令码

        __u32            flags; //标志位，比如TF_ONE_WAY代表异步，即不等待Server端回复
        pid_t        sender_pid;  //发送端进程的pid
        uid_t        sender_euid; //发送端进程的uid
        binder_size_t    data_size;    //data数据的总大小
        binder_size_t    offsets_size; //IPC对象的大小

        union {
            struct {
                binder_uintptr_t    buffer; //数据区起始地址
                binder_uintptr_t    offsets; //数据区IPC对象偏移量
            } ptr;
            __u8    buf[8];
        } data;   //RPC数据
    };
```
6.  binder_transaction.
```
BC_TRANSACTION,BC_REPLY会调用该方法.

reply的过程会找到目标线程.
非reply过程一般找到目标进程.

```