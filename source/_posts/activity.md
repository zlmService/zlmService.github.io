---
title: Activity
date: 2016-05-07 22:02:34
tags: Activity
---
	exporte="true"表示可以被其他应用启动
	Task：相当于存放Activity的栈
### 四种启动模式：

1.Standard
	
	：每次启动Activity的时候，这种模式下每次创建新的Activity不会创建新的Task，
	只会将新的Activity添加到原有的Task中
	A-B-A-B-C 依次打开 就会创建5个Activity

2.SingleTop  使用场景如新闻类或者阅读类App的内容页面。
	
	：与第一个基本相似，但是如果要添加的Activity已经在栈顶 就不会创建新的，而是复用
	A-A-A-A-A 打开5个Activity 如果名字是一样的 并且它是在栈顶， 就只会创建一个A的实例
	A-B-A-B-B 打开5个Activity 只会创建四个Activity 因为最后一个B是在栈顶 所以就直接调用不会在创建一个新的Activity

3.SingleTask 如浏览器的主界面
	
	：同一个栈中 只能有一个Activity的实例
	⑴如果要启动的不存在，系统将创建新的 并且加到栈顶
	⑵如果要启动的Activity已经在栈顶，就不会创建新的，而是复用  和SingleTop一样
	⑶如果存在，但是没有位于栈顶，系统就把位于该Activity上面的所有Activity移除，从而使目标Activity转入栈顶
	A-B-C-A 在创建一个A的时候  如果有这个Activity，就不会创建 只会把这个Activity所在的栈上面的所有Activity给清除出去，也就是 把B-C给清楚出去 从而显示这个 A
	
4.SingleInstance 如打开打电话 发短信界面
	
	：无论从那个Task中启动目标Activity，只会创建一个目标实例，并使用一个全新的Task栈来装载该Activity实例  类似于单例模式 每一个栈中 只能存放一个Activity
	⑴如果不存在，就创建一个新的Task栈，创建一个新的Activity 并加入到栈顶
	⑵如果目标Activity已经存在，无论它位于哪个程序中，无论它位于哪个Task中，系统都会把该Activity所在的Task转到前台，从而使用该Activity展示出来。
	A-B-C 就会创建三个Task栈 来状态对应的Activity
---
### 生命周期：

	1.onCreate()
	创建Activity的时候调用 只会调用一次
	2.onStart()
	启动Activity的时候调用 
	3.onResume()
	获取焦点的时候调用，也就是与用户交互的时候
	4.onPause()
	失去焦点的时候调用
	5.onStop()
	暂停的时候调用
	6.onDestroy()
	销毁Activity的时候调用 只会调用一次
	7.onRestart()	
	重新运行的Activity的时候调用

	- onCreate和onDestroy代表Activty的创建和销毁
	- onStart和onStop代表设备屏幕的点亮和熄灭 (按下home键)
	- onResume和onPause代表Activity是否在前台（来短信）
	启动一个Activity onCreate()-->onStart()-->onResume()
	销毁一个Activity onPause()-->onStop()-->onDestroy()
	失去焦点后，又重新获取焦点	onPause（）-->onStop()-->onRestart()-->onStart()-->onResume()


### 四种状态：

1.活动状态：Activity处于界面最顶端，获取焦点

2.暂停状态：失去焦点，但是用户可见，

3.停止状态：被完全遮挡，但是保留所有的状态和成员信息 相当于按下home见，程序在后台运行

4.销毁状态：被销毁了 ，


### 当Activity异常终止的时候

	系统会调用onSaveInstanceState来保存Activity的状态
	可以在里面存入一些需要保存的数据，当Activity被重新创建的时候，可以用onRestoreInstanceState来恢复之前保存的数据。

### 当Activity进行屏幕翻转的时候 

	如果不想重新创建Activity就把
	android:configChanges="orientation


### Activity之间传值

	 startActivityForResult(intent, 1); //打开第二个Activity  请求码是 1；
	setResult(2,intent);// 返回码 是2   intent里面存储的是一个传递过来的数据
	然后在第一个Activity中 
	重写onActivityResult（int requestCode, int resultCode, Intent data）方法
	 if(requestCode==1&&resultCode==2)判断（第一次是请求码，第二个是返回码）
	如果相等 就调用 data.getStringExtra(“key”);来获取数据

### Intent的属性

动作(Action),数据(Data),分类(Category),类型(Type),组件(Compent)以及扩展信(Extra)。其中最常用的是Action属性和Data属性。

- 原则上一个Intent不应该既是显示调用又是隐式调用，如果二者同时存在的话，以显示调用为主。 
- 需要同时匹配过滤列表中的action，category，data信息，否则匹配失败
- 一个列表中可以有多个action，category，data，只有一个Intent同时匹配action类别，category类别，data类别才算完全匹配，只有完全匹配，才能成功启动Activity，一个Activity可以有多个Intent-filter，一个Intent只要匹配上任何一组Intent-filter就能成功启动该Activity。
	
	

	