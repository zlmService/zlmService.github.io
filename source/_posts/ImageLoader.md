---
title: ImageLoader
date: 2016-05-07 23:29:51
tags: ImageLoader原理
---
 

- 在application中配置初始化

	ImageLoader.getInstance().init（ImageLoaderConfiguration configuration）
	getInstance 是一个懒汉式单例模式，获取ImageLoader的对象 双层是否为 null 判断提高性能
	
	init的作用就是 开启一个引擎，也就是任务分发器，负责分发 是加载并显示图片任务 又或者是 处理并显示图片任务 给具体的线程池去执行，


- 加载图片时

	配置完以后 我们在需要加载图片的地方 调用
    ImageLoader.getInstance().displayImage(image, holder.img_profile);
	先通过这一句话来进行加载图片  
	
	它是通过一个displayImage的方法 进行加载图片  这个方法有很多重载
	它们都是调用了一个最全参数的displayImage  这个方法参数6个参数 分别是
	图片的路径，用来显示图片的控件，在application中定义的DisplayImageOptions的，图片加载的监听方法，图片的宽高，图片加载进度的回调接口


- 判断参数 ，并且内存中有这张图片

	这个方法的内部会先判断 这些参数 哪些为null，如果image为null 就抛非法参数异常，判断图片的uri是否为null，如果为null 就用之前配置的uri为null的占位图来展示出来，如果不为null 它会先从内存中获取图片的信息  看看内存里面是否有这张图片， 通过根据图片的uri来生成图片在内存中存的key，来取出key对应的图片信息，如果图片不为null 并且它没有被回收掉，开启一个处理并显示图片的任务（ProcessAndDisplayImageTask）他就会将图片按照选项配置的大小，缩放等等信息进行处理 最终 显示到指定的控件中，


- 如果内存中没有
	
	他会先显示一张 正在加载的占位图
	然后开启一个 加载并显示图片的任务（LoadAndDisplayImageTask）他其实就是一个继承Runnable的类


- LoadingAndDisplayTask

	在它的run方法里 他又判断了一遍
	在判断一遍 内存中是否有这个图片（防止多线程并发） 
	然后 他又调用了 tryLoadBitmap方法


- tryLoadBitmap

	他会 通过 uri 获取 磁盘上的文件
	如果这个文件不为null并且存在 就把这个文件 处理 解码后 变成一个bitmap对象
	
	如果这个内存卡上也没有这个文件
	就通过uri网络加载这个图片
	
	然后根据uri获取一个inputStream流 把图片下载下来，根据配置的大小信息等，如果图片需要缓存
	就让uri+图片的宽高当file的名字 来生成一个file文件 就把这个文件存储到内存卡上
	如果根据uri获取的inputStream 是一个null的  就显示一个 网络加载失败的 占位图

	最后根据存储的这个文件 获取他的 路径 然后 处理解码 变成一个bitmap对象 最终显示到控件上去
	
#### 总的来说 ImageLoader分成三种情况 

		第一种是 直接在内存中获取到图片然后进行图片处理 显示到控件上
		第二种 在内存中获取不到，在内存卡中进行获取，然后得到文件路径 进行 解码处理 变成bitmap对象 显示到控件上
		第三种 内存和内存卡都获取不到，就从网路上进行加载 如果需要缓存，就用uri+图片宽高当file文件的名字 然后存储到内存卡中， 然后通过路径 获取这个图片 进行解码处理 变成bitmap对象 显示到控件上
	
----



MemoryCache的key的样式

	[imageUri]_[width]x[height]

### 加载图片的流程

1、displayImage(uri,ImageView)

- Uri 图片的路径
- ImageAware 用来显示图片的控件，相当于Imageview
- DisplayImageOptions 在application中定义的那些显示的选项，比如说错误或者加载中的占位图等
- ImageLoadingListener 图片加载中的监听方法
- Imagesize 图片的宽高，如果不设置，默认为Imageview的宽高
- ImageLoadingProgressListener  用来加载图片的进度信息

	public void displayImage(String uri, ImageAware imageAware, DisplayImageOptions options,
			ImageSize targetSize, ImageLoadingListener listener, ImageLoadingProgressListener progressListener) {
		checkConfiguration();
		if (imageAware == null) {
			throw new IllegalArgumentException(ERROR_WRONG_ARGUMENTS);
		}
		if (listener == null) {
			listener = defaultListener;
		}
		if (options == null) {
			options = configuration.defaultDisplayImageOptions;
		}

		if (TextUtils.isEmpty(uri)) {
			//先判断图片的uri是否为空
			engine.cancelDisplayTaskFor(imageAware);
			listener.onLoadingStarted(uri, imageAware.getWrappedView());
			if (options.shouldShowImageForEmptyUri()) {
				//如果为空，就看是否配置的有占位图
				imageAware.setImageDrawable(options.getImageForEmptyUri(configuration.resources));
			} else {
				imageAware.setImageDrawable(null);
			}
			listener.onLoadingComplete(uri, imageAware.getWrappedView(), null);
			return;
		}

		if (targetSize == null) {
			//设置图片的宽高信息
			targetSize = ImageSizeUtils.defineTargetSizeForView(imageAware, configuration.getMaxImageSize());
		}
		

		//重点：首先从内存中获取该图片信息
		//第一步，根据图片的uri来生成图片在内存中存的key，默认情况下key是唯一的
		String memoryCacheKey = MemoryCacheUtils.generateKey(uri, targetSize);
		engine.prepareDisplayTaskFor(imageAware, memoryCacheKey);

		listener.onLoadingStarted(uri, imageAware.getWrappedView());
		//第二步 根据上面的key获取内存中的图片信息
		Bitmap bmp = configuration.memoryCache.get(memoryCacheKey);
		if (bmp != null && !bmp.isRecycled()) {
			//如果图片不为空并且图片没有被回收的情况下
			L.d(LOG_LOAD_IMAGE_FROM_MEMORY_CACHE, memoryCacheKey);

			if (options.shouldPostProcess()) {
				//接着就是用来处理并显示图片到Imageview中
				ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,
						options, listener, progressListener, engine.getLockForUri(uri));
				ProcessAndDisplayImageTask displayTask = new ProcessAndDisplayImageTask(engine, bmp, imageLoadingInfo,
						defineHandler(options));
				if (options.isSyncLoading()) {
					displayTask.run();
				} else {
					engine.submit(displayTask);
				}
			} else {
				options.getDisplayer().display(bmp, imageAware, LoadedFrom.MEMORY_CACHE);
				listener.onLoadingComplete(uri, imageAware.getWrappedView(), bmp);
			}
		} else {
			//内存中不存在该图片的情况吧
			if (options.shouldShowImageOnLoading()) {
				//显示加载中的占位图
				imageAware.setImageDrawable(options.getImageOnLoading(configuration.resources));
			} else if (options.isResetViewBeforeLoading()) {
				imageAware.setImageDrawable(null);
			}
			
			ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,
					options, listener, progressListener, engine.getLockForUri(uri));
			//用来加载并显示图片的task
			LoadAndDisplayImageTask displayTask = new LoadAndDisplayImageTask(engine, imageLoadingInfo,
					defineHandler(options));
			if (options.isSyncLoading()) {
				displayTask.run();
			} else {
				engine.submit(displayTask);
			}
		}
	}

### LoadingAndDisplayTask

	@Override
	public void run() {
		if (waitIfPaused()) return;
		if (delayIfNeed()) return;

		ReentrantLock loadFromUriLock = imageLoadingInfo.loadFromUriLock;
		L.d(LOG_START_DISPLAY_IMAGE_TASK, memoryCacheKey);
		if (loadFromUriLock.isLocked()) {
			L.d(LOG_WAITING_FOR_IMAGE_LOADED, memoryCacheKey);
		}

		loadFromUriLock.lock();
		Bitmap bmp;
		try {
			checkTaskNotActual();
			//双重判断，继续判断内存中该图片信息是否存在
			bmp = configuration.memoryCache.get(memoryCacheKey);
			if (bmp == null || bmp.isRecycled()) {
				//如果图片不存在或者图片被回收
				//重点方法
				bmp = tryLoadBitmap();

				if (bmp == null) return; // listener callback already was fired

				checkTaskNotActual();
				checkTaskInterrupted();

				if (options.shouldPreProcess()) {
					L.d(LOG_PREPROCESS_IMAGE, memoryCacheKey);
					bmp = options.getPreProcessor().process(bmp);
					if (bmp == null) {
						L.e(ERROR_PRE_PROCESSOR_NULL, memoryCacheKey);
					}
				}

				if (bmp != null && options.isCacheInMemory()) {
					L.d(LOG_CACHE_IMAGE_IN_MEMORY, memoryCacheKey);
					configuration.memoryCache.put(memoryCacheKey, bmp);
				}
			} else {
				loadedFrom = LoadedFrom.MEMORY_CACHE;
				L.d(LOG_GET_IMAGE_FROM_MEMORY_CACHE_AFTER_WAITING, memoryCacheKey);
			}

			if (bmp != null && options.shouldPostProcess()) {
				L.d(LOG_POSTPROCESS_IMAGE, memoryCacheKey);
				bmp = options.getPostProcessor().process(bmp);
				if (bmp == null) {
					L.e(ERROR_POST_PROCESSOR_NULL, memoryCacheKey);
				}
			}
			checkTaskNotActual();
			checkTaskInterrupted();
		} catch (TaskCancelledException e) {
			fireCancelEvent();
			return;
		} finally {
			loadFromUriLock.unlock();
		}

		DisplayBitmapTask displayBitmapTask = new DisplayBitmapTask(bmp, imageLoadingInfo, engine, loadedFrom);
		runTask(displayBitmapTask, syncLoading, handler, engine);
	}

#### tryLoadBitmap()
  	```java
	//先从文件系统中获取该图片文件
	File imageFile = configuration.diskCache.get(uri);
			if (imageFile != null && imageFile.exists() && imageFile.length() > 0) {
				L.d(LOG_LOAD_IMAGE_FROM_DISK_CACHE, memoryCacheKey);
				//如果文件系统中存在的话，就从内存卡上加载该图片并显示
				loadedFrom = LoadedFrom.DISC_CACHE;

				checkTaskNotActual();
				//把文件信息转变成bitmap
				bitmap = decodeImage(Scheme.FILE.wrap(imageFile.getAbsolutePath()));
			}
			if (bitmap == null || bitmap.getWidth() <= 0 || bitmap.getHeight() <= 0) {
				//如果bitmap不存在
				L.d(LOG_LOAD_IMAGE_FROM_NETWORK, memoryCacheKey);
				//就从网络上加载图片信息
				loadedFrom = LoadedFrom.NETWORK;

				String imageUriForDecoding = uri;
				//重点方法 tryCacheImageOnDisk  缓存图片到内存卡上
				if (options.isCacheOnDisk() && tryCacheImageOnDisk()) {
					//下载完并且存储好之后再次获取内存卡上的图片信息
					imageFile = configuration.diskCache.get(uri);
					if (imageFile != null) {
						imageUriForDecoding = Scheme.FILE.wrap(imageFile.getAbsolutePath());
					}
				}

				checkTaskNotActual();
				bitmap = decodeImage(imageUriForDecoding);
				//此处表明内存、内存卡、以及网路上都不存在该图片信息
				//回调图片加载失败的监听方法
				if (bitmap == null || bitmap.getWidth() <= 0 || bitmap.getHeight() <= 0) {
					fireFailEvent(FailType.DECODING_ERROR, null);
				}
			}
	```

### tryCacheImageOndisk

	private boolean tryCacheImageOnDisk() throws TaskCancelledException {
		L.d(LOG_CACHE_IMAGE_ON_DISK, memoryCacheKey);

		boolean loaded;
		try {
			//下载图片
			loaded = downloadImage();
			if (loaded) {
				int width = configuration.maxImageWidthForDiskCache;
				int height = configuration.maxImageHeightForDiskCache;
				if (width > 0 || height > 0) {
					L.d(LOG_RESIZE_CACHED_IMAGE_FILE, memoryCacheKey);
					//此处是根据设置的大小，改变图片要缓存到内存卡上的大小并且把图片缓存下来
					resizeAndSaveImage(width, height); // TODO : process boolean result
				}
			}
		} catch (IOException e) {
			L.e(e);
			loaded = false;
		}
		return loaded;
	}

### DownloadImage

	private boolean downloadImage() throws IOException {
		//根据图片的uri获取图片的流，然后存储图片到内存卡上
		InputStream is = getDownloader().getStream(uri, options.getExtraForDownloader());
		if (is == null) {
			L.e(ERROR_NO_IMAGE_STREAM, memoryCacheKey);
			return false;
		} else {
			try {
				return configuration.diskCache.save(uri, is, this);
			} finally {
				IoUtils.closeSilently(is);
			}
		}
	}

### ImageDownloader -> 实现 BaseImageDownloader获取图片流的具体东西


