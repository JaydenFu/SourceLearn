#   IO多路复用--epoll机制

参考学习:

*   http://www.cnblogs.com/Anker/p/3265058.html
*   http://www.cnblogs.com/Anker/archive/2013/08/17/3263780.html
*   http://gityuan.com/2015/12/06/linux_epoll/


该篇文章是学习Handler消息机制Native层的基础.

epoll使用一个文件描述符管理多个文件描述符.将用户指定监听的文件描述符的事件存放在内核的一个事件表中

epoll操作过程涉及3个接口:
1.  int epoll_create(int size)
创建一个epoll句柄,size表示可以监听的文件描述符的最大个数.这个只是初始值.是可以动态变化的.但是epoll本身也是会占用一个fd值,在/proc/进程id/fd/下可以查看到该
fd.所以,在使用完epoll后,需要调用close关闭,防止fd耗尽.

2.  int epoll_ctl(int epollFd,int op,int fd,struct epoll_evnet *event)
该函数表示epoll的事件注册
epollFd:第一步epoll_create()过程返回的值
op代表操作类型:
    *   EPOLL_CTL_ADD:  注册新的fd到epollFd中.
    *   EPOLL_CTL_MOD:  修改已经注册的fd的监听事件
    *   EPOLL_CTL_DEL:  从epollFd中删除一个fd.
fd: 表示要监听的文件描述符.
event:表示要监听的事件.epoll_event是个结构体:
```
struct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};
其中 events有以下7个值:
    *   EPOLLIN:    表示对应的文件描述符可以读（包括对端SOCKET正常关闭)
    *   EPOLLOUT:   表示对应的文件描述符可以写
    *   EPOLLPRI:   表示对应的文件描述符有紧急数据可读
    *   EPOLLERR:   表示对应的文件描述符发生错误
    *   EPOLLHUP:   表示对应的文件描述符被挂断
    *   EPOLLET:    表示EPOLL设为边缘触发.
    *   EPOLLONESHOT:表示只监听一次事件,当监听万这次事件之后,如果还需要继续监听的话,需要再次添加.
```

3.  int epoll_wait(int epollFd, struct epoll_event * events, int maxevents, int timeout)
等待事件的产生,events用来接收内核传递出来的事件.maxevents表示events的大小.maxevents不能大于创建epoll时的size.timeout超时时间.
(毫秒,0会立即返回,-1将不确定,也有说法说是永久阻塞),该函数返回需要处理的事件数目,如返回0表示已超时.

