---
title: LayoutInflater
date: 2016-05-26 15:40:46
tags:
---
### setFactory方法的使用
	
#### 可以将布局中的控件替换成另一个控件

	public class MainActivity extends AppCompatActivity
	{
    private static final String TAG = "MainActivity";

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        LayoutInflaterCompat.setFactory(LayoutInflater.from(this), new LayoutInflaterFactory()
        {
            @Override
            public View onCreateView(View parent, String name, Context context, AttributeSet attrs)
            {
                Log.e(TAG, "name = " + name);
                int n = attrs.getAttributeCount();
                for (int i = 0; i < n; i++)
                {
                    Log.e(TAG, attrs.getAttributeName(i) + " , " + attrs.getAttributeValue(i));
                }

                return null;
            }
        });
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

更新v7包的时候，忽然我们的TextView就支持了一些属性，比如tint属性，以前是不支持的，怎么能够做到使用v7包，然后就能支持且向下兼容的呢？
	
	//系统其实是利用setFactory 把TextView替换成AppCompatTextView
	switch (name) {
	    case "TextView":
	        view = new AppCompatTextView(context, attrs);
	        break;
	    case "ImageView":
	        view = new AppCompatImageView(context, attrs);
	        break;
	    case "Button":
	        view = new AppCompatButton(context, attrs);
	        break;
	    case "EditText":
	        view = new AppCompatEditText(context, attrs);
	        break;
	   //...
	}

但由于AppCompatActivity中调用了setFactory方法，所以当我们调用的时候会有一些影响，比如无法使用替换后的AppCompatTextView，也没有办法使用新特性。

解决方法：在onCreateView中 添加两行代码

	我们可以自己设置factory中，依然可以保证appcompat中创建View的代码的执行。
	AppCompatDelegate delegate = getDelegate();
    View view = delegate.createView(parent, name, context, attrs);

#### 高效统一设置app中所有字体

    LayoutInflaterCompat.setFactory(LayoutInflater.from(this), new LayoutInflaterFactory()
    {
        @Override
        public View onCreateView(View parent, String name, Context context, AttributeSet attrs)
        {
            View view ;
		//如果是继承TextView的自定义的View
        1.//if(name.equals("自定义View") view = new ...
        //或者
        2.if (1 == name.indexOf('.')//表明是自定义View
	        {
	           inflater.createView(name,null,attrs);
	        }
        AppCompatDelegate delegate = getDelegate();
        if(view == null)
          view = delegate.createView(parent, name, context, attrs);


            //可以将布局中所有TextView控件的字体改变
            if (view!=null && (view instanceof TextView)){
                ((TextView) view).setTypeface(Typeface.SERIF);
            }

            return view;
        }
    });

