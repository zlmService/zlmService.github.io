---
title: View的工作原理
date: 2016-05-09 21:23:53
tags: 源自Android开发艺术探索总结
---

### MeasureSpec：测量规格
- 转换过程：对于**DecorView**（顶级View）来说，MeasureSpec由窗口的尺寸和自身的LayoutParams来共同确定的；对于**普通的View**，其MeasureSpec由父容器的MeasureSpec和自身的LayoutParams来共同确定的。
- **SpecMode**：测量模式
	1. UNSPECIFIED：父容器不做任何限制，要多大给多大，一般用于系统内部
	2. EXACTLY：父容器已经检测出View所需要的精确大小，相当于match_parent
	3. AT__MOST：父容器指定了一个可用大小即SpecSize，相当于wrap_content
- **SpecSize**：某种测量模式下的规格大小

- **普通View的SpasureSpec的创建过程:**
	- 红色：代表父容器的SpecMode类别
	- 绿色：代表普通View的LayoutParams的设置
	- 黄色：代表普通View的SpecMode的类别
	- 蓝色：代表普通View的SpecSize
	- parentSize：代表父容器中目前可使用的大小
	
	
![普通View的MeasureSpec的创建规则](http://i.imgur.com/zPTyZOg.png)

	
1. 当View采用固定宽高时：无论父容器的MeasureSpec是什么，View都是精确模式并且大小遵循LayoutParams中的大小。
2. 当View是match_parent：①如果**父类是最大模式**，那么View也是最大模式并且大小**不会超过父容器的剩余空间** 。②如果**父类是精准模式**，那么View也是精准模式并且大小**就是父容器的剩余空间**。
3. 当View是wrap_content：不管父类是什么模式，View总是最大化并且大小不能超过父容器的剩余空间。
	
### Measure过程：
1.view的Measure过程：解决在自定义View布局中使用 wrap-content 相当于match-parent

**原因：**当View在布局中使用wrap-content，那么它的specMode是AT_MOST模式，这种模式下，它的宽高等于specSize,也就是父容器当前剩余的空间大小。

		//源码
	public static int getDefaultSize(int size, int measureSpec) {
	// 将我们获得的最小值赋给result
    int result = size;

	// 从measureSpec中解算出测量规格的模式和尺寸
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

	/*
	 * 根据测量规格模式确定最终的测量尺寸
	 */
    switch (specMode) {
	-------------------
	//如果是UNSPECIFIED模式下，
	当我们没有设置background属性的情况下，size就是我们所设置的值，当我们没有设置 默认是0
	//如果我们设置了background，size就是minWidth和设置的background的最小宽度这两者中的最大值。
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
	-------------------------
	从这可以看出 为什么设置wrap_content还是match_parent
	因为它的宽高等于specSize，而这个specSize就是父容器当前剩余的空间大小
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
	}

**解决办法：重写子类的onMeasure方法**

	 @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
	 int measuredWidth;//定义一个默认宽高，
     int measuredHeight;
	//获取测量的模式和大小
	int widthSpaceSize = MeasureSpec.getSize(widthMeasureSpec);
    int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
    int heightSpaceSize = MeasureSpec.getSize(heightMeasureSpec);
    int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
	//如果 宽和高的模式都是AT_MOST（表示父容器不知道具体多大）
	 if (widthSpecMode == MeasureSpec.AT_MOST && heightSpecMode == MeasureSpec.AT_MOST) {
	//就子View自己设置测量尺寸（比如是一个ImageView,可以获取bitmap的宽高来当做这个控件的大小）
	 setMeasuredDimension(measuredWidth, measuredHeight);
	} else if (heightSpecMode == MeasureSpec.AT_MOST) {
	//如果宽不确定，就重新设置宽，高不确定就设置高
	 setMeasuredDimension(widthSpaceSize, childView.getMeasuredHeight());
	} else if (widthSpecMode == MeasureSpec.AT_MOST) {
	    setMeasuredDimension(measuredWidth, heightSpaceSize);
        }
    }

2.ViewGroup的measure过程

 - 它是一个抽象类，提供了一个measureChildren的方法，这个方法内部就是取出所有的子元素，然后获取每个子元素的LayoutParams,然后子元素自己去实现具体的onMeasure方法，他会通过变量来记录每一个子元素测量后的宽高，每测量一个子元素，这个变量就会增加，增加的部分主要是（子元素的宽高+子元素的margin），当所有子元素测量完毕后，它也就计算出了自己的大小。

### layout过程：
Layout的作用就是ViewGroup用来确定子元素的位置，layout（）确定view本身的位置，而onLayout（）确定所有子元素的位置。当ViewGroup的位置确定后，就会在onLayout中遍历所有的子元素并调用其layout方法，在子元素的layout方法中又会调用onLayout方法来确定自身的位置。

- getMeasureWidth和getWidth的区别：在默认情况下它们都是相等的，但是如果是重写了View的layout方法 又进行了赋值，getWidth就会是 重新赋值的这个宽度，
	
		这种情况下，getWidth就会比getMeasureWidth多100
		view.layout(left,top,right+100,bottom+100);

### draw过程：
 	就是把View绘制到屏幕上：
	1. 绘制背景background.draw(canvas)
	2. 绘制自己(onDraw)
	3. 绘制children(dispatchDraw)
	4. 绘制装饰(onDrawScrollBars)
	
- View的绘制过程传递是通过dispatchDraw来实现的，它会遍历调用所有子元素的draw方法，如此draw事件就一层一层传递下去。
- setWillNotDraw：如果一个View不需要绘制任何内容，就设置为true以后，系统就会进行相应的优化。默认情况下①View没有启这个优化标记位，②ViewGroup启用了这个标记位。所以当我们需要一个ViewGroup通过onDraw来绘制内容时，就需要关闭这个标记位。

		 public void setWillNotDraw(boolean willNotDraw) {
		        setFlags(willNotDraw ? WILL_NOT_DRAW : 0, DRAW_MASK);
		    }

### 自定义View

- 直接继承View或者ViewGroup的控件，padding默认是无法生效的，需要自己处理。
	
		  @Override
		    protected void onDraw(Canvas canvas) {
		        super.onDraw(canvas);
		        int paddingBottom = getPaddingBottom();
		        int paddingLeft = getPaddingLeft();
		        int paddingRight = getPaddingRight();
		        int paddingTop = getPaddingTop();
		        int width = getMeasuredWidth() - paddingLeft - paddingRight;
		        int height = getMeasuredHeight() - paddingTop - paddingBottom;
		        int radius = Math.min(width, height) / 2;
		        canvas.drawCircle(paddingLeft + width / 2, paddingTop + height / 2, radius, paint);
		    }

- 自定义属性：

	1. 在values创建自定义属性xml：

			<?xml version="1.0" encoding="utf-8"?>
			<resources>
		   		 <declare-styleable name="CircleView">
		       		 <attr name="circle_color" format="color"/>
		   		 </declare-styleable>
			</resources>
	2. 在自定义View类中进行代码编写：

			//在自定义View的构造函数中 获取自定义属性的文件，然后获取文件中CircleView_circle_color的这个属性，
			如果布局中没有设置，就用黄色当默认值，然后recycle（）来实现资源。
			 TypedArray typedArray = context.obtainStyledAttributes(attrs,R.styleable.CircleView);
		     mColor = typedArray.getColor(R.styleable.CircleView_circle_color, Color.YELLOW);
		     typedArray.recycle();
	3. 在布局中应用：
			
			//先在布局中进行声明，app是自定义属性的前缀 可以随意起，但是在自定义View中的自定义属性 前缀必须和这里的一致
			xmlns:app="http://schemas.android.com/apk/res-auto"

				<com.demo.zlm.viewsample.CircleView
			        android:layout_width="100dp"
			        android:layout_height="100dp"
			        android:layout_margin="20dp"
					//使用自定义属性
			       app:circle_color="#f00"
			        android:background="#000"
		        android:padding="20dp" />

### 自定义View属性相关细节

#### ObtainStyledAttributes四个参数的详细作用

		TypedArray typedArray = context.obtainStyledAttributes(
				AttributeSet set,
				int[] atts，
				int defStyleAttr,
				int defStyleRes);

defStyleRes参数 当defStyleAttr为0时，布局文件中也没有进行设置，就使用它指定的style文件

	我们在Style.xml里面编写
    <style name="test">
        <item name="t1">false</item>
        <item name="t2">3333</item>
        <item name="t3">139dp</item>
    </style>
	//然后设置进去	
    TypedArray typedArray = context.obtainStyledAttributes(
	attrs, R.styleable.CircleView，0，R.style.test);
	//当我们不在布局文件中设置任何属性的时候，它会从这个style中读取相关属性
	
defStyleAttr 可以在theme中修改样式，比如在系统主题中，想修改所有textView或者Button的样式
	
	我们先在attrs.xml里面添加一个引用格式的属性
    <attr name="test2" format="reference"></attr>

	然后在styles.xml里面找到我们所使用的theme，添加一条item
    <style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <item name="test2">@style/test</item>
    </style>

    <style name="test">
        <item name="t1">true</item>
        <item name="t2">8888</item>
        <item name="t3">29dp</item>
    </style>

    //然后设置进去	
    TypedArray typedArray = context.obtainStyledAttributes(
	attrs, R.styleable.CircleView，R.attr.test2，0);
	//他就会读取这里面的属性
使用场景：当我们需要切换不同主题时，会发现控件的样式也会发生一些改变，就是因为不同的主题，设置了不同的style。当我们自定义View有很多属性的时候，可以提供默认的Style，然后让用户去设置到theme里面即可。

总结：只有当defStyleAttr设置为0或者当前的theme中没有找到相关属性的情况下。才会从defStyleRes中获取，所以defStyleAttr的优先级更高

#### AttributeSet、defStyleAttr、defStyleRes、Threm 属性的优先级

	  //定义属性
	<declare-styleable name="custom_attrs">
	    <attr name="custom_color1" format="color"></attr>
	    <attr name="custom_color2" format="color"></attr>
	    <attr name="custom_color3" format="color"></attr>
	    <attr name="custom_color4" format="color"></attr>
	    <attr name="custom_color5" format="color"></attr>
	</declare-styleable>

这一步是设置Theme的默认值

	//定义theme可配置style
	<attr name="custom_style" format="reference"></attr>	
	
	 <style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
      //配置style
    <item name="custom_style">@style/custom_theme</item>
      //直接在主题中指定
    <item name="custom_color1">#ff444444</item>
    <item name="custom_color2">#ff444444</item>
    <item name="custom_color3">#ff444444</item>
    <item name="custom_color4">#ff444444</item>
    <item name="custom_color5">#ff444444</item>
    </style>

    //主题中配置的style
    <style name="custom_theme">
    <item name="custom_color1">#ff222222</item>
    <item name="custom_color2">#ff222222</item>
    <item name="custom_color3">#ff222222</item>
    </style>

设置第四个参数的默认值

	 //定义默认style
	<style name="default_style">
	    <item name="custom_color1">#ff333333</item>
	    <item name="custom_color2">#ff333333</item>
	    <item name="custom_color3">#ff333333</item>
	    <item name="custom_color4">#ff333333</item>
	</style>

自定义一个style 在布局中直接引用

	   //直接在layout文件中引用的style，最后会被放到set中
	  <style name="myStyle">
	    <item name="custom_color1">#ff111111</item>
	    <item name="custom_color2">#ff111111</item>
	  </style>


		//当我们在布局文件中 对自定义View进行设置
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        app:custom_color1="#ff000000"
        style="@style/myStyle"
最后结果

	custom_color1=#ff000000 //布局文件中直接指定，优先级最高 （set）
	custom_color2=#ff111111 //布局同通过style指定，也包含在set中，优先级第二
	custom_color3=#ff222222 //布局通过主题中配置风格style（defStyleAttr）
	custom_color4=#ff444444 //由系统Theme直接指定的
	custom_color5=#ff444444

set>defStyleAttr(主题可配置样式)>defStyleRes(默认样式)>NULL(主题中直接指定)

这个优先级需要说明一点，defStyleRes只有在defStyleAttr为0或者主题中没有配置时，才会生效；所以上面例子中
custom_color4=#ff444444 而不是333333，因为此时的defStyleAttr我们配置了。

#### 自定义View构造函数的两种写法的区别

//第一种写法

	    public AttrButton(Context context) {
	        this(context,null);
	    }
	
	    public AttrButton(Context context, AttributeSet attrs) {
	        this(context, attrs,0);
	    }
	
	    public AttrButton(Context context, AttributeSet attrs, int defStyleAttr) {
	        super(context, attrs, defStyleAttr);
	        init();
	    }

//第二种写法

	    public AttrButton(Context context) {
	        super(context);
	        init();
	    }
	
	    public AttrButton(Context context, AttributeSet attrs) {
	        super(context, attrs);
	        init();
	    }
	
	    public AttrButton(Context context, AttributeSet attrs, int defStyleAttr) {
	        super(context, attrs, defStyleAttr);
	        init();
	    }


区别1：当需要设置obtainStyledAttributes的第三个参数时，一般就使用第一种方式，会在两个参数的构造函数中，去调用三个桉树的构造函数，同时传入defStyleAttr。

区别2：当你系统已有控件的话，比如继承Button，使用第一种方式会覆盖掉Button默认在theme里面设置的style，相对于第二种写法更合适
