---
title: Handler
date: 2016-05-07 23:05:11
tags:
---


	创建Handler的时候就会 在里面 得到当前线程的保存的Looper实例，
	而Looper里面又有一个MessageQueue对象，每一个线程只能有一个Looper实例 
	每一个Looper里面也只有一个MessageQueue实例
	获取到以后，handler就和MessageQueue关联上了， 然后当它执行 sendMessage的时候
	就会把消息传送到MessageQueue里面去  Looper 通过loop方法负责从MessageQueue里面去取Message   
	把取出来的Message 在传递个Handler ，Handler接收并且负责处理


#### Looper ：主要是 prepare和loop方法
	prepare（）方法
		主要是 在当前线程中保存一个Looper实例，然后在该实例中保存一个MessageQueue实例， 
		这个方法只会调用一次 所有Looper 在一个线程中只会存在一个
	loop()方法
		会让当前线程进入一个无限循环，从MessageQueue实例中不断读取消息，然后把消息传递给Handler

#### Handler：主要负责发送消息到MessageQueue中， 然后接收从Looper传递过来的信息
	handler的构造函数会获取当前线程 保存的Looper对象，
	从而根据这个looper对象获取他里面保存的MessageQueue实例，
	这样就保证了handler和MessageQueue关联上了

#### MessageQueue：负责存储Handler发送过来的信息 底部采用队列 先进先出

---
##### Message.obtain()

	获取Message的实例时候 推荐使用 Message.obtain()
	因为Message内部会有一个Message池 用于Message的复用，避免了new Message的时候重新分配内存。

##### handler用法：
	
	handler不光可以用来更新UI  还可以在子线程中创建一个Handler实例，
	然后使用这个handler实例去任何其他线程中发送消息，
	最终处理的代码都会在你创建的这个Handler实例都线程中运行

##### Handler的post方法创建的线程和UI线程有什么关系

	其实这个Runnable并没有创建一个线程，而是发送了一条消息，只不过是把这个Runable对象当成callback（回调）属性，赋值给了Message  然后还是 把消息添加到MessageQueue中去，和上面的一样。

---
### 内存泄漏问题
- 原因：因为Handler是一个内部类，它会获得外部类Activity的引用，如果Activity被用户关掉了，但是Handler有一个延时发送的信息 还没有执行，所以表面是关闭了Activity 释放了资源，实际上其实，Handler还在调用着它的引用，这样就导致了内存泄漏，

- 解决：使用静态内部类，然后在该类里使用弱引用来指向所在的Activity。
- 如果Activity关闭的时候，也要remove掉MessageQueue中的Message消息和Runnable

		@Override  
		 public void onDestroy() {  
	     mHandler.removeMessages(MESSAGE_1);  
	     mHandler.removeMessages(MESSAGE_2);  
	
	     mHandler.removeCallbacks(mRunnable);  
	
	     // ... ...  
	 	}
		
		或者是
		@Override  
		public void onDestroy() {  
		    //  If null, all callbacks and messages will be removed.  
		    mHandler.removeCallbacksAndMessages(null);  
		}


