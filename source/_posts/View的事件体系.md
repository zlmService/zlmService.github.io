---
title: View的事件体系
date: 2016-05-09 21:23:22
tags: 源自Android开发艺术探索第三章总结
---
#### getX()和getRawX()的区别：
第一个是返回当前View左上角的x坐标，第二个是返回View相对于手机屏幕左上角的x坐标。

#### VelocityTracker:
速度追踪，可用于监听滑动的速度，加载列表的数据多少。

#### GestureDetector:
手势检测，用于辅助检测用户的单击、滑动、长按、双击等行为。如果只是监听滑动相关的事件在onTouchEvent中实现；如果要监听双击这种行为的话，那么就使用GestureDetector。

#### TouchSlop：
系统所能识别出的最小滑动距离。ViewConfiguration.get((getConText)).getScaledTouchSlop()

#### View的滑动：
1. **scrollTo/scrollBy**：操作简单，适合对View内容的滑动。只能改变View内容的位置，不能改变View在布局中的位置。比如：ImageView改变了src的位置，但是background位置并没有改变
2. **动画**:操作简单，适用于对没有交互的View和实现复杂的动画效果，比如：Button有一个点击事件，但是当滑动过后，新的位置无法触发onClick事件，而单击原始的空白位置仍然可以触发onClick事件。
3. **LatoutParams**:操作稍微麻烦，适用于和用户有交互的View，

#### 弹性滑动：
**大致思想：**将一次大的滑动分成若干次小的滑动，并在一个时间段内完成。

**实现方法：**它们三个只滑动了内容，并没有滑动本身。因为内部都使用了scrollTo实现滑动效果。

1. **Scroller**:它本身不能实现View的滑动，他需要配合View的computeScroll方法才能完成弹性滑动的效果，它不断地让View重绘，而每一次重绘起始时间都会有一个时间间隔，通过这个时间间隔得到View当前的滑动位置，然后通过scrollTO方法来完成View的滑动，View每一次的重绘都导致View进行小幅度的滑动，而多次的小幅度就组成了弹性滑动。
	- 大致流程：当创建了一个Scroller对象，然后调用startScroll方法（这个方法其实就是保存我们传递过来的参数），然后调用invalidate（）方法（这个方法会导致View重绘，他会自动去调用computeScroll（）方法，这个方法是一个空方法需要我们去实现）
	
			@Override
			public void computeScroll() {
				//这个方法就是会根据时间的流逝的百分比来计算出需要滑动的值，当滑动还未结束就返回true，结束就返回false
			    if (scroller.computeScrollOffset()) {
					//判断返回值如果是true 也就是滑动未结束，然后就进行滑动，
			        scrollTo(scroller.getCurrX(),scroller.getCurrY());
					//重新绘制。
			        postInvalidate();
			    }
			}

2. **动画**：我们可以在onAnimationUpdate方法中实现其他想要的操作。
 
		  //动画 实现弹性滑动
	        ValueAnimator animator = ValueAnimator.ofInt(0,1).setDuration(1000);
	        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
	            @Override
	            public void onAnimationUpdate(ValueAnimator animation) {
	                float fraction=animation.getAnimatedFraction();
	                System.out.println("fraction:"+fraction);
	                btn.scrollTo(0+(int)(-100*fraction),0);
	            }
	        });
	        animator.start();

3. **使用延时策略**：通过一个循环，不断的发送消息，然后在消息中进行View的滑动，从而实现弹性滑动的效果。
   - Handler+sendEmptyMessageDelayed（）：
   
			  handler=new Handler(){
	            int count=0;
	            @Override
	            public void handleMessage(Message msg) {
	                super.handleMessage(msg);
	                switch (msg.what){
	                    case 1: {
	                        count++;
	                        if (count <= 30) {
							//分成30份的小幅度滑动最终组合成一个大幅度的滑动
	                            System.out.println(count);
	                            float fraction = count / (float) 30;
	                            int scrollX = (int) (fraction * 100);
	                            System.out.println(scrollX);
	                            btn.scrollTo(-scrollX, 0);
								//每隔33毫秒 滑动一次
	                            handler.sendEmptyMessageDelayed(1, 33);
	                        }
	                        break;
	                    }
	                }
	            }
	        };   	

   - View的postDelayed（）
   - 线程的sleep（）
  
#### View的事件分发机制
**分发流程**：当事件产生后，会由Activity先接收，然后调用自身的dispatchTouchEvent方法进行分发，分发到Window（唯一实现类：PhoneWindow，它用来控制顶级View的外观和行为策略）中，然后再传递到View（先传递到DecorView：它继承FrameLayout，它内部会获取我们setContentView设置的View，然后传递给它的子View）中。期间如果某个进行了拦截（onInterceptTouchEvent=true）就会调用自身的onTouchEvent进行消耗事件，如果不拦截最终会传递到最底层的View中，然后消耗事件，如果onTouchEvent返回false（表示不消耗事件），这个事件就会回传给上层，直到某一层消耗了这个事件。


- **如何不消耗事件**：View的onTouchEvent默认都会消耗事件（返回值为true），但是当clickable和longclickable同时为false的时候，这个View是不会消耗事件的，比如button的clickable默认为true，但是TextView的clickable默认为false，一般longclickable默认为false。

- **setOnClickListener：**当view设置setOnClickListener的时候，它会自动将view的clickable设置为true，同理setOnLongClickListener的时候，也会自动把longclickable设置为true。

- **onTouchListener和onTouchEvent的区别**：
	- 相同点：
	onTouch()与onTouchEvent()都是用户处理触摸事件的API。
	- 不同点：
	(01)，onTouch()是View专门提供给用户的接口，目的是为了方便用户自己处理触摸事件。而onTouchEvent()是Android系统自己实现的接口。
	(02)，onTouch()的优先级比onTouchEvent()的优先级更高。
	dispatchTouchEvent()中分发事件的时候，会先将事件分配给onTouch()进行处理，然后才分配给onTouchEvent()进行处理。 如果onTouch()对触摸事件进行了处理，并且返回true；那么，该触摸事件就不会分配在分配给onTouchEvent()进行处理了。只有当onTouch()没有处理，或者处理了但返回false时，才会分配给onTouchEvent()进行处理。

- **enabled不会影响view消耗事件**：当一个View处于不可用的状态下（enabled=false），它照样会消耗事件，尽管它看起来不可用。只要clickable或者longclickable有一个为true，那么它的onTouchEvent就会返回为true

#### View的事件冲突：
**场景：**

1. 外部滑动方向和内部滑动方向不一致
2. 外部滑动方向和内部滑动方向一致
3. 上面两种情况的嵌套

**解决方式：**

- 方向不一致：

	1. 外部拦截法： 当事件发生后，先经过父容器进行拦截，然后判断是否需要消耗此事件，如果不需要此事件就不拦截。**注意**：外部拦截的时候，**Action_Down**必须返回为false，因为如果返回true代表拦截Action_Down事件，那么后续的Action_Move和Action_UP也会直接交给父容器进行处理;**Action_Move:**进行判断如果需要就拦截返回true，反之返回false;**Action_UP：**也必须返回false：如果拦截这个事件，那么子View就接收不到，也就不会触发Click点击事件。

			//父容器的方法：
		    @Override
		    public boolean onInterceptTouchEvent(MotionEvent ev) {
		        int x = 0,y = 0;
		        switch (ev.getAction()){
		            case MotionEvent.ACTION_DOWN:
		                x= (int) ev.getX();
		                y= (int) ev.getY();
		                return false;
		            case MotionEvent.ACTION_MOVE:
		//         比如解决横向和竖向滑动的冲突，需要拦截左右滑动的事件
		//          如果 滑动x坐标大于滑动y坐标，表示左右滑动，拦截事件 返回true
		                if (Math.abs(x)>Math.abs(y)){
		                    return true;
		                }else{
		                    return false;
		                }
		            case MotionEvent.ACTION_UP:
		                return false;
		        }

	2. 内部拦截法：指父容器不拦截所有事件，全部传递到子元素中，如果子元素需要此事件就消耗掉，否则就给父容器进行处理。**注意：**requestDisallowInterceptTouchEvent(true);表示父容器会跳过onInterceptTouchEvent回调，也就是说，在down之后的move，up等等 父容器都不能进行拦截
		
			//子元素的方法：
			  @Override
		    public boolean dispatchTouchEvent(MotionEvent ev) {
		        int x = 0,y = 0;
		        switch (ev.getAction()){
		            case MotionEvent.ACTION_DOWN:
		               getParent().requestDisallowInterceptTouchEvent(true);//表示父容器会跳过onInterceptTouchEvent回调，也就是说，在down之后的move，up等等 父容器都不能进行拦截
		                break;
		            case MotionEvent.ACTION_MOVE:
		                //判断 入容器需要此类事件
		                if (Math.abs(x)>Math.abs(y)) {
		                    getParent().requestDisallowInterceptTouchEvent(false);//返回给父容器
		                }
		                break;
		
		            case MotionEvent.ACTION_UP:
		                break;
		        }
		        return super.dispatchTouchEvent(ev);
		    }
	
			//父容器的拦截方法：
			   //表示 父容器拦截除了Action_Down以外的所有事件.
			    // 这样子View调用 getParent().requestDisallowInterceptTouchEvent(false);父容器才能继续拦截所需的事件.
			    // 不能拦截Action_Down:因为如果拦截Action_Down，那么所有事件都无法传递到子View中去，这样内部拦截就无法其作用.
			    @Override
			    public boolean onInterceptTouchEvent(MotionEvent ev) {
			        if (ev.getAction()==MotionEvent.ACTION_DOWN){
			            return  false;
			        }else{
			            return true;
			        }
			    }

- 方向一致：判断子View是否滑动到当前View顶部并且继续向下滑动，这时候父容器拦截事件，并进行处理