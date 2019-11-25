> 总结一下在android 9 系统使用前台服务的坑



### 1. [Android 8.0 通知](https://developer.android.com/about/versions/oreo/android-8.0#notifications) 

[在前台运行服务](https://developer.android.google.cn/guide/components/services#Foreground)

我们知道，创建前台服务，必须在状态栏，为用户提供通知 。

而在android 8.0 通知有了相关的变化，引入了 Notification channels。

从 Android 8.0（API 级别 26）开始，所有通知都必须分到一个渠道，否则通知将不会显示。

#### 1.1  启动服务

```
  Intent startIntent = new Intent(getApplicationContext(), CustomService.class);
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
     startForegroundService(startIntent);
     } else {
         startService(startIntent);
     }
```

#### 1.2  创建通知渠道+推到前台

```
    // 在Service的onCreate中调用
    private void initStartForegroundService() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        
        // 创建 NotificationChannel
            NotificationManager manager = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
            NotificationChannel channel = new NotificationChannel(CHANNEL_ID, "前台服务", NotificationManager.IMPORTANCE_HIGH);
            manager.createNotificationChannel(channel);
            
            // 创建 Notification
            Intent intent = new Intent(this, Main2Activity.class);
            PendingIntent pi = PendingIntent.getActivity(this, 0, intent, 0);
            
            // 注意Builder第二个参数，与NotificationChannel第一个参数要对应
            Notification notification = new Notification.Builder(this, CHANNEL_ID)
                    .setContentTitle("This is content title")
                    .setContentText("This is content text")
                    .setWhen(System.currentTimeMillis())
                    .setSmallIcon(R.mipmap.ic_launcher)
                    .setLargeIcon(BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher))
                    .setContentIntent(pi)
                    .build();
                    
            // 推到前台
            startForeground(1, notification);
        }
    }
```

也可以参考

[Create a notification channel](https://developer.android.com/training/notify-user/channels.html#CreateChannel)



### 2. [Android 9 前台服务](https://developer.android.com/about/versions/pie/android-9.0-changes-28?hl=zh-CN#fg-svc)

> 这个坑是Android 9 带来的，但是解决起来比较简单

如果应用以 Android 9 或更高版本为目标平台并使用前台服务，则必须请求 [`FOREGROUND_SERVICE`](https://developer.android.com/reference/android/Manifest.permission.html?hl=zh-CN#FOREGROUND_SERVICE) 权限。这是[普通权限](https://developer.android.com/guide/topics/permissions/normal-permissions.html?hl=zh-CN)，因此，系统会自动为请求权限的应用授予此权限。

如果以 Android 9 或更高版本为目标平台的应用尝试创建前台服务且未请求 [`FOREGROUND_SERVICE`](https://developer.android.com/reference/android/Manifest.permission.html?hl=zh-CN#FOREGROUND_SERVICE)，则系统会抛出 [`SecurityException`](https://developer.android.com/reference/java/lang/SecurityException.html?hl=zh-CN)。

在代码中，也就是说 AndroidManifest.xml 中声明下面的权限

```
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
```







