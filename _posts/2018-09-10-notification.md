---
layout: post
comments: true
title: "Android学习笔记——Notification和RemoteView"
description: "Android学习笔记——Notification和RemoteView"
category: Android
tags: [Android]
---


这部分主要涉及的知识点：    
（1）RemoteView    
（2）PendingIntent    
（3）Notification    

<!--more-->

任玉刚的《Android开发艺术探索》里有第5章“理解RemoteViews”讲的比较详细。    

### RemoteView    

应用场景：主要是用在通知栏Notification、桌面小部件AppWidgetProvider场景

内部机制：跨进程显示并更新View

支持的View的类型有限：    
Layout -- FrameLayout, LinearLayout, RelativeLayout, GridLayout    
View -- AnalogClock, Button, Chronometer, ImageButton, ImageView, ProgressBar, TextView, ViewFlipper, ListView, GridView, StackView, AdapterViewFlipper, ViewStub    



```
public class RemoteViews implements Parcelable    
private abstract static class Action implements Parcelable    
```

RemoteViews维护了一个List<Action>，使用了反射机制，每一个set方法实际上就是一个对View的操作的Action。

### PendingIntent    

PendingIntent.getActivity(Context context, int requestCode, Intent intent, int flags)

PendingIntent.getService(Context context, int requestCode, Intent intent, int flags)

PendingIntent.getBroadcast(Context context, int requestCode, Intent intent, int flags)

flags参数：

`FLAG_ONE_SHOT`    
`FLAG_NO_CREATE`    
`FLAG_CANCEL_CURRENT`    
`FLAG_UPDATE_CURRENT`    

### Notification    

示例代码：    

    RemoteViews notificationView = new RemoteViews(context.getPackageName(), R.layout.xxx);
    notificationView.setTextViewText(R.id.title, title);
    notificationView.setTextViewText(R.id.content, content);
    notificationView.setImageViewResource(R.id.icon, icon);
      
    PendingIntent contentIntent = PendingIntent.getActivity(context, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT);

    Notification notification = new NotificationCompat.Builder(context)
      .setCustomContentView(notificationView)
      .setContentIntent(contentIntent)
      ....
      .build();

    NotificationManager nm = (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);
    nm.notify(id, notification);


target 26开始，通知栏部分涉及到NotificationChannel概念，即Notification的显示需要先创建NotificationChannel，通过NotificationChannel来控制Notification的显示逻辑。

官方文档：    
[https://developer.android.com/reference/android/app/PendingIntent](https://developer.android.com/reference/android/app/PendingIntent)    
[https://developer.android.com/reference/android/widget/RemoteViews](https://developer.android.com/reference/android/widget/RemoteViews)    
[https://developer.android.com/reference/android/app/Notification](https://developer.android.com/reference/android/app/Notification)    
[https://developer.android.com/reference/android/app/NotificationChannel](https://developer.android.com/reference/android/app/NotificationChannel)    
