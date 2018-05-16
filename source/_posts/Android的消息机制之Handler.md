---
title: Android的消息机制之Handler
date: 2017-11-10 16:11:22
tags:
---

# 什么是Handler?
*Handler:*Handler是Android消息机制的上层接口,是一个消息分发对象,进行发送和处理消息,Handler的运行需要底层的*MessageQueue*和*Looper*的支撑.
*作用:*调度消息,将一个任务切换到某个指定的线程中去执行.(从本质上来说,Handler并不是专门用于更新,只是常被开发者用来更新UI).

# 为什么要有Handler?

因为Android规定访问UI只能在主线程中进行,在子线程中访问UI,程序就会抛出异常.ViewRootImpl对UI操作做了验证.

```
  void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
```

假设我们在子线程从网络获取数据,然后又想访问UI,如果没有Handler,那么将没办法将访问UI的工作切换到主线程.所以:*就是为了解决在子线程中无法访问UI的矛盾*.

# 为什么不允许在子线程访问UI

主要是因为Android的UI控件不是线程安全的,如果在多线程中并发访问会导致不可预期的后果.如果加上锁机制会使UI访问的逻辑变得复杂,同时锁机制会降低访问的效率.

# Handler的使用

## post(Runnable)
```
      new Thread(new Runnable() {
            @Override
            public void run() {
                //我可以做耗时任务
                handler.post(new Runnable() {
                    @Override
                    public void run() {
                        //我可以更新UI
                    }
                });
            }
        });
```

# sendMessage(Message)

```
 private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            if (msg.what == 1) {
                // data:我是从网络获取的
                String data = (String) msg.obj;
                //我可以更新UI

            }
        }
    };
    private void downloadAndUpdateUI() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                //从网络获取User信息
                String data = "我是从网络获取的";
                //将获取的数据通过Message包装起来,使用Handler发送过去
                Message message = new Message();
                message.what = 1;//消息标志
                message.obj = data;
                mHandler.sendMessage(message);
            }
        });

    }
```

# Handler引起的内存泄漏

## 内存泄漏会如何发生的

 当使用内部类或匿名内部类的方式创建Handler时,Handler对象会隐式的持有一个外部类对象的引用.

比如:在耗时任务中开启了一个子线程进行网络请求,如果任务还未结束,但是Activity被finish掉了,此时GC回收Activity并不会成功.因为子线程没有执行结束,子线程持有Handler的引用,而Handler又持有Activity的引用,直接回导致Activity对象无法被GC回收.

## 解决方法
### 1.将Handler声明为静态内部类,因为静态内部类不会持有外部类的引用,所以不会导致外部类实例出现内存泄漏

### 2.在Handler中添加对外部Activity的弱引用.由于Handler被声明为静态内部类,不再持有外部类对象的引用,导致无法在handlerMessage()中操作Activity中的对象,所以需要在Handler中增加一个对Activity的弱引用.

```
 private final MyHandler mHandler = new MyHandler(this);  
  
    private static class MyHandler extends Handler {  
        private final WeakReference<MainActivity> mActivity;  
  
        public MyHandler(MainActivity activity) {  
            this.mActivity = new WeakReference<MainActivity>(activity);  
        }  
  
        @Override  
        public void handleMessage(Message msg) {  
            MainActivity mainActivity = mActivity.get();  
            if (mainActivity == null) {  
                return;  
            }  
            // your code here  
        }  
    }  
```

# Handler如何与Looper,MessageQueue合作的

![]("https://github.com/huangdashun/blog/blob/master/source/_posts/Handler/Handler%E7%9A%84%E9%80%9A%E4%BF%A1.png")
 

## MessageQueue
 中文翻译: 消息队列,但是它内部是采用*单链表*的数据结构来存储消息列表.

### 插入(enqueueMessage)

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

### 读取(next)

 是一个无线循环的方法,如果队列中没有消息,那么next方法会一直阻塞在这里.
那么什么时候退出循环呢,当mQuitting 为true时,那什么时候为true,当Looper调用quit或quitSafely来终止消息循环时.

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
```

## Looper
中文翻译:消息循环,Looper会以无限循环的形式去查找MessageQueue是否有新消息.

线程默认是没有Looper的,所以想使用Handler就必须为线程创建Handler,UI线程也就是ActivityThread,它被创建时会初始化Looper,所以在主线程中可以使用Handler.

```
        new Thread(new Runnable() {
            @Override
            public void run() {
                Looper.prepare();
                Handler handler = new Handler();
                Looper.loop();
            }
        });
```

Looper提供了prepareMainLooper,主要给主线程也就是ActivityThread创建Looper使用的.

在任何地方都可以通过getMainLooper获取到主线程的Looper.

### Looper的loop方法

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

            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
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
```

唯一跳出循环的办法是MessageQueue的next方法返回了null,当Looper的quit或者quitSafely方法被调用时,Looper就会调用MessageQueue的quit方法来通知消息队列退出.

### Quit和QuitSafely的区别

quit会直接退出Looper.
quitSafely只是设定一个标记,等消息队列的消息全部处理完毕后才会安全的退出.

## Handler的工作原理

handler的用法前面已经介绍过.具体方法如下.

*Handler.post(Runnable r)*

```
public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
```

```
  public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
```

```
  public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```

```
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
```

```
  private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

看到这里最后又走到了 * queue.enqueueMessage* ,然后loop就会在MessageQueue获取到该消息.最后由Looper将该消息交给Handler处理,则Handler的dispatchMessage会被调用.

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

1.检测Message的callback是否为null,不为null就由handlerCallback来处理.
也就是处理的Handler的post方法所传递的Runnable参数

```
 private static void handleCallback(Message message) {
        message.callback.run();
    }
```

2.检验mCallback是否为null,不为null就调用mCallback的handleMessage方法来处理消息. mCallback是个接口.

```
  public interface Callback {
        public boolean handleMessage(Message msg);
    }
```

如果想不派生子类创建一个Handler的实例就可以用它.
`Handler mHandler = new Handler(callback);`

3.最后调用handleMessage方法来处理.

## ThreadLocal
作用:可以在每个线程中存储数据,读写操作仅局限于各自线程的内部,通过ThreadLocal可以轻松获取每个线程的Looper,可以在不同的线程中互不干扰的存储并提供数据.

# 主线程中的Looper.loop()一直无限循环为什么不会造成ANR？

## ANR造成的原因：
1. 当前的事件没有机会得到处理
2. 当前的事件正在处理，但没有及时完成。

## 源码分析
### ActivityThread源码

```
    public static final void main(String[] args) {
        ...
        //创建Looper和MessageQueue
        Looper.prepareMainLooper();
        ...
        //轮询器开始轮询
        Looper.loop();
        ...
    }
```
#### Looper.loop()方法


```
   while (true) {
       //取出消息队列的消息，可能会阻塞
       Message msg = queue.next(); // might block
       ...
       //解析消息，分发消息
       msg.target.dispatchMessage(msg);
       ...
    }
```
**反证：如果main方法中没有looper循环，那么主线程一运行完毕就会退出，肯定不行。所以main方法主要做的就是消息循环。**

### 那么为什么这个死循环不会造成ANR

因为 Android 的是由事件驱动的，looper.loop() 不断地接收事件、处理事件，每一个点击触摸或者说 Activity 的生命周期都是运行在 Looper.loop() 的控制之下，如果它停止了，应用也就停止了。只能是某一个消息或者说对消息的处理阻塞了Looper.loop()，而不是 Looper.loop() 阻塞它。

**所以我们的代码其实都是在该循环里执行的，当然不会阻塞。**

#### handleMessage方法部分源码


```
public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
            case LAUNCH_ACTIVITY: {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
                r.packageInfo = getPackageInfoNoCheck(r.activityInfo.applicationInfo, r.compatInfo);
                handleLaunchActivity(r, null);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }
            break;
            case RELAUNCH_ACTIVITY: {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityRestart");
                ActivityClientRecord r = (ActivityClientRecord) msg.obj;
                handleRelaunchActivity(r);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }
            break;
            case PAUSE_ACTIVITY:
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                handlePauseActivity((IBinder) msg.obj, false, (msg.arg1 & 1) != 0, msg.arg2, (msg.arg1 & 2) != 0);
                maybeSnapshot();
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                break;
            case PAUSE_ACTIVITY_FINISHING:
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                handlePauseActivity((IBinder) msg.obj, true, (msg.arg1 & 1) != 0, msg.arg2, (msg.arg1 & 1) != 0);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                break;
            ...........
        }
    }
```
可以看到Activity的生命周期都是依靠主线程的Looper.loop，当收到不同Message采取不同的措施。

**总结：Looer.loop()方法可能会引起主线程的阻塞，但只要它的消息循环没有被阻塞，能一直处理事件就不会产生ANR异常。**
