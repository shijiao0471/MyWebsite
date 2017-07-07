---
title: Android Handler 消息循环机制源码分析等
---
### 等:

注意标题最后的等，这个等并不属于`Handler`消息循环机制，首先我想说的是在实际使用`Handler`中容易出现的问题，
然后再说消息循环机制。

- 内存泄漏

##### 什么是内存泄漏？

Java中的对象(非基本数据类型)通过`new`关键字分配内存空间，正常情况下对象使用完毕将会释放。
但是由于某种原因(短生命周期对象持有一个长生命周期对象)导致
该段内存无法释放，一直占用内存单元，我们的程序就无法继续使用该内存。我们就称无法被释放的内存为泄漏内存。


##### `Handler`为什么会造成内存泄漏

平时我们怎么声明一个`Handler`呢？ 通过下面的方式？
```
Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            dosomething();
        }
    };
```
这样写其实AndroidStudio 会抛出一个警告，说你这样写会造成内存泄漏。为什么这样的写法会造成内存泄漏呢？
`Java中非静态内部类会持有外部类的引用`。如果内部类的生命周期比外部类长，那么外部类就容易造成内存泄漏。
回到上面的代码，我们定义的mHandler对象其实是一个匿名类，他会隐式的持有外部类的引用，他可以使用外部类的成员变量
，这样就佐证了之前说的非静态内部类会持有外部类的引用。
这时候如果子线程使用mHandler将message发送到MessageQueue中并等待执行的过程长，如果这时候activity已经执行了finish方法
我们希望的是activity在执行了onDestroy方法后，activity的相关资源被销毁回收，但由于mHandler隐式的持有activity的引用
，那么将导致activity对象无法被gc掉，activity相关资源与组件也无法被回收，造成内存泄露。

解决办法：
1. 使用static静态声明

这样的话mHandler将无法持有外部activity的引用，但也导致了在handleMessage方法内不能访问activity中的对象了，这时候我们
需要在Handler对象的声明中增加一个对activity的弱引用

```
static class MyHandler extends Handler {
    WeakReference<Activity > mActivityReference;
    MyHandler(Activity activity) {
        mActivityReference= new WeakReference<Activity>(activity);
    }
    @Override
    public void handleMessage(Message msg) {
        final Activity activity = mActivityReference.get();
        if (activity != null) {
            mImageView.setImageBitmap(mBitmap);
        }
    }
}
```

小tips：*什么是WeakReference？*
>WeakReference弱引用，与强引用（即我们常说的引用）相对，它的特点是，GC在回收时会忽略掉弱引用，即就算有弱引用指向某对象，但只要该对象没有被强引用指向（实际上多数时候还要求没有软引用，但此处软引用的概念可以忽略），该对象就会在被GC检查到时回收掉。对于上面的代码，用户在关闭Activity之后，就算后台线程还没结束，但由于仅有一条来自Handler的弱引用指向Activity，所以GC仍然会在检查的时候把Activity回收掉。这样，内存泄露的问题就不会出现了。

2. 在onDestroy中释放handler
```
mHandler.removeCallbacks();
mHandler.removeMessages();
mHandler.removeCallbacksAndMessages();
```


### Android Handler 消息循环机制源码分析:

Android源码相关类：

```
Handler    Looper    MessageQueue    ThreadLocal    Message
```

我们平时使用Handler一般是这样的：
```
    //1. 主线程
    Handler handler = new MyHandler();

    //2. 非主线程
    HandlerThread handlerThread = new HandlerThread("handlerThread");
    handlerThread.start();
    Handler handler = new Handler(handlerThread.getLooper());

    //发送消息
    handler.sendMessage(msg);
    
    //接收消息
    static class MyHandler extends Handler {
        //对于非主线程处理消息需要传Looper，主线程有默认的sMainLooper
        public MyHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
    }
```

那么现在就从非主线程启动handler看看android的运行机制吧。

handlerThread.start()的时候，实际上创建了一个用于消息循环的Looper和消息队列MessageQueue，同时启动了消息循环，
并将这个循环传给Handler，这个循环会从MessageQueue中依次取任务出来执行。用户若要执行某项任务，
只需要调用handler.sendMessage即可，这里做的事情是将消息添加到MessageQueue中。对于主线程也类似，
只是主线程sMainThread和sMainLooper不需要我们主动去创建，程序启动的时候Application就创建好了，我们只需要创建Handler即可。

#### Handler
当handler调用sendMessage()方法or别的发送消息的方法，其实最后都是调用了sendMessageAtTime()
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

    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }

    /**
     * Handle system messages here.
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

```
其实sendMessageAtTime方法就是将一个message消息放入了MessageQueue这个队列。不对，不要被MessageQueue这个名字误导了，
MessageQueue管理的是一个Message的单项链表而并非队列，那么MessageQueue中的message是什么时候被取出来的呢，这时候就需要将一下Looper了。

#### Looper
一个线程中只能有一个looper
```
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```
这里用到了ThreadLocal这个类，在每个线程中保存looper信息。

looper的核心功能其实是在 loop()这个方法中
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
我们看到这个方法中有一个for(;;)意思和while(true)一样，不停的从MessageQueue中取数据，然后 msg.target.dispatchMessage(msg);
其实是调用了handler中的dispatchMessage方法，最终调用了我们实现的handleMessage。
Looper主要作用：
1. 与当前线程绑定，保证一个线程只会有一个Looper实例，同时一个Looper实例也只有一个MessageQueue。
2. loop()方法，不断从MessageQueue中去取消息，交给消息的target属性的dispatchMessage去处理。

#### MessageQueue
MessageQueue 主要的方法有(源码有点多我就不贴出来了，感兴趣的可以自己去看)：
```
enqueueMessage() 1
next() 2
removeMessages() 3

```
1. 消息入队方法enqueueMessage(Message msg, long when)。其处理过程如下：

待入队的Message标记为InUse，when赋值,若消息链表mMessages为空为空，或待入队Message执行时间小于mMessage链表头，则待入队Message添加到链表头
若不符合以上条件，则轮询链表，根据when从低到高的顺序，插入链表合适位置。

2. 消息轮询

next()依次从MessageQueue中取出Message

3. 移除消息

removeMessages()可以移除消息，做的事情实际上就是将消息从链表移除，同时将移除的消息添加到消息池，提供循环复用
