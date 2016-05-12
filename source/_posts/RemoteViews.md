---
title: RemoteViews
date: 2016-05-11 16:52:32
tags: 源自Android开发艺术探索总结
---

### RemoteViews:
远程View，在其他进程中显示并更新View，在安卓①Notification通知栏，②AppWidgetProvider桌面小部件中都应用到了RemoteViews。

### Notification：
- 普通通知栏：使用系统的样式

	    //发送普通通知
	    public void sendNotification(View v){
	        Intent intent=new Intent(this,Main2.class);
	        PendingIntent pendingIntent=PendingIntent.getActivity(this,0,intent,PendingIntent.FLAG_UPDATE_CURRENT);
	        Notification notification=new Notification.Builder(this)
	                .setContentText("hello,world")
	                .setContentTitle("hello")
	                .setSmallIcon(R.mipmap.ic_launcher)
	                .setTicker("测试")
	                .setContentIntent(pendingIntent)
	                .build();
	        NotificationManager notificationManager= (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
	        notificationManager.notify(1,notification);
	    }

- 自定义通知栏：当使用自定义通知的时候，就使用到了RemoteViews

		  //发送自定义通知
	    public void sendNotification2(View v){
			//通过RemoteViews加载自定义的布局文件
	        RemoteViews remoteViews=new RemoteViews(getPackageName(),R.layout.layout_notification);
	        //设置自定义布局的TextView的字体颜色
	        remoteViews.setTextColor(R.id.text, Color.RED);
	        Intent intent=new Intent(this,Main2.class);
	        PendingIntent pend=PendingIntent.getActivity(this,0,intent,PendingIntent.FLAG_UPDATE_CURRENT);
	        //设置点击事件
	        remoteViews.setOnClickPendingIntent(R.id.image,pend);
	        Notification notification=new Notification.Builder(this)
	                .setContent(remoteViews)
	                .setSmallIcon(R.mipmap.ic_launcher)
	                .setTicker("测试2")
	                .build();
	        NotificationManager notificationManager= (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
	        notificationManager.notify(2,notification);
	    }

### AppWidgetProvider

1. **简介**：用于实现桌面小部件的类，本质就是一个广播。

2. **例子：**[桌面小部件例子](https://github.com/zlmService/RemoteViews)

3. **注意:** 因为它本质是一个广播组件，所以需要声明，
			
   	<receiver android:name=".MyAppWidgetProvider">
        <meta-data
            android:name="android.appwidget.provider"
            android:resource="@xml/appwidget_provider_info">
        </meta-data>
        <intent-filter>
		第一个action用于识别小部件的单击行为
		第二个action作为标识必须存在，否则无法出现在手机小部件列表中。
            <action android:name="com.zlm.action.CLICK"/>
            <action android:name="android.appwidget.action.APPWIDGET_UPDATE"/>
        </intent-filter>
	</receiver>

4. **方法**：
 
	- onEnable：当小部件第一次添加到桌面的时候调用，可添加多次，但只在第一次调用。
	- onUpdate：当每次被添加或者更新的时候都会调用，每个周期小部件都会自动更新一次
	
			<?xml version="1.0" encoding="utf-8"?>
			<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
			android:initialLayout="@layout/widget"
			//初始化布局和大小。
			android:minHeight="84dp"
			android:minWidth="84dp"
			//每个一个周期，就会自动更新一次
			android:updatePeriodMillis="5000" >
			</appwidget-provider>
	- onDeleted：每删除一次就调用一次。
	- onDisable：当最后一个该类型的小部件被删除时调用。
	- onReceive：广播内置方法，用于分发具体的事件给其他方法，内部通过不同的Action来分别调用对应的onEnable、onUpdate等方法。
	
### PendingIntent
- 对比：pending待定，等待，即在将来某个不确定的时间发生，而Intent是立即发生。
- **匹配规则：**当两个PendingIntent它们内部的Intent相同并且requestCode也相同，那么这两个PendingIntent就是相同的。而**Intent的匹配规则**：当两个Intent的ComponentName（组件名称）和intent-filter相同，那么这两个Intent就相同。

	 	Intent intent=new Intent();
        intent.setComponent(new ComponentName(this,Main2.class));
        intent.setComponent(new ComponentName(this,"com.demo.zlm.remoteviewssample.Main2"));
- **FLAG：**
	- **FLAG-ONE-SHOT**：当前描述的PendingIntent只能使用一次，然后自动被cancel，后续有相同的PendingIntent，它们的send方法就会调用失败。对于通知栏来说：后续的通知无法打开
	- **FLAG-NO-CREATE**：PendingIntent不会主动创建，如果PendingIntent之前不存在，那么getActivity返回null，它无法单独使用
	- **FLAG-CANCEL-CURRENT**：如果当前描述的PendingIntent已经存在，那么它们会被cancel，然后系统会创建一个新的PendingIntent。对于通知栏来说：那些被cancel的消息点击后将不会打开。
	- **FLAG-UPDATE-CURRENT**：如果当前描述的PendingIntent已经存在，那么它们会被更新。
- **notify(1,notification)**：
	1. 如果第一个参数一样的话，那么不管PendingIntent是否匹配，后续的通知会把前面的覆盖掉。
	2. 如果第一个参数每次都不同，那么当PendingIntent不匹配的时候（匹配指：intent和requestCode都相同），无论采用哪个标记位，它们之间都不互相干扰；如果PendingIntent匹配，就按照FLAG标记位的规则。
	
### RemoteViews的内部机制：

- 通过Action的概念去支持View的跨进程通信，Action实现Parcelable,每一个Action代表一个View的操作，当我们在应用中每调用一个set方法（比如：remoteViews.setTextColor(R.id.text, Color.RED)RemoteViews内部就添加了一个对应的Action，并加入到一个集合中，内部有一个ArrayList集合专门负责进行存储，RemoteViews的apply方法会去遍历这个集合中所有的Action，并调用它们的apply方法，具体的View更新操作由Action对象的apply来完成的。
- 当我们调用set方法的时候并不会立即更新它们的界面，而必须要通过NotificationManager的notify方法以及AppWidgetManager的updateAppWidget才能更新它们的界面。它们其实是通过RemoteViews的apply和reapply方法进行加载，当第一次加载的时候，只会调用apply，而后续的更新界面时则会调用reapply方法。它们的区别是apply：来加载布局并更新界面，而reapply方法只会更新界面。

### RemoteViews单击事件
- setOnClickPendingIntent：只用于给普通View设置单击事件，不能给集合（ListView，StackView）进行设置。
- 如果要给ListView设置单击事件，通过将setPendingIntentTemplate和setOnClickFillInIntent组合使用才可以。

### 使用场景：
- 如果我们需要一个应用更新另一个应用界面的时候，我们可以使用AIDL，但是面对频繁的更新AIDL会变得比较复杂，如果是只对界面更新，里面的View都是RemoteViews支持的View且没有自定义的View时候，我们可以选择RemoteViews来完成。



