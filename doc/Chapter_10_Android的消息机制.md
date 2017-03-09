**Android的消息机制**

[TOC]

> Android的消息机制主要是指Handler的运行机制，Handler的运行需要底层的MessageQueue和Looper的支撑。
MessageQueue是消息队列，它的内部存储了一组消息，以队列的形式对外提供插入和删除的工作，它并不是真的队列，采用单链表的数据结构来存储消息队列。
Looper是消息循环，因为MessageQueue只是一个消息的存储单元，它不能去处理消息，而Looper就填补了这个功能，Looper会以无限循环的形式去查找是否有新消息，如果有的话就处理消息，否则就等待，Looper是ThreadLocal（不是线程），它的作用是可以在每个线程中存储数据。
ThreadLocal可以在不同的线程中互不干扰地存储并提供数据，通过ThreadLocal可以轻松获取每一个线程的Looper。默认情况下没有Looper的，如果需要使用Hanlder就必须为线程创建Looper。（由于ActivityThread被创建就会初始化Looper，这就是主线程中默认可以使用Handler的原因。）


# Android的消息机制的概述

Handler的主要作用是将一个任务切换到某个特定的线程中去执行。Android规定访问ＵＩ只能在主线程中进行，如果在线程中访问UI，那么程序就会抛出异常。

```
源码解析：
void checkThread(){
	if(mThread != Thread.currentThread()){
		throw new CalledFromWrongThreadException(“Only the original thread that created a view hierarchy can touch its views.”);
	}
}
```

因为有上面的限制，导致必须在主线程中访问UI，但是Android又建议不要在主线程中进行耗时操作，否则会导致程序无法响应ANR。

Android的UI控件不是线程安全的，如果再多线程中并发访问可能会导致UI处于不可预期的状态。

为什么系统不对UI空间的访问加上锁机制呢？缺点有两个：
1. 首先加上锁机制会让UI访问的逻辑变得复杂；
2. 其次锁机制会降低UI访问的效率，因为锁机制会阻塞某些线程的进行。

**最高效的处理方法就是采用单线程模型来处理UI操作。**


Handler的工作原理：首先Handler创建会采用当前线程的Looper来构建内部的消息循环系统，如果当前线程没有Looper会报错。（Can’t create handler inside thread that has not called Looper.prepare()）。解决方案就是为当前线程创建Looper即可，或者在一个Looper的线程中创建Handler也行。Hanlder创建完毕后，其内部的Looper以及MessageQueue就可以和Handler一起协同工作了，然后通过Handler的post方法将一个Runnable投递到Handler内部的Looper中去处理，也可以通过Handler的send方法发送一个消息，这个消息同样会在Looper中处理。（post方法最终也是通过send方法来完成）

send方法的工作过程：当Handler的send方法被调用时，它会调用MessageQueue的enqueueMessage方法将这个消息放出消息队列中，然后Looper发现有新的消息到来时，就会处理这个消息，最终消息中的Runnable或者Handler的handleMessage方法就会被调用。（注意Looper是运行在创建Handler所在的线程中，这样Handler中的业务逻辑被切换到创建Handler所在的线程去执行了。）

**Handler的工作原理：**

![](/media/Chapter_10/Handler的工作原理.png)

# Android的消息机制分析

### 1.ThreadLocal的工作原理

ThreadLocal是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，数据存储以后，只有在指定线程中可以获取到存储数据，对于其他线程来说则无法获取到数据。
使用场景少，一般当某些数据是以线程为作用于并且不同线程具有不同的数据副本的时候，就可以考虑采用ThreadLocal。ThreadLocal另一个使用场景是复杂逻辑下的对象传递。比如监听器的传递，有些时候一个线程中的任务过于复杂，这可能表现为函数调用栈比较深以及代码入口的多样性，在这种情况下，我们又需要监听能贯穿整个线程在执行过程、采用ThreadLocal可以让监听器作为线程内的全局对象而存在，在线程内部只要通过get方法就可以获取到监听器。（也可以将监听器作为静态变量供线程访问。）
不同线程访问同一个ThreadLocal的get方法，ThreadLocal内部会从 各自的线程中取出一个数组，然后再从数组中根据当前的ThreadLocal的索引去查找对应的value值。（这就是为什么ThreadLocal可以在不同线程中维护一套数据的副本，并且彼此互不干扰。）

**分析ThreadLocal的内部实现：**
它是一个泛型类，它的定义为public class ThreadLocal<T>。
ThreadLocal的set方法，如下所示：
```
public void set(T value) {
    Thread currentThread = Thread.currentThread();
    Values values = values(currentThread);
    if (values == null) {
        values = initializeValues(currentThread);
    }
    values.put(this, value);
}
```

首先会通过values方法来获取当前线程中的ThreadLocal数据。在Thread类的内部有一个成员专门用于保存线程的ThreadLocal的数据：ThreadLocal.Values localValues，因此获取当前的ThreadLocal数据就变得异常简单了。如果localValues的值为null，那么就需要对其进行初始化，初始化后再将ThreadLocal的值进行存储。localValues内部的一个数据：private Object[] table，这个就是保存ThreadLocal的值。

如下是ThreadLocal的值如何在LocalValues中进行存储：
```
void put(ThreadLocal<?> key, Object value) {
    cleanUp();

    // Keep track of first tombstone. That's where we want to go back
    // and add an entry if necessary.
    int firstTombstone = -1;

    for (int index = key.hash & mask;; index = next(index)){
        Object k = table[index];

        if (k == key.reference) {
            // Replace existing entry.（替换已存在的key）
            table[index + 1] = value;
            return;
        }

        if (k == null) {
            if (firstTombstone == -1) {
                // Fill in null slot.
                table[index] = key.reference;(标记对象的下一个位置)
                table[index + 1] = value;(标记值)
                size++;
                return;
            }

            // Go back and replace first tombstone.
            table[firstTombstone] = key.reference;
            table[firstTombstone + 1] = value;
            tombstones--;
            size++;
            return;
        }

        // Remember first tombstone.
        if (firstTombstone == -1 && k == TOMBSTONE) {
            firstTombstone = index;
        }
    }
}
```

ThreadLocal的值在table数据中的存储位置总是为ThreaLocal的reference字段所标识的对象的下一个位置。最终ThreaLocal的值将会被存储在table数据中：table[index + 1] = value。

get方法分析：
```
public T get() {
    // Optimized for the fast path.
    Thread currentThread = Thread.currentThread();
    Values values = values(currentThread);
    if (values != null) {
        Object[] table = values.table;
        int index = hash & values.mask;
        if (this.reference == table[index]) {
            return (T) table[index + 1];
        }
    } else {
        values = initializeValues(currentThread);
    }

    return (T) values.getAfterMiss(this);
}
```
ThreaLocal同样取出当前线程的localValues对象，如果这个对象为null那么就返回初始值，初始值由ThreaLocal的initialValue方法描述，默认情况下返回null。必要情况下可以重写这个方法。默认实现：
```
protected T initialValue() {
    return null;
}
```
如果localValues对象不为null，那就取出它的table数据并找出ThreadLocal的reference对象在table数组的位置，然后table数组中的下一个位置所存储的数据就是TreadLocal的值。


### 2.消息列表的工作原理

MessageQueue主要包含两个操作：插入和读取。
读取操作本身会伴随删除操作，插入和读取对应的方法分别是enqueueMessage和next，其中enqueueMessage的作用是往消息队列中插入一条消息，而next的作用是从消息队列中取出一条消息并将其从消息队列中移除。（单链表的数据结构来维护消息队列。）
enqueueMessage源码：
```
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    synchronized (this) {
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }
        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```
next方法的实现：
```
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }
            ......
        }
        .......
    }
}
```
这是一个无限循环的方法，如果消息队列中没有消息，那么next方法会一直阻塞在这里。当有新消息到来时，next方法会返回这条消息并将其从单链表中移除。

### 3.Looper的工作原理

Looper在Android的消息机制中扮演着消息循环的角色，它会不停地从MessageQueue中查看是否有新消息，如果有新消息就会立即处理，否则就一直阻塞在那里。
它的构造方法中它会创建一个MessageQueue，然后将当前线程保存起来。

源码：
```
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

Handler的工作需要Looper，否则会报错。因此需要通过Looper.prepare()即可为当前线程创建一个Looper，通过Looper.loop()开启消息循环。
Looper除了prepare方法外，还提供了prepareMainLooper方法，这个方法主要给主线程（ActivityThread）创建Looper使用，其本质也是通过prepare方法来实现。因为主线程的Looper比较特殊，所以Looper提供了一个getMainLooper方法，通过它可以在任何地方获取到主线程的Looper。
Looper提供了quit（直接退出）和quitSafeLy（已有消息处理完毕再退出）来退出一个Looper。建议在不需要的时候终止Looper。
Looper的loop方法，这个方法被调用消息循环系统才真正起作用。

源码：
```
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {（结束循环）
            // No message indicates that the message queue is quitting.
            return;
        }
        // This must be in a local variable, in case a UI event sets the logger
        Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        msg.target.dispatchMessage(msg);（消息处理）
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
        // Make sure that during the course of dispatching the
        // identity of the thread wasn't corrupted.
        final long newIdent = Binder.clearCallingIdentity();
        if (ident != newIdent) {
            Log.wtf(TAG, "Thread identity changed from 0x"
                    + Long.toHexString(ident) + " to 0x"
                    + Long.toHexString(newIdent) + " while dispatching to "
                    + msg.target.getClass().getName() + " "
                    + msg.callback + " what=" + msg.what);
        }
        msg.recycleUnchecked();
    }
}
```
Loop方法是一个死循环，唯一跳出循环的方式就是MessageQueue的next方法返回null。Looper必须退出，否则loop方法会无限循环下去。Loop方法会调用MessageQueue的next方法来获取新消息，而next是一个阻塞操作，当没有消息时，next方法会一直阻塞在那里，导致loop方法一直阻塞。
如果MessageQueue的next方法返回了新的消息，Looper会处理这个消息：msg.target.dispatchMessage(msg)；msg.target(发送这个消息的Handler)，它发送的消息最终会交给它的dispatchMessage来处理。不同的是，Hanlder的dispatchMessage方法是创建Hanlder时所使用的Looper中执行，这样就成功将代码逻辑切换到指定的线程中执行了。

### 4.Handler的工作原理
Handler的工作主要包含消息的发送和接收过程。消息发送可以通过post和send的一系列方法来实现，post最终也是通过send的一系列方法实现。

```
public final boolean sendMessage(Message msg){
    return sendMessageDelayed(msg, 0);
}
public final boolean sendMessageDelayed(Message msg, long delayMillis){
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
Handler发送的消息过程仅仅是向消息队列中插入了一条消息，MessageQueue的next方法就会返回这条消息给Looper，Looper收到消息后就开始处理了，最终消息由Looper交由Handler处理。（即Handler的dispatchMessage方法会被调用，这时Handler就进入了处理消息的阶段。）

dispatchMessage源码：
```
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

首先，检查Message的CallBack是否为null，不为null就通过handleCallback来处理消息。Message的Callback是一个Runnable对象，实际上就是Handler的post方法所传递的Runnable参数。

```
private static void handleCallback(Message message) {
    message.callback.run();
}
```

其次，检车mCallBack是否为null，不为null就调用mCallBack的handleMessage方法来处理消息。
```
public interface Callback {
    public boolean handleMessage(Message msg);
}
```
在日常开发中Handler最常见的方式就是派生一个Handler的子类并重写其handleMessage方法来处理具体的消息，而Callback给我们提供了另外一种使用Handler的方式，当我们不想派生子类时，就可以通过Callback来实现。
最后调用Handler的handleMessage方式来处理消息。
Handle的处理过程归纳为一个流程图：

![](/media/Chapter_10/Handler的处理过程.png)

Handler还有一个特殊的构建方法，就是通过一个特定的Looper来构建Handler。

```
public Handler(Looper looper) {
    this(looper, null, false);
}
```
下面代码解释在没有Looper的子线程中创建Handler会引发程序异常的原因。

```
public Handler(Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
        }
    }
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```
# 主线程的消息循环

Android的主线程就是ActivityThread，主程序的入口方法为main，在main方法中系统会通过Looper.prepareMainLooper()来创建主线程的Looper以及MessageQueue，并通过Looper.loop()来开启主线程的消息循环。

源码：
```
public static void main(String[] args) {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
    SamplingProfilerIntegration.start();
    // CloseGuard defaults to true and can be quite spammy.  We
    // disable it here, but selectively enable it later (via
    // StrictMode) on debug builds, but using DropBox, not logs.
    CloseGuard.setEnabled(false);
    Environment.initForCurrentUser();
    // Set the reporter for event logging in libcore
    EventLogger.setReporter(new EventLoggingReporter());
    AndroidKeyStoreProvider.install();
    // Make sure TrustedCertificateStore looks in the right place for CA certificates
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);
    Process.setArgV0("<pre-initialized>");
    Looper.prepareMainLooper();
    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }
    // End of event ActivityThreadMain.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

主线程需要一个Handler和消息队列进行交互，这个Handler就是ActivityThread.H，它内部定义了一组消息类型，主要包含了四大组件的启动和停止等过程。

```
private class H extends Handler {
    public static final int LAUNCH_ACTIVITY         = 100;
    public static final int PAUSE_ACTIVITY          = 101;
    public static final int PAUSE_ACTIVITY_FINISHING= 102;
    public static final int STOP_ACTIVITY_SHOW      = 103;
    public static final int STOP_ACTIVITY_HIDE      = 104;
    public static final int SHOW_WINDOW             = 105;
    public static final int HIDE_WINDOW             = 106;
    public static final int RESUME_ACTIVITY         = 107;
    public static final int SEND_RESULT             = 108;
    public static final int DESTROY_ACTIVITY        = 109;
    public static final int BIND_APPLICATION        = 110;
    public static final int EXIT_APPLICATION        = 111;
    public static final int NEW_INTENT              = 112;
    public static final int RECEIVER                = 113;
    public static final int CREATE_SERVICE          = 114;
    public static final int SERVICE_ARGS            = 115;
    public static final int STOP_SERVICE            = 116;

    public static final int CONFIGURATION_CHANGED   = 118;
    public static final int CLEAN_UP_CONTEXT        = 119;
    public static final int GC_WHEN_IDLE            = 120;
    public static final int BIND_SERVICE            = 121;
    public static final int UNBIND_SERVICE          = 122;
    public static final int DUMP_SERVICE            = 123;
    public static final int LOW_MEMORY              = 124;
    public static final int ACTIVITY_CONFIGURATION_CHANGED = 125;
    public static final int RELAUNCH_ACTIVITY       = 126;
    public static final int PROFILER_CONTROL        = 127;
    public static final int CREATE_BACKUP_AGENT     = 128;
    public static final int DESTROY_BACKUP_AGENT    = 129;
    public static final int SUICIDE                 = 130;
    public static final int REMOVE_PROVIDER         = 131;
    public static final int ENABLE_JIT              = 132;
    public static final int DISPATCH_PACKAGE_BROADCAST = 133;
    public static final int SCHEDULE_CRASH          = 134;
    public static final int DUMP_HEAP               = 135;
    public static final int DUMP_ACTIVITY           = 136;
    public static final int SLEEPING                = 137;
    public static final int SET_CORE_SETTINGS       = 138;
    public static final int UPDATE_PACKAGE_COMPATIBILITY_INFO = 139;
    public static final int TRIM_MEMORY             = 140;
    public static final int DUMP_PROVIDER           = 141;
    public static final int UNSTABLE_PROVIDER_DIED  = 142;
    public static final int REQUEST_ASSIST_CONTEXT_EXTRAS = 143;
    public static final int TRANSLUCENT_CONVERSION_COMPLETE = 144;
    public static final int INSTALL_PROVIDER        = 145;
    public static final int ON_NEW_ACTIVITY_OPTIONS = 146;
    public static final int CANCEL_VISIBLE_BEHIND = 147;
    public static final int BACKGROUND_VISIBLE_BEHIND_CHANGED = 148;
    public static final int ENTER_ANIMATION_COMPLETE = 149;
}
```

ActivityThread通过ApplicationThread和AMS进行进程间通讯，AMS以集成见通信的方式完成ActivityThread的请求后会回调ApplicationThread中的Binder方法，然后ApplicationThread会向H发送消息，H收到消息后会将ApplicationThread中的逻辑切换到ActivityThread中去执行，即切换到主线程中去执行，这个就是主线程的消息循环模型。
