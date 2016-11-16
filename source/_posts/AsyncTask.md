---
title: AsyncTask解析
date: 2016-07-29 15:44:19
tags: [Android]
---

# AsyncTask用法

```java
    private class SaveImageTask extends AsyncTask<Bitmap, String, String> {
        @Override
        protected void onPreExecute() {
            Helper.showToast("正在分享，请稍等...");
        }
        
        @Override
        protected String doInBackground(Bitmap... params) {
            //耗时操作
            return String; 
        }

        @Override
        protected void onPostExecute(String result) {
            super.onPostExecute(result);
        }
    }
    
    //使用
    new SaveImageTask().execute(loadedImage);
```

# 源码分析

AsyncTask类总共600行左右，大概看下内部实现。初始化：

```java
    private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }
    
    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
```

初始化了两个变量，一个Callable mWorker，一个FutureTask mFuture，FutureTask实现了Runnable和Future。将mWorker作为参数传递给了mFuture的构造方法，关于FutureTask和Callable，简单的来说就是Future里面封装了Callable，执行Future实际上执行的是Callable的call方法，Future的get()可以从Callable拿到执行的结果。Callable的Call()会去执行抽象方法doInBackground(mParams);并把传进来的mParams带过去，doInBackground需要我们自己实现，去做一些耗时的操作。然后拿到结果执行postResult(result)方法，这个待会再说。继续往下看

```java
FutureTask的run方法
public void run() {
        try {
            //构造方法里的Callable参数
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);//可以通过get()拿到结果
            }
        } finally {
        }
    }
```

执行AsyncTask的方法

```java
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }

    mStatus = Status.RUNNING;

    onPreExecute();

    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}
```

每个任务在完成前只能执行一次，然后执行onPreExecute();的抽象方法，在使用的AsyncTask中实现做一些准备操作，然后将传进来的params参数付给mWorker的mParams，还记得mWorker吗，是一个Callable。exec.execute(mFuture);实际上被调用的是mWorker的call()。这里有个默认的Executor sDefaultExecutor，看下实现

```java
    
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
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

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
    
```

mTasks是一个先进先出的队列存储Runnable对象，offer方法加到队尾，poll()从队头取。第一次mActive肯定是null，所以走到scheduleNext取出一个用THREAD_POOL_EXECUTOR去执行。下一次再调用execute()，这是mActive不为空，所以就不会执行scheduleNext()，但是由于有try finally的存在，所以下一次scheduleNext();是在本次run方法执行完，也就是说要等待本次耗时操作执行完才可以进行下一次耗时操作。也对应了SerialExecutor这个名字，串行执行。实际的Executor是THREAD_POOL_EXECUTOR，看下实现
    
```java
    
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    private static final int KEEP_ALIVE = 1;

    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
```

定义了一个线程池，同时运行线程数CPU数+1，线程池总大小CPU数 * 2 + 1，在之前的版本这两个数字分别是5和128。虽然定义了可以同时运行那么多线程，但是由于SerialExecutor的存在，它会强制串行并发，所以实际上只有一个线程在跑，所以也就不存在任务数超过线程池总大小会蹦的问题了。SerialExecutor是AsyncTask提供给开发者的一种默认实现，我们也可以通过``public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,Params... params)``方法传进去一个自己定义的线程池，这样就可以并行并发了。

刚才说到最终的执行时在mWorker的call()去执行具体的耗时操作，执行完了调用postResult()方法，看下实现

```java
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```

这就很明了了，通过Handler把result结果发出去。看下Handler的实现

```java

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
```

InternalHandler是一个主线程上的Handler，也就是发消息到主线程，刚才发过来的Result就被发送的主线程了，最后调用AsyncTask的finish方法，看下

```java
    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```

如果取消了执行onCancelled(result)回调，否则执行onPostExecute(result)。并把状态置为FINISHED。整个流程也就走完了。

# 总结

本文分析了AsyncTask的原理，一句话概括，AsyncTask封装了线程池和Handler，线程池跑耗时任务、Handler向主线程发消息。如果不是很多任务的话就用HandlerThread来做就行了；任务多并且不是那么耗时的可以考虑用用AsyncTask，不过还是建议自己写线程池。

