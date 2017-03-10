**Android的线程和线程池**

[TOC]


从用途上分，线程分为主线程(主要处理界面和界面相关的事情)和子线程(子线程则往往用于执行耗时操作)。

**从使用场景分：**
1. AsyncTask封装了线程池和Handler，它主要是为了方便开发者在子线程中更新UI。
2. HandlerThread是一种具有消息循环的线程，它内部可以使用Handler。
3. IntentService是一个服务，系统对其进行封装使其可以更方便地执行后台任务，IntentService内部采用HandlerThread来执行任务，当任务执行完毕后IntentService会自动退出。IntentService作用很像一个后台线程，但是IntentService是一种服务，它不容易被系统杀死从而可以尽量保证任务的执行，而如果是一个后台线程，由于这个时候进程中没有活动的四大组件，那么这个进程的优先级就会非常低，会容易被系统杀死，这就是IntentService的有点。

@线程在操作系统调度是最小的单元，同时线程又是一种受限的系统资源（线程不可能无限制地产生，并且线程的创建和销毁都会有相应的开销）

# 主线程和子线程

主线程是指进程所拥有的线程，在Java中默认情况下一个进程只有一个线程，这个线程就是主线程。作用是运行四大组件以及处理它们和用户的交互。
子线程也叫工作线程，除了主线程以外的线程都是子线程。作用是执行耗时任务，比如网络请求、I/O操作等。
NetworkOnMainThreadException是从Android3.0开始需要声明的，主要为了避免主线程由于被耗时操作所阻塞从而出现ANR现象。


# Android的线程形态

### AsyncTask

AsyncTask是一种轻量级的异步人物类，它可以在线程中执行后台任务，然后把执行的进度和记过传递给主线程并在主线程中更新UI。（AsyncTask封装了Thread和Handler，通过AsyncTask可以更加方便地执行后台任务以及在线程中访问UI，但AsyncTask并不适合进行特别耗时的后台任务，对于特别耗时的任务来说，建议使用线程池。）

`Public abstract class AsyncTask<Params, Progress, Result>。`

4个核心方法：
1. onPreExecute()，在主线程中执行，在异步任务执行之前，此方法会被调用，一般可以用于做一些准备工作。
2. doInBackGround(Params… params)，在线程池中执行，此方法用于执行异步任务，params参数表示异步任务的输入的参数。在此方法中可以通过publishProgress方法来更新任务的进度，publishProgress方法会调用onProgressUpdate方法。另外此方法需要返回计算结果给onPostExecute方法。
3. onProgressUpdate(Progress … values)，在主线程中执行，当后台任务的执行进度发生改变时此方法会被调用。
4. onPostExecute(Result result)，在主线程中执行，在异步任务执行之后，此方法会被调用，其中result参数是后台任务的返回值，即doInBackground的返回值。

**AsyncTask在具体的使用过程中也是有一些条件限制的，主要有如下几点：**

1. AsyncTask的类必须在主线程中加载。
Android4.1及以上版本已经被系统自动完成。在Android5.0的源码中，在ActivityThread的main方法，它会调用AsyncTask的init方法。
2. AsyncTask的对象必须在主线程中创建
3. Execute方法必须在UI线程调用。
4. 不要在程序中直接调用onPreExecute()、onPostExecute、doInBackground和onProgressUpdate方法。
5. 一个AsyncTask对象只能执行一次，即只调用一次execute，否则会报运行异常。
6. 在Android1.6之前，AsyncTask是串行执行任务的，Android1.6的时候AsyncTask开始采用线程池处理并行任务，但是从Android3.0开始，为了避免AsyncTask所带来的并发错误，AsyncTask又采用一个线程来串行执行任务。但在Android3.0过后的版本，可以通过AsyncTask的executeOnExecutor方法来并行执行任务。

### AsyncTask的工作原理

从execute方法开始，execute方法会调用executeOnExecutor方法。
```
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
    mStatus = Status.RUNNING;（保存状态）
    onPreExecute();（最先执行）
    mWorker.mParams = params;
    exec.execute(mFuture);（线程池执行）
    return this;
}
```
sDefaultExecutor是一个串行的线程池，一个进程中所有的AsyncTask全部在这串行的线程池中排队执行。onPreExecute方法最先被执行，然后线程池开始执行。
```
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
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
SerialExecutor的实现可以分析AsyncTask的排队执行过程，默认情况下AsyncTask是串行执行的。
AsyncTask中有两个线程池（SerialExecutor和THREAD_POOL_EXECUTOR）和一个Handler（InternalHandler），其中线程池SerialExecutor用于任务的排队，而线程池THREAD_POOL_EXECUTOR用于真正地执行任务，InternalHandler用于将执行环境 从线程池切换到主线程，本质是线程的调用过程。
```
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
```
mWorker的call方法中，首先将mTaskInvoked设为true，表示当前任务已经被调用过了，然后在执行AsyncTask的doInBackground方法，接着讲其返回值传递给postResult方法。
```
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```
postResult方法会通过mHandler发送一个MESSAGE_POST_RESULT的消息。
```
private static InternalHandler sHandler;

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
sHandler是一个静态的Handler对象，为了能够将执行环境切换到主线程，sHandler这个对象必须在主线程中chuangjian 。要求AsyncTask的类必须在主线程加载，否则同一个进程中的AsyncTask都将无法正常工作。sHandler收到MESSAGE_POST_RESULT这个消息后会调用AsyncTask的finish方法。
```
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```
@ Android4.1.1上执行，串行执行。

@ Android2.3.3上执行，并行执行。

@ 为了让AsyncTask可以在Android3.0及以上的版本上并行，可以采用AsyncTask的executeOnExecutor方法（并行执行），需要注意的是这个方法Android3.0新添加的方法，并不能在低版本上使用。

### HandlerThread

HandlerThread继承Thread，它是一种可以使用Handler的Thread。实现在run方法中通过Looper.prepare()来创建消息队列，并且通过Looper.loop()来开启消息循环，这样就允许HandlerThread中创建Handler了。
@由于HandlerThread的run方法是一个无限循环，因此当明确不需要再使用HandlerThread时，可以通过它的quit或quitSafely方法来终止线程的执行。（良好习惯）

### IntentService

IntentService是一种特殊的Service，它集成Service并且它是一个抽象类。IntentService可用于执行后台耗时的任务，当任务执行后它会自定停止。IntentService比较适合执行一些高优先级的后台任务。

IntentService封装了HandlerThread和Handler。
```
public void onCreate() {
    // TODO: It would be nice to have an option to hold a partial wakelock
    // during processing, and to have a static startService(Context, Intent)
    // method that would launch the service & hand off a wakelock.

    super.onCreate();
    HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
    thread.start();

    （1）mServiceLooper = thread.getLooper();
    （2）mServiceHandler = new ServiceHandler(mServiceLooper);
}
```
mServiceHandler发送的消息最终都会在HandlerThread中执行。IntentService在onStartCommand中处理每一个后台任务的Intent。
```
public void onStart(Intent intent, int startId) {
    Message msg = mServiceHandler.obtainMessage();
    msg.arg1 = startId;
    msg.obj = intent;
    mServiceHandler.sendMessage(msg);
}
```
IntentService仅仅是通过mServiceHandler发送一个消息，这个消息会在HandlerThread中被处理。mServiceHandler收到消息后，会将Intent对象传递给onHandlerIntent方法去处理。采用stopSelf(int startId)相等而不是stopSelf()来停止服务，那是因为stopSelf()会立即停止服务，而这个时候可能还有其他消息未处理，stopSelf(int startIf)则会等待所有的消息都处理完毕才终止服务。（一般来说，stopSelf(int startId)在尝试停止服务之前会判断最近启动服务的次数是否和startId相等，如果相等就立即停止服务，不相等则不停止服务。）
```
private final class ServiceHandler extends Handler {
    public ServiceHandler(Looper looper) {
        super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
        onHandleIntent((Intent)msg.obj);（处理IntentService传递的消息）
        stopSelf(msg.arg1);
    }
}
```

# Android的线程池

线程池的有点：
1. 重用线程池中的线程，避免因为线程的创建和销毁所带来的性能开销。
2. 能有效控制线程池的最大并发数，避免大量的线程之间因互相互抢占系统资源而导致的阻塞现场。
3. 能够对线程进行简单的管理，并提供定时执行以及指定间隔循环执行等功能。
Android中的线程池的概念是来源于Java中的Executor，Executor是一个接口，真正的线程池的实现为ThreadPoolExecutor。由于都是直接或间接通过配置ThreadPoolExecutor来实现的。

### ThreadPoolExecutor

常用的构造方法分析：

`public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory)`

1. corePoolSize

线程池的核心线程数，默认情况下，核心线程会在线程池中一直存活，即时它们处于闲置状态。如果将ThreadPoolExecutor的allowCoreThreadTimeOut属性设置为true，那么闲置的核心线程在等待新任务到来时会有超时策略，这个时间间隔由keepAliveTime所指定，当等待时间超过keepAliveTime所指定的时长后，核心线程就会被终止。

2. maximunPoolSize

线程池所能容纳的最大线程数，当活动线程数达到这个数值后，后续的新任务将会被阻塞。

3. keepAliveTime

非核心线程限制时的超时时长，超过这个时长，非核心线程就会被回收。当ThreadPoolExecutor的allowCoreThreadTimeOut属性设置为true，keepAliveTime同样会作用于核心线程。

4. unit(枚举)

用于指定keepAliveTime参数的时间单位，当用的有TimeUnit.MILLSECONDS(毫秒)、TimeUnit.SECONDS(秒)以及TimeUnit.MINUTES(分钟)等。

5. workQueue

线程中的任务队列，通过线程池的execute方法提交的Runnable对象会存在在这个参数中。

6. threadFactory

线程工厂，为线程池提供创建新线程的功能。ThreadFactory是一个接口，它只有一个方法：Thread new Thread(Runnable r)；


**ThreadPoolExecutor执行任务时大致遵循如下规则：**
1. 如果线程池中的线程数量未达到核心线程的数量，那么会启动一个核心线程来执行任务。
2. 如果线程池中的线程数量已经达到或超过核心线程的数量，那么任务会被插入到任务队列中排队等待执行。
3. 如果再步骤2中无法将任务插入到任务队列中，这往往是由于任务队列已满，这个时候如果线程数量未达到线程池规定的最大值，那么会立即启动一个非核心线程来执行任务。
4. 如果步骤3中的线程已经达到线程池规定的最大值，那么就会拒绝执行任务，ThreadPoolExecutor会调用RejectedExecutionHandler的rejectedExecution方法来通知调用者。

AsyncTask中线程池的配置情况：
```
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

/**
 * An {@link Executor} that can be used to execute tasks in parallel.
 */
public static final Executor THREAD_POOL_EXECUTOR
        = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
```
配置后的线程池规则如下：
1. 核心线程数等于CPU核心数+1
2. 线程池的最大线程数为CPU核心数的2倍+1
3. 核心线程无超时机制，非核心线程在限制时的超时时间为1秒
4. 任务队列的容量为128

### 线程池的分类

线程池分为四类，FixedThreadPool、CachedThreadPoolExecutor、ScheduledThreadPool以及SingleThreadExecutor。

1. FixedThreadPool

通过Executors的newFixedThreadPool方法来创建。它是一种线程数量固定的线程池，当线程处于空闲状态是，它们并不会被回收，除非线程池被关闭。当所有线程都处于活动状态时，新任务都会处于等待状态，直到有线程空闲出来。由于FixedThreadPoolExecutor只有核心线程并且这些核心线程不会被回收，它能更加快速的响应外界的请求。（核心线程没有超时机制，任务队列也没有大小限制。）

```
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

2. CachedThreadPoolExecutor

通过Executors的newCachedThreadPool方法来创建。它是一种线程数不定的线程池，它只有非核心线程，并且最大线程数为Integer.MAX_VALUE（任意大）。当线程池处于活动状态时，线程池会创建新的线程来处理新任务，否则就会利用空闲的线程来处理新任务。它的空闲线程都有超时机制（60秒）。相比FixedThreadPool，不同在于CachedThreadPool的任务队列其实相当于一个空集合。根据特性分析，它比较适合执行大量耗时较少的任务。当整个线程池都处于限制状态是，线程池中的线程都会超时而被停止，实际上是没有任何线程的，它几乎是不占用任何系统资源的。

```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

3. ScheduledThreadPool

通过Executors的newScheduledThreadPool方法来创建。它的核心线程数量是固定的，而非核心线程数是没有限制的，并且当非核心线程闲置时会立即回收。主要用于执行定时任务和固定周期的宠物任务。

```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}

public static ScheduledExecutorService newScheduledThreadPool(
        int corePoolSize, ThreadFactory threadFactory) {
    return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
}
```

4. SingleThreadExecutor

通过Executors的newSingleThreadExecutor方法来创建。这类线程池内部只有一个核心线程，它确保所有任务都在同一个线程中按顺序执行。意义在于统一所有的外界任务到一个线程中，任务之间不需要处理线程同步的问题。

```
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

系统配置的4中线程池的典型使用方法：

```
Runnable commond = new Runnable() {
    @Override
    public void run() {
        SystemClock.sleep(2000);
    }
};

ExecutorService fixedThreadPool = Executors.newFixedThreadPool(4);
fixedThreadPool.execute(commond);

ExecutorService cachedThreadPool = Executors.newCachedThreadPool();
cachedThreadPool.execute(commond);

ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(4);
// 2000ms后执行command
scheduledThreadPool.schedule(commond, 2000, TimeUnit.MILLISECONDS);
// 延迟10ms后，每隔1000ms执行一次commond
scheduledThreadPool.scheduleAtFixedRate(commond, 10, 1000, TimeUnit.MILLISECONDS);

ExecutorService singleThreadExecutor = Executors.newSingleThreadExecutor();
singleThreadExecutor.execute(commond);
```
