#   Activity的启动流程总结

首先总结一下在Activity整个启动流程中涉及到的一些重要类：
*   Instrumentation:主要负责监控应用程序和系统之间的交互
*   ActivityManagerService:主要负责启动和调度应用程序组件，管理任务栈，通知ANR等
*   ActivityStarter：该类控主要制如何打开一个Activity。对于Intent和flags的处理都是在该类中进行。
*   ActivityRecord/ActivityClientRecord:都是用来描述一个启动的Activity。ActivityRecord在ActivityManagerService中使用，
    ActivityClientRecord在ActivityManagerService中使用。
*   TaskRecord: Activity任务栈，内部维护一个ArrayList<ActivityRecord>。
*   ActivityStack:  该类主要负责任务栈的状态及管理操作,内部维护一个ArrayList<TaskRecord>。
*   ActivityStackSupervisor:内部持有一个ActivityStack.ActivityStack内部也持有ActivityStackSuperVisor.可以当成是ActivityStack的辅助类。
*   ActivityThread: 该类描述一个应用程序进程，系统每启动一个进程都会加载一个ActivityThread实例。对于Activity,Service的生命周期方法都是在该类中调度。
*   ApplicationThread:ActivityThread的一个内部类，是一个Binder本地对象。AMS主要就是通过它来让应用程序执行相应的操作。
*   LoadedApk:  用来描述一个已加载的APK文件。应用程序进程在启动一个Activity时，需要先将它所属的APK文件加载进来，以便访问里面的资源。


重要步骤：
1.  ActivityStarter.startActivityMayWait:
    1.  解析Intent，通过PackageManager找到目标Activity.

2.  ActivityStarter.startActivityLocked:
    1.  创建一个新的ActivityRecord

3.  ActivityStarter.startActivityUnchecked:
    1.  根据传入的变量重新初始化ActivityStarter的成员变量.
    2.  根据启动模式，Intent的flag参数计算Activity的任务栈

4.  ActivityStack.startActivityLocked,ActivityStack.resumeTopActivityUnchecked
    1.  基本都是任务栈相关的操作

5.  ActivityStack.startPausingLocked
    1.  通过ApplicationThreadProxy调度，让当前显示的Activity执行onPause。

6.  ActivityStackSupervisor.startSpecificActivityLocked
    1.  如果进程进程启动，则执行realStartActivityLocked方法启动Activity
    2.  如果进程还没有启动，则通过AMS.startProcessLocked方法启动进程

7.  ActivityStackSupervisor.realStartActivityLocked
    1.  通过ApplicationThreadProxy调度,执行ActivityThread的handLaunchActivity方法，完成Activity的onCreate,onStart,onResume。
