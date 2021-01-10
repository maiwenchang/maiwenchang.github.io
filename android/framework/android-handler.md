# 理解Handler机制

Android中的Handler机制由Looper，MessageQueue，Handler和Message共同作用组成。Handler机制在Android中非常重要，个人觉得需要充分理解了Handler机制，才能理解Android系统中调度任务和处理事件的基本方式。它就像现实中的事件处理机制，举个例子，比如我们要申请公租房：

1. 先准备好资料；
2. 在政府网站系统上，向户政部门提出申请；
3. 资料提交后，记录到网站系统的数据库中，等待处理；
4. 网站系统持续从数据库中读取出申请资料，并发给各部门处理，当读到了你的公租房申请资料时，就把它发给户政部门；
5. 户政部门收到了公租房申请资料，对它审核并反馈结果。

那么问题来了，在上面的例子中，Looper，MessageQueue，Handler和Message分别对应什么角色？

## 解读Looper类







### ThreadLocal







## 解读MessageQueue类





## 解读Message类







## 解读Handler类

Handler的一些特性：

-  Handler允许我们向它关联的线程发送消息，或者post一个Runnable到它关联的线程。

 * 每个Handler只能关联一个线程和该线程里的消息队列。
 * 当你创建Handler时，它默认会关联到当前线程；如果你在构造方案里传递另一个线程的Looper，那就关联到另一个线程。
 * 根据上面的特性，它发送消息和Runnable到关联线程的消息队列，并在关联的线程处理他们。
 * 

Handler源码：

```java
package android.os;

/**
 * <p>There are two main uses for a Handler: (1) to schedule messages and
 * runnables to be executed as some point in the future; and (2) to enqueue
 * an action to be performed on a different thread than your own.
 * 
 * <p>Scheduling messages is accomplished with the
 * {@link #post}, {@link #postAtTime(Runnable, long)},
 * {@link #postDelayed}, {@link #sendEmptyMessage},
 * {@link #sendMessage}, {@link #sendMessageAtTime}, and
 * {@link #sendMessageDelayed} methods.  The <em>post</em> versions allow
 * you to enqueue Runnable objects to be called by the message queue when
 * they are received; the <em>sendMessage</em> versions allow you to enqueue
 * a {@link Message} object containing a bundle of data that will be
 * processed by the Handler's {@link #handleMessage} method (requiring that
 * you implement a subclass of Handler).
 * 
 * <p>When posting or sending to a Handler, you can either
 * allow the item to be processed as soon as the message queue is ready
 * to do so, or specify a delay before it gets processed or absolute time for
 * it to be processed.  The latter two allow you to implement timeouts,
 * ticks, and other timing-based behavior.
 * 
 * <p>When a
 * process is created for your application, its main thread is dedicated to
 * running a message queue that takes care of managing the top-level
 * application objects (activities, broadcast receivers, etc) and any windows
 * they create.  You can create your own threads, and communicate back with
 * the main application thread through a Handler.  This is done by calling
 * the same <em>post</em> or <em>sendMessage</em> methods as before, but from
 * your new thread.  The given Runnable or Message will then be scheduled
 * in the Handler's message queue and processed when appropriate.
 */
public class Handler {
    
    /*
     * 把这个变量设置成ture，在构造方法会检测子类是不是非静态匿名内部类Handler，
     * 它们可能会造成内存泄漏。检测后没有报异常，只是打印了一条Log。
     */
    private static final boolean FIND_POTENTIAL_LEAKS = false;

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
     * 同步执行一个任务
     * 
     * If the current thread is the same as the handler thread, then the runnable
     * runs immediately without being enqueued.  Otherwise, posts the runnable
     * to the handler and waits for it to complete before returning.
     * </p><p>
     * This method is dangerous!  Improper use can result in deadlocks.
     * Never call this method while any locks are held or use it in a
     * possibly re-entrant manner.
     * </p><p>
     * This method is occasionally useful in situations where a background thread
     * must synchronously await completion of a task that must run on the
     * handler's thread.  However, this problem is often a symptom of bad design.
     * Consider improving the design (if possible) before resorting to this method.
     * </p><p>
     * One example of where you might want to use this method is when you just
     * set up a Handler thread and need to perform some initialization steps on
     * it before continuing execution.
     * </p><p>
     * If timeout occurs then this method returns <code>false</code> but the runnable
     * will remain posted on the handler and may already be in progress or
     * complete at a later time.
     * </p><p>
     * When using this method, be sure to use {@link Looper#quitSafely} when
     * quitting the looper.  Otherwise {@link #runWithScissors} may hang indefinitely.
     * (TODO: We should fix this by making MessageQueue aware of blocking runnables.)
     * </p>
     *
     * @param r The Runnable that will be executed synchronously.
     * @param timeout The timeout in milliseconds, or 0 to wait indefinitely.
     *
     * @return Returns true if the Runnable was successfully executed.
     *         Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     *
     * @hide This method is prone to abuse and should probably not be in the API.
     * If we ever do make it part of the API, we might want to rename it to something
     * less funny like runUnsafe().
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

        BlockingRunnable br = new BlockingRunnable(r);
        return br.postAndWait(this, timeout);
    }

    /**
     * Pushes a message onto the end of the message queue after all pending messages
     * before the current time. It will be received in {@link #handleMessage},
     * in the thread attached to this handler.
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
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

    final Looper mLooper;
    final MessageQueue mQueue;
    final Callback mCallback;
    final boolean mAsynchronous;
    IMessenger mMessenger;

    private static final class BlockingRunnable implements Runnable {
        private final Runnable mTask;
        private boolean mDone;

        public BlockingRunnable(Runnable task) {
            mTask = task;
        }

        @Override
        public void run() {
            try {
                mTask.run();
            } finally {
                synchronized (this) {
                    mDone = true;
                    notifyAll();
                }
            }
        }

        public boolean postAndWait(Handler handler, long timeout) {
            if (!handler.post(this)) {
                return false;
            }

            synchronized (this) {
                if (timeout > 0) {
                    final long expirationTime = SystemClock.uptimeMillis() + timeout;
                    while (!mDone) {
                        long delay = expirationTime - SystemClock.uptimeMillis();
                        if (delay <= 0) {
                            return false; // timeout
                        }
                        try {
                            wait(delay);
                        } catch (InterruptedException ex) {
                        }
                    }
                } else {
                    while (!mDone) {
                        try {
                            wait();
                        } catch (InterruptedException ex) {
                        }
                    }
                }
            }
            return true;
        }
    }
}
```