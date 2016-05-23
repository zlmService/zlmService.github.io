---
title: Android的线程和线程池
date: 2016-05-20 14:19:30
tags: 源自Android开发艺术探索总结
---

### AsyncTask
#### 简介：
是一种轻量级的异步任务类，它可以在线程池中执行后台任务，然后把执行的进度和最终结果传递到主线程，并在主线程中更新UI。AsyncTask内部封装了Thread和Handler，可以方便快捷的执行后台任务以及在主线程中访问UI，但是不适合进行特别耗时的后台任务。

	public abstract class AsyncTask<Params,Progress,Result>
AsyncTask是一个抽象的泛型类，它提供了三个参数，Params代表参数的类型，Progress表示后台任务的执行进度的类型，Result表示返回值类型。如果AsyncTask不需要传递具体的参数时，也可以用Void来代替。

#### 核心方法：
AsyncTask提供了四个核心方法：

- **onPreExecute（）**：在主线程中执行，在异步任务执行之前，此方法调用，用来做一些初始化准备工作。
- **doInBackground（Params...params）**：在线程池总执行，此方法用于执行异步任务。在此方法中可以通过publishProgress方法来更新任务的进度，publishProgress会调用onProgressUpdate（），另外此方法的返回值会调用onPostExecute（）。
- **onProgressUpdate（Progress...values）**：在主线程中执行，当后台任务的执行进度发生改变时此方法会被调用。
- **onPostExecute（Result result）**：在主线程中执行，在异步任务完成之后，会调用这个方法，其中Result参数就是doInBackground（）的返回值。
- **还有onCancelled（）**：在主线程中执行，当异步任务被取消的时候被调用，这个时候onPostExecute（）不会被调用。

#### 注意事项：
- AsyncTask的类必须在主线程中加载，这个过程在Android4.1以上已经被系统自动完成。
- AsyncTask的对象必须在主线程中创建。
- execute方法必须在主线程调用。
- 一个AsyncTask对象只能调用一次execute（）。
- Andorid3.0开始，为避免AsyncTask所带来的并发问题，又采用一个线程来串行执行任务，但是我们可以通过executeOnExecutor（）来并行执行任务。

#### AsyncTask原理：
- 从execute方法开始，它又调用了executeOnExecutor（）方法。

		 public final AsyncTask<Params, Progress, Result> execute(Params... params) {
	        return executeOnExecutor(sDefaultExecutor, params);
	     }
- 上面的sDefaultExecutor其实是一个串行的线程池，一个进程中所有的AsyncTask全部在这个串行的线程池中排队执行

		 public final AsyncTask<Params, Progress, Result> executeOnExecutor
		(Executor exec,Params... params) {
	
			mStatus = Status.RUNNING;
	
	        onPreExecute();
			
	        mWorker.mParams = params;
	        exec.execute(mFuture);
	
	        return this;
		}

- onPreExecute（）方法会首先执行，然后线程池开始执行

		sDefaultExecutor= new SerialExecutor()
	    private static class SerialExecutor implements Executor {
	        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
	        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }
- 上述代码中，系统首先会把AsyncTask的Params参数封装成FutureTask对象，FutureTask是一个并发类，execute（）会把FutureTask对象插入到任务队列mTasks中，如果这个时候没有正在活动的AsyncTask任务，就调用scheduleNext（）方法来执行下一个AsyncTask任务，同时当一个AsyncTask任务执行完以后也会调用scheduleNext（）方法来执行下一个，从这可以看出，AsyncTask是串行执行的。

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }

-从这可以看出，SerialExecutor只是负责用于任务的排队，而真正执行任务的是THREAD_POOL_EXECUTOR这个线程池


    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);//表示当前任务已经被调用，


                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);//然后会执行doInBackground方法

                Binder.flushPendingCommands();
                return postResult(result);//接着会将返回值传递给postResult方法
            }
        };

	//FutureTask
	public void run() {
		  result = c.call();
- 由于FutureTask的call方法会调用call方法，因此mWorker的call方法最终会在线程池中执行


	    private static class InternalHandler extends Handler {
	        public InternalHandler() {
	            super(Looper.getMainLooper());
	        }
	
	        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
	        @Override
	        public void handleMessage(Message msg) {
	            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
	            switch (msg.what) {
	                case MESSAGE_POST_RESULT:
	                    // There is only one result
	                    result.mTask.finish(result.mData[0]);
	                    break;
	                case MESSAGE_POST_PROGRESS:
	                    result.mTask.onProgressUpdate(result.mData);
	                    break;
	            }
	        }
	    }

- 可以发现，InternalHandler是一个静态的Handler对象，由于静态成员会在加载类的时候进行初始化，所以要求AsyncTask的类必须在主线程中加载。当postResult方法会通过Handler发送一个MESSAGE_POST_RESULT消息，然后handler收到这个消息后，会调用AsyncTask的finish方法


	    private void finish(Result result) {
	        if (isCancelled()) {
	            onCancelled(result);
	        } else {
	            onPostExecute(result);
	        }
	        mStatus = Status.FINISHED;
	    }
- 判断AsyncTask是否被取消执行了，如果取消就调用onCancelled（）否则就调用onPostExecute()方法。

### HandlerThread
**简介**：是一种具有消息循环的线程，在它的内部可以使用Handler

	  @Override
	    public void run() {
	        mTid = Process.myTid();
	        Looper.prepare();
	        synchronized (this) {
	            mLooper = Looper.myLooper();
	            notifyAll();
	        }
	        Process.setThreadPriority(mPriority);
	        onLooperPrepared();
	        Looper.loop();
	        mTid = -1;
	    }
- 在run方法中，通过Looper.prepare()方法来创建消息队列，并通过Looper.loop方法来开启消息循环。
- 由于HandlerThread的run方法时一个无限循环的方法的，当确定不使用HandlerThread的时候，就调用它的quit或者quitSafely方法来终止线程的执行。这两个方法内部分别调用的是Looper的quit和quitSafely方法。	

### IntentService
**简介** 内部采用HandlerThread来执行，当任务完成后，就自动关闭。由于它是一个服务，所以它比一般的后台线程优先级要高，所以IntentService比较适合执行一些高优先级的后台任务。

- 当HandlerThread第一次启动时，它的onCreate方法会调用，里面会创建一个HandlerThread，然后使用它的Looper来构造一个Handler对象，这样通过Handler发送的消息就都在HandlerThread中执行了，

	  	@Override
	    public void onCreate() 
	        super.onCreate();
			//先创建一个HandlerThread 他的run方法会创建一个Looper
	        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
	        thread.start();
			//然后获取它里面的Looper对象
	        mServiceLooper = thread.getLooper();
			//根据Looper对象 在创建一个handler
	        mServiceHandler = new ServiceHandler(mServiceLooper);
	    }

- 每执行一个任务，就会启动一次IntentService，然后它的onStartCommand方法就会被调用一次，IntentService在onStartCommand方法中处理每个后台任务的Intent。onStartCommand方法中调用了onStart方法

		 @Override
		    public void onStart(Intent intent, int startId) {
		        Message msg = mServiceHandler.obtainMessage();
		        msg.arg1 = startId;
		        msg.obj = intent;
		        mServiceHandler.sendMessage(msg);
		    }
- 这个方法仅仅是用mServiceHandler发送了一个消息，然后mServiceHandler收到这个消息，会传给onHandlerThread方法去处理。这里的Intent就是startService传过来的Intent对象，在onHandleIntent方法中，可以根据不同的Intent做不同的处理。

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
- 当onHandlerIntent方法执行完以后，会通过stopSelf（int startId）方法停止服务，为什么采用带参数的stopSelf（int startId），因为这个方法不会直接停止服务，而是等待所有的消息都处理完毕后才终止服务，stopSelf（）会立刻停止服务

### 线城池

优点：

1. 重用线程池中的线程
2. 能有效控制线城池中的最大并发数，避免大量线程之间因互相抢占资源而导致阻塞的现象。
3. 能够对线程进行简单的管理，并提供定时执行以及指定间隔循环执行等功能。
	
#### ThreadPoolExeutor
ThreadPoolExecutor是Executor的真正实现类，他的构造方法提供一系列的参数来配置线程池。

	    public ThreadPoolExecutor(int corePoolSize,
	                              int maximumPoolSize,
	                              long keepAliveTime,
	                              TimeUnit unit,
	                              BlockingQueue<Runnable> workQueue,
	                              ThreadFactory threadFactory)
参数：

1. corePoolSize：线程池中的核心线程数，默认情况下，核心线程会一直存活，即便是闲置状态。但是如果将ThreadPoolExecutor的allowCoreThreadTimeOut的属性设置为true以后，那么闲置的核心线程会有一个闲置的时间期限（由keepAliveTime指定），一旦超出这个时间期限，那么核心线程也被会终止。
2. maximumPoolSize：线程池中的最大线程数，当活动线程的数量达到这个值后，后续的任务就会被阻塞。
3. keepAliveTime：非核心线程闲置时间的期限，如果超过这个期限，系统就会回收非核心线程。当allowCoreThreadTimeOut的属性设置为true以后，keepAliveTime指定的时间也作用于核心线程。
4. unit：用于指定keepAliveTime参数时间的单位，是一个枚举（毫秒，秒，分钟等）。
5. workQueue：线程池中的任务队列，通过线程池的execute方法提交的Runnable对象会存储在这个参数中。
6. threadFactory：线程工厂，为线程池提供创建新线程的功能，它是一个接口，只有一个方法：Thread newThread(Runnable r)。

ThreadPoolExecutor的大致流程：
1. 如果线程池中的线程未达到核心线程的数量，那么会直接开启一个核心线程来执行任务，
2. 如果线程中的数量达到或者超过核心线程的数量，那么任务会被插入到任务队列中排队等待执行。
3. 如果在步骤2中无法将任务插入到任务队列中去（可能是任务队列满了），这个时候如果线程数量没有达到线程池中的最大数，那么就开启一个非核心线程去执行这个任务。
4. 在步骤3中 如果线程数量已经达到了线程池中的最大数，那么就拒绝执行此任务，会调用RejectedExeutionHandler的rejectedExcution方法来通知调用者。

#### 线程池的分类
他们都是直接或者间接通过配置ThreadPoolExecutor来实现自己的功能

- FixedThreadPool

	     Executors.newFixedThreadPool(1);通过Executor的newFixedThreadPool方法来创建
		**简介**：它是一种线程数量固定的线程池，里面只有核心线程并且不会被回收，
		另外任务队列也没有大小限制它可以更加快速的响应外界的请求。

- CachedThreadPool

       	Executors.newCachedThreadPool();创建
		**简介**：它是一种线程数量不固定的线程池，它只有非核心线程，并且最大线程数为Integer.MAX_VALUE。也可以说是接近无限大。
		当线程中的线程都处于活动状态时，线程池会创建新的线程来处理新任务，否则就会利用空闲线程来处理任务。
		线程池中的空闲空闲线程有超时机制，超过60秒后的闲置线程就会被回收掉。
		可以把它看做是一个空集合，任何来的任务都可以立即执行，当整个线程池处于闲置状态时，里面的线程都会超时而被停止，
		这个时候它里面是没有线程的，几乎不占任何系统资源。所以它适用于执行大量的耗时较少的任务，

- ScheduledThreadPool

		 Executors.newScheduledThreadPool(1);创建
		核心线程是固定的，而非核心线程是没有闲置的，并且当非核心线程闲置时会立即回收
		这类线程池主要用于执行定时任务和具有固定周期的重复任务。

- SingleThreadExecutor

	    Executors.newSingleThreadExecutor();创建
		内部只有一个核心线程，它确保所有任务都能在同一个线程中按照顺序执行
		它可以统一外界的任务到一个线程中，这使得这些任务之间不需要处理同步问题
