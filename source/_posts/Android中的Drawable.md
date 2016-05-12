---
title: Android中的Drawable
date: 2016-05-12 11:09:25
tags: 源自Android开发艺术探索总结
---

### Drawable简介：
- Drawable有很多种，它们都表示一种图像的概念，但是它们又不全是图片，通过颜色也可以构造出各式各样的图像的效果。在实际开发中，Drawable常常被用来当做View的背景使用。
- Drawable的内部宽高 通过getIntrinsicIdth和getIntrinsicHeight这两个方法可以获取它们，但是并不是所有的Drawable都有内部宽高，比如一个图片所形成的Drawable，它的内部宽高就是这个图片的宽高，但是一个颜色所形成Drawable，它就没有内部宽高。注意的是：内部宽高不等同它的大小，当用作View背景的时候，Drawable会被拉伸至View的同等大小。

### Drawable的分类：

- **BitmapDrawable**：对应<bitmap>标签，它表示的就是一站图片，也支持.9图片

		<bitmap xmlns:android="http://schemas.android.com/apk/res/android"
		    android:src="@mipmap/ic_launcher"//图片资源的Id
		    android:antialias="true"//是否开启抗锯齿 
		    android:dither="true"//是否开启抖动，当图片和手机像素不一致时，可以智能的改变为手机支持的色彩模式
		    android:filter="true"//是否开启过滤效果，当图片被拉伸时，可以保持较好的显示效果
		    android:gravity="center"//当图片小于容器尺寸的时候，可以对图片进行定位
		    android:tileMode="repeat" >//平铺效果，开启这个效果，gravity就会失效
		</bitmap>
- **ShapeDrawable**：对应<shape>标签，它表示通过颜色来构造的图形，既可以是纯色，也可以是渐变效果的

		<shape xmlns:android="http://schemas.android.com/apk/res/android"
		    android:shape="rectangle">//表示图形的形状：oval（椭圆）line（横线）ring（圆形）默认是rectangle（矩形）
		    <corners android:radius="20sp"/>//表示shape四个角的角度，只适用于矩形
		    <gradient android:startColor="#f00"//表示渐变效果 和solid互相排斥
		        android:centerColor="#0f0"
		        android:endColor="#00f"/>
		   	<solid android:color="#f0f"/> //表示纯色填充
		    <size android:height="50dp" //表示图形大小，默认情况下shape是没有固定宽高的，
			如果设置了size，那么size的宽高就是shape的固定宽高，
			但是作为View的背景时，shape还会被拉伸或缩小为view的大小
		        android:width="50dp"/>
			//表示 描边，边框
		    <stroke android:width="2sp" 描边的宽度
		        android:color="#ff0"	颜色
		        android:dashGap="2sp"  设置这属性后，边框由实现变成虚线 描边虚线的线段宽度
		        android:dashWidth="10sp"/> 组成虚线的线段之间间隔
		</shape>

- **LayerDrawable**：对应<layer-list>标签，表示一种层次化的Drawable集合，将不同的Drawable放置不同的层上面从而达到一种叠加后的效果，一个集合中可以包含多个item，一个item相当于一个Drawable。效果类似于FrameLayout 后来的会覆盖到之前的上面。

		<layer-list xmlns:android="http://schemas.android.com/apk/res/android">		
			//可以引用一个已有的Drawable资源，也可以自定义一个
		    <item android:drawable="@drawable/bitmap_drawable">
		        <shape>
		            <solid android:color="#0ac39e" />
		        </shape>
		    </item>
			//代表偏移量
		    <item android:bottom="25dp">
		        <shape>
		            <solid android:color="#ff0" />
		        </shape>
		    </item>
		
		    <item android:bottom="50dp">
		        <shape>
		            <solid android:color="#f00" />
		        </shape>
		    </item>
		</layer-list>

- **StateListDrawable**：对应<selector>标签，它表示的也是Drawable集合，每个Drawable都对应View的一种状态，这样系统可以根据View的状态来选择合适的Drawable。

	**constantSize**：表示固有大小是否不随着其状态的改变而改变，因为随着状态的不同，它们的Drawable也不同，而不同的Drawable有不同的大小，true就表示固有大小保持不变，默认为false。固有大小：内部所有的Drawable的固有大小的最大值。

		//常用属性
		<selector xmlns:android="http://schemas.android.com/apk/res/android" android:constantSize="true">
		//用户选择了View  可以在View的点击事件中 View.setSelected(true);
		    <item android:drawable="@drawable/image1" android:state_selected="true"/>
		//用户按下状态，比如Button按下后没有松开的状态
		    <item  android:drawable="@drawable/image1" android:state_pressed="true" />
		//表示View已经获取焦点 te.setFocusable();
		    <item android:state_focused="true"/>
		//表示用户选中了View,适于用CheckBox这类View
		    <item android:state_checked= "true"/>
		//表示View处于可用状态
		    <item android:state_enabled="true"/>
		//表示默认状态，一般来说，应该把默认状态写到最下面，当上面所有都匹配不到的情况下，系统就会选择默认的item
		    <item android:drawable="@mipmap/ic_launcher"/>
		</selector>

- **LevelListDrawable**：根据不同的等级，会切换对应的Drawable,如果用来作为ImageView的前景Drawable，可以通过ImageView的setImageLevel方法来切换Drawable
		
		<level-list xmlns:android="http://schemas.android.com/apk/res/android">
			//等级范围是 0~10000 
		    <item
		        android:drawable="@drawable/image1"
				android:minLevel="0" 
		        android:maxLevel="10" />
		    <item
		        android:drawable="@mipmap/ic_launcher"
				android:minLevel="11" 
		        android:maxLevel="20" />
		</level-list>

- **TransitionDrawable**： 它用于实现两个Drawable之间淡入淡出的效果。
		
		<transition xmlns:android="http://schemas.android.com/apk/res/android">
		    <item android:drawable="@drawable/shape_drawable_gradient_linear" />
		    <item android:drawable="@drawable/shape_drawable_gradient_radius" />
		</transition>

然后通过代码方式进行变换

		textView= (TextView) findViewById(R.id.text_transition);
		TransitionDrawable drawable = (TransitionDrawable) textView.getBackground();
	    drawable.startTransition(2000);//淡入淡出运行的的时间
		drawable.reverseTransition(2000);//返过来运行淡入淡出

- **InsetDrawable**：它可以将其他的Drawable内嵌到自己当中，并可以在四周留出一定间距。当一个View希望自己的背景比自己实际区域小的时候，可以使用它来实现。
		
		<inset xmlns:android="http://schemas.android.com/apk/res/android"
		    android:insetTop="15dp"
		    android:insetBottom="15dp"
		    android:insetLeft="15dp"
		    android:insetRight="15dp">
		        <shape android:shape="oval">
		            <solid android:color="#f0f" />
		        </shape>
		</inset>

- **ScaleDrawable**：它可以根据自己的等级将指定的Drawable缩放到一定比例
	
		<scale xmlns:android="http://schemas.android.com/apk/res/android"
		    android:drawable="@drawable/image1"
		    android:scaleGravity="center"//位置
		    android:scaleHeight="70%" //缩放比例
		    android:scaleWidth="70%">
		</scale>
还必须通过代码进行等级设置 因为不设置的等级，默认等级为0，这样ScaleDrawable就不会显示出来，同时缩放大小受等级的大小的影响。10000表示不缩放。级别越大 内部的Drawable越大，缩放比例越大，内部的Drawable越小。

        im3= (ImageView) findViewById(R.id.image_scale);
        ScaleDrawable scaleDrawable= (ScaleDrawable) im3.getBackground();
        scaleDrawable.setLevel(5555);

- **ClipDrawable**： 它可以根据自己当前的等级来裁剪另一个Drawable，裁剪后只是Drawable变化了，但是空间还是占原来的大小

		
		<clip xmlns:android="http://schemas.android.com/apk/res/android"
		    android:clipOrientation="vertical"//裁剪的方向
		    android:drawable="@drawable/image1"
		    android:gravity="bottom">//表示如果是 垂直裁剪，那么从顶部开始剪
			//裁剪的方向是由这两个属性组合一起控制的
		</clip>

通过代码进行等级设置， 0代表完全裁剪，10000代表不裁剪，等级越大，裁剪的比例越小

        im4= (ImageView) findViewById(R.id.image_clip);
        ClipDrawable clipDrawable= (ClipDrawable) im4.getBackground();
        clipDrawable.setLevel(5000);