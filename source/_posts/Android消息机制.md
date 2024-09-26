---
title: Android消息机制
date: 2020-10-04 21:26:43
categories: 
- Android
- 系统源码
tag: 
- Android
- 系统源码
---

### Handler,MessageQueue,Runnable和Looper

描述：Looper不断的从MessageQueue中获取一个Message，然后由Handler来处理。

### Handler

- 每个Thread只对应一个Looper;
- 每个Looper只对应一个MessageQueue;
- 每个MessageQueue中有N个Message;
- 每个Mesaage最多指定一个Handler来处理事件。

可以推出Thread和Handler是一对多关系。

```java
public class Handler {
    final Looper mLooper; 
    final MessageQueue mQueue;
    final Callback mCallback;
    final boolean mAsynchronous;
     /**
     * Subclasses must implement this to receive messages.
     */
    public void handleMessage(Message msg) {
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
}
```

Looper从MessageQueue中取出一个Message后，首先会调用Handler.dispatchMessage进行消息派发，由上面方法可以看出有三种处理情况，

1、Message是通过Runnable发送的，这个Runnable会封装成Message赋值给Message的CallBack，分发消息的时候会先执行这个Runnable;

2、如何Handle以Handler(Callback callback，)这种带有 Callback 的参数初始化会执行这个实现里的handleMessage方法；

最后才执行handle里的handleMessage方法。

### 发送消息

```java
  public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```

延迟多长时间发送，内部通过当前时间+延时时长计算出具体的时间点，然后发送到消息队列。

具体发送到消息队列的方法如下：

```java
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

由上面的方法可以看出，消息队列并不是队列，而是单链表结构。


### Looper死循环为什么不会导致应用卡死，会耗费大量资源吗？

从前面的主线程、子线程的分析可以看出，Looper会在线程中不断的检索消息，如果是子线程的Looper死循环，一旦任务完成，用户应该手动退出，而不是让其一直休眠等待。（引用自Gityuan）线程其实就是一段可执行的代码，当可执行的代码执行完成后，线程的生命周期便该终止了，线程退出。而对于主线程，我们是绝不希望会被运行一段时间，自己就退出，那么如何保证能一直存活呢？简单做法就是可执行代码是能一直执行下去的，死循环便能保证不会被退出，例如，binder 线程也是采用死循环的方法，通过循环方式不同与 Binder 驱动进行读写操作，当然并非简单地死循环，无消息时会休眠。**Android是基于消息处理机制的，用户的行为都在这个Looper循环中，我们在休眠时点击屏幕，便唤醒主线程继续进行工作**。

主线程的死循环一直运行是不是特别消耗 CPU 资源呢？ 其实不然，这里就涉及到 Linux  pipe/epoll机制，简单说就是在主线程的 MessageQueue 没有消息时，便阻塞在 loop 的 queue.next() 中的 nativePollOnce() 方法里，此时主线程会释放 CPU 资源进入休眠状态，直到下个消息到达或者有事务发生，通过往 pipe 管道写端写入数据来唤醒主线程工作。这里采用的 epoll 机制，是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。 所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。

#### epoll机制

epoll机制在Handler中的应用，在主线程的 MessageQueue 没有消息时，便阻塞在 loop 的 queue.next() 中的 nativePollOnce() 方法里，最终调用到epoll_wait()进行阻塞等待。此时主线程会释放 CPU 资源进入休眠状态，直到下个消息到达或者有事务发生，通过往 pipe 管道写端写入数据来唤醒主线程工作。这里采用的 epoll 机制，是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。 所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。

### ThreadLocal

```java
public final class Looper {
    // sThreadLocal.get() will return null unless you've called prepare().
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    
     private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    
     public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
}
```

ThreadLocal可以实现Looper在线程中的存取，Handler在初始化的时候通过Looper回去Looper中的消息队列。如果不使用，那么系统就必须提供一个全局的哈希表供Handlder查询指定线程的Looper。

### Handler的同步屏障机制

如果有一个紧急的Message需要优先处理，该怎么做？这其实涉及到架构方面的设计了，通用场景和特殊场景的设计。你可能会想到`sendMessageAtFrontOfQueue()`这个方法，实际也远远不只是如此，**Handler中加入了同步屏障这种机制，来实现[异步消息优先]执行的功能**。

postSyncBarrier()发送同步屏障，removeSyncBarrier()移除同步屏障

同步屏障的作用可以理解成拦截同步消息的执行，主线程的 Looper 会一直循环调用 MessageQueue 的 `next()` 来取出队头的 Message 执行，当 Message 执行完后再去取下一个。当 `next()` 方法在取 Message 时发现队头是一个同步屏障的消息时，就会去遍历整个队列，只寻找设置了异步标志的消息，如果有找到异步消息，那么就取出这个异步消息来执行，否则就让 `next()` 方法陷入阻塞状态。如果 `next()` 方法陷入阻塞状态，那么主线程此时就是处于空闲状态的，也就是没在干任何事。所以，如果队头是一个同步屏障的消息的话，那么在它后面的所有同步消息就都被拦截住了，直到这个同步屏障消息被移除出队列，否则主线程就一直不会去处理同步屏幕后面的同步消息。

而所有消息默认都是同步消息，只有手动设置了异步标志，这个消息才会是异步消息。另外，同步屏障消息只能由内部来发送，这个接口并没有公开给我们使用。

Choreographer 里所有跟 message 有关的代码，你会发现，**都手动设置了异步消息的标志**，所以这些操作是不受到同步屏障影响的。这样做的原因可能就是为了尽可能保证上层 app 在接收到屏幕刷新信号时，可以在第一时间执行遍历绘制 View 树的工作。

Choreographer 过程中的动作也都是异步消息，这样可以确保 Choreographer 的顺利运转，也确保了第一时间执行 doTraversal（doTraversal → performTraversals 就是执行 view 的 layout、measure、draw），这个过程中如果有其他同步消息，也无法得到处理，都要等到 doTraversal 之后。

因为主线程中如果有太多消息要执行，而这些消息又是根据时间戳进行排序，如果不加一个同步屏障的话，那么遍历绘制 View 树的工作就可能被迫延迟执行，因为它也需要排队，那么就有可能出现当一帧都快结束的时候才开始计算屏幕数据，那即使这次的计算少于 16.6ms，也同样会造成丢帧现象。

那么，**有了同步屏障消息的控制就能保证每次一接收到屏幕刷新信号就第一时间处理遍历绘制 View 树的工作么？**

只能说，同步屏障是尽可能去做到，但并不能保证一定可以第一时间处理。因为，同步屏障是在 `scheduleTraversals()` 被调用时才发送到消息队列里的，也就是说，只有当某个 View 发起了刷新请求时，在这个时刻后面的同步消息才会被拦截掉。如果在 `scheduleTraversals()` 之前就发送到消息队列里的工作仍然会按顺序依次被取出来执行。

https://juejin.cn/post/6893791473121280013

