2 Handler原理是什么，简单说下

1 为什么不能在子线程更新UI？

2一个线程有几个looper? 如何保证，又可以有几个Handler

3 handler内存泄漏的原因，其他内部类为什么没有这个问题

4为什么主线程可以new Handler 其他子线程可以吗 怎么做

5 Message可以如何创建？哪种效果更好，为什么？

 6 Handler中的生产者-消费者设计模式你理解不？

7 子线程中维护Looper在消息队列无消息的时候处理方案是怎么样的

8 既然存在多个Handler往MessageQueue中添加数据（发消息时各个Handler处于不同线程），内部如何保证安全

9 我们使用Message是应该如何创建它

10 Looper死循环为什么不会导致线程卡死

11 关于ThreadLocal，谈谈你的理解？

11 使用Hanlder的postDealy()后消息队列会发生什么变化？

12 为什么不能在子线程更新UI？





Handler不是解决线程通信问题，而是解决了线程切换的问题。



| Handler源码整体框架                            |
| ---------------------------------------------- |
| Handler实现线程间通信的方案思想                |
| Handler源码面试问题全解析                      |
| Loop消息泵机制                                 |
| Loop死循环是在UI线程，那为什么会不阻塞UI线程呢 |
| Linux pipe/epoll机制                           |
| pipe管道与CPU休眠状态                          |
| Message回收池设计，单项链表详解                |
| Message的回收与复用机制                        |
| 项目巩固(手写Handler消息机制,面试再也不怕了)   |





### Handler原理是什么，简单说下

Android系统主要通过MessageQueue、Loper和Handler 三个类来实现Android应用程序的消息处理 机制，其中，MesageQueue 类用来描述消息队列;Looper类用来创建消息队列，以及进人消息循环; Handl er类用来发送消息和处理消息。



### 创建线程消息队列

```java
public final class Looper {
		......
    //ThreadLocal用来保存线程中的Looper对象.我们可以将Looper类的静态成员变量sThreadlocal理解为一个线程局部变量，或者一个HashMap，每一个创建了消息队列的Android应用程序线程在里面都有一个关联的Looper对象。当我们调用这个静态成员变量的成函数get或者Looper类的静态成员函数myLooper时，就可以获得与当前线程关联的一个1ooper对象;而当我们调用这个静态成员变量的成员两数set时，就可以将一个Looper对象与当前线程关联起来。 
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
  	final MessageQueue mQueue;
  	private static Looper sMainLooper;  
   	......
  	public static void prepare() {
        prepare(true);
    }
	
  	//在Looper类的静态成员两数prepare中，第12行的if语句检查当前线程是否已经有一个Looper对象了。如果有，那么第13行代码就会抛出一个异常;否则，第15行代码就会首先创建—-个Looper对象，然后将这个Looper对象保存在Looper类的静态成员变量sThreadlocal中。
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
  //在Looper类的静态成员两数preparreMainlooper中，第19行代码首先调用Looper类的静态成员丽数prepare在当前线程中创建一个Iooper对象，接着第20行代码调用Looper类的静态成员两数seMainLooper将这个Looper对象保存在Looper类的静态成员变量mMainlooper中。
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }

    public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }

    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }

  	......
}
```

一个Looper对象在创建的过程中，会在内部创建—个MessageQueve对象，并且保存在它的成员

```java
public final class Looper {
  .....
  private Looper(boolean quitAllowed) {
      mQueue = new MessageQueue(quitAllowed);
  }
  .....
}
```

一个MessageQueve对象在创建的过程中，又会在C++层中创建一个NativeMesageQueue对象，这是通过调用MessageQueue类的成员两数nativelnit来实现的，如下所示。



```java
public final class MessageQueue {
	private long mPtr;
  MessageQueue(boolean quitAllowed) {
      mQuitAllowed = quitAllowed;
      mPtr = nativeInit();
  }
	private native static long nativeInit();
}
```

MesageQueue类的成员两数nativelnit 是一个JNI 方法，它是由C++层中的两数android_os_MesageQueue_ nativelnit 来实现的，如下所示。

```c++
static void android_os_MessageQueue_nativeInit(UNIEnv* env,jobjectobj){
  NativevessageQueue *nativeMessageQueve=new NativeMessageQueue();
  androia_os_MessageQueue_setNativeMessageQueue(env,obj,nativeMessageQueue)
}
```

第2行代码首先在C++层中创建一个NativeMessageQueve对象，接着第5行代码调用两数android_os_MessageQueue_seVativeMessageQueue将它与参数obj所描述的一个Java层的MesageQueve对象关联起来。两数android_os_MessageQueue_setNativeMessageQueve的实现如下所示。

```c++
static void android_os_MessageQueue_setNativeMessageQueue(JNIEnv*env,jobject messageQueueobj,NativeMessageQueue* nativeMessageQueue ) {
  
	env->SetIntField(messageQueueobj, gMessageoueueClassInfo.mPtr,
	reinterpret_casts<jint>(nativeMessageQueue));
}
```

一个NativeMesageQueue对象在创建的过程中，又会在内部创建一个C+层的Looper对象，如下所示。

```c++
NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}
```

Loper类的静态成员两数setForThread将它与当前线程关联起来。 一个C++层的Looper对象在创建的过程中，又会在内部创建一个管道，如下所示

```c++
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mPolling(false), mEpollFd(-1), mEpollRebuildRequired(false),
        mNextRequestSeq(0), mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    LOG_ALWAYS_FATAL_IF(mWakeEventFd < 0, "Could not make wake event fd: %s",
                        strerror(errno));

    AutoMutex _l(mLock);
    rebuildEpollLocked();
}

void Looper::rebuildEpollLocked() {
    // Close old epoll instance if we have one.
    if (mEpollFd >= 0) {
#if DEBUG_CALLBACKS
        ALOGD("%p ~ rebuildEpollLocked - rebuilding epoll set", this);
#endif
        close(mEpollFd);
    }

    // Allocate the new epoll instance and register the wake pipe.
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
   	.....
    struct epoll_event eventItem;

    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);
		......
}
```



### 线程消息循环过程

### 线程消息发送过程

### 线程消息处理过程



