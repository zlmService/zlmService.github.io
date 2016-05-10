---
title: Service
date: 2016-05-07 23:19:28
tags: Android
---
	IPC--：进程内部通信
	AIDL：跨进程通信
### 两种启动方式：

startService()

	onCreate()-->onStartCommand()-->onDestroy()
	这种方式和调用者没有任何关系，只要是没有执行onStopService（）服务就会一直运行
	
bindSerVice()

	onCreate()-->onBind()-->onUnbind()-->onDestroy()
	这种方式 是和调用者绑定在一块，当调用者退出，他也会调用onDestroy（）
	如果多次调用bindService（）还是只会调用一次onBind（）但是如调用多次unBindService（）就会报错


通常会 Start-->Bind-->unbind-->stop 这样即使Service在unbind之后还能保证服务存在	


#### 防止SerVice被强制停止
可能会创建2个Service 然后他们互相监听，如果监听到 其中一个停止掉了，就启动另一个Service

可以通过 onStartCommand 设置Frag =START_REDELIVER_INTENT 就是如果Service被杀死了，系统会重新创建Service并且调用onStartCommand（）所有被挂起的intent都会一次传递


### IntentService
	1.IntentService 内部只有一个工作线程来完成耗时操作，只需实现onhandleIntent
	2.完成后会自动停止服务
	3.如果同时执行多个任务，会以队列的方式，一次执行
	4.通常使用该类来完成本App内部中的一些耗时工作。

