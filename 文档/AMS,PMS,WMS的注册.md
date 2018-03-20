#   ActivityManagerService,PackageManagerService,WindowManagerService的注册.

系统服务都是在系统进程中被注册的.

1.  SystemServer的main方法: 该方法会从zygote调用.
```
    public static void main(String[] args) {
        new SystemServer().run();
    }
```
2.  SystemServer的run方法:
```
 private void run() {
        try {
            ......
            // Initialize the system context.
            createSystemContext();

            // Create the system service manager.
            mSystemServiceManager = new SystemServiceManager(mSystemContext);
            LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
        } finally {
            ...
        }

        // Start services.
        try {
            //启动一些核心的系统服务
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
        } catch (Throwable ex) {
            throw ex;
        } finally {
        ....
        }

        // Loop forever.
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
3.  AMS,PMS在startBootstrapServices中被启动.
```
 private void startBootstrapServices() {

        Installer installer = mSystemServiceManager.startService(Installer.class);


        //1.创建ActivityManagerService.
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
        ......

        //2.创建PackageManagerService
        //在main方法内部会通过addService将PMS注册到ServiecManager
        //ServiceManager.addService("package", m);

        mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        mFirstBoot = mPackageManagerService.isFirstBoot();
        mPackageManager = mSystemContext.getPackageManager();

        ......

        //在该方法内部会通过addService将AMS注册到ServiceManager.
        //ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);

        mActivityManagerService.setSystemProcess();
        .....
    }
```
4.  WMS在startOtherService中被创建以及注册到ServiceManager.
```
    wm = WindowManagerService.main(context, inputManager,
            mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
            !mFirstBoot, mOnlyCore);
    ServiceManager.addService(Context.WINDOW_SERVICE, wm);
```