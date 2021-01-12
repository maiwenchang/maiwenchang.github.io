# 理解Handler机制

Android中的Handler机制由Looper，MessageQueue，Handler和Message共同作用组成。Handler机制在Android中非常重要，个人觉得需要充分理解了Handler机制，才能理解Android系统中调度任务和处理事件的基本方式。它就像现实中的事件处理机制，举个例子，比如我们要申请公租房：

1. 先准备好资料；
2. 在政府网站系统上，向户政部门提出申请；
3. 资料提交后，记录到网站系统的数据库中，等待处理；
4. 网站系统持续从数据库中读取出申请资料，并发给各部门处理，当读到了你的公租房申请资料时，就把它发给户政部门；
5. 户政部门收到了公租房申请资料，对它审核并反馈结果。

那么问题来了，在上面的例子中，Looper，MessageQueue，Handler和Message分别对应什么角色？

## 解读Handler类

Handler主要功能有两个: 
- 安排计划任务在未来的某个时间点执行;
- 把任务交给其他线程执行。

Handler的一些特性：

-  Handler允许我们向它关联的线程发送消息，或者post一个Runnable到它关联的线程。
-  每个Handler只能关联一个线程和该线程里的消息队列。
-  当你创建Handler时，它默认会关联到当前线程；如果你在构造方案里传递另一个线程的Looper，那就关联到另一个线程。
-  根据上面的特性，它发送消息和Runnable到关联线程的消息队列，并在关联的线程处理他们。

Handler源码：

```java
package android.os;

public class Handler {
    
    /*
     * 把这个变量设置成ture，在构造方法会检测子类是不是非静态匿名内部类Handler，
     * 它们可能会造成内存泄漏。检测后没有报异常，只是打印了一条Log。
     */
    private static final boolean FIND_POTENTIAL_LEAKS = false;
  
    final Looper mLooper;//关联的looper
    final MessageQueue mQueue;//关联的MessageQueue
    final Callback mCallback;//处理任务的Callback
    final boolean mAsynchronous;//是否异步Handler
    IMessenger mMessenger;//跨进程通信的信使

    /**
     * 可以给Handler设置一个Callback去处理消息，如果不想新建一个类继承Handler的话
     */
    public interface Callback {
        public boolean handleMessage(Message msg);
    }
    
    /**
     * 也可以继承Handler，复写这个方法去处理消息
     */
    public void handleMessage(Message msg) {
    }
    
    /**
     * 在这里分发消息
     * 优先级：Message.callback -> Handler.callback -> Handler.handleMessage
     */
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

    /**
     * 默认构造方法，会关联到当前线程，PS：如果当前线程没有Looper，无法处理消息
     */
    public Handler() {
        this(null, false);
    }

    /**
     * 传入一个callback去处理消息
     */
    public Handler(Callback callback) {
        this(callback, false);
    }

    /**
     * 传入一个Looper，会关联到该Looper所在的线程
     * PS：每个线程都最多只有一个属于自己的Looper，这个不会有人不知道吧？
     */
    public Handler(Looper looper) {
        this(looper, null, false);
    }
    
    ...

    /**
     * 参数async标记是否异步处理消息，默认是false；
     * 该方法用@hide标注，代表不希望开发者调用
     * 
     * 关于异步消息：
     * 异步消息意味着拦截执行；
     * 相对与同步消息来讲，它不需要遵循全局顺序；
     * 异步消息不是同步屏障的目标；
     *
     * @hide
     */
    public Handler(boolean async) {
        this(null, async);
    }

    /**
     * callback与异步的合体
     *
     * @hide
     */
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            //在这里检查子类是否非静态匿名内部类
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                    //打印一条警告日志
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        //检查关联的线程有没有Looper在执行
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
    
    ...

    /**
     * 从Message池中获取消息，而不是new一个，官方建议这么做，可以做到Message的复用
     */
    public final Message obtainMessage()
    {
        return Message.obtain(this);
    }

    ...

    /**
     * 加入一个Runnable到消息到消息队列等待执行
     */
    public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
    
    ...

    /**
     * 同步执行一个任务：
     * 如果当前线程就是Handler关联的线程，立马执行Runnable任务，无须排队；否则
     * 切换到Handler所在的线程执行，并等待返回值
     * 
     * 这个方法很危险!  不正当使用会造成死锁。
     *
     * @hide 该方法容易被滥用，所以藏起来了。
     */
    public final boolean runWithScissors(final Runnable r, long timeout) {
        if (r == null) {
            throw new IllegalArgumentException("runnable must not be null");
        }
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout must be non-negative");
        }

        if (Looper.myLooper() == mLooper) {
            r.run();
            return true;
        }

       //BlockingRunnable会阻塞线程执行，设置了超时的时间以免runnable发生异常造成死锁，
       //try-catch了run方法的执行，在finaly里释放了锁。
        BlockingRunnable br = new BlockingRunnable(r);
        return br.postAndWait(this, timeout);
    }

    /**
     * 发送一条消息到MessageQueue
     */
    public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }

   ...

    final IMessenger getIMessenger() {
        synchronized (mQueue) {
            if (mMessenger != null) {
                return mMessenger;
            }
            mMessenger = new MessengerImpl();
            return mMessenger;
        }
    }

    private final class MessengerImpl extends IMessenger.Stub {
        public void send(Message msg) {
            msg.sendingUid = Binder.getCallingUid();
            Handler.this.sendMessage(msg);
        }
    }

    private static void handleCallback(Message message) {
        message.callback.run();
    }
}
```



## 解读Looper类

看一下Looper的初始化：

```java
class LooperThread extends Thread {
     public Handler mHandler;

     public void run() {
         Looper.prepare();

         mHandler = new Handler() {
             public void handleMessage(Message msg) {
                 // 在这里处理消息
             }
         };

         Looper.loop();
     }
 }
```



Looper的源码：

```java
package android.os;

/**
  *  Looper为线程提供消息循环能力，线程默认是没有Looper的，需要创建一个Looper并调用Looper.prepare()，
  *  并且调用Looper.loop()开始循环，然后通过Handler发送和处理消息。
  */
public final class Looper {
    private static final String TAG = "Looper";

    // sThreadLocal.get() 会返回null除非你已经调用了prepare().
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    // 主线程的Looper
    private static Looper sMainLooper;
    //
    final MessageQueue mQueue;
    final Thread mThread;

    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
      
    }

    /**
     * Initialize the current thread as a looper, marking it as an
     * application's main looper. The main looper for your application
     * is created by the Android environment, so you should never need
     * to call this function yourself.  See also: {@link #prepare()}
     */
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }

    /**
     * Returns the application's main looper, which lives in the main thread of the application.
     */
    public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }

    /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
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
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;

            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            final long start = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
            final long end;
            try {
                msg.target.dispatchMessage(msg);
                end = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            if (slowDispatchThresholdMs > 0) {
                final long time = end - start;
                if (time > slowDispatchThresholdMs) {
                    Slog.w(TAG, "Dispatch took " + time + "ms on "
                            + Thread.currentThread().getName() + ", h=" +
                            msg.target + " cb=" + msg.callback + " msg=" + msg.what);
                }
            }

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

    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }

    /**
     * Return the {@link MessageQueue} object associated with the current
     * thread.  This must be called from a thread running a Looper, or a
     * NullPointerException will be thrown.
     */
    public static @NonNull MessageQueue myQueue() {
        return myLooper().mQueue;
    }

    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }

    /**
     * Returns true if the current thread is this looper's thread.
     */
    public boolean isCurrentThread() {
        return Thread.currentThread() == mThread;
    }

    /**
     * Control logging of messages as they are processed by this Looper.  If
     * enabled, a log message will be written to <var>printer</var>
     * at the beginning and ending of each message dispatch, identifying the
     * target Handler and message contents.
     *
     * @param printer A Printer object that will receive log messages, or
     * null to disable message logging.
     */
    public void setMessageLogging(@Nullable Printer printer) {
        mLogging = printer;
    }

    /** {@hide} */
    public void setTraceTag(long traceTag) {
        mTraceTag = traceTag;
    }

    /** {@hide} */
    public void setSlowDispatchThresholdMs(long slowDispatchThresholdMs) {
        mSlowDispatchThresholdMs = slowDispatchThresholdMs;
    }

    /**
     * Quits the looper.
     * <p>
     * Causes the {@link #loop} method to terminate without processing any
     * more messages in the message queue.
     * </p><p>
     * Any attempt to post messages to the queue after the looper is asked to quit will fail.
     * For example, the {@link Handler#sendMessage(Message)} method will return false.
     * </p><p class="note">
     * Using this method may be unsafe because some messages may not be delivered
     * before the looper terminates.  Consider using {@link #quitSafely} instead to ensure
     * that all pending work is completed in an orderly manner.
     * </p>
     *
     * @see #quitSafely
     */
    public void quit() {
        mQueue.quit(false);
    }

    /**
     * Quits the looper safely.
     * <p>
     * Causes the {@link #loop} method to terminate as soon as all remaining messages
     * in the message queue that are already due to be delivered have been handled.
     * However pending delayed messages with due times in the future will not be
     * delivered before the loop terminates.
     * </p><p>
     * Any attempt to post messages to the queue after the looper is asked to quit will fail.
     * For example, the {@link Handler#sendMessage(Message)} method will return false.
     * </p>
     */
    public void quitSafely() {
        mQueue.quit(true);
    }

    /**
     * Gets the Thread associated with this Looper.
     *
     * @return The looper's thread.
     */
    public @NonNull Thread getThread() {
        return mThread;
    }

    /**
     * Gets this looper's message queue.
     *
     * @return The looper's message queue.
     */
    public @NonNull MessageQueue getQueue() {
        return mQueue;
    }

    /**
     * Dumps the state of the looper for debugging purposes.
     *
     * @param pw A printer to receive the contents of the dump.
     * @param prefix A prefix to prepend to each line which is printed.
     */
    public void dump(@NonNull Printer pw, @NonNull String prefix) {
        pw.println(prefix + toString());
        mQueue.dump(pw, prefix + "  ", null);
    }

    /**
     * Dumps the state of the looper for debugging purposes.
     *
     * @param pw A printer to receive the contents of the dump.
     * @param prefix A prefix to prepend to each line which is printed.
     * @param handler Only dump messages for this Handler.
     * @hide
     */
    public void dump(@NonNull Printer pw, @NonNull String prefix, Handler handler) {
        pw.println(prefix + toString());
        mQueue.dump(pw, prefix + "  ", handler);
    }

    /** @hide */
    public void writeToProto(ProtoOutputStream proto, long fieldId) {
        final long looperToken = proto.start(fieldId);
        proto.write(LooperProto.THREAD_NAME, mThread.getName());
        proto.write(LooperProto.THREAD_ID, mThread.getId());
        proto.write(LooperProto.IDENTITY_HASH_CODE, System.identityHashCode(this));
        mQueue.writeToProto(proto, LooperProto.QUEUE);
        proto.end(looperToken);
    }

    @Override
    public String toString() {
        return "Looper (" + mThread.getName() + ", tid " + mThread.getId()
                + ") {" + Integer.toHexString(System.identityHashCode(this)) + "}";
    }
}

```





## 解读MessageQueue类

```java
/*
 * Copyright (C) 2006 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package android.os;

import android.annotation.IntDef;
import android.annotation.NonNull;
import android.os.MessageQueueProto;
import android.util.Log;
import android.util.Printer;
import android.util.SparseArray;
import android.util.proto.ProtoOutputStream;

import java.io.FileDescriptor;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.util.ArrayList;

/**
 * Low-level class holding the list of messages to be dispatched by a
 * {@link Looper}.  Messages are not added directly to a MessageQueue,
 * but rather through {@link Handler} objects associated with the Looper.
 *
 * <p>You can retrieve the MessageQueue for the current thread with
 * {@link Looper#myQueue() Looper.myQueue()}.
 */
public final class MessageQueue {
    private static final String TAG = "MessageQueue";
    private static final boolean DEBUG = false;

    // True if the message queue can be quit.
    private final boolean mQuitAllowed;

    @SuppressWarnings("unused")
    private long mPtr; // used by native code

    Message mMessages;
    private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
    private SparseArray<FileDescriptorRecord> mFileDescriptorRecords;
    private IdleHandler[] mPendingIdleHandlers;
    private boolean mQuitting;

    // Indicates whether next() is blocked waiting in pollOnce() with a non-zero timeout.
    private boolean mBlocked;

    // The next barrier token.
    // Barriers are indicated by messages with a null target whose arg1 field carries the token.
    private int mNextBarrierToken;

    private native static long nativeInit();
    private native static void nativeDestroy(long ptr);
    private native void nativePollOnce(long ptr, int timeoutMillis); /*non-static for callbacks*/
    private native static void nativeWake(long ptr);
    private native static boolean nativeIsPolling(long ptr);
    private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);

    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }

    @Override
    protected void finalize() throws Throwable {
        try {
            dispose();
        } finally {
            super.finalize();
        }
    }

    // Disposes of the underlying message queue.
    // Must only be called on the looper thread or the finalizer.
    private void dispose() {
        if (mPtr != 0) {
            nativeDestroy(mPtr);
            mPtr = 0;
        }
    }

    /**
     * Returns true if the looper has no pending messages which are due to be processed.
     *
     * <p>This method is safe to call from any thread.
     *
     * @return True if the looper is idle.
     */
    public boolean isIdle() {
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            return mMessages == null || now < mMessages.when;
        }
    }

    /**
     * Add a new {@link IdleHandler} to this message queue.  This may be
     * removed automatically for you by returning false from
     * {@link IdleHandler#queueIdle IdleHandler.queueIdle()} when it is
     * invoked, or explicitly removing it with {@link #removeIdleHandler}.
     *
     * <p>This method is safe to call from any thread.
     *
     * @param handler The IdleHandler to be added.
     */
    public void addIdleHandler(@NonNull IdleHandler handler) {
        if (handler == null) {
            throw new NullPointerException("Can't add a null IdleHandler");
        }
        synchronized (this) {
            mIdleHandlers.add(handler);
        }
    }

    /**
     * Remove an {@link IdleHandler} from the queue that was previously added
     * with {@link #addIdleHandler}.  If the given object is not currently
     * in the idle list, nothing is done.
     *
     * <p>This method is safe to call from any thread.
     *
     * @param handler The IdleHandler to be removed.
     */
    public void removeIdleHandler(@NonNull IdleHandler handler) {
        synchronized (this) {
            mIdleHandlers.remove(handler);
        }
    }

    /**
     * Returns whether this looper's thread is currently polling for more work to do.
     * This is a good signal that the loop is still alive rather than being stuck
     * handling a callback.  Note that this method is intrinsically racy, since the
     * state of the loop can change before you get the result back.
     *
     * <p>This method is safe to call from any thread.
     *
     * @return True if the looper is currently polling for events.
     * @hide
     */
    public boolean isPolling() {
        synchronized (this) {
            return isPollingLocked();
        }
    }

    private boolean isPollingLocked() {
        // If the loop is quitting then it must not be idling.
        // We can assume mPtr != 0 when mQuitting is false.
        return !mQuitting && nativeIsPolling(mPtr);
    }

    /**
     * Adds a file descriptor listener to receive notification when file descriptor
     * related events occur.
     * <p>
     * If the file descriptor has already been registered, the specified events
     * and listener will replace any that were previously associated with it.
     * It is not possible to set more than one listener per file descriptor.
     * </p><p>
     * It is important to always unregister the listener when the file descriptor
     * is no longer of use.
     * </p>
     *
     * @param fd The file descriptor for which a listener will be registered.
     * @param events The set of events to receive: a combination of the
     * {@link OnFileDescriptorEventListener#EVENT_INPUT},
     * {@link OnFileDescriptorEventListener#EVENT_OUTPUT}, and
     * {@link OnFileDescriptorEventListener#EVENT_ERROR} event masks.  If the requested
     * set of events is zero, then the listener is unregistered.
     * @param listener The listener to invoke when file descriptor events occur.
     *
     * @see OnFileDescriptorEventListener
     * @see #removeOnFileDescriptorEventListener
     */
    public void addOnFileDescriptorEventListener(@NonNull FileDescriptor fd,
            @OnFileDescriptorEventListener.Events int events,
            @NonNull OnFileDescriptorEventListener listener) {
        if (fd == null) {
            throw new IllegalArgumentException("fd must not be null");
        }
        if (listener == null) {
            throw new IllegalArgumentException("listener must not be null");
        }

        synchronized (this) {
            updateOnFileDescriptorEventListenerLocked(fd, events, listener);
        }
    }

    /**
     * Removes a file descriptor listener.
     * <p>
     * This method does nothing if no listener has been registered for the
     * specified file descriptor.
     * </p>
     *
     * @param fd The file descriptor whose listener will be unregistered.
     *
     * @see OnFileDescriptorEventListener
     * @see #addOnFileDescriptorEventListener
     */
    public void removeOnFileDescriptorEventListener(@NonNull FileDescriptor fd) {
        if (fd == null) {
            throw new IllegalArgumentException("fd must not be null");
        }

        synchronized (this) {
            updateOnFileDescriptorEventListenerLocked(fd, 0, null);
        }
    }

    private void updateOnFileDescriptorEventListenerLocked(FileDescriptor fd, int events,
            OnFileDescriptorEventListener listener) {
        final int fdNum = fd.getInt$();

        int index = -1;
        FileDescriptorRecord record = null;
        if (mFileDescriptorRecords != null) {
            index = mFileDescriptorRecords.indexOfKey(fdNum);
            if (index >= 0) {
                record = mFileDescriptorRecords.valueAt(index);
                if (record != null && record.mEvents == events) {
                    return;
                }
            }
        }

        if (events != 0) {
            events |= OnFileDescriptorEventListener.EVENT_ERROR;
            if (record == null) {
                if (mFileDescriptorRecords == null) {
                    mFileDescriptorRecords = new SparseArray<FileDescriptorRecord>();
                }
                record = new FileDescriptorRecord(fd, events, listener);
                mFileDescriptorRecords.put(fdNum, record);
            } else {
                record.mListener = listener;
                record.mEvents = events;
                record.mSeq += 1;
            }
            nativeSetFileDescriptorEvents(mPtr, fdNum, events);
        } else if (record != null) {
            record.mEvents = 0;
            mFileDescriptorRecords.removeAt(index);
        }
    }

    // Called from native code.
    private int dispatchEvents(int fd, int events) {
        // Get the file descriptor record and any state that might change.
        final FileDescriptorRecord record;
        final int oldWatchedEvents;
        final OnFileDescriptorEventListener listener;
        final int seq;
        synchronized (this) {
            record = mFileDescriptorRecords.get(fd);
            if (record == null) {
                return 0; // spurious, no listener registered
            }

            oldWatchedEvents = record.mEvents;
            events &= oldWatchedEvents; // filter events based on current watched set
            if (events == 0) {
                return oldWatchedEvents; // spurious, watched events changed
            }

            listener = record.mListener;
            seq = record.mSeq;
        }

        // Invoke the listener outside of the lock.
        int newWatchedEvents = listener.onFileDescriptorEvents(
                record.mDescriptor, events);
        if (newWatchedEvents != 0) {
            newWatchedEvents |= OnFileDescriptorEventListener.EVENT_ERROR;
        }

        // Update the file descriptor record if the listener changed the set of
        // events to watch and the listener itself hasn't been updated since.
        if (newWatchedEvents != oldWatchedEvents) {
            synchronized (this) {
                int index = mFileDescriptorRecords.indexOfKey(fd);
                if (index >= 0 && mFileDescriptorRecords.valueAt(index) == record
                        && record.mSeq == seq) {
                    record.mEvents = newWatchedEvents;
                    if (newWatchedEvents == 0) {
                        mFileDescriptorRecords.removeAt(index);
                    }
                }
            }
        }

        // Return the new set of events to watch for native code to take care of.
        return newWatchedEvents;
    }

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

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }

    void quit(boolean safe) {
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) {
                return;
            }
            mQuitting = true;

            if (safe) {
                removeAllFutureMessagesLocked();
            } else {
                removeAllMessagesLocked();
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
            nativeWake(mPtr);
        }
    }

    /**
     * Posts a synchronization barrier to the Looper's message queue.
     *
     * Message processing occurs as usual until the message queue encounters the
     * synchronization barrier that has been posted.  When the barrier is encountered,
     * later synchronous messages in the queue are stalled (prevented from being executed)
     * until the barrier is released by calling {@link #removeSyncBarrier} and specifying
     * the token that identifies the synchronization barrier.
     *
     * This method is used to immediately postpone execution of all subsequently posted
     * synchronous messages until a condition is met that releases the barrier.
     * Asynchronous messages (see {@link Message#isAsynchronous} are exempt from the barrier
     * and continue to be processed as usual.
     *
     * This call must be always matched by a call to {@link #removeSyncBarrier} with
     * the same token to ensure that the message queue resumes normal operation.
     * Otherwise the application will probably hang!
     *
     * @return A token that uniquely identifies the barrier.  This token must be
     * passed to {@link #removeSyncBarrier} to release the barrier.
     *
     * @hide
     */
    public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

    private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }

    /**
     * Removes a synchronization barrier.
     *
     * @param token The synchronization barrier token that was returned by
     * {@link #postSyncBarrier}.
     *
     * @throws IllegalStateException if the barrier was not found.
     *
     * @hide
     */
    public void removeSyncBarrier(int token) {
        // Remove a sync barrier token from the queue.
        // If the queue is no longer stalled by a barrier then wake it.
        synchronized (this) {
            Message prev = null;
            Message p = mMessages;
            while (p != null && (p.target != null || p.arg1 != token)) {
                prev = p;
                p = p.next;
            }
            if (p == null) {
                throw new IllegalStateException("The specified message queue synchronization "
                        + " barrier token has not been posted or has already been removed.");
            }
            final boolean needWake;
            if (prev != null) {
                prev.next = p.next;
                needWake = false;
            } else {
                mMessages = p.next;
                needWake = mMessages == null || mMessages.target != null;
            }
            p.recycleUnchecked();

            // If the loop is quitting then it is already awake.
            // We can assume mPtr != 0 when mQuitting is false.
            if (needWake && !mQuitting) {
                nativeWake(mPtr);
            }
        }
    }

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

    boolean hasMessages(Handler h, int what, Object object) {
        if (h == null) {
            return false;
        }

        synchronized (this) {
            Message p = mMessages;
            while (p != null) {
                if (p.target == h && p.what == what && (object == null || p.obj == object)) {
                    return true;
                }
                p = p.next;
            }
            return false;
        }
    }

    boolean hasMessages(Handler h, Runnable r, Object object) {
        if (h == null) {
            return false;
        }

        synchronized (this) {
            Message p = mMessages;
            while (p != null) {
                if (p.target == h && p.callback == r && (object == null || p.obj == object)) {
                    return true;
                }
                p = p.next;
            }
            return false;
        }
    }

    boolean hasMessages(Handler h) {
        if (h == null) {
            return false;
        }

        synchronized (this) {
            Message p = mMessages;
            while (p != null) {
                if (p.target == h) {
                    return true;
                }
                p = p.next;
            }
            return false;
        }
    }

    void removeMessages(Handler h, int what, Object object) {
        if (h == null) {
            return;
        }

        synchronized (this) {
            Message p = mMessages;

            // Remove all messages at front.
            while (p != null && p.target == h && p.what == what
                   && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }

            // Remove all messages after front.
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && n.what == what
                        && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }

    void removeMessages(Handler h, Runnable r, Object object) {
        if (h == null || r == null) {
            return;
        }

        synchronized (this) {
            Message p = mMessages;

            // Remove all messages at front.
            while (p != null && p.target == h && p.callback == r
                   && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }

            // Remove all messages after front.
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && n.callback == r
                        && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }

    void removeCallbacksAndMessages(Handler h, Object object) {
        if (h == null) {
            return;
        }

        synchronized (this) {
            Message p = mMessages;

            // Remove all messages at front.
            while (p != null && p.target == h
                    && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }

            // Remove all messages after front.
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }

    private void removeAllMessagesLocked() {
        Message p = mMessages;
        while (p != null) {
            Message n = p.next;
            p.recycleUnchecked();
            p = n;
        }
        mMessages = null;
    }

    private void removeAllFutureMessagesLocked() {
        final long now = SystemClock.uptimeMillis();
        Message p = mMessages;
        if (p != null) {
            if (p.when > now) {
                removeAllMessagesLocked();
            } else {
                Message n;
                for (;;) {
                    n = p.next;
                    if (n == null) {
                        return;
                    }
                    if (n.when > now) {
                        break;
                    }
                    p = n;
                }
                p.next = null;
                do {
                    p = n;
                    n = p.next;
                    p.recycleUnchecked();
                } while (n != null);
            }
        }
    }

    void dump(Printer pw, String prefix, Handler h) {
        synchronized (this) {
            long now = SystemClock.uptimeMillis();
            int n = 0;
            for (Message msg = mMessages; msg != null; msg = msg.next) {
                if (h == null || h == msg.target) {
                    pw.println(prefix + "Message " + n + ": " + msg.toString(now));
                }
                n++;
            }
            pw.println(prefix + "(Total messages: " + n + ", polling=" + isPollingLocked()
                    + ", quitting=" + mQuitting + ")");
        }
    }

    void writeToProto(ProtoOutputStream proto, long fieldId) {
        final long messageQueueToken = proto.start(fieldId);
        synchronized (this) {
            for (Message msg = mMessages; msg != null; msg = msg.next) {
                msg.writeToProto(proto, MessageQueueProto.MESSAGES);
            }
            proto.write(MessageQueueProto.IS_POLLING_LOCKED, isPollingLocked());
            proto.write(MessageQueueProto.IS_QUITTING, mQuitting);
        }
        proto.end(messageQueueToken);
    }

    /**
     * Callback interface for discovering when a thread is going to block
     * waiting for more messages.
     */
    public static interface IdleHandler {
        /**
         * Called when the message queue has run out of messages and will now
         * wait for more.  Return true to keep your idle handler active, false
         * to have it removed.  This may be called if there are still messages
         * pending in the queue, but they are all scheduled to be dispatched
         * after the current time.
         */
        boolean queueIdle();
    }

    /**
     * A listener which is invoked when file descriptor related events occur.
     */
    public interface OnFileDescriptorEventListener {
        /**
         * File descriptor event: Indicates that the file descriptor is ready for input
         * operations, such as reading.
         * <p>
         * The listener should read all available data from the file descriptor
         * then return <code>true</code> to keep the listener active or <code>false</code>
         * to remove the listener.
         * </p><p>
         * In the case of a socket, this event may be generated to indicate
         * that there is at least one incoming connection that the listener
         * should accept.
         * </p><p>
         * This event will only be generated if the {@link #EVENT_INPUT} event mask was
         * specified when the listener was added.
         * </p>
         */
        public static final int EVENT_INPUT = 1 << 0;

        /**
         * File descriptor event: Indicates that the file descriptor is ready for output
         * operations, such as writing.
         * <p>
         * The listener should write as much data as it needs.  If it could not
         * write everything at once, then it should return <code>true</code> to
         * keep the listener active.  Otherwise, it should return <code>false</code>
         * to remove the listener then re-register it later when it needs to write
         * something else.
         * </p><p>
         * This event will only be generated if the {@link #EVENT_OUTPUT} event mask was
         * specified when the listener was added.
         * </p>
         */
        public static final int EVENT_OUTPUT = 1 << 1;

        /**
         * File descriptor event: Indicates that the file descriptor encountered a
         * fatal error.
         * <p>
         * File descriptor errors can occur for various reasons.  One common error
         * is when the remote peer of a socket or pipe closes its end of the connection.
         * </p><p>
         * This event may be generated at any time regardless of whether the
         * {@link #EVENT_ERROR} event mask was specified when the listener was added.
         * </p>
         */
        public static final int EVENT_ERROR = 1 << 2;

        /** @hide */
        @Retention(RetentionPolicy.SOURCE)
        @IntDef(flag=true, value={EVENT_INPUT, EVENT_OUTPUT, EVENT_ERROR})
        public @interface Events {}

        /**
         * Called when a file descriptor receives events.
         *
         * @param fd The file descriptor.
         * @param events The set of events that occurred: a combination of the
         * {@link #EVENT_INPUT}, {@link #EVENT_OUTPUT}, and {@link #EVENT_ERROR} event masks.
         * @return The new set of events to watch, or 0 to unregister the listener.
         *
         * @see #EVENT_INPUT
         * @see #EVENT_OUTPUT
         * @see #EVENT_ERROR
         */
        @Events int onFileDescriptorEvents(@NonNull FileDescriptor fd, @Events int events);
    }

    private static final class FileDescriptorRecord {
        public final FileDescriptor mDescriptor;
        public int mEvents;
        public OnFileDescriptorEventListener mListener;
        public int mSeq;

        public FileDescriptorRecord(FileDescriptor descriptor,
                int events, OnFileDescriptorEventListener listener) {
            mDescriptor = descriptor;
            mEvents = events;
            mListener = listener;
        }
    }
}

```





## 解读Message类

```java
/*
 * Copyright (C) 2006 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package android.os;

import android.os.MessageProto;
import android.util.TimeUtils;
import android.util.proto.ProtoOutputStream;

/**
 *
 * Defines a message containing a description and arbitrary data object that can be
 * sent to a {@link Handler}.  This object contains two extra int fields and an
 * extra object field that allow you to not do allocations in many cases.
 *
 * <p class="note">While the constructor of Message is public, the best way to get
 * one of these is to call {@link #obtain Message.obtain()} or one of the
 * {@link Handler#obtainMessage Handler.obtainMessage()} methods, which will pull
 * them from a pool of recycled objects.</p>
 */
public final class Message implements Parcelable {
    /**
     * User-defined message code so that the recipient can identify
     * what this message is about. Each {@link Handler} has its own name-space
     * for message codes, so you do not need to worry about yours conflicting
     * with other handlers.
     */
    public int what;

    /**
     * arg1 and arg2 are lower-cost alternatives to using
     * {@link #setData(Bundle) setData()} if you only need to store a
     * few integer values.
     */
    public int arg1;

    /**
     * arg1 and arg2 are lower-cost alternatives to using
     * {@link #setData(Bundle) setData()} if you only need to store a
     * few integer values.
     */
    public int arg2;

    /**
     * An arbitrary object to send to the recipient.  When using
     * {@link Messenger} to send the message across processes this can only
     * be non-null if it contains a Parcelable of a framework class (not one
     * implemented by the application).   For other data transfer use
     * {@link #setData}.
     *
     * <p>Note that Parcelable objects here are not supported prior to
     * the {@link android.os.Build.VERSION_CODES#FROYO} release.
     */
    public Object obj;

    /**
     * Optional Messenger where replies to this message can be sent.  The
     * semantics of exactly how this is used are up to the sender and
     * receiver.
     */
    public Messenger replyTo;

    /**
     * Optional field indicating the uid that sent the message.  This is
     * only valid for messages posted by a {@link Messenger}; otherwise,
     * it will be -1.
     */
    public int sendingUid = -1;

    /** If set message is in use.
     * This flag is set when the message is enqueued and remains set while it
     * is delivered and afterwards when it is recycled.  The flag is only cleared
     * when a new message is created or obtained since that is the only time that
     * applications are allowed to modify the contents of the message.
     *
     * It is an error to attempt to enqueue or recycle a message that is already in use.
     */
    /*package*/ static final int FLAG_IN_USE = 1 << 0;

    /** If set message is asynchronous */
    /*package*/ static final int FLAG_ASYNCHRONOUS = 1 << 1;

    /** Flags to clear in the copyFrom method */
    /*package*/ static final int FLAGS_TO_CLEAR_ON_COPY_FROM = FLAG_IN_USE;

    /*package*/ int flags;

    /*package*/ long when;

    /*package*/ Bundle data;

    /*package*/ Handler target;

    /*package*/ Runnable callback;

    // sometimes we store linked lists of these things
    /*package*/ Message next;

    private static final Object sPoolSync = new Object();
    private static Message sPool;
    private static int sPoolSize = 0;

    private static final int MAX_POOL_SIZE = 50;

    private static boolean gCheckRecycle = true;

    /**
     * Return a new Message instance from the global pool. Allows us to
     * avoid allocating new objects in many cases.
     */
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }

    /**
     * Same as {@link #obtain()}, but copies the values of an existing
     * message (including its target) into the new one.
     * @param orig Original message to copy.
     * @return A Message object from the global pool.
     */
    public static Message obtain(Message orig) {
        Message m = obtain();
        m.what = orig.what;
        m.arg1 = orig.arg1;
        m.arg2 = orig.arg2;
        m.obj = orig.obj;
        m.replyTo = orig.replyTo;
        m.sendingUid = orig.sendingUid;
        if (orig.data != null) {
            m.data = new Bundle(orig.data);
        }
        m.target = orig.target;
        m.callback = orig.callback;

        return m;
    }

    /**
     * Same as {@link #obtain()}, but sets the value for the <em>target</em> member on the Message returned.
     * @param h  Handler to assign to the returned Message object's <em>target</em> member.
     * @return A Message object from the global pool.
     */
    public static Message obtain(Handler h) {
        Message m = obtain();
        m.target = h;

        return m;
    }

    /**
     * Same as {@link #obtain(Handler)}, but assigns a callback Runnable on
     * the Message that is returned.
     * @param h  Handler to assign to the returned Message object's <em>target</em> member.
     * @param callback Runnable that will execute when the message is handled.
     * @return A Message object from the global pool.
     */
    public static Message obtain(Handler h, Runnable callback) {
        Message m = obtain();
        m.target = h;
        m.callback = callback;

        return m;
    }

    /**
     * Same as {@link #obtain()}, but sets the values for both <em>target</em> and
     * <em>what</em> members on the Message.
     * @param h  Value to assign to the <em>target</em> member.
     * @param what  Value to assign to the <em>what</em> member.
     * @return A Message object from the global pool.
     */
    public static Message obtain(Handler h, int what) {
        Message m = obtain();
        m.target = h;
        m.what = what;

        return m;
    }

    /**
     * Same as {@link #obtain()}, but sets the values of the <em>target</em>, <em>what</em>, and <em>obj</em>
     * members.
     * @param h  The <em>target</em> value to set.
     * @param what  The <em>what</em> value to set.
     * @param obj  The <em>object</em> method to set.
     * @return  A Message object from the global pool.
     */
    public static Message obtain(Handler h, int what, Object obj) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.obj = obj;

        return m;
    }

    /**
     * Same as {@link #obtain()}, but sets the values of the <em>target</em>, <em>what</em>,
     * <em>arg1</em>, and <em>arg2</em> members.
     *
     * @param h  The <em>target</em> value to set.
     * @param what  The <em>what</em> value to set.
     * @param arg1  The <em>arg1</em> value to set.
     * @param arg2  The <em>arg2</em> value to set.
     * @return  A Message object from the global pool.
     */
    public static Message obtain(Handler h, int what, int arg1, int arg2) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.arg1 = arg1;
        m.arg2 = arg2;

        return m;
    }

    /**
     * Same as {@link #obtain()}, but sets the values of the <em>target</em>, <em>what</em>,
     * <em>arg1</em>, <em>arg2</em>, and <em>obj</em> members.
     *
     * @param h  The <em>target</em> value to set.
     * @param what  The <em>what</em> value to set.
     * @param arg1  The <em>arg1</em> value to set.
     * @param arg2  The <em>arg2</em> value to set.
     * @param obj  The <em>obj</em> value to set.
     * @return  A Message object from the global pool.
     */
    public static Message obtain(Handler h, int what,
            int arg1, int arg2, Object obj) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.arg1 = arg1;
        m.arg2 = arg2;
        m.obj = obj;

        return m;
    }

    /** @hide */
    public static void updateCheckRecycle(int targetSdkVersion) {
        if (targetSdkVersion < Build.VERSION_CODES.LOLLIPOP) {
            gCheckRecycle = false;
        }
    }

    /**
     * Return a Message instance to the global pool.
     * <p>
     * You MUST NOT touch the Message after calling this function because it has
     * effectively been freed.  It is an error to recycle a message that is currently
     * enqueued or that is in the process of being delivered to a Handler.
     * </p>
     */
    public void recycle() {
        if (isInUse()) {
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            return;
        }
        recycleUnchecked();
    }

    /**
     * Recycles a Message that may be in-use.
     * Used internally by the MessageQueue and Looper when disposing of queued Messages.
     */
    void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }

    /**
     * Make this message like o.  Performs a shallow copy of the data field.
     * Does not copy the linked list fields, nor the timestamp or
     * target/callback of the original message.
     */
    public void copyFrom(Message o) {
        this.flags = o.flags & ~FLAGS_TO_CLEAR_ON_COPY_FROM;
        this.what = o.what;
        this.arg1 = o.arg1;
        this.arg2 = o.arg2;
        this.obj = o.obj;
        this.replyTo = o.replyTo;
        this.sendingUid = o.sendingUid;

        if (o.data != null) {
            this.data = (Bundle) o.data.clone();
        } else {
            this.data = null;
        }
    }

    /**
     * Return the targeted delivery time of this message, in milliseconds.
     */
    public long getWhen() {
        return when;
    }

    public void setTarget(Handler target) {
        this.target = target;
    }

    /**
     * Retrieve the a {@link android.os.Handler Handler} implementation that
     * will receive this message. The object must implement
     * {@link android.os.Handler#handleMessage(android.os.Message)
     * Handler.handleMessage()}. Each Handler has its own name-space for
     * message codes, so you do not need to
     * worry about yours conflicting with other handlers.
     */
    public Handler getTarget() {
        return target;
    }

    /**
     * Retrieve callback object that will execute when this message is handled.
     * This object must implement Runnable. This is called by
     * the <em>target</em> {@link Handler} that is receiving this Message to
     * dispatch it.  If
     * not set, the message will be dispatched to the receiving Handler's
     * {@link Handler#handleMessage(Message Handler.handleMessage())}.
     */
    public Runnable getCallback() {
        return callback;
    }

    /**
     * Obtains a Bundle of arbitrary data associated with this
     * event, lazily creating it if necessary. Set this value by calling
     * {@link #setData(Bundle)}.  Note that when transferring data across
     * processes via {@link Messenger}, you will need to set your ClassLoader
     * on the Bundle via {@link Bundle#setClassLoader(ClassLoader)
     * Bundle.setClassLoader()} so that it can instantiate your objects when
     * you retrieve them.
     * @see #peekData()
     * @see #setData(Bundle)
     */
    public Bundle getData() {
        if (data == null) {
            data = new Bundle();
        }

        return data;
    }

    /**
     * Like getData(), but does not lazily create the Bundle.  A null
     * is returned if the Bundle does not already exist.  See
     * {@link #getData} for further information on this.
     * @see #getData()
     * @see #setData(Bundle)
     */
    public Bundle peekData() {
        return data;
    }

    /**
     * Sets a Bundle of arbitrary data values. Use arg1 and arg2 members
     * as a lower cost way to send a few simple integer values, if you can.
     * @see #getData()
     * @see #peekData()
     */
    public void setData(Bundle data) {
        this.data = data;
    }

    /**
     * Sends this Message to the Handler specified by {@link #getTarget}.
     * Throws a null pointer exception if this field has not been set.
     */
    public void sendToTarget() {
        target.sendMessage(this);
    }

    /**
     * Returns true if the message is asynchronous, meaning that it is not
     * subject to {@link Looper} synchronization barriers.
     *
     * @return True if the message is asynchronous.
     *
     * @see #setAsynchronous(boolean)
     */
    public boolean isAsynchronous() {
        return (flags & FLAG_ASYNCHRONOUS) != 0;
    }

    /**
     * Sets whether the message is asynchronous, meaning that it is not
     * subject to {@link Looper} synchronization barriers.
     * <p>
     * Certain operations, such as view invalidation, may introduce synchronization
     * barriers into the {@link Looper}'s message queue to prevent subsequent messages
     * from being delivered until some condition is met.  In the case of view invalidation,
     * messages which are posted after a call to {@link android.view.View#invalidate}
     * are suspended by means of a synchronization barrier until the next frame is
     * ready to be drawn.  The synchronization barrier ensures that the invalidation
     * request is completely handled before resuming.
     * </p><p>
     * Asynchronous messages are exempt from synchronization barriers.  They typically
     * represent interrupts, input events, and other signals that must be handled independently
     * even while other work has been suspended.
     * </p><p>
     * Note that asynchronous messages may be delivered out of order with respect to
     * synchronous messages although they are always delivered in order among themselves.
     * If the relative order of these messages matters then they probably should not be
     * asynchronous in the first place.  Use with caution.
     * </p>
     *
     * @param async True if the message is asynchronous.
     *
     * @see #isAsynchronous()
     */
    public void setAsynchronous(boolean async) {
        if (async) {
            flags |= FLAG_ASYNCHRONOUS;
        } else {
            flags &= ~FLAG_ASYNCHRONOUS;
        }
    }

    /*package*/ boolean isInUse() {
        return ((flags & FLAG_IN_USE) == FLAG_IN_USE);
    }

    /*package*/ void markInUse() {
        flags |= FLAG_IN_USE;
    }

    /** Constructor (but the preferred way to get a Message is to call {@link #obtain() Message.obtain()}).
    */
    public Message() {
    }

    @Override
    public String toString() {
        return toString(SystemClock.uptimeMillis());
    }

    String toString(long now) {
        StringBuilder b = new StringBuilder();
        b.append("{ when=");
        TimeUtils.formatDuration(when - now, b);

        if (target != null) {
            if (callback != null) {
                b.append(" callback=");
                b.append(callback.getClass().getName());
            } else {
                b.append(" what=");
                b.append(what);
            }

            if (arg1 != 0) {
                b.append(" arg1=");
                b.append(arg1);
            }

            if (arg2 != 0) {
                b.append(" arg2=");
                b.append(arg2);
            }

            if (obj != null) {
                b.append(" obj=");
                b.append(obj);
            }

            b.append(" target=");
            b.append(target.getClass().getName());
        } else {
            b.append(" barrier=");
            b.append(arg1);
        }

        b.append(" }");
        return b.toString();
    }

    void writeToProto(ProtoOutputStream proto, long fieldId) {
        final long messageToken = proto.start(fieldId);
        proto.write(MessageProto.WHEN, when);

        if (target != null) {
            if (callback != null) {
                proto.write(MessageProto.CALLBACK, callback.getClass().getName());
            } else {
                proto.write(MessageProto.WHAT, what);
            }

            if (arg1 != 0) {
                proto.write(MessageProto.ARG1, arg1);
            }

            if (arg2 != 0) {
                proto.write(MessageProto.ARG2, arg2);
            }

            if (obj != null) {
                proto.write(MessageProto.OBJ, obj.toString());
            }

            proto.write(MessageProto.TARGET, target.getClass().getName());
        } else {
            proto.write(MessageProto.BARRIER, arg1);
        }

        proto.end(messageToken);
    }

    public static final Parcelable.Creator<Message> CREATOR
            = new Parcelable.Creator<Message>() {
        public Message createFromParcel(Parcel source) {
            Message msg = Message.obtain();
            msg.readFromParcel(source);
            return msg;
        }

        public Message[] newArray(int size) {
            return new Message[size];
        }
    };

    public int describeContents() {
        return 0;
    }

    public void writeToParcel(Parcel dest, int flags) {
        if (callback != null) {
            throw new RuntimeException(
                "Can't marshal callbacks across processes.");
        }
        dest.writeInt(what);
        dest.writeInt(arg1);
        dest.writeInt(arg2);
        if (obj != null) {
            try {
                Parcelable p = (Parcelable)obj;
                dest.writeInt(1);
                dest.writeParcelable(p, flags);
            } catch (ClassCastException e) {
                throw new RuntimeException(
                    "Can't marshal non-Parcelable objects across processes.");
            }
        } else {
            dest.writeInt(0);
        }
        dest.writeLong(when);
        dest.writeBundle(data);
        Messenger.writeMessengerOrNullToParcel(replyTo, dest);
        dest.writeInt(sendingUid);
    }

    private void readFromParcel(Parcel source) {
        what = source.readInt();
        arg1 = source.readInt();
        arg2 = source.readInt();
        if (source.readInt() != 0) {
            obj = source.readParcelable(getClass().getClassLoader());
        }
        when = source.readLong();
        data = source.readBundle();
        replyTo = Messenger.readMessengerOrNullFromParcel(source);
        sendingUid = source.readInt();
    }
}

```



## ThreadLocal

ThreadLocal源码：

```java
public class ThreadLocal<T> {
    /**
     * ThreadLocals rely on per-thread linear-probe hash maps attached
     * to each thread (Thread.threadLocals and
     * inheritableThreadLocals).  The ThreadLocal objects act as keys,
     * searched via threadLocalHashCode.  This is a custom hash code
     * (useful only within ThreadLocalMaps) that eliminates collisions
     * in the common case where consecutively constructed ThreadLocals
     * are used by the same threads, while remaining well-behaved in
     * less common cases.
     */
    private final int threadLocalHashCode = nextHashCode();

    /**
     * The next hash code to be given out. Updated atomically. Starts at
     * zero.
     */
    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    /**
     * The difference between successively generated hash codes - turns
     * implicit sequential thread-local IDs into near-optimally spread
     * multiplicative hash values for power-of-two-sized tables.
     */
    private static final int HASH_INCREMENT = 0x61c88647;

    /**
     * Returns the next hash code.
     */
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }

    /**
     * Returns the current thread's "initial value" for this
     * thread-local variable.  This method will be invoked the first
     * time a thread accesses the variable with the {@link #get}
     * method, unless the thread previously invoked the {@link #set}
     * method, in which case the {@code initialValue} method will not
     * be invoked for the thread.  Normally, this method is invoked at
     * most once per thread, but it may be invoked again in case of
     * subsequent invocations of {@link #remove} followed by {@link #get}.
     *
     * <p>This implementation simply returns {@code null}; if the
     * programmer desires thread-local variables to have an initial
     * value other than {@code null}, {@code ThreadLocal} must be
     * subclassed, and this method overridden.  Typically, an
     * anonymous inner class will be used.
     *
     * @return the initial value for this thread-local
     */
    protected T initialValue() {
        return null;
    }

    /**
     * Creates a thread local variable. The initial value of the variable is
     * determined by invoking the {@code get} method on the {@code Supplier}.
     *
     * @param <S> the type of the thread local's value
     * @param supplier the supplier to be used to determine the initial value
     * @return a new thread local variable
     * @throws NullPointerException if the specified supplier is null
     * @since 1.8
     */
    public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
        return new SuppliedThreadLocal<>(supplier);
    }

    /**
     * Creates a thread local variable.
     * @see #withInitial(java.util.function.Supplier)
     */
    public ThreadLocal() {
    }

    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

    /**
     * Returns {@code true} if there is a value in the current thread's copy of
     * this thread-local variable, even if that values is {@code null}.
     *
     * @return {@code true} if current thread has associated value in this
     *         thread-local variable; {@code false} if not
     */
    boolean isPresent() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        return map != null && map.getEntry(this) != null;
    }

    /**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * @return the initial value
     */
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            map.set(this, value);
        } else {
            createMap(t, value);
        }
        if (this instanceof TerminatingThreadLocal) {
            TerminatingThreadLocal.register((TerminatingThreadLocal<?>) this);
        }
        return value;
    }

    /**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            map.set(this, value);
        } else {
            createMap(t, value);
        }
    }

    /**
     * Removes the current thread's value for this thread-local
     * variable.  If this thread-local variable is subsequently
     * {@linkplain #get read} by the current thread, its value will be
     * reinitialized by invoking its {@link #initialValue} method,
     * unless its value is {@linkplain #set set} by the current thread
     * in the interim.  This may result in multiple invocations of the
     * {@code initialValue} method in the current thread.
     *
     * @since 1.5
     */
     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null) {
             m.remove(this);
         }
     }

    /**
     * Get the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param  t the current thread
     * @return the map
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    /**
     * Create the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the map
     */
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }

    /**
     * Factory method to create map of inherited thread locals.
     * Designed to be called only from Thread constructor.
     *
     * @param  parentMap the map associated with parent thread
     * @return a map containing the parent's inheritable bindings
     */
    static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
        return new ThreadLocalMap(parentMap);
    }

    /**
     * Method childValue is visibly defined in subclass
     * InheritableThreadLocal, but is internally defined here for the
     * sake of providing createInheritedMap factory method without
     * needing to subclass the map class in InheritableThreadLocal.
     * This technique is preferable to the alternative of embedding
     * instanceof tests in methods.
     */
    T childValue(T parentValue) {
        throw new UnsupportedOperationException();
    }

    /**
     * An extension of ThreadLocal that obtains its initial value from
     * the specified {@code Supplier}.
     */
    static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

        private final Supplier<? extends T> supplier;

        SuppliedThreadLocal(Supplier<? extends T> supplier) {
            this.supplier = Objects.requireNonNull(supplier);
        }

        @Override
        protected T initialValue() {
            return supplier.get();
        }
    }

    /**
     * ThreadLocalMap is a customized hash map suitable only for
     * maintaining thread local values. No operations are exported
     * outside of the ThreadLocal class. The class is package private to
     * allow declaration of fields in class Thread.  To help deal with
     * very large and long-lived usages, the hash table entries use
     * WeakReferences for keys. However, since reference queues are not
     * used, stale entries are guaranteed to be removed only when
     * the table starts running out of space.
     */
    static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

        /**
         * The initial capacity -- MUST be a power of two.
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;

        /**
         * The number of entries in the table.
         */
        private int size = 0;

        /**
         * The next size value at which to resize.
         */
        private int threshold; // Default to 0

        /**
         * Set the resize threshold to maintain at worst a 2/3 load factor.
         */
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }

        /**
         * Increment i modulo len.
         */
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }

        /**
         * Decrement i modulo len.
         */
        private static int prevIndex(int i, int len) {
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }

        /**
         * Construct a new map initially containing (firstKey, firstValue).
         * ThreadLocalMaps are constructed lazily, so we only create
         * one when we have at least one entry to put in it.
         */
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }

        /**
         * Construct a new map including all Inheritable ThreadLocals
         * from given parent map. Called only by createInheritedMap.
         *
         * @param parentMap the map associated with parent thread.
         */
        private ThreadLocalMap(ThreadLocalMap parentMap) {
            Entry[] parentTable = parentMap.table;
            int len = parentTable.length;
            setThreshold(len);
            table = new Entry[len];

            for (Entry e : parentTable) {
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                    if (key != null) {
                        Object value = key.childValue(e.value);
                        Entry c = new Entry(key, value);
                        int h = key.threadLocalHashCode & (len - 1);
                        while (table[h] != null)
                            h = nextIndex(h, len);
                        table[h] = c;
                        size++;
                    }
                }
            }
        }

        /**
         * Get the entry associated with key.  This method
         * itself handles only the fast path: a direct hit of existing
         * key. It otherwise relays to getEntryAfterMiss.  This is
         * designed to maximize performance for direct hits, in part
         * by making this method readily inlinable.
         *
         * @param  key the thread local object
         * @return the entry associated with key, or null if no such
         */
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.refersTo(key))
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }

        /**
         * Version of getEntry method for use when key is not found in
         * its direct hash slot.
         *
         * @param  key the thread local object
         * @param  i the table index for key's hash code
         * @param  e the entry at table[i]
         * @return the entry associated with key, or null if no such
         */
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                if (e.refersTo(key))
                    return e;
                if (e.refersTo(null))
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }

        /**
         * Set the value associated with key.
         *
         * @param key the thread local object
         * @param value the value to be set
         */
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.refersTo(key)) {
                    e.value = value;
                    return;
                }

                if (e.refersTo(null)) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }

        /**
         * Remove the entry for key.
         */
        private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.refersTo(key)) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }

        /**
         * Replace a stale entry encountered during a set operation
         * with an entry for the specified key.  The value passed in
         * the value parameter is stored in the entry, whether or not
         * an entry already exists for the specified key.
         *
         * As a side effect, this method expunges all stale entries in the
         * "run" containing the stale entry.  (A run is a sequence of entries
         * between two null slots.)
         *
         * @param  key the key
         * @param  value the value to be associated with key
         * @param  staleSlot index of the first stale entry encountered while
         *         searching for key.
         */
        private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

            // Back up to check for prior stale entry in current run.
            // We clean out whole runs at a time to avoid continual
            // incremental rehashing due to garbage collector freeing
            // up refs in bunches (i.e., whenever the collector runs).
            int slotToExpunge = staleSlot;
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                if (e.refersTo(null))
                    slotToExpunge = i;

            // Find either the key or trailing null slot of run, whichever
            // occurs first
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                // If we find key, then we need to swap it
                // with the stale entry to maintain hash table order.
                // The newly stale slot, or any other stale slot
                // encountered above it, can then be sent to expungeStaleEntry
                // to remove or rehash all of the other entries in run.
                if (e.refersTo(key)) {
                    e.value = value;

                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    // Start expunge at preceding stale entry if it exists
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                // If we didn't find stale entry on backward scan, the
                // first stale entry seen while scanning for key is the
                // first still present in the run.
                if (e.refersTo(null) && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            // If key not found, put new entry in stale slot
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            // If there are any other stale entries in run, expunge them
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }

        /**
         * Expunge a stale entry by rehashing any possibly colliding entries
         * lying between staleSlot and the next null slot.  This also expunges
         * any other stale entries encountered before the trailing null.  See
         * Knuth, Section 6.4
         *
         * @param staleSlot index of slot known to have null key
         * @return the index of the next null slot after staleSlot
         * (all between staleSlot and this slot will have been checked
         * for expunging).
         */
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }

        /**
         * Heuristically scan some cells looking for stale entries.
         * This is invoked when either a new element is added, or
         * another stale one has been expunged. It performs a
         * logarithmic number of scans, as a balance between no
         * scanning (fast but retains garbage) and a number of scans
         * proportional to number of elements, that would find all
         * garbage but would cause some insertions to take O(n) time.
         *
         * @param i a position known NOT to hold a stale entry. The
         * scan starts at the element after i.
         *
         * @param n scan control: {@code log2(n)} cells are scanned,
         * unless a stale entry is found, in which case
         * {@code log2(table.length)-1} additional cells are scanned.
         * When called from insertions, this parameter is the number
         * of elements, but when from replaceStaleEntry, it is the
         * table length. (Note: all this could be changed to be either
         * more or less aggressive by weighting n instead of just
         * using straight log n. But this version is simple, fast, and
         * seems to work well.)
         *
         * @return true if any stale entries have been removed.
         */
        private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.refersTo(null)) {
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);
            return removed;
        }

        /**
         * Re-pack and/or re-size the table. First scan the entire
         * table removing stale entries. If this doesn't sufficiently
         * shrink the size of the table, double the table size.
         */
        private void rehash() {
            expungeStaleEntries();

            // Use lower threshold for doubling to avoid hysteresis
            if (size >= threshold - threshold / 4)
                resize();
        }

        /**
         * Double the capacity of the table.
         */
        private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (Entry e : oldTab) {
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }

        /**
         * Expunge all stale entries in the table.
         */
        private void expungeStaleEntries() {
            Entry[] tab = table;
            int len = tab.length;
            for (int j = 0; j < len; j++) {
                Entry e = tab[j];
                if (e != null && e.refersTo(null))
                    expungeStaleEntry(j);
            }
        }
    }
}
```



## 总结

发送消息流程：



处理消息流程：





