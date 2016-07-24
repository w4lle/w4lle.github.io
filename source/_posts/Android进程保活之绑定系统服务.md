---
title: Android进程保活之绑定系统服务
date: 2016-07-24 20:25:32
tags: [Android, 进程保活]
---

# 进程保活

有些业务需要service在后台持续的运行，所以就要有后台保活机制，包括lowMemory防杀和自启。

## 防杀机制

基本就是提高进程优先级，保证在低内存时进程不被有限杀死，常用的方法就是利用系统bug提高进程优先级，灰色保活手段。

## 后台自启

大概包括

 - Receiver拉起
 - AlarmManager拉起
 - 双进程互相守护
 - 利用推送SDK拉起进程
 
以上说的这几种是常用的方法，对于原生和没有深度定制的ROM有一定作用，但是对于像小米、魅族等这类深度定制的系统来说效果不是很好。

## 绑定系统服务

系统提供一些系统级的Service，比如AccessibilityService辅助服务、NotificationListenerService用于监听通知消息。这篇文章主要讲下怎样利用NotificationListenerService用于监听通知消息实现进程保活。

首先，如果想使用系统service，必须要用户手动点开权限。我们可以添加一个设置的入口直接跳到系统设置通知权限的界面，显示提示用户需要开启权限。

```java
    public static void goNLPermission(Context context) {
        try {
            Intent intent = new Intent("android.settings.ACTION_NOTIFICATION_LISTENER_SETTINGS");
            context.startActivity(intent);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

界面就不截图了，然后我们app中要有一个权限的状态检查用于提示用户是否开启了通知显示权。

```java
/**
 * 检查通知使用权
 */
public static boolean checkNotificationPermission(Context context) {
    String pkg = context.getPackageName();
    String flat = Settings.Secure.getString(context.getContentResolver(), "enabled_notification_listeners");
    boolean enabled = flat != null && flat.contains(pkg);
    return enabled;
}
```

注册，声明运行在com.package.pedometer进程中

```java
<service
    android:name=".service.MyListenerService"
    android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE"
    android:process=":pedometer">
    <intent-filter>
        <action android:name="android.service.notification.NotificationListenerService"/>
    </intent-filter>
</service>
```

然后需要继承NotificationListenerService实现两个方法

```java
/**
 * 监听系统通知，需要用户手动开启权限，那么该进程可以不死
 */
@TargetApi(Build.VERSION_CODES.JELLY_BEAN_MR2)
public class MyListenerService extends NotificationListenerService {
    @Override
    public void onNotificationPosted(StatusBarNotification sbn) {
    }

    @Override
    public void onNotificationRemoved(StatusBarNotification sbn) {
    }
}
```

方法中什么都不用实现，MyListenerService所在的进程就拥有了跟NotificationListenerService一样的权限。可以试下手动强制停止进程会发现怎么都结束不掉，强制停止的按钮一直可以点击。可以使用
```
adb shell ps | grep ***
```
命令查看进程是否存活。

```java
cat /proc/12869/oom_adj
0
```
查看进程的oom_adj的值为0，那么我们知道在灰色保活手段上基本就是提高oom_adj的值，越小越不容易被杀死，这种方法一步到位。

那么怎么应用到我们自己的service中呢？很简单，将service跟MyListenerService运行在同一个进程就可以了。

```java
<service
    android:name=".service.StepCounterService"
    android:exported="true"
    android:process=":pedometer">
</service>
```

这样基本就可以保证service所在的进程不被杀死了。当然，如果ROM厂商在系统级别拦截掉了，这种方法也会无效了。

# 总结

如果要用这种方法进程保活，那么你的app肯定是有显示通知栏的通知，不然用户谁这么傻去给你开这么权限呢。其次，显示通知栏消息就是前台进程了，用户始终可以看到，再配合service自启的几种方法，基本就可以保证我们的进程不死了。当然，这种方法也不能说绝对管用，在深度定制面前，一切都是渣渣。验证下来，魅族基本都好用，小米4c不太好用，小米其他机型没有测试。

# 参考

[论Android应用进程长存的可行性][1]
[关于 Android 进程保活，你所需要知道的一切][2]


  [1]: http://blog.csdn.net/aigestudio/article/details/51348408
  [2]: http://www.jianshu.com/p/63aafe3c12af#