##ReentrantLock

## 为什么会有ReentrantLock，synchronized不够吗？

相比于synchronized，ReentrantLock增加了一些高级功能，

**等待可中断**：等待可中断是指当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待，改为处理其他事情，可中断特性对处理执行时间非常长的同步块很有帮助。

**可实现公平锁**：公平锁是指多个线程在等待同一个锁时，必须按照申请锁的时间顺序来依次获得锁；而非公平锁则不保证这一点，在锁被释放时，任何一个等待锁的线程都有机会获得锁。synchronized中的锁是非公平的，ReentrantLock默认情况下也是非公平的，但可以通过带布尔值的构造函数要求使用公平锁。

**以及锁可以绑定多个条件**：锁绑定多个条件是指一个ReentrantLock对象可以同时绑定多个Condition对象，而在synchronized中，锁对象的wait（）和notify（）或notifyAll（）方法可以实现一个隐含的条件，如果要和多于一个的条件关联的时候，就不得不额外地添加一个锁，而ReentrantLock则无须这样做，只需要多次调用newCondition（）方法即可。



## 源码解析

一句话概要：

通过CAS判断是否已经被锁住，如果锁住将会加入到双端队列中，也就是两个Node 头尾链表，开始自旋等待，自旋一定次数后将通过LockSupport.park()将线程挂起，等待上一个线程执行完成后唤醒头节点。



```java
private transient volatile Node head;

private transient volatile Node tail;
static final class Node {
    // 排他锁的标识
    static final Node EXCLUSIVE = null;
    // 证明是否失效了
    static final int CANCELLED =  1;
    // 证明后续节点要被唤醒
    static final int SIGNAL    = -1;
 
    // Node对象存储标识的地方
    volatile int waitStatus;

  	//双向队列
    volatile Node prev;
    volatile Node next;

  	// 返回前屈节点
    final Node predecessor() {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }
}
```



### 构造

```java
public ReentrantLock() {
    sync = new NonfairSync();
}
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

默认是非公平锁、可以通过设置参数 true，使用公平锁

```java
public void lock() {
    sync.acquire(1);
}

public final void acquire(int arg) {
  // 尝试获取锁，获取失败加入队列，加入成功则
  if (!tryAcquire(arg) &&
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}



private Node addWaiter(Node mode) {
  Node node = new Node(mode);
	// 循环两次，插入到尾部
  for (;;) {
    Node oldTail = tail;
    if (oldTail != null) {
      node.setPrevRelaxed(oldTail);
      if (compareAndSetTail(oldTail, node)) {
        oldTail.next = node;
        return node;
      }
    } else {
      initializeSyncQueue();
    }
  }
}


final boolean acquireQueued(final Node node, int arg) {
  //阻塞是false失败的
  boolean interrupted = false;
  try {
    //死循环，这个死循环是一直自旋，获得锁用的
    for (;;) {
      //拿到刚阻塞的线程的前驱节点
      final Node p = node.predecessor();
      //如果它的前驱节点是那个头节点和获取锁成功的情况下
      if (p == head && tryAcquire(arg)) {
        setHead(node);
        p.next = null; // help GC
         //返回Interrupted判断，即阻塞失败了
        return interrupted;
      }
      //这个是获取锁失败了，他就会返回true进行阻塞
      if (shouldParkAfterFailedAcquire(p, node))
        interrupted |= parkAndCheckInterrupt();
    }
  } catch (Throwable t) {
    cancelAcquire(node);
    if (interrupted)
      selfInterrupt();
    throw t;
  }
}

```











## 公平锁

```java
// 公平的 tryAcquire
protected final boolean tryAcquire(int acquires) {
    // 当前调用的线程
    final Thread current = Thread.currentThread();
  	// 获取当前是否是锁住的状态
    int c = getState();
    if (c == 0) {
      	// 如果没有这个节点，尝试获取锁，
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 重入状态
    else if (current == getExclusiveOwnerThread()) {
      	// status状态进行叠加
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```




## 非公平锁

```java
// 非公平的tryAcquire
final boolean nonfairTryAcquire(int acquires) {
  // 当前调用的线程
  final Thread current = Thread.currentThread();
  // 获取当前是否是锁住的状态
  int c = getState();
  if (c == 0) {
    // 尝试获取
    if (compareAndSetState(0, acquires)) {
      // 获取成功设置为独占
      setExclusiveOwnerThread(current);
      return true;
    }
  }
  //可重入的情况，
  else if (current == getExclusiveOwnerThread()) {
    // status状态进行叠加
    int nextc = c + acquires;
    if (nextc < 0) // overflow
      throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
  }
  return false;
}
```

