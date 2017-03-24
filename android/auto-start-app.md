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

## How to prevent app auto start
