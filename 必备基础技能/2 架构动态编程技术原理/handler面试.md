# 码牛最硬核Handler面试题深度解析]

## 1.简述Handler的实现原理

**Android 应用**是通过消息驱动运行的，在 Android 中一切皆消息，包括触摸事件，视图的绘制、显示和刷新等等都是消息。Handler 是消息机制的上层接口，平时开发中我们只会接触到 Handler 和 Message，内部还有 MessageQueue 和 Looper 两大助手共同实现消息循环系统。

**（1）Handler** 通过Handler的sendXXX或者postXXX来发送一个消息，这里要注意post(Runnable r)方法也会将Runnable包装成一个Message，代码如下：

```
 public final boolean post(Runnable r){
    	return  sendMessageDelayed(getPostMessage(r), 0);
    }
    public final boolean postDelayed(Runnable r, long delayMillis){
    	return sendMessageDelayed(getPostMessage(r), delayMillis);
    }
private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    } 
```

从代码中可以看到将Runnable赋值给了Message.callback了。最终sendXXX和postXXX都会调用到sendMessageAtTime，代码如下：

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

在这个方法中最终调用了enqueueMessage方法，这里注意将this赋值给了Message.target，而此处this就是Handler。

```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        //转到 MessageQueue 的 enqueueMessage 方法
        return queue.enqueueMessage(msg, uptimeMillis);
    } 
```

enqueueMessage方法最终调用了MessageQueue的enqueueMessage方法，将消息放入队列。 **（2）MessageQueue** MessageQueue是一个优先级队列，核心方法是enqueueMessage和next方法，也就是将插入队列，将消息取出队列的操作。 之所以说MessageQueue是一个优先级队列是因为enqueueMessage方法中会根据Message的执行时间来对消息插入，这样越晚执行的消息会被插入到队列的后边。

而next方法是一个死循环，如果队列中有消息，则next方法会将Message移除队列并返回该Message，如果队列中没有消息该方法则会处于阻塞状态。

**（3）Looper** Looper可以理解为一个消息泵，Looper的核心方法是loop。注意loop方法的第一行会首先通过myLooper来得到当前线程的Looper,接着拿到Looper中的MessageQueue，然后开启一个死循环，它会不断的通过MessageQueue的next方法将消息取出来，并执行。代码如下：

```
public static void loop() {
        final Looper me = myLooper();// 这里要特别注意，是从ThreadLocal中拿到当前线程的Looper。
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        for (;;) {
            //从 MessageQueue 中取消息
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
			//通过 Handler 分发消息
            msg.target.dispatchMessage(msg);
            //回收消息
            msg.recycleUnchecked();
        }
    } 
```

可以看到在取出Message后则会调用Message.target调用dispatchMessage方法，这里target就是Handler，它是在Handler的enqueueMessage时赋值的。紧接着将Message进行了回收。 接下来再回到Handler看dispatchMessage，代码如下：

```
 public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            //通过 handler.postXxx 形式传入的 Runnable
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                //以 Handler(Handler.Callback) 写法
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            //以 Handler(){} 内存泄露写法
            handleMessage(msg);
        }
    } 
```

## 2.一个线程有几个Handler？一个线程有几个Looper？如何保证？

Handler的个数与所在线程无关，可以在线程中实例化任意多个Handler。一个线程中只有一个Looper。Looper的构造方法被声明为了private，我们无法通过new关键字来实例化Looper，唯一开放的可以实例化Looper的方法是prepare。prepare方法的源码如下：

```
 public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    } 
```

我们知道ThreadLocal是一个线程内部的数据存储类，当某个线程调用prepare方法的时候，会首先通过ThreadLocal检查这个线程是否已经创建了Looper,如果还没创建，则实例化Looper并将实例化后的Looper保存到ThreadLocal中，而如果ThreadLocal中已经保存了Looper，则会抛出一个RuntimeException的异常。那么意味着在一个线程中最多只能调用一次prepare方法，这样就保证了Looper的唯一性。

## 3.Handler线程是如何切换的？

（1）假设现在有一个线程A，在A线程中通过Looper.prepare和Looper.loop来开启Looper,并且在A线程中实例化出来一个Handler。Looper.prepare()方法被调用时会为会初始化Looper并为ThreadLocal 设置Looper，此时ThreadLocal中就存储了A线程的Looper。另外MessageQueue也会在Looper中被初始化。

（2）接着当调用Loop.loop方法时，loop方法会通过myLooper得到A线程中的Looper，进而拿到Looper中的MessageQueue，接着开启死循环等待执行MessageQueue中的方法。 （3）此时，再开启一个线程B，并在B线程中通过Handler发送出一个Message，这个Message最终会通过sendMessageAtTime方法调用到MessageQueue的equeueMessage方法将消息插入到队列。

（3）由于Looper的loop是一个死循环，当MessageQueue中被插入消息的时候，loop方法就会取出MessageQueue中的消息，并执行callback。而此时，Looper是A线程的Looper,进而调用的Message或者Handler的Callback都是执行在A线成中的。以此达到了线程的切换。

## 4.Handler内存泄漏的原因是什么？如何解决?

通常在使用Handler的时候回通过匿名内部类的方式来实例化Handler，而非静态的匿名内部类默认持有外部类的引用，即匿名内部类Handler持有了外部类。而导致内存泄漏的根本原因是是因为Handler的生命周期与宿主的生命周期不一致。

比如说在Activity中实例化了一个非静态的匿名内部类Handler，然后通过Handler发送了一个延迟消息，但是在消息还未执行时结束了Activity，此时由于Handler持有Activity，就会导致Activity无法被GC回收，也就是出现了内存泄漏的问题。

**解决方式**：可以把Handler声明为静态的匿名内部类，但这样一来，在Handler内部就没办法调用到Activity中的非静态方法或变量。那么最终的解决方案可以使用静态内部类 + 弱引用来解决。代码如下：

```
public class MainActivity extends AppCompatActivity {

    private MyHandler mMyHandler = new MyHandler(this);

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    private void handleMessage(Message msg) {

    }

    static class MyHandler extends Handler {
        private WeakReference<Activity> mReference;

        MyHandler(Activity reference) {
            mReference = new WeakReference<>(reference);
        }

        @Override
        public void handleMessage(Message msg) {
            MainActivity activity = (MainActivity) mReference.get();
            if (activity != null) {
                activity.handleMessage(msg);
            }
        }
    }

    @Override
    protected void onDestroy() {
        mMyHandler.removeCallbacksAndMessages(null);
        super.onDestroy();
    }
} 
```

## 5.主线程为什么不用初始化Looper？

**答：因为应用在启动的过程中就已经初始化主线程Looper了。**

每个java应用程序都是有一个main方法入口，Android是基于Java的程序也不例外。Android程序的入口在ActivityThread的main方法中：

```
public static void main(String[] args) {
    ...
 // 初始化主线程Looper
    Looper.prepareMainLooper();
    ...
    // 新建一个ActivityThread对象
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);
 
    // 获取ActivityThread的Handler，也是他的内部类H
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
 
    ...
    Looper.loop();
 // 如果loop方法结束则抛出异常，程序结束
    throw new RuntimeException("Main thread loop unexpectedly exited");
} 
```

main方法中先初始化主线程Looper，新建ActivityThread对象，然后再启动Looper，这样主线程的Looper在程序启动的时候就跑起来了。我们不需要再去初始化主线程Looper。

## 6.Handler如何保证MessageQueue并发访问安全？

**答：循环加锁，配合阻塞唤醒机制。**

我们可以发现MessageQueue其实是“生产者-消费者”模型，Handler不断地放入消息，Looper不断地取出，这就涉及到死锁问题。如果Looper拿到锁，但是队列中没有消息，就会一直等待，而Handler需要把消息放进去，锁却被Looper拿着无法入队，这就造成了死锁。Handler机制的解决方法是**循环加锁**。在MessageQueue的next方法中：

```
Message next() {
   ...
    for (;;) {
  ...
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            ...
        }
    }
} 
```

我们可以看到他的等待是在锁外的，当队列中没有消息的时候，他会先释放锁，再进行等待，直到被唤醒。这样就不会造成死锁问题了。

那在入队的时候会不会因为队列已经满了然后一边在等待消息处理一边拿着锁呢？这一点不同的是MessageQueue的消息没有上限，或者说他的上限就是JVM给程序分配的内存，如果超出内存会抛出异常，但一般情况下是不会的。

## 7.Handler的阻塞唤醒机制是怎么回事？

**答： Handler的阻塞唤醒机制是基于Linux的阻塞唤醒机制。**

这个机制也是类似于handler机制的模式。在本地创建一个文件描述符，然后需要等待的一方则监听这个文件描述符，唤醒的一方只需要修改这个文件，那么等待的一方就会收到文件从而打破唤醒。和Looper监听MessageQueue，Handler添加message是比较类似的。

## 8.能不能让一个Message加急被处理？/ 什么是Handler同步屏障？

**答：可以 / 一种使得异步消息可以被更快处理的机制**

如果向主线程发送了一个UI更新的操作Message，而此时消息队列中的消息非常多，那么这个Message的处理就会变得缓慢，造成界面卡顿。所以通过同步屏障，可以使得UI绘制的Message更快被执行。

什么是同步屏障？这个“屏障”其实是一个Message，插入在MessageQueue的链表头，且其target==null。Message入队的时候不是判断了target不能为null吗？不不不，添加同步屏障是另一个方法：

```
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}
 
private int postSyncBarrier(long when) {
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;
 
        Message prev = null;
        Message p = mMessages;
        // 把当前需要执行的Message全部执行
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        // 插入同步屏障
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
```

可以看到同步屏障就是一个特殊的target，哪里特殊呢？target==null，我们可以看到他并没有给target属性赋值。**那这个target有什么用呢？看next方法：**

```
Message next() {
    ...
 
    // 阻塞时间
    int nextPollTimeoutMillis = 0;
    for (;;) {
        ...
        // 阻塞对应时间 
        nativePollOnce(ptr, nextPollTimeoutMillis);
  // 对MessageQueue进行加锁，保证线程安全
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            /**
            *  1
            */
            if (msg != null && msg.target == null) {
                // 同步屏障，找到下一个异步消息
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // 下一个消息还没开始，等待两者的时间差
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 获得消息且现在要执行，标记MessageQueue为非阻塞
                    mBlocked = false;
                    /**
              *  2
              */
                    // 一般只有异步消息才会从中间拿走消息，同步消息都是从链表头获取
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    msg.markInUse();
                    return msg;
                }
            } else {
                // 没有消息，进入阻塞状态
                nextPollTimeoutMillis = -1;
            }
 
            // 当调用Looper.quitSafely()时候执行完所有的消息后就会退出
            if (mQuitting) {
                dispose();
                return null;
            }
            ...
        }
        ...
    }
} 
```

这个方法我在前面讲过，我们重点看一下关于同步屏障的部分，看注释1的地方的代码：

```
if (msg != null && msg.target == null) {
    // 同步屏障，找到下一个异步消息
    do {
        prevMsg = msg;
        msg = msg.next;
    } while (msg != null && !msg.isAsynchronous());
} 
```

如果遇到同步屏障，那么会循环遍历整个链表找到标记为异步消息的Message，即isAsynchronous返回true，其他的消息会直接忽视，那么这样异步消息，就会提前被执行了。注释2的代码注意一下就可以了。

**注意，同步屏障不会自动移除，使用完成之后需要手动进行移除，不然会造成同步消息无法被处理**。从源码中可以看到如果不移除同步屏障，那么他会一直在那里，这样同步消息就永远无法被执行了。

有了同步屏障，那么唤醒的判断条件就必须再加一个：**MessageQueue中有同步屏障且处于阻塞中，此时插入在所有异步消息前插入新的异步消息**。这个也很好理解，跟同步消息是一样的。如果把所有的同步消息先忽视，就是插入新的链表头且队列处于阻塞状态，这个时候就需要被唤醒了。看一下源码：

```
boolean enqueueMessage(Message msg, long when) {
    ...
 
    // 对MessageQueue进行加锁
    synchronized (this) {
        ...
        if (p == null || when == 0 || when < p.when) {
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            /**
            * 1
            */
            // 当线程被阻塞，且目前有同步屏障，且入队的消息是异步消息
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                /**
                * 2
                */
                // 如果找到一个异步消息，说明前面有延迟的异步消息需要被处理，不需要被唤醒
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; 
            prev.next = msg;
        }
  
        // 如果需要则唤醒队列
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
} 
```

同样，这个方法我之前讲过，把无关同步屏障的代码忽视，看到注释1处的代码。如果插入的消息是异步消息，且有同步屏障，同时MessageQueue正处于阻塞状态，那么就需要唤醒。而如果这个异步消息的插入位置不是所有异步消息之前，那么不需要唤醒，如注释2。

**那我们如何发送一个异步类型的消息呢？有两种办法：**

- 使用异步类型的Handler发送的全部Message都是异步的
- 给Message标志异步

Handler有一系列带Boolean类型的参数的构造器，这个参数就是决定是否是异步Handler：

```
public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    // 这里赋值
    mAsynchronous = async;
} 
```

但是异步类型的Handler构造器是标记为hide，我们无法使用，所以我们使用异步消息只有通过给Message设置异步标志：

```
public void setAsynchronous(boolean async) {
    if (async) {
        flags |= FLAG_ASYNCHRONOUS;
    } else {
        flags &= ~FLAG_ASYNCHRONOUS;
    }
} 
```

**但是！！！！** ，其实同步屏障对于我们的日常使用的话其实是没有多大用处。因为设置同步屏障和创建异步Handler的方法都是标志为hide，说明谷歌不想要我们去使用他。所以这里同步屏障也作为一个了解，可以更加全面地理解源码中的内容。

**Android面试的风向已经变了！**

以前，你只要了解四大组件、视图、网络请求，你就能拿到一份薪资丰厚的offer；如果你精通Java基础、知道如何处理OOM、组件生命周期原理、熟悉Android源码、架构体系，进阿里腾讯可能都很轻松。

现在面试是越来越难了！别说阿里、头条、腾讯等大厂，就连很多中小型公司面试问的都是原理