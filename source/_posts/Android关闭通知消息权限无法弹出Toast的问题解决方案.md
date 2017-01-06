---
title: Android关闭通知消息权限无法弹出Toast的解决方案
date: 2016-03-27 19:47:28
tags: [Android]
---


# 背景

在之前一段时间里经常有用户反馈无法看到薄荷app弹出的消息提示，导致用户对一些使用场景感到很困惑。猜测可能是由于用户关闭了薄荷app的通知消息的权限导致的问题，测试后发现果真如此。

# 原因

跟踪Toast的源代码，``make``方法省略，做了一些初始化的工作，``show``方法

```java
public void show() {
　　if (mNextView == null) {
　　　　throw new RuntimeException("setView must have been called");
　　}

　　INotificationManager service = getService();
　　String pkg = mContext.getPackageName();
　　TN tn = mTN;
　　tn.mNextView = mNextView;

　　try {
　　　　service.enqueueToast(pkg, tn, mDuration);
　　} catch (RemoteException e) {
　　　　// Empty
　　}
}

static private INotificationManager getService() {
    if (sService != null) {
      return sService;
    }
    sService = INotificationManager.Stub.asInterface(ServiceManager.getService("notification"));
     return sService;
}
```

熟悉``binder``通讯的同学应该都知道，其实调用了``NotificationManagerService.service.enqueueToast``方法进入toast队列，进行相应的逻辑处理后回调给Toast中的``TN``，``TN``其实就是一个``aidl``的``stub``实现，相当于``Client``端，用来接收``Service``端发来的消息。看下``TN``中的show方法

```java
public void handleShow() { 
			... 
            mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);  
            mWM.removeView(mView);  
            mWM.addView(mView, mParams);  
            trySendAccessibilityEvent();  
            ...
            }  
        }  
```

通过``WinwodManager``添加一个``view``显示提示消息。
总结来说就是toast的显示过程通过IPC通讯由``NotificationManagerService``维护一个toast队列，然后通知给Toast中的客户端``TN``调用``WindowManager``添加view。
那么，如果关闭通知栏消息权限，会影响``NotificationManagerService``队列的逻辑处理过程，导致不能通知``TN``显示出视图。

# 解决

通过上面的分析，我们可以绕过``NotificationManagerService``，我们自己维护一个toast队列，处理相关的逻辑，进行显示，定时取消。关键代码

```java
    private static void activeQueue() {
        BooheeToast toast = mQueue.peek();
        if (toast == null) {
            mAtomicInteger.decrementAndGet();
        } else {
            mHanlder.post(toast.mShow);
            mHanlder.postDelayed(toast.mHide, toast.mDuration);
            mHanlder.postDelayed(mActivite, toast.mDuration);
        }
    }
```
``mQueue``维护了``Toast``的队列，队列采用``FIFO``调度，每次调用``show()``方法触发``activeQueue()``方法，从队列中取出toast进行显示，然后定时取消。

# 总结

Google把Toast视为系统级别的消息通知，其实是不合理的，在app前台可见的情况下，有些关键信息需要提供给用户，如果关闭了消息通知权限，那么用户就会看不到这些关键的提示就会很困惑，直接影响用户体验，并且在Android5.0之后在设置中就可以关闭app的消息权限，Toast作为一个系统级别的消息提示设计者真的挺缺心眼的。虽然MD中有SnackBar作为消息提示，其原理就是在当前界面找到根视图，在根视图``addView()``，达到弹出提示的目的，但是请原谅我的审美不够高，SnackBar真的很丑，并且最近Google又支持BottomSheet，两个叠加到一起，我的天，画面太美，我不敢看。还有就是必须要拿到当前的content，对于用Application的context弹出toast的方案来说适配起来修改比较麻烦。很明显，Toast就应该是app级的消息提示，我们的解决方案也正是这么做的。

# 参考

* 《Android开发艺术探索》第八章 理解Window和WindowManager
* [解决小米MIUI系统上后台应用没法弹Toast的问题](http://caizhitao.com/2016/02/09/android-toast-compat/)
* [Toast](http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/java/android/widget/Toast.java)
* [NotificationManagerService](http://androidxref.com/6.0.1_r10/xref/frameworks/base/services/core/java/com/android/server/notification/NotificationManagerService.java)
