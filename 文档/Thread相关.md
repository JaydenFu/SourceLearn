#   Thread相关

1.  创建线程的3种方式:
```
    *   继承Thread类创建线程类.重写run方法
         new Thread(){
                   @Override
                   public void run() {
                       super.run();
                       //do something
                   }
               }.start();

    *   继承Runnable类.重写run方法
        Runnable runnable = new Runnable() {
                    @Override
                    public void run() {
                        //do something
                    }
                };
        new Thread(runnable).start();

    *   通过FutureTask和Callable
       Callable<Object> callable = new Callable<Object>() {
                   @Override
                   public Object call() throws Exception {
                       //do thing
                       return null;
                   }
               };
       FutureTask<Object> futureTask = new FutureTask<Object>(callable);
       new Thread(futureTask).start();
```

2.  Object.wait  Thread.sleep
    调用wait需要先持有对应object的锁.而sleep不需要.
    wait会释放锁.sleep不会.