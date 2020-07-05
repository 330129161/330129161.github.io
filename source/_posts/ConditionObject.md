title: ConditionObject源码解析(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/cs/cs-14.jpg

toc: true 

tags:

  - 源码
  - java
  - 并发

categories: 

  - [源码,并发] 

date: 2019-08-07 23:10:01

---

## 概述

`ConditionObject`是`AQS`的内部类，是一个单向队列，提供了条件锁的同步实现。<!--more--> 在一个AQS同步器中，可以定义多个`ConditionObject`对象，因此AQS中可以存在多个等待队列，根据每个队列的条件不同，控制线程的挂起与唤醒。

## 结构特点

1.实现了`Condition`中`await()`,`signal()`等方法。

2.实现了 `Serializable` 接口， 表示 `ArrayList` 支持序列化的功能 ，可用于网络传输

![](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/ConditionObject.png)

## 重要属性

`ConditionObject`是一个FIFO单向队列，队列中每个节点都记录着当前线程的引用。外部类`AQS`中`node`节点的`nextWaiter`指向队列中下一个等待节点。

```java
public class ConditionObject implements Condition, java.io.Serializable {
    //等待队列头，Node节点在外部类AQS中
    private transient Node firstWaiter;
	//等待队列尾
    private transient Node lastWaiter;
    
    //异常标识符，针对响应中断的await方法，如果用户中断动作发生在signal后，会返回REINTERRUPT
    //表示不会对中断做出响应，只会保留线程中断状态
    private static final int REINTERRUPT =  1;
    //抛出异常标识，针对响应中断的await方法，如果用户中断动作发生在signal前，会抛出InterruptedException响应中断
    private static final int THROW_IE    = -1;
    
}
```

## 常用方法

### 构造方法

```java
//无参构造方法，在AQS中创建一个等待队列
public ConditionObject() { }
```

### await方法

```java
//释放持有的锁，阻塞当前线程，不响应中断
public final void awaitUninterruptibly() {
    //添加一个新的等待节点
    Node node = addConditionWaiter();
    //释放锁
    int savedState = fullyRelease(node);
    boolean interrupted = false;
    //如果node已经在同步队列中，跳出while
    while (!isOnSyncQueue(node)) {
        //将当前线程阻塞，通过signal等方法唤醒后，会判断当前线程在阻塞期间是否被中断
        //进入下一次循环，此时node通过signal等方法后，最终被设置到同步队列中了
        LockSupport.park(this);
        if (Thread.interrupted())
            //从这里就可以看出，该方法是不响应中断的
            interrupted = true;
    }
    //acquireQueued(node, savedState) 方法，在AQS中已经解析过
    //被其他线程通过signal等方法唤醒后，会通过acquireQueued从同步队列中获取锁
    //如果获取锁失败，判断线程是否被中断，标记中断
    if (acquireQueued(node, savedState) || interrupted)
        selfInterrupt();
}

//阻塞等待，将当前线程加入等待队列中，并释放当前锁
//当其他线程调用signal()会重新请求锁，线程被中断有两种情况
//1.线程是在调用signal方法之前被中断，那么抛出异常，响应中断
//2.线程是在调用notity方法之后被中断，那么不会响应中断,只会保留中断状态
public final void await() throws InterruptedException {
    //如果线程被中断，直接响应
    if (Thread.interrupted())
        throw new InterruptedException();
    //方法同上awaitUninterruptibly
    Node node = addConditionWaiter();
    //释放等待节点对应的锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    //检查当前节点是否在同步队列中
    while (!isOnSyncQueue(node)) {
        //阻塞当前线程，等待被唤醒
        LockSupport.park(this);
        //检查是否是由于被中断而唤醒，如果是，则跳出循环
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //在同步队列中获取锁，如果线程中断且中断的方式不是抛出异常，则设置中断后续的处理方式设置为REINTERRUPT
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    //遍历等待队列，移除所有被取消的节点
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        //通过interruptMode判断，线程中断后要处理的模式，是直接设置当前线程的中断状态
        //还是直接抛出InterruptedException
        reportInterruptAfterWait(interruptMode);
}



//阻塞超时到指定秒之后，针对中断的响应与await()方法一致
public final long awaitNanos(long nanosTimeout)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    //记录获取锁超时时间
    final long deadline = System.nanoTime() + nanosTimeout;
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        if (nanosTimeout <= 0L) {
            //将node节点添加到同步队列中
            transferAfterCancelledWait(node);
            break;
        }
        //跟独占锁中获取锁超时方法一样，如果距离最后超时时间<spinForTimeoutThreshold,
        //那么不会阻塞，直接进入下一次检查
        if (nanosTimeout >= spinForTimeoutThreshold)
            //阻塞指定时长
            LockSupport.parkNanos(this, nanosTimeout);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
        nanosTimeout = deadline - System.nanoTime();
    }
    //跟await方法一样
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
    //返回超时剩余时间
    return deadline - System.nanoTime();
}

//阻塞超时到指定日期之前，针对中断的响应与await()方法一致，逻辑基本与awaitNanos一致
public final boolean awaitUntil(Date deadline)
    throws InterruptedException {
    //获取指定日期毫秒数
    long abstime = deadline.getTime();
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    boolean timedout = false;
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        if (System.currentTimeMillis() > abstime) {
            timedout = transferAfterCancelledWait(node);
            break;
        }
        //阻塞到指定时间
        LockSupport.parkUntil(this, abstime);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
    return !timedout;
}


//阻塞到指定时长，可以设置时长单位，逻辑基本与await()方法一致
public final boolean await(long time, TimeUnit unit)
    throws InterruptedException {
    //计算超时时间
    long nanosTimeout = unit.toNanos(time);
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    final long deadline = System.nanoTime() + nanosTimeout;
    boolean timedout = false;
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        if (nanosTimeout <= 0L) {
            timedout = transferAfterCancelledWait(node);
            break;
        }
        if (nanosTimeout >= spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanosTimeout);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
        nanosTimeout = deadline - System.nanoTime();
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
    return !timedout;
}
```

### addConditionWaiter方法

```java
//添加一个等待节点，如果队尾节点已经被移动到同步队列中，
//那么遍历等待队列移除所有移动到同步队列中的节点
private Node addConditionWaiter() {
    //记录尾节点
    Node t = lastWaiter;
    //如果等待队列存在，并且节点的状态不为Node.CONDITION（说明该节点被其他线程唤醒移动到同步队列中了）
    if (t != null && t.waitStatus != Node.CONDITION) {
        //将所有已经移动到同步队列中的节点从等待队列中移除
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    //创建一个等待节点
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        //如果等待队列不存在，那么设置等待队列头
        firstWaiter = node;
    else
        //将创建的等待节点链接到等待队列尾
        t.nextWaiter = node;
    //将队尾指向创建的新节点
    lastWaiter = node;
    return node;
}
```

### unlinkCancelledWaiters方法

```java
//将所有已经移动到同步队列中的节点从等待队列中移除
private void unlinkCancelledWaiters() {
    //记录等待队列中的头节点
    Node t = firstWaiter;
    //记录状态为Node.CONDITION遍历的最后一个节点
    Node trail = null;
    //遍历等待队列
    while (t != null) {
        Node next = t.nextWaiter;
        //如果该节点已经被转移到同步队列中
        if (t.waitStatus != Node.CONDITION) {
            //从队列中移出
            t.nextWaiter = null;
            //trail == null，说明移除的是队头
            if (trail == null)
                //记录新的队头
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            //next == null说明当前已经到队尾了
            if (next == null)
                //设置队尾
                lastWaiter = trail;
        }
        else
            //更新trail
            trail = t;
        t = next;
    }
}
```

### checkInterruptWhileWaiting方法

```java
private int checkInterruptWhileWaiting(Node node) {
    //1.Thread.interrupted()判断线程中断状态
    //2.如果线程被中断，通过transferAfterCancelledWait可以知道当前线程被中断是发生在notify之后还是之前，true表示之前，false表示之后
    //3.如果线程已经在同步队列中了，返回THROW_IE，否则返回REINTERRUPT
    //4.如果线程未被中断返回0
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
    0;
}
```

### reportInterruptAfterWait方法

```java
//根据标识判断中断方式，是设置线程中断状态，还是直接抛出InterruptedException
private void reportInterruptAfterWait(int interruptMode)
    throws InterruptedException {
    if (interruptMode == THROW_IE)
        //抛出异常
        throw new InterruptedException();
    else if (interruptMode == REINTERRUPT)
        //设置线程中断状态
        selfInterrupt();
}
```

### signal方法

```java
//唤醒节点，将头节点从等待队列转移到同步队列中
public final void signal() {
    //在独占锁模式下，状态是否被占用，使用该方法需要先获取锁
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        //唤醒队头
        doSignal(first);
}

//唤醒所有节点
public final void signalAll() {
    //在独占锁模式下，状态是否被占用，使用该方法需要先获取锁
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        //唤醒所有节点
        doSignalAll(first);
}

//唤醒等待节点
private void doSignal(Node first) {
    do {
        //如果下一个等待节点不存在，那么将当前等待节点置为null
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        //将当前节点从等待队列中移除
        first.nextWaiter = null;
        //将节点从等待队列转移到同步队列中，如果此次操作失败，说明操作的节点已经被其他的线程唤醒
        //（或者在await方法fullyRelease中，释放锁失败，节点状态被设置为取消），那么直接寻找下一个节点
        //如果下一个等待节点存在，那么继续唤醒下一个线程，直到成功或者下一个节点不存在为止
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}

//通知所有的等待节点，将等待队列中所有节点转移到同步队列中
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```

### ConditionObject中其他方法

```java
//是否为当前同步器持有
final boolean isOwnedBy(AbstractQueuedSynchronizer sync) {
    return sync == AbstractQueuedSynchronizer.this;
}

//等待队列是否存在，也就是说是否还有线程处于wait状态
protected final boolean hasWaiters() {
    //在独占锁模式下，状态是否被占用，也就是说使用此方法，需要先获取锁
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    //判断等待队列中，是否存在Node.CONDITION节点
    for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
        if (w.waitStatus == Node.CONDITION)
            return true;
    }
    return false;
}

//获取等待队列的中Node.CONDITION等待节点的长度
protected final int getWaitQueueLength() {
    //在独占锁模式下，状态是否被占用，也就是说使用此方法，需要先获取锁
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int n = 0;
    //统计节点为Node.CONDITION的数量
    for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
        if (w.waitStatus == Node.CONDITION)
            ++n;
    }
    return n;
}

//获取所有等待节点的线程
protected final Collection<Thread> getWaitingThreads() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    ArrayList<Thread> list = new ArrayList<Thread>();
    for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
        if (w.waitStatus == Node.CONDITION) {
            Thread t = w.thread;
            if (t != null)
                list.add(t);
        }
    }
    return list;
}
```



### 外部类AQS方法

#### transferForSignal方法

```java
//将当前节点从等待队列转移到同步队列中
final boolean transferForSignal(Node node) {
    //通过cas将节点的WaitStatus设置为0，如果失败说明线程已经中断了(也可能释放锁失败，被取消)，返回false
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    //通过doSignal等方法唤醒线程后，会将节点的WaitStatus设置为0，之后加入到同步节点末尾
    //添加到同步队列末尾，返回原同步队列中的尾节点
    Node p = enq(node);
    //判断原尾节点waitStatus，如果ws > 0，说明为CANCELLED状态，节点被取消
    //设置原尾节点waitStatus为Node.SIGNAL，表示被释放后，要唤醒当前节点
    //如果设置waitStatus失败，说明节点的线程已经被取消
    //此时原尾节点p的ws值为1（判断了ws>0之后被取消的），unpark后，后续同步队列通过自旋，获取锁
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

#### isHeldExclusively方法

```java
//在独占锁模式下，状态是否被占用。留给子类实现，如果返回false，也就是说未被占用
//那么使用过if (!isHeldExclusively()) throw new IllegalMonitorStateException()；的方法
//最终都会抛出IllegalMonitorStateException，因此我们在使用这些方法之前，必须先获取锁
protected boolean isHeldExclusively() {
    throw new UnsupportedOperationException();
}
```



#### fullyRelease方法

```java
//释放持有当前节点的所有锁
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        //获取当前锁状态
        int savedState = getState();
        //释放锁
        if (release(savedState)) {
            failed = false;
            //返回锁状态
            return savedState;
        } else {
            //释放锁失败，抛出异常
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            //如果释放锁失败，设置node节点等待状态为Node.CANCELLED，表示被取消
            //被取消后，该节点一样会被转移到同步队列中
            node.waitStatus = Node.CANCELLED;
    }
}
```

#### isOnSyncQueue方法

```java
//是否在同步队列中
final boolean isOnSyncQueue(Node node) {
    //1.node.waitStatus == Node.CONDITION，表示当前节点在等待队列中
    //2.node.prev == null，等待队列是单向节点，没有前驱节点的，说明该节点在等待队列中
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    //如果有后继者，说明当前当前节点一定在同步队列中（等待队列中的节点只有nextWaiter）
    if (node.next != null) // If has successor, it must be on queue
        return true;
    //node.next == null，也有可能当前node在同步队列中的末尾，去同步队列中查找（从尾节点开始）
    return findNodeFromTail(node);
}

//判断节点是否在同步队列中存在，从尾节点开始查找(效率比较高，如果没有其他线程获取锁的情况下，那么尾节点可能就直接匹配上当前节点了)
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}

```

#### transferAfterCancelledWait方法

```java
//将节点转移到同步队列中
final boolean transferAfterCancelledWait(Node node) {
    //将节点的WaitStatus设置为0,如果通过checkInterruptWhileWaiting中调用此方法，
    //成功则说明中断发生在signal调用之前，（因为signal方法会将状态设置为0，此处应该会失败），那么加入到同步队列尾端并返回true，从这里我们知道，不管什么时候调用中断，该节点始终会被转移到同步队列中
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        //加入到同步队列末尾
        enq(node);
        return true;
    }
   	//如果上面通过cas设置失败，判断是否在同步队列中
    while (!isOnSyncQueue(node))
        //退出线程竞争，保证资源不会一直被当前线程占用
        Thread.yield();
    //到这步说明其他线程可能通过signal等方法已经将node转移到同步队列中了
    return false;
}
```

## 总结

上面已经解析了`ConditionObject`、以及相关联的外部类`AQS`中的方法。如果已经看过上一章[AbstractQueuedSynchronizer的解析]("https")，再看这章就容易得多。按照惯例，我们将所有方法串起来做一个总结。

条件等待：

1. 使用`await`方法，将线程等待。需要注意的是，在使用`await`方法之前，我们需要获取重入锁。调用`await`方法后，通过`addConditionWaiter`创建一个等待节点A，并加入到等待队列末尾。
2. 节点入队后，通过`fullyRelease`方法，释放当前线程持有的重入锁。
3. 释放锁后，会以自旋的方式通过`isOnSyncQueue`方法，判断A节点，是否被转移到同步队列中。
4. 如果A节点，还未转入同步队列，那么通过`LockSupport.park`阻塞当前线程，等待被唤醒。
5. 被唤醒后，通过`checkInterruptWhileWaiting`方法，检查是否是由于被中断而唤醒，如果被中断唤醒，会通过`transferAfterCancelledWait`判断是在调用`signal`之前被中断，还是调用`signal`之后被中断。如果是，则跳出循环，退出自旋。
6. 如果判断是通过`signal`方式唤醒，那么会进入下一次自旋，此时节点A已经被转移到同步队列中了，也会退出自旋。
7. 退出自旋后，通过`acquireQueued`方法，自旋从同步队列中获取锁(上一章AQS中已经详细解析了，`acquireQueued`方法)。`acquireQueued`方法结束后，判断记录的`interruptMode`中断模式，如果`interruptMode != THROW_IE`，那么将`interruptMode = REINTERRUPT`，表示方法结束后，只需要记录中断状态就行。
8. 如果A后面还有下一个等待节点。那么通过`unlinkCancelledWaiters`方法，将等待节点中所有被取消的节点清除

条件唤醒：

1. 通过使用`signal`方法，唤醒节点线程。调用`signal`方法后，会通过`isHeldExclusively`判断当前是否已经持有独占锁。如果没有就会抛出`IllegalMonitorStateException`异常。从这里就可以看出，要想使用`signal`必须先获取重入锁。
2. 唤醒等待节点头A，如果节点A没有后驱节点，那么将A节点置为null，否则后驱节点记录为新的头节点，并将A节点从等待队列中移除。
3. `transferForSignal`中通过cas设置A的`waitStatus`为0。如果设置失败，说明线程已经中断了(也可能释放锁失败，A节点被取消了)，返回false后，`signal`会向等待队列后面查找节点，直到节点被转移成功为止。
4. cas设置成功后，会将A节点转移到同步队列队尾，并返回旧的队尾。如果旧的队尾的`waitStatus`> 0(表示被取消)，或者设置`waitStatus`为`Node.SIGNAL`失败，那么A将不会被唤醒，那么通过`LockSupport.unpark`唤醒当前线程(此时程序会执行上面的第4步，判断是否中断，自旋获取锁)，后续同步队列通过自旋，获取锁。

