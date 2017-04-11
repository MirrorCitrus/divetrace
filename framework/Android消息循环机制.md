# Android消息循环机制: Handler, MessageQueue， Message和Looper

## 概述

- 总述

Handler-Looper-Message共同构建了Android的异步消息处理的机制。它负责为一个线程构建消息循环（可以理解为一种“死循环”），并提供了线程之间通过发送消息来互相通信和执行任务的模型。其中，Looper负责和当前线程绑定并执行消息循环；Handler负责发送消息和在对应的线程当中接收并处理消息；Message负责封装需要处理的消息，从来源上主要分为系统消息、输入消息、用户消息。从类型上可以分为静态消息(例如我把设置了message.what等静态量的消息称为静态消息)、动态消息（我把postRunnable这种方式称为动态消息）。

- 不得不说的主线程

应用程序的主线程就是一个含有消息循环的线程。我们最多的也是构建和主线程通讯的Handler，然后向主线程发送消息，或者是发送需要在主线程执行的任务。
当然，这个不仅限于主线程，知道了Looper.prepare和Looper.loop的用法，我们就可以为自己构建的线程创建消息循环。而HandlerThread则是这个操作的最佳实践：系统帮助你封装了一个具有消息循环的Thread，避免了一些问题。

- 进阶

消息循环的本质：Linux的pipe/epoll机制

## 源码分析
### Looper.prepare()和Looper.loop()

Looper.prepare()方法的主要作用就是创建一个和当前线程绑定的Looper实例。创建的方式借助于ThreadLocal。

```
public final class Looper {
   
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
}
```

Looper.loop()方法则是具体的消息循环：基本可以理解为：在线程内部以类似于“死循环”的方式不断取消息(`MessageQueue#next()`)然后调用dispatchMessage。而msg的target就是Handler，所以说，是在looper对应的线程中，调用Handler的dispatchMessage方法。

```
public static void loop() {
    final Looper me = myLooper();
    final MessageQueue queue = me.mQueue;

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity ();

    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            return;
        }
        msg.target .dispatchMessage(msg);

        // Make sure that during the course of dispatching the
        // identity of the thread wasn't corrupted.
        final long newIdent = Binder.clearCallingIdentity ();
        if (ident != newIdent) {
            Log.wtf( TAG, "Thread identity changed from 0xxxx to 0xxxxx while dispatching to messagexx");
        }

        msg.recycleUnchecked();
    }
}

```

### 消息的send和handle的过程

handler的各种sendxxx和post的方法，最终其实都调了enqueueMessage，我们看看实现：其实基本可以认为，就是把message放到列表中，然后需要唤醒的时候调用nativeWake。

```
boolean enqueueMessage(Message msg, long when) {
    synchronized (this) {
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg. next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p. target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p. when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev. next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake( mPtr);
        }
    }
    return true ;
}

```

## 主线程消息循环的构建

应用（进程）的主线程由ZygoteInit启动，入口是ActivityThread.main()方法。ActivityThread本身并不是一个线程，只是某种和应用的主线程有莫大的关联。

ActivityThread.main()方法的主要作用是构建ActivityThread的实例，并构建主线程消息循环。消息循环的构建过程和普通线程无异，区别仅在于Looper实例创建后要不要赋值给Looper的静态变量sMainLooper以及是否允许退出，这也是Looper.prepareMainLooper()和Looper.prepare()的差异。

```
    public static void main(String[] args) {
       
        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

# References












