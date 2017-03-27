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

[Job API]: https://developer.android.com/reference/android/app/job/package-summary.html
