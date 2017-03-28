# Auto Start App

This article is about how to make your app auto start when android device boot, and how to prevent this if you are a system developer who cares the performance.

## How to make your app auto start

### Use broadcast receiver

Usually you can use the following actions with your broadcast receiver to make your app auto start after the device boot.

- android.intent.action.BOOT_COMPLETED
- android.intent.action.USER_PRESENT
- android.net.conn.CONNECTIVITY_CHANGE

> NOTE: For action `android.intent.action.BOOT_COMPLETE`, you shoud hold the `android.permission.RECEIVE_BOOT_COMPLETED` permission.

> NOTE: For action `android.net.conn.CONNECTIVITY_CHANGE`, apps targeting Android 7.0 (API level 24) and higher do not receive this broadcast if they declare the broadcast receiver in their manifest. Apps will still receive broadcasts if they register their BroadcastReceiver with Context.registerReceiver() and that context is still valid. 

Sample Code:
```
<receiver android:name="com.demo.BootReceiver">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>

    <intent-filter>
        <action android:name="android.net.conn.CONNECTIVITY_CHANGE" />
    </intent-filter>

    <intent-filter>
        <action android:name="android.intent.action.USER_PRESENT" />
    </intent-filter>
</receiver>
```

### Use Persistable Job

Since Google add [Job][Job API] related API from API level 21, aka Lollipop 5.0, Our developer then have a efficient way to schedule future work. But it also give opportunities to make malicious apps auto start with the system. Once you start the app the first time, it will schedule a periodic persistable job to keep itself alive even you reboot your device.

How to achieve this purpose? First you need a class extends JobService. You should override `onStartJob(JobParameters)` and `onStopJob(JobParameters)`.

```
public class PeriodicJobService extends JobService {
   @Override
    public boolean onStartJob(JobParameters params) {
        // TODO: do something here
        return true;
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        // TODO: do something here
        return true;
    }

    // Helper method
    public static void startRepeatJob(Context context) {
        ComponentName serviceName = new ComponentName(context, PeriodicJobService.class);
        JobInfo jobInfo = new JobInfo.Builder(jobId, serviceName)
                .setPersisted(true)
                .setPeriodic(INTERVAL_TIME) // 1 min
                .build();
        JobScheduler s = (JobScheduler) context.getSystemService(Context.JOB_SCHEDULER_SERVICE);
        int result = s.schedule(jobInfo);
        Log.d(TAG, "schedule job " + jobId + " " + ((result == JobScheduler.RESULT_SUCCESS) ? "ok" : "failed"));
    }
}
```

A JobService must declare permission `android.permission.BIND_JOB_SERVICE` in file AndroidManifest.xml.

```
<service android:name=".PeriodicJobService"
    android:permission="android.permission.BIND_JOB_SERVICE"
    android:enabled="true"
    android:exported="true">
</service>
```

Schedule a persistable job when user manually start your app. A persistable job will also work even you reboot your device.

```
public class MainActivity extends AppCompatActivity {
    public static final String TAG = "DebugJobScheduler";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        PeriodicJobService.startRepeatJob(this);
    }
}
```

> Note: If you want your job interval more precise, don't rely on `setPeriodic(long)` method, it will determine by the system, it only guarantee your job fire no more than once within the setting interval. 
> You shoud reschedule your job when the previous job finished by just call `jobFinished(JobParameters, false)` and pass `false` to the second parameter.

```
public class PeriodicJobService extends JobService {
   @Override
    public boolean onStartJob(JobParameters params) {
        new AsyncTask<JobParameters, Void, JobParameters[]>() {
            @Override
            protected JobParameters[] doInBackground(JobParameters... jobParameters) {
                return jobParameters;
            }

            @Override
            protected void onPostExecute(JobParameters[] jobParameters) {
                for (JobParameters params : jobParameters) {
                    jobFinished(params, false);
                }
                startRepeatJob(PeriodicJobService.this);
            }
        }.execute(params);
        return true;
    }

    ......
}
```


## How to prevent app auto start

You may consider to prevent app auto start behavior if you are a platform developer who cares the performance of the system. According to the way that the app used to auto start, you can take steps to prevent it process be bring up.

### Prevent start process by Broadcast

Just stop start process when deliver broadcast to receives.

```
// frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java

final void processNextBroadcast(boolean fromMsg) {
    ......

    // TODO: prevent process start up here if needed
    if (shouldPreventStartProcess(info, r)) {
        return;
    }

    if ((r.curApp=mService.startProcessLocked(targetProcess,
                    info.activityInfo.applicationInfo, true,
                    r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
                    "broadcast", r.curComponent,
                    (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, false, false))
                            == null) {
                ......
            }

            mPendingBroadcast = r;
            mPendingBroadcastRecvIndex = recIdx;
        }
}

private shouldPreventStartProcess(ResolveInfo info, BroadcastRecord r) {
    boolean stop = false;

    // TODO: make your decision here

    // the following lines is important to make broadcast work properly
    if (stop) {
        r.queue.logBroadcastReceiverDiscardLocked(r);
        r.queue.finishReceiverLocked(r, r.resultCode, r.resultData, r.resultExtras,
                r.resultAbort, true);
        r.queue.scheduleBroadcastsLocked();
        r.state = BroadcastRecord.IDLE;
    }

    return stop;
}
```

### Prevent start process by JobServiceContext

Start process for bringing up service has two major ways, you can call `startService(...)` or `bindService(...)`. Our JobService use the later. The difficulty is how to distinguish this bindService call if from JobServiceContext but not normal usages.

All service start by JobServiceContext should extend from `JobService`, this give us a chance to stop that process being startup in `ActiveServices.bringUpServiceLocked(...)`. But here we choose another way to accomplish our goal.

```
// frameworks/base/services/core/java/com/android/server/job/JobServiceContext.java

boolean executeRunnableJob(JobStatus job) {
    synchronized(mLock) {
        ......
        final Intent intent = new Intent().setComponent(job.getServiceComponent());
                boolean binding = mContext.bindServiceAsUser(intent, this,
                        Context.BIND_AUTO_CREATE | Context.BIND_NOT_FOREGROUND,
                        new UserHandle(job.getUserId()));
        // TODO: we add a extra indicator to the intent, and we use it in ActiveService
        intent.putExtra("extra_from_jobservice", true);
        ......
    }
}

// frameworks/base/services/core/java/com/android/server/am/ActiveServices.java

int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
        String resolvedType, IServiceConnection connection, int flags,
        String callingPackage, int userId) throws TransactionTooLargeException {
    ......
    if ((flags&Context.BIND_AUTO_CREATE) != 0) {
        s.lastActivity = SystemClock.uptimeMillis();

        // TODO: get our indicator here, and pass to bringUpServiceLocked(...)
        boolean fromJobService = service.getBooleanExtra("extra_from_jobservice", false);
        if (bringUpServiceLocked(s, service.getFlags(), callerFg, false, fromJobService) != null) {
            return 0;
        }
    }
    ......
}

private final String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
         boolean whileRestarting) throws TransactionTooLargeException {
    return bringUpServiceLocked(r, intentFlags, execInFg, whileRestarting, false);
}

private final String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
        boolean whileRestarting, boolean fromJobService) throws TransactionTooLargeException {
    ......
    // Not running -- get it started, and enqueue this service record
    // to be executed when the app comes up.
    if (app == null) {
        // TODO: prevent process startup here
        if (fromJobService) {
            String msg = "Unable to launch app "
                    + r.appInfo.packageName + "/"
                    + r.appInfo.uid + " for service "
                    + r.intent.getIntent() + ": from job service";
            Slog.w("chenwei_as", msg);
            bringDownServiceLocked(r);
            return msg;
        }
        if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                "service", r.name, false, isolated, false)) == null) {
            ......
        }
        ......
    }
    ......
}
```

> Note: The implementation of the aforementioned ways to prevent start process is ugly, but the principle is correct I think. Our goal is to stop process startup cause by receivers or services.

[Job API]: https://developer.android.com/reference/android/app/job/package-summary.html