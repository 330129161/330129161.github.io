title: AbstractQueuedSynchronizer源码解析(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/book.png

toc: true 

tags:

  - 源码
  - java
  - 并发

categories: 

  - [源码,并发] 

date: 2019-12-25 20:41:01

---

## 概述

`AQS(AbstractQueuedSynchronizer)`以`CLH`锁为基础而设计，是并发编程中一个重要的框架类，用于构建锁和其他的同步组件。我们熟知的`ReentrantLock`、`Semaphore`、`CountDownLatch`等就是基于`AQS`实现的。`AQS`定义两种资源共享方式：`Exclusive`（独占，只有一个线程能执行，如`ReentrantLock`）和`Share`（共享，多个线程可同时执行，如`Semaphore/CountDownLatch`）。<!--more--> 

> CLH锁是一种基于链表的可扩展、高性能、公平的自旋锁，申请线程只在本地变量上自旋，它不断轮询前驱的状态，如果发现前驱释放了锁就结束自旋。

## 结构特点

1. 继承自`AbstractOwnableSynchronizer`，为创建锁和相关同步器提供了基础。
2. `AQS`中还有一个`ConditionObject`内部类，提供了条件锁的同步实现，详见[ConditionObject源码解析](https://)。

![AbstractQueuedSynchronizer结构图](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/AbstractQueuedSynchronizer.png)

## 重要属性

```java
//node节点
static final class Node {
    //标记符，表示在共享模式下的同步节点	
    static final Node SHARED = new Node();
    
	//标记符，表示在独占模式下的同步节点
    static final Node EXCLUSIVE = null;
    
	//等待状态值，表示同步队列中等待的线程等待超时或被中断
    static final int CANCELLED =  1;	
    
	//等待状态值，当前驱节点释放了同步状态后，会唤醒后驱节点
    static final int SIGNAL = -1;
    
	//等待状态值，与Condition等待队列相关，表示该标识的结点处于等待队列中，结点的线程等待在Condition队列上
    //当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将从等待队列转移到同步队列中，等待获取同步锁
    static final int CONDITION = -2;
    
	//与共享模式相关，在共享模式中，该状态标识结点的线程处于可运行状态
    static final int PROPAGATE = -3;
	
    //初始化状态，通过addWaiter传入的节点，waitStatus的值会为初始值0
    volatile int waitStatus;

    //当前节点的前驱节点
    volatile Node prev;

    //当前节点的后驱节点，当前节点在释放同步状态后会唤醒后驱节点
    volatile Node next;

    //节点关联的线程，构造时被初始化、用完后置空
    volatile Thread thread;

    //Condition等待队列中指向下一个等待节点
    Node nextWaiter;

    //判断当前节点是否为共享节点
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    //获取前驱节点
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }
	
    //创建一个空节点，用于设置队列头
    Node() { 
    }
	
    //创建一个新的同步节点，储存传入的线程与同步节点，addWaiter中创建一个新同步节点会用到这个构造方法
    //此时node的waitStatus = 0
    Node(Thread thread, Node mode) {
        this.nextWaiter = mode;
        this.thread = thread;
    }
	//创建一个新的等待节点，储存传入的线程与等待状态，这个方法会在Condition等待队列使用
    Node(Thread thread, int waitStatus) {
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}

//同步队列中的头节点，通过volatile修饰，保证多线程中的可见性
private transient volatile Node head;

//同步队列中的尾节点，通过volatile修饰，保证多线程中的可见性
private transient volatile Node tail;

//同步状态，state>0，说明当前线程持有锁，对于独占锁来说，state代表着冲入次数，对于共享锁来说，state就是持有锁的线程数量
//当state=0时，表示没有线程在使用资源，继承AQS的子类，通过state来判断获得锁的状态
private volatile int state;

//超时时间间隔阈值，在获取超时锁中使用，当前获取锁时间距离超时时间不足此间隔
//那么没必要对当前线程阻塞，直接快速获取锁操作
static final long spinForTimeoutThreshold = 1000L;
```

## 常用方法

### state方法

```java
//返回当前的同步状态值。state通过volatile，保证可见
protected final int getState() {
        return state;
}
//设置同步状态值
protected final void setState(int newState) {
        state = newState;
}
//通过cas方式设置state的值，成功返回true，否则返回false
protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

### 独占锁的获取

```java
//通过该方法获取同步状态，对中断不响应，线程获取锁失败后，会进入同步队列中，
//后续对线程进行中断操作，不会将线程从队列中移除
public final void acquire(int arg) {
    //1.tryAcquire(arg)尝试获取锁，成功返回
    //2.获取锁失败，通过addWaiter(Node.EXCLUSIVE)构建独占式的节点，加入到同步队列的尾端，
    //3.acquireQueued(addWaiter(Node.EXCLUSIVE), arg)，通过自旋尝试从同步队列中获取锁
    //如果获取不到，则阻塞节点中的线程
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

//中断当前线程
static void selfInterrupt() {
    Thread.currentThread().interrupt();
}

//通过该方法获取同步状态，对中断响应
public final void acquireInterruptibly(int arg)
            throws InterruptedException {
    //如果线程被中断，抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        //获取锁响应中断
        doAcquireInterruptibly(arg);
}

//典型的模板方法，留给子类实现
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}

//添加同步节点
private Node addWaiter(Node mode) {
    //创建一个新节点，记录当前线程，节点类型
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    //记录尾节点
    Node pred = tail;
    //如果尾节点存在，快速入队
    if (pred != null) {
        //设置当前节点的前驱节点未原来的尾节点
        node.prev = pred;
        //通过cas设置node为新的尾节点
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //如果尾节点不存在，或者快速入队失败，那么通过enq将node设置到队尾
    enq(node);
    //返回新节点
    return node;
}

//添加到同步节点末端，返回原尾节点
private Node enq(final Node node) {
    //cas自旋，失败后快速重试
    for (;;) {
        //记录队列尾
        Node t = tail;
        //如果为null，说明当前队列未初始化
        if (t == null) { // Must initialize
            //通过cas设置队列头
            if (compareAndSetHead(new Node()))
                //成功后设置队列尾,此时head = tail,且都为空节点
                //继续执行一下次循环，会将node节点，链接到队尾
                //这么做的目的是为了保证头节点为空节点
                tail = head;
        } else {
            //要入队的节点前驱指向尾节点
            node.prev = t;
            //通过cas设置入队节点为尾节点
            if (compareAndSetTail(t, node)) {
                //原未节点的next指向新的尾节点
                t.next = node;
                //返回原尾节点
                return t;
            }
        }
    }
}

//设置头节点
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}

//在同步队列中尝试获取锁
final boolean acquireQueued(final Node node, int arg) {
    //失败标记
    boolean failed = true;
    try {
        //中断标记
        boolean interrupted = false;
        //自旋
        for (;;) {
            //获取前驱节点，如果为null,立刻抛出NullPointException
            final Node p = node.predecessor();
            //如果当前节点的前驱节点为头节点，那么尝试获取锁
            if (p == head && tryAcquire(arg)) {
                //获取锁成功，设置当前节点为新的头节点
                setHead(node);
                //将原头节点的后驱节点指向置为null，便于GC回收
                p.next = null; // help GC
                //设置失败标记为false，代表着获取锁成功
                failed = false;
                //返回false，tryAcquire中if (!tryAcquire(arg) 				&&acquireQueued(addWaiter(Node.EXCLUSIVE), arg))会直接返回
                return interrupted;
            }
            //无限重试，直到将前驱节点的waitStatus设置为-1，设置为-1后阻塞当前线程，等待前驱节点唤醒
            //唤醒后会继续执行上面获取的方法，直到获取锁成功或者被阻塞
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                //parkAndCheckInterrupt()返回true
                interrupted = true;
        }
    } finally {
        //如果标记失败为true,会取消获取锁
        if (failed)
            //取消获取锁，在当前方法中不会执行，因为没有响应中断,无法跳出上面的for循环
            //只有获取锁成功后，才会跳出，但是这时候，failed = false
            cancelAcquire(node);
    }
}

//获取锁，响应中断，方法跟上面acquireQueued差不多
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                //与acquireQueued中不同的地方，如果线程被中断了，会立马抛出InterruptedException
                //这就是说doAcquireInterruptibly响应中断的原因
                throw new InterruptedException();
        }
    } finally {
        //被中断后，取消获取锁
        if (failed)
            cancelAcquire(node);
    }
}

//将node节点的前驱节点的waitStatus置为-1，如果前驱节点状态为取消，那么将所有被取消的前驱节点从
//队列中去除
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //记录前驱节点waitStatus状态值
    int ws = pred.waitStatus;
    //如果前驱节点状态为Node.SIGNAL，返回true，后续会通过parkAndCheckInterrupt()将当前线程挂起
    if (ws == Node.SIGNAL)
        return true;
    //如果前驱节点被取消，那么将前驱节点从队列中去除，链接前驱节点的上一个节点，
    //通过while继续判断上一个前驱节点，移除所有被取消的节点
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        //如果前驱节点为0或者是共享状态，那么直接设置前驱节点为Node.SIGNAL
        //表示当前驱节点中记录的线程被释放后，要唤醒下一个节点记录的线程
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    //如果ws>0,表示去除了所有被取消的节点，但是这一步还不能保证前驱节点的waitStatus值为Node.SIGNAL，
    //返回false后，acquireQueued中自旋会再一次调用此方法
    return false;
}

//通过LockSupport将当前线程阻塞，唤醒后会返回interrupted的一个状态值
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}

//取消获取锁
private void cancelAcquire(Node node) {
    //如果节点不存在直接返回
    if (node == null)
        return;
	//将节点的线程置为null
    node.thread = null;
    //记录前驱节点
    Node pred = node.prev;
    //如果前驱节点被取消，那么跳过，直到找到未被取消的前驱节点
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    //记录最后一个未被取消节点的后驱节点
    Node predNext = pred.next;
    //将当前节点的waitStatus值置为1，也就是取消状态
    node.waitStatus = Node.CANCELLED;
	//如果当前当前节点是尾节点，那么设置前驱节点为尾节点
    if (node == tail && compareAndSetTail(node, pred)) {
        //设置尾节点成功后，将前驱节点的后驱置为null
        compareAndSetNext(pred, predNext, null);
    } else {
        int ws;
        //1.pred != head，当前节点的前驱节点,此时pred.waitStatus<=0
        //2.如果pred的状态不为为SIGNAL，那么通过cas设置前驱节点的waitStatus为Node.SIGNAL表示该节点记录的线程被释放时，要唤醒下一个节点
        //前驱节点的记录的线程不为null
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            //记录当前节点的后驱节点
            Node next = node.next;
            //如果后驱节点不为null,且没有被取消
            if (next != null && next.waitStatus <= 0)
                //那么链接前后驱节点
                compareAndSetNext(pred, predNext, next);
        } else {
            //如果前驱节点为头节点，并且没有被阻塞，且状态为Node.SIGNAL，那么唤醒后驱节点
            unparkSuccessor(node);
        }
		//断开node链接，帮助回收
        node.next = node; // help GC
    }
}

//唤醒后驱节点
private void unparkSuccessor(Node node) {
    //记录ws状态，cancelAcquire方法中，通过node.waitStatus = Node.CANCELLED;已经将waitStatus设置为1了
    int ws = node.waitStatus;
    if (ws < 0)
        //将waitStatus设置为初始值0
        compareAndSetWaitStatus(node, ws, 0);
	//记录后驱节点
    Node s = node.next;
    //如果后驱节点为null，或者被取消
    if (s == null || s.waitStatus > 0) {
        s = null;
        //从当前node位置一直向后查找不为null且没有被取消的节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    //如果后续的正常节点存在，那么释放后续节点
    if (s != null)
        LockSupport.unpark(s.thread);
}

//获取超时锁
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    //如果获取锁失败，那么去同步队列中获取锁
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}

//获取锁超时，响应中断，超过指定超时时间nanosTimeout还没有获取到锁，那么获取锁失败，从同步队列中将node移除，返回false
private boolean doAcquireNanos(int arg, long nanosTimeout)
    throws InterruptedException {
    //如果超时时间<=0,直接返回false
    if (nanosTimeout <= 0L)
        return false;
    //获取等待过期时间
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        //自旋
        for (;;) {
            //跟acquireQueued方法中一样处理
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            //获取剩余时间
            nanosTimeout = deadline - System.nanoTime();
            //如果剩余时间<=0,返回fasle，获取锁失败
            if (nanosTimeout <= 0L)
                //获取锁超时，返回false，此时failed = true，会进入到finally块中
                return false;
            //将前驱节点的waitStatus设置为-1
            //如果nanosTimeout>1000L,则需要阻塞，如果nanosTimeout<=1000
            //那么直接进行下一次自旋获取锁，spinForTimeoutThreshold的时候很短，
            //这么短的时间就没有必要进行阻塞操作了，而是快速进入下一次自旋
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                //阻塞当前线程，直到nanosTimeout，进入下一次自旋，
                //会进入if (nanosTimeout <= 0L)，返回false
                LockSupport.parkNanos(this, nanosTimeout);
            //如果线程在阻塞期间被中断了，抛出InterruptedException
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        //中断线程或者锁超时后，取消获取锁操作
        if (failed)
            cancelAcquire(node);
    }
}
```

### 共享锁的获取

```java
public final void acquireShared(int arg) {
    //尝试获取共享锁
    if (tryAcquireShared(arg) < 0)
        //如果获取锁失败，到队列中获取共享锁
        doAcquireShared(arg);
}

//获取共享锁，响应中断
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }

//尝试获取共享锁，留给子类实现
protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}

//获取共享锁，不响应中断
private void doAcquireShared(int arg) {
    //创建一个共享锁状态节点，添加到队列末端
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        //自旋
        for (;;) {
            //获取前驱节点
            final Node p = node.predecessor();
            if (p == head) {
                //如果当前节点为头节点，直接获取锁
                int r = tryAcquireShared(arg);
                //如果获取锁成功
                if (r >= 0) {
                    //acquireQueued中，这步仅仅通过setHead(node)设置了队列头
                    //共享锁中，获取到共享锁之后，除了要设置队头以外，还需要将判断是否还有剩余资源，如果						//有则唤醒后续共享锁
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    //如果当前线程中断，设置当前线程状态为中断
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            //获取锁失败后，将node节点的前驱节点的waitStatus设置为-1，阻塞当前线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                //中断后设置interrupted = true
                interrupted = true;
        }
    } finally {
        //如果标识失败为true,则取消获取锁，此方法不响应中断，不会进入cancelAcquire方法
        if (failed)
            cancelAcquire(node);
    }
}

//获取共享锁，响应中断
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                //响应中断的关键，抛出异常，会进入finally块中，执行cancelAcquire方法
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

//获取共享锁超时，逻辑基本与排他锁一致
private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    //与排他锁不同的一点，获取排他锁超时时上面也介绍过
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
            }
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            //超时后，判断是否被中断，抛出异常
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private void setHeadAndPropagate(Node node, int propagate) {
    //记录头节点
    Node h = head; // Record old head for check below
    //设置新的队列头
    setHead(node);
    //1.propagate > 0,说明当前还有剩余资源，释放后继节点
    //2.原头节点h == null || h.waitStatus < 0的时候，说明propagate = 0，头不存在或者为waitStatus<0说明后继节点需要被唤醒
    //3.h=head，通过上面设置新的头部后，此时再进行一遍操作2
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        //如果当前节点的后继节点不存在，或者后续节点为共享状态节点，那么释放锁
        if (s == null || s.isShared())
            //释放共享锁
            doReleaseShared();
    }
}
```

### 释放锁

```java
public final boolean release(int arg) {
    //释放锁
    if (tryRelease(arg)) {
        Node h = head;
        //释放锁成功后，如果头节点不为null(如果为null的话说明同步队列不存在)，
        //h.waitStatus != 0，如果h.waitStatus==0，说明没有后续节点，改变当前节点的状态，也不需要唤醒后续节点
        if (h != null && h.waitStatus != 0)
            //唤醒后驱节点
            unparkSuccessor(h);
        //释放锁成功
        return true;
    }
    //释放锁失败
    return false;
}

//尝试释放锁，留给子类实现
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}

public final boolean releaseShared(int arg) {
    //尝试释放锁
    if (tryReleaseShared(arg)) {
        //如果失败，在同步队列中释放锁
        doReleaseShared();
        return true;
    }
    return false;
}

//释放共享锁
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        //如果头不存在，且头尾不相等（头尾相等说明当前同步队列中只有一个头节点）
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            //如果头结点的waitStatus为Node.SIGNAL，那么需要唤醒后续节点
            if (ws == Node.SIGNAL) {
                //将头节点waitStatus置为初始值
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            //如果当前waitStatus状态为0，将当前状态设置为Node.PROPAGATE，表示当前线程处于可运行状态
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        //如果头节点没有被其他线程改变，说明当前操作已经成功了，直接跳出循环
        if (h == head)                   // loop if head changed
            break;
    }
}
```

## 总结

上面解析了AQS中获取锁、释放锁使用CLH队列管理分配锁的流程。我们针对整个获取锁、释放锁的流程做一个总结。

**获取锁:**

1. 通过`acquire`方法获取锁，真正获取锁的操作在`tryAcquire`方法中，直接使用`tryAcquire`会抛出`UnsupportedOperationException`异常，目的是留给子类实现。
2. 通过`tryAcquire`方法成功后，会直接返回，如果失败，会通过`addWaiter`方法创建一个新节点加入到同步队列末尾，该节点保存着当前线程以及当前节点的模式（独占或共享）
3. 通过`addWaiter`方法将节点加入到同步队列末尾后，会通过`acquireQueued`方法，不断检测当前节点前驱节点的状态(**获取共享锁，会执行`doAcquireShared`方法，逻辑基本跟`acquireQueued`中方法一致，不过`acquireQueued`某个节点获取锁成功后，通过`setHead`方法将被释放的节点置为队头,而`doAcquireShared`方法会通过`setHeadAndPropagate`设置队头的同时，将获取锁的消息传播到下一个共享锁节点，释放下一个共享锁节点**)
4. 如果前驱节点为头节点，那么尝试获取锁，获取锁成功后，会将当前节点置为头部节点，失败会进入下一次自旋
5. 如果前驱节点不为头节点，那么会通过`shouldParkAfterFailedAcquire`方法，将当前节点的前驱节点的waitStatus置为 -1 (表示前驱节点被释放后，需要唤醒后续节点，将前驱节点置为-1期间，还会检测同步队列中节点状态，将所有被取消的节点从队列中移除)，完成后会通过`parkAndCheckInterrupt`方法阻塞当前线程。
6. 通过`parkAndCheckInterrupt`方法中`LockSupport.park`阻塞线程后，等待`release`方法或者中断唤醒，唤醒后返回中断状态。
7. 如果我们使用的`acquire`方法不响应中断，那么返回中断状态之后，只会保留线程的中断状态
8. 如果我们使用的`acquire`方法响应中断，那么返回中断状态为true之后(表示被中断唤醒)，那么会直接抛出`InterruptedException`异常，随即执行`finally`块中的`cancelAcquire`取消获取锁
9. `cancelAcquire`方法中，会从当前节点向头节点寻找最近一个未被取消的前驱节点A (从前后向前，也就是跳过前面被取消的节点)， 然后将当前取消节点的`waitStatus`置为1(表示被取消)，如果被取消的节点是同步队列的尾节点，那么直接从A开始截断。
10. 如果当前节点不是尾部节点，判断A是否为头节点，如果不是头节点，那么将A的`waitStatus`设置为-1
11. 如果A是头节点，说明当前被取消的节点，是同步节点的第一个要唤醒的节点，那么被取消后，下一个节点就是目前要被唤醒的节点，通过`unparkSuccessor`唤醒下一个节点。

 **释放锁：**

1. 通过`release`方法释放锁，跟`acquire`一样，主要释放锁的操作在`tryRelease`方法中，留给子类实现的。
2. `release`成功返回后，会通过`unparkSuccessor`唤醒下一个节点
3. `unparkSuccessor`如果当前节点`waitStatus`< 0,会将当前节点的`waitStatus`设置为0，如果失败说明当前节点被取消了
4. 记录下一个节点s,如果s存在并且`s.waitStatus > 0`节点没有被取消，那么通过`LockSupport.unpark`释放s的线程(节点释放后，上面获取锁第6步会往下执行)，如果是共享锁，释放锁后，会执行上面获取锁第2步中的操作，将锁传递给下一个共享节点

AQS中获取锁、释放锁流程的总结到这里就结束了。本章只解析了AQS同步器锁的获取与释放，下一章解析AQS中的条件锁`ConditionObject`[传送门]("http://")。



