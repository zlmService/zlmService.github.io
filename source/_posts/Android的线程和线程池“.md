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

