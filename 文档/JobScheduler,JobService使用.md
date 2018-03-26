#   JobScheduler,JobService使用
JobScheduler,JobService是API21以上才能使用的任务调度系统.
它的作用是用来调度应用在一定时间之后或者一定条件之下才执行相应的操作.让应用延后批处理一些不重要的操作.
```
基本使用:
        JobScheduler jobscheduler = (JobScheduler) context.getSystemService(Context.JOB_SCHEDULER_SERVICE);
        PersistableBundle bundle = new PersistableBundle();
        bundle.putString("task","task1");
        bundle.putLong("taskId",1000);
        JobInfo jobinfo = new JobInfo
                .Builder(1,new ComponentName(context,MyJobService.class))
                .setMinimumLatency(10000)//设置最小延迟时间
                .setOverrideDeadline(15000)//设置最大延迟时间.当到达这个时间后,即使其他条件没达到,也会执行任务.
                .setRequiresCharging(false)//设置是否需要在充电期间,才执行任务.默认false
//                .setPeriodic(5000)//定期执行.不能和setMinimumLatency或者setOverrideDeadline同时设置,否则会报错
                .setRequiresDeviceIdle(false)//是否要在设备idle模式下,才执行任务.默认false.
                .setRequiredNetworkType(JobInfo.NETWORK_TYPE_NONE)//设置网络状况.JobInfo.NETWORK_TYPE_NONE:不管是否有网都执行.JobInfo.NETWORK_TYPE_ANY:任何一种网络.都执行.JobInfo.NETWORK_TYPE_UNMETERED:非蜂窝网络执行.
                .setPersisted(false)//在手机重启后,是否还需要执行任务.需要权限android.Manifest.permission#RECEIVE_BOOT_COMPLETED.
                .setExtras(bundle)//设置数据
                .build();
        Log.d("fxj","schedule:"+ SystemClock.uptimeMillis());
        jobscheduler.schedule(jobinfo);
//        jobscheduler.cancel(1);//取消任务


    public class MyJobService extends JobService {

        @Override
        public boolean onStartJob(final JobParameters params) {
            Log.d("fxj","onStartJob: "+SystemClock.uptimeMillis());
            int jobId = params.getJobId();
            final String task = params.getExtras().getString("task");
            final long taskId = params.getExtras().getLong("taskId");
            Log.d("fxj","jobId: "+jobId+",task: "+task+", taskId: "+taskId);
            new Thread(new Runnable() {
                @Override
                public void run() {
                    Log.d("fxj","开始处理任务:taskId:"+taskId+",task: "+task);
                    SystemClock.sleep(10000);
                    Log.d("fxj","处理任务完成:taskId:"+taskId+",task: "+task);
                    jobFinished(params,false);//通过jobFinished告诉jobManager.任务已经完成
                }
            }).start();
    //        startActivity(new Intent(getBaseContext(), Main2Activity.class).setFlags(Intent.FLAG_ACTIVITY_NEW_TASK));
            return true;
        }

        @Override
        public boolean onStopJob(JobParameters params) {
            //停止,不是结束。jobFinished不会直接触发onStopJob
            //必须在“onStartJob之后，jobFinished之前”取消任务，才会在jobFinished之后触发onStopJob
            Log.d("fxj","onStopJob: " +SystemClock.uptimeMillis());
            return true;
        }
    }

注意事项:
    *   需要在manifest的service中声明android.permission.BIND_JOB_SERVICE权限

```