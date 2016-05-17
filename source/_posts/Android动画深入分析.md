---
title: Android动画深入分析
date: 2016-05-12 20:50:03
tags: 源自Android开发艺术探索总结
---

### 动画的种类　
View动画，帧动画，属性动画

### 补间动画
平移，旋转，缩放，透明度

- **xml方式实现**：

		<set xmlns:android="http://schemas.android.com/apk/res/android"
			android:duration="500"//运行动画的周期
			android:fillAfter="true">//动画结束以后View是否停留在结束的位置，默认为false
			//透明度 1代表不透明，0代表完全透明
		    <alpha android:fromAlpha="0.1"
		        android:toAlpha="1"/>
			//旋转，前两个是角度， 后两个是旋转的中心轴，默认是控件的中心位置。  
		    <rotate android:fromDegrees="0"
		        android:toDegrees="180"
		        android:pivotX=""
		        android:pivotY=""/>
			//缩放，0代表完全缩放，1代表本身大小，最后两个是缩放的中心轴
		    <scale android:fromXScale="1"
		        android:toXScale="0.5"
		        android:fromYScale="1"
		        android:toYScale="0.5"
		        android:pivotY=""
		        android:pivotX=""/>
			//平移 从哪到哪，0代表
		    <translate android:fromXDelta="0"
		        android:fromYDelta="0"
		        android:toYDelta="1"
		        android:toXDelta="1"/>
		</set>
		
		Animation animation = AnimationUtils.loadAnimation(this, R.anim.scale);
	    image.startAnimation(animation);

- **代码方式实现**：
	
        ScaleAnimation scaleAnimation=new ScaleAnimation(0.1f,1f,0.1f,1f); 
        scaleAnimation.setDuration(1000);//动画运行的时间
		//动画的监听器
        scaleAnimation.setAnimationListener(new 
        image.startAnimation(scaleAnimation);

- **补间动画监听器**：

		Animation.AnimationListener() {
            @Override
            public void onAnimationStart(Animation animation) {
                //动画启动前
            }
            @Override
            public void onAnimationEnd(Animation animation) {
                //完成动画后 
            }
            @Override
            public void onAnimationRepeat(Animation animation) {
                //重新运行动画
            }
        });

### 帧动画

	<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
	    android:oneshot="false">
	oneshot代表着是否只展示一遍，设置为false会不停的循环播放动画
	<item android:drawable="@mipmap/ic_launcher" android:duration="500"/>
	<item android:drawable="@mipmap/ic_launcher" android:duration="500"/>
	<item android:drawable="@mipmap/ic_launcher" android:duration="500"/>
	</animation-list>
	//然后设置为View的北京 并通过Drawable来播放动画
     image.setBackgroundResource(R.drawable.anim);
        AnimationDrawable background = (AnimationDrawable) image.getBackground();
        background.start();

### View动画的特殊使用场景
比如在ViewGroup中可以控制子元素的出场效果，在Activity中可以实现不同Activity之间的切换效果。

- **LayoutAnimation**：用于ViewGroup。为子元素添加出场动画

		delay：指延时多长时间，如果动画运行时间是300ms，那么0.5表示延时150ms才能运行动画。
		animationOrder：子元素出场顺序，normal|reverse|random 顺序|反序|随机
		<layoutAnimation xmlns:android="http://schemas.android.com/apk/res/android"
		    android:delay="0.5"
		    android:animationOrder="normal"
		    android:animation="@anim/scale">
		</layoutAnimation>

然后为ViewGroup添加：

	android:layoutAnimation="@anim/layout_anim"

代码设置ListView子元素出场动画

        Animation anim=AnimationUtils.loadAnimation(this,R.anim.layout_anim);
        LayoutAnimationController controller=new LayoutAnimationController(anim);
        controller.setDelay(0.5f);
        controller.setOrder(LayoutAnimationController.ORDER_NORMAL);
        ls.setLayoutAnimation(controller);

- **Activity的切换效果**：
		
		//参数：1、Activity被打开时，所需要的动画资源的Id
			   2、Activity被暂停时，所需要的动画资源的Id
		//注意：overridePendingTransition必须位于startActivity和finish方法的后面，否则动画效果将不起作用。
		Intent intent=new Intent(this,Main2Activity.class);
		startActivity(intent);
		overridePendingTransition(R.anim.scale,R.anim.scale);
			
		@Override
	    public void finish() {
	        super.finish();
	        overridePendingTransition(R.anim.scale,R.anim.scale);
	    }

- **Fragment切换动画**：

        FragmentTransaction fragmentTransaction = getFragmentManager().beginTransaction();
        fragmentTransaction.setCustomAnimations(R.anim.scale,R.anim.scale);

### 属性动画
在一个时间间隔内完成对象从一个属性值到另一个属性值的改变。

        //使用属性动画 平移
        ObjectAnimator.ofFloat(image,"translationX",image.getWidth()).start();
        
        //修改背景颜色
        ValueAnimator valueAnimator=ObjectAnimator.ofInt(image,"backgroundColor",0xFFFF8080,0xFF8080FF);
        valueAnimator.setDuration(2000);
        valueAnimator.setStartDelay(500);//动画延迟多次时间后执行
        valueAnimator.setEvaluator(new ArgbEvaluator());
		//重复的次数 -1代表无限循环
        valueAnimator.setRepeatCount(ValueAnimator.INFINITE);
		//循环的模式 连续重复（123，123，123，123）和逆向重复（123，321，123，321）
        valueAnimator.setRepeatMode(ValueAnimator.RESTART);
        valueAnimator.start();

        //动画集合
        AnimatorSet set=new AnimatorSet();
        set.playTogether(
                ObjectAnimator.ofFloat(image,"rotationX",0,360),
                ObjectAnimator.ofFloat(image,"rotationY",0,180),
                ObjectAnimator.ofFloat(image,"rotation",0,-180),
                ObjectAnimator.ofFloat(image,"alpha",1,0.25f),
                ObjectAnimator.ofFloat(image,"scaleX",1,1.5f),
                ObjectAnimator.ofFloat(image,"scaleY",1,0.5f)
        );
        set.setDuration(3000);
        set.start();

		 AnimatorSet set=new AnimatorSet();
		 set.setInterpolator(new AccelerateDecelerateInterpolator());
			//三个同时执行
	        set.playTogether(oa1,oa2,oa3);
			//延迟执行
	        set.setStartDelay(200);
			//按照顺序执行
	        set.playSequentially(oa1,oa2,oa3);
	        //1和2 一块执行
	        set.play(oa1).with(oa2);
	        //3 在2的后面执行
	        set.play(oa3).after(oa2);
	        set.setDuration(2000);
	        set.start();

- **属性动画的监听器**：
		
		//监听动画整个过程，动画每播放一帧，它就调用一次
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                
            }
        });
		
		//监听动画的开始，结束，取消，重复
		valueAnimator.addListener(new Animator.AnimatorListener() {
	        @Override
	        public void onAnimationStart(Animator animation) {}
	
	        @Override
	        public void onAnimationEnd(Animator animation) {}
	
	        @Override
	        public void onAnimationCancel(Animator animation) {}
	        
	        @Override
	        public void onAnimationRepeat(Animator animation) {}
	    });

		//AnimatorUpdateListener：我们可以选择性的继承我们想要方法
		valueAnimator.addListener(new Animator.AnimatorUpdateListener(){
			
		}	

       


### 插值器和估值器
- **TimeInterpolator：**时间插值器，作用是根据时间流逝的百分比来计算当前属性值改变的百分比。

		1. LinearInterpolator：线性插值器（匀速动画）
		2. AccelerateDecelerateInterpolator：加速减速插值器（两头慢中间快）
		3. DecelerateInterpolator：减速插值器（越来越慢）
		4. AccelerateInterpolator：加速插值器（越来越快）等
	
	
- **TypeEvalutors**:类型估值器，作用是根据当前属性改变的百分比来计算改变后的属性值。

		1. IntEvaluator：属性的值类型为int；
		2. FloatEvaluator：属性的值类型为float；
		3. ArgbEvaluator：属性的值类型为十六进制颜色值；
		4. TypeEvaluator：一个接口，可以通过实现该接口自定义Evaluator

---

### 属性动画的原理：
属性动画要求动画作用的对象提供该属性的get和set方法，属性动画根据外界传递该属性的初始值和最终值，以动画的效果多次去调用set方法，每次传递给set方法的值都不一样，确切来说，根据时间的推移，所传递的值越来越接近最终值：

1. object必须要提供setAbc方法，如果动画的时候没有提供初始值，那么还要提供getAbc方法，因为系统要去取abc属性的初始值（如果不满足，系统直接Crash）
2.  object的setAbc方法对属性abc所作的改变必须能够通过某种方法反映出来，比如会带来UI的改变等（这条不满足，动画就没有效果，但系统不会Crash）


**为什么我们对ImageView的width属性做动画会没有效果？**

		ObjectAnimator.ofInt(imageView,"width",400).setDuration(500).start();
**原因**：ImageView内部没有setWidth和getWidth方法()。

**解决办法：**

1. 给你的对象加上set和get方法，如果你有权限的情况下

		由于ImageView是android内部封装好的，所有我们不能自己加set和get方法。

2. 用一个类来包装原始对象，间接为其提供get和set方法

		//当我们在调用这句话的时候，只需要把ImageView对象传递过去就可以实现效果。	
		ObjectAnimator.ofInt(new ViewWrapper(imageView),"width",400).setDuration(500).start();
	
		//创建一个类，然后构造函数中，接收ImageView，然后间接的提供getWidth和setWidth方法 
		 public static class ViewWrapper {
	        private View view;
	
	        public ViewWrapper(View v) {
	            this.view = v;
	        }
	        public int getWidth(){
	            return view.getLayoutParams().width;
	        }
	        public void setWidth(int width){
	            view.getLayoutParams().width=width;
	            view.requestLayout();
	        }
	    }
		

3. 采用ValueAnimator，监听动画过程，自己实现属性的改变

	    public void performAnimate(final View target, final int start, final int end){
	        ValueAnimator valueAnimator=ValueAnimator.ofInt(1,100);
	        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
	            private IntEvaluator mEvaluator=new IntEvaluator();
	            @Override
	            public void onAnimationUpdate(ValueAnimator animation) {
	                //获取当前动画的进度值，整型，1~100之间
	                int currentValue = (int) animation.getAnimatedValue();
	                System.out.println("currentValue:"+currentValue);
	                //获取当前进度占整个动画过程的比例 浮点型 0~1之间
	                float fracation=animation.getAnimatedFraction();
	                System.out.println("fracation:"+fracation);
	                //调用整型估值器，通过比例计算出宽度，然后赋值给ImageView，
	                target.getLayoutParams().width=mEvaluator.evaluate(fracation,start,end);
	                //重新测量和布局
	                target.requestLayout();
	        /**
	         * requestLayout:当View确定已经不在适合现有的区域时，调用这个方法，
	         * 告诉父容器重新重新调用它的onMeasure和onLayout方法 来对自己重新设置位置
	         * invalidate：View本身调用迫使View重画
	         * */
	            }
	        });
	        valueAnimator.setDuration(3000).start();
	    }

	 	//然后调用这个方法，把ImageView，开始宽度，最终宽度传入进去。
		 performAnimate(image,image.getWidth(),300);


### 使用动画的注意事项

1. **OOM问题**：主要出现在帧动画中，尽量避免使用帧动画。
2. **内存泄漏**：在属性动画中有一类无限循环的动画，这类动画需要在Activity退出时进行停止，否则就会造成Activity无法释放，而从内存泄漏。View动画不会存在此问题。
3. **View动画的问题**：只是对View的影像做动画，并不是真正改变View的状态，有时候会出现View无法隐藏的现象，这个时候只要调用view.clearAnimation（）即可。
4. **不要使用px**：尽量使用dp，使用px会导致不同的设备上有不同的效果。
5. **动画元素的交互**：在3.0之前，无论是什么动画，新位置都无法触发单击事件，同时老位置仍然可以触发单击事件。从3.0之后，属性动画的单击事件触发位置为移动后的位置，但是View动画仍然在原位置。
6. **硬件加速**：使用动画的过程中，建议开启硬件加速，这样会提高动画的流畅性。

### 硬件加速
默认情况下 高级别开启后，低级别也开启

1. **application级别开启**：开启后，即可为整个应用程序开启硬件加速。

		<application android:hardwareAccelerated="true" ...>

2. **Activity级别开启**：false表示这一个Activity不开启硬件加速。

	  	<activity android:hardwareAccelerated="false" />

3. **window级别开启**：window不支持硬件加速关闭

        getWindow().setFlags(
            WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED,
            WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED);

4. **View级别关闭**： view不支持硬件加速开启

		myView.setLayerType(View.LAYER_TYPE_SOFTWARE, null);