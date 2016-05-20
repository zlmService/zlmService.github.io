---
title: Window和WindowManager
date: 2016-05-17 12:20:26
tags: 源自Android开发艺术探索总结
---
### Window
Window表示一个窗口的概念，创建一个Window通过WindowManager来完成。WindowManager是外界访问Window的入口，Window的具体试下位于WindowManagerService中。Android所有的视图都是通过Window来呈现的，不管是Activity、Dialog还是Toast，他们的视图都是附加在Window上的。因此Window实际是View的直接管理者。

-  **使用WindowManage添加一个Window**：将一个Button添加到屏幕（100，300）的位置上。
	
	    public void onButtonClick(View v) {
	        if (v == mCreateWindowButton) {
	            mFloatingButton = new Button(this);
	            mFloatingButton.setText("click me");
	            mLayoutParams = new WindowManager.LayoutParams(
	                    LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT, 0, 0,
	                    PixelFormat.TRANSPARENT);
		其中flags和type比较重要
	            mLayoutParams.flags = LayoutParams.FLAG_NOT_TOUCH_MODAL
	                    | LayoutParams.FLAG_NOT_FOCUSABLE
	                    | LayoutParams.FLAG_SHOW_WHEN_LOCKED;
	            mLayoutParams.type = LayoutParams.TYPE_SYSTEM_ERROR;
	            mLayoutParams.gravity = Gravity.LEFT | Gravity.TOP;
	            mLayoutParams.x = 100;
	            mLayoutParams.y = 300;
	            mFloatingButton.setOnTouchListener(this);
	            mWindowManager.addView(mFloatingButton, mLayoutParams);
	        }
	    }

##### LayoutParams.flags：表示Window的属性，有很多选项代表不同的特性。

- **FLAG_NOT_FOCUSABLE**：表示window不需要获取焦点，也不需要接收各种输入事件，此标记会同时启动FLAG_NOT_TOUCH_MODAL，最终事件会直接传递给下层具有焦点的Window。
- **FLAG_NOT_TOUCH_MODAL**：系统会将当前Window区域以外的单击事件传递给底层的Window，当前Window区域以内的单击事件则自己处理，这个标记很重要，一般来说都要开启，否则其他Window无法收到单击事件。
- **FLAG_SHOW_WHEN_LOCKED**：可以让Window显示在锁屏的界面上。

##### LayoutParams.type:表示Window的类型。Window有三种类型，分别是应用Window，子Window，系统Window。
- **应用Window**：应用Window对应着一个Activity。层级范围：1~99
- **子Window**：不能单独存在，它需要附属在特定的父Window之中。比如Dialog就是一个子Window。层级范围：1000~1999
- **系统Window**：需要声明权限才能创建的Window，比如Toast和系统状态栏都是系统Window。层级范围：2000~2999。

		如果想要Window位于所有Window的最顶层，那么采用较大的层级即可，
		也就是创建一个系统Window，只需要：
		LayoutParams.type = LayoutParams.TYPE_SYSTEM_ERROR;
		同时声明权限即可
		<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />

### WindowManager
WindowManager提供的功能很简单，常用的只有三个方法，即添加View，更新View，和删除View。这三个方法在ViewManager中，而WindowManager继承了ViewManager。WindowManager操作Window更像是操作Window中的View。

	//它可以创建一个Window并向其添加View
	public void addView(View view, ViewGroup.LayoutParams params);
	//更新Window中的View
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    //删除Window中的View
	public void removeView(View view);

- **跟随手指拖动Window：**只需要根据手指的位置来设定LayoutParams中的X和Y即可改变Window的位置。

	    @Override
	    public boolean onTouch(View v, MotionEvent event) {
	        int rawX = (int) event.getRawX();
	        int rawY = (int) event.getRawY();
	        switch (event.getAction()) {
	            case MotionEvent.ACTION_MOVE: {
	                mLayoutParams.x = rawX;
	                mLayoutParams.y = rawY;
	                mWindowManager.updateViewLayout(mFloatingButton, mLayoutParams);
	                break;
	            }
	        }
	        return false;

### Window的添加过程
- 通过WindowManager.addView实现，WindowManager是一个接口，真是实现类是WindowManagerImpl
- 而WindowManager又交给了WindowManagerGlobal来处理，这个类内部会创建几个集合，分别存储①View（Window对应的所有的View），②LayoutParams（布局参数），③DyingView（调用了remoteView方法但是还没有完成删除操纵的Window对象），④ViewRootImpl（负责更新界面并完成Window的添加过程，View的绘制过程是由它来完成的）。
- ViewRootImpl内部有一个IWindowSession，它是以个Binder对象，真正的实现类是Session，也就是Window的添加过程是是一次IPC调用。Session内部通过WindowManagerService来完成Window的添加。
- 这样一来Window的添加请求就交给WindowManagerService去处理，WindowManager和WindowManagerService交互是一个IPC过程。Window的具体实现都位于WindowManagerService中。

### Window的删除过程
- 删除过程和添加过程一样，都是通过WindowManagerImpl后，在通过WindowManagerGlobal来实现
- WIndowManagerGlobal的removeView会获取待删除View的索引，然后遍历集合找到这个View，然后调用removeViewLocked方法进行删除。
- WindowManager删除有两种情况，分别是异步删除和同步删除，在Die（）方法中如果是判断为异步删除，就发送一个请求删除的消息，把这个View加入到DyingView集合中，然后返回，ViewRootImpl的Handler会接收到发送过来的消息并进行处理调用doDie（），如果是同步删除，就不发送消息直接进行处理调用doDie（）。
- doDie（）内部会调用disPatchDetachedFromWindow方法。真正删除View的逻辑会在这个方法中实现。

### Window的更新过程
- 通过更新LayoutParams，然后对View重新布局，包括测量，布局，重绘，除了View本身的重绘，还会更新Window的视图，这个过程最终是由WindowManagerService的relayoutWindow方法来具体完成的，它同样是一个IPC过程。

---
### 各级别window创建过程
#### Activity的Window创建中视图如何附属在Window上

获取PhoneWindow

	public Window makeNewWindow(Context context){
		return new PhoneWindow(context)//从这可以看出，Window的实现类确实是PhoneWindow，
	}

通过PhoneWindow实现，如果没有DecorView，就创建它，然后把View添加到DecorView的ContenParent中，最终回调onContentChanged方法通知Activity视图已经发生了改变，然后在onPause方法之后调用makeVisible，这时候就能看到视图了。

    void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm = getWindowManager();
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded = true;
    }
    mDecor.setVisibility(View.VISIBLE);
	}

#### Dialog的Window创建

如果是Dialog的Window创建，普通Dialog必须采用Activity的context，如果采用了Application的就会报错，
原因是因为没有应用token所导致的，而token一般只有Activity拥有
但是如果指定对话框的Window为系统类型（2000~2999）就可以正常弹出对话框，因为系统Window不需要token，系统window需要加权限。


	 public void onDialogClick(View v){
        Dialog dialog = new Dialog(getApplicationContext());
        TextView textView = new TextView(this);
        textView.setText("this is toast!");
        dialog.setContentView(textView);
        dialog.getWindow().setType(LayoutParams.TYPE_SYSTEM_ERROR);
        dialog.show();
    }

#### Toast的Window创建
- 因为有定时取消，所以采用Handler来完成，意味着Toast只能在有Looper的线程中弹出，因为Handler需要使用Looper才能完成切换线程的功能
- 内部有一个集合，负责存储所有的Toast，最多只能存储50个