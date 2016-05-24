---
title: Bitmap的加载和Cache
date: 2016-05-23 22:44:22
tags: 源自Android开发艺术探索总结
---
### Bitmap的高效加载
**为什么采取高效加载**：因为当我们需要显示的图片远远小于图片本身的大小的时候，就会很浪费内存资源，为了提高Bitmap的性能，我们需要通过BitmapFactory.Options可以按照一定的采样率来加载缩小后的图片，这样一定程度上避免了OOM。

**如何获取采样率**：

1. 将BitmapFactory.Options的inJustDecodeBounds参数设置为true并加载图片
2. 从Options中获取图片原始宽高
3. 结合目标View的所需大小计算出采样率inSampleSize
4. 将inJustDecodeBounds参数设置为false，重新加载图片

		//Bitmap高效加载
	    public Bitmap BitmapCache(Resources res, int resId, int reqWidth, int reqHeight) {
	        BitmapFactory.Options options = new BitmapFactory.Options();
	        options.inJustDecodeBounds = true;//为true时，代表只会解析图片的宽和高，并不会真正的加载图片
	        BitmapFactory.decodeResource(res, resId, options);//解析图片
	        //设置完缩放比例后 在解析图片
	        options.inSampleSize = calculateInSampleSize(options, reqHeight, reqWidth);
	        options.inJustDecodeBounds=false;
	        //最后返回解析后的Bitmap
	        return BitmapFactory.decodeResource(res,resId,options);
	    }
	    //设置缩放比例
	    private int calculateInSampleSize(BitmapFactory.Options options, int reqHeight, int reqWidth) {
	        int inSampleSize = 1;
	        int width = options.outWidth;//从BitmapFactory.Options中取出图片的原始宽高
	        int height = options.outHeight;
	        //根据传过来的参数（需要展现的宽高）和图片实际的宽高进行对比
	        if (width > reqWidth || height > reqHeight) {
	            int halfWidth = width / 2;
	            int halfHeight = height / 2;
	            while ((halfHeight / inSampleSize) >= reqHeight && (halfWidth / inSampleSize) >= reqWidth) {
	                inSampleSize *= 2; // 缩放比例一般都是2的倍数存在
	            }
	        }
	        return inSampleSize;
	    }

### Bitmap的缓存策略
缓存减少用户流量的使用，同时大大加快了应用的流畅度。
一般采用2级缓存，当用户需要访问一张图片时，首先会去内存中找，如果没有就去存储设备上找，还没有就去网络上加载这张图片。
缓存机制内部通过LRU(最近最少)算法来完成,它分为LruCache和DiskLruCache分别对应内存缓存和储存设备缓存。

#### LruCache
是一个泛型类，内部采用以个LinkedHashMap存储外界缓存对象。 

        //获取当前进程的大小
        int maxMemory= (int) (Runtime.getRuntime().maxMemory()/1024);
        //当前进程的8分之一当做缓存的大小
        int cacheSize=maxMemory/8; //单位为KB
        LruCache<String,Bitmap> bitmapLruCache=new LruCache<String,Bitmap>(cacheSize){
            @Override
            //sizeof方法作用是 计算缓存对象的大小
            protected int sizeOf(String key, Bitmap bitmap) {
                return bitmap.getRowBytes()*bitmap.getHeight()/1024;
            }
            @Override
			//LruCache移除旧缓存时会调用，在里面完成一些资源回收的工作
            protected void entryRemoved(boolean evicted, String key, Bitmap oldValue, Bitmap newValue) {
                super.entryRemoved(evicted, key, oldValue, newValue);
            }
        };
		
		
		bitmapLruCache.get(key)//获取缓存对象
		bitmapLruCache.put(key,bitmap)//添加缓存对象
		bitmapLruCache.remove(key)//删除缓存对象

### DiskLruCache
用于存储设备缓存，它通过将缓存对象写入文件系统从而实现缓存的效果。
#### DiskLruCache的创建
1. 通过open方法创建自身

        File cacheDir = getCacheDir();
        if (cacheDir.exists()) {
            cacheDir.mkdirs();
        }
            /**
             * 参数：1.表示磁盘存储存放文件的路径
             *       2.表示应用的版本号，设置为1即可，当版本号发生改变的时候，会清空之前所有的缓存
             *       3.表示单个节点对应的数据的个数，设置为1即可
             *       4.缓存的总大小。
             */
        mDiskLruCache = mDiskLruCache.open(cacheDir, 1, 1, DISK_CACHE_SIZE);
		
2. 通过Editor完成缓存的添加操作

        DiskLruCache.Editor editor = mDiskLruCache.edit(MyUtils.hashKeyFormUrl(url));
        if (editor != null) {
            //由于open方法中设置了一个节点只能有一个数据，所以常量设置为0即可。
            OutputStream outputStream = editor.newOutputStream(DISK_CACHE_INDEX);
			//下载图片 下载成功后返回tru
            if (MyUtils.downUrlToStream(url, outputStream)) {
                editor.commit();//通过这个方法 才真正的存储到手机里
            } else {
                //如果下载过程出现异常，可以通过这个方法进行回退整个操作
                editor.abort();
            }
            mDiskLruCache.flush();

3. 通过Snapshot完成缓存的查找

        DiskLruCache.Snapshot snapshot=mDiskLruCache.get(key);
            if (snapshot!=null){
                FileInputStream fileInputStream= (FileInputStream) snapshot.getInputStream(DISK_CACHE_INDEX);
                FileDescriptor fileDescriptor=fileInputStream.getFD();
                bitmap=mImageResizer.decodeSampledBitmapFromFileDescriptor(fileDescriptor,200,200);
                if (bitmap!=null){
                    imageView.setImageBitmap(bitmap);
                }