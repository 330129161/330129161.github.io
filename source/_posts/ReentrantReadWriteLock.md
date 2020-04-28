title: ReentrantReadWriteLock源码解析(jdk1.8)

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

`ReentrantReadWriteLock`基于AQS实现的读写锁，写锁使用排他锁实现，读锁使用共享锁实现。

## 结构特点

1. `ReentrantReadWriteLock`继承自`ReadWriteLock`，读锁使用共享锁，写锁使用排它锁
2. `ReentrantReadWriteLock`实现了 `Serializable` 接口， 表示 `ReentrantReadWriteLock`支持序列化的功能 ，可用于网络传输。

![](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/ReentrantReadWriteLock.png)

## 重要属性

```java
//读锁
private final ReentrantReadWriteLock.ReadLock readerLock;
//写锁
private final ReentrantReadWriteLock.WriteLock writerLock;
//继承自AQS做为内部的同步器
final Sync sync;
```

### Sync内部类

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 6317671515068378041L;

    static final int SHARED_SHIFT   = 16;
    //
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    //最大值为2^16 - 1
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    //2^16 - 1，通过EXCLUSIVE_MASK可以将state的低16位取出来，详见exclusiveCount()方法
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

    //state的高16位代表着取读锁的获取次数，包括重入次数，获取到读锁一次加 1，释放掉读锁一次减 1
    //此使通过右移16位后获取读锁的获取次数
    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
    //state的低16位代表着写锁的获取次数，因为写锁是独占锁，同时只能被一个线程获得，所以它代表重入次数
    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

    //保持计数器，用来记录每个线程持有的读锁数量(读锁重入)
    static final class HoldCounter {
        int count = 0;
        final long tid = getThreadId(Thread.currentThread());
    }

    //ThreadLocalHoldCounter继承自ThreadLocal，记录每个线程的HoldCounter计数器
    static final class ThreadLocalHoldCounter
        extends ThreadLocal<HoldCounter> {
        public HoldCounter initialValue() {
            //初始化创建一个HoldCounter计数器
            return new HoldCounter();
        }
    }

	// 本地线程计数器，记录者各个线程读锁的数量
    private transient ThreadLocalHoldCounter readHolds;

    //将最后一次获取读锁的线程的 HoldCounter计数器缓存到这里
    private transient HoldCounter cachedHoldCounter;

    //第一个获取获取读锁的线程
    private transient Thread firstReader = null;
    //第一个获取读锁的线程的锁数量
    private transient int firstReaderHoldCount;

    //Sync默认构造方法
    Sync() {
        //创建当前锁对象现成的HoldCounter计数器
        readHolds = new ThreadLocalHoldCounter();
        //设置state,初始为0
        setState(getState());
    }

    //写锁是否应该被阻塞，留给子类实现
    abstract boolean readerShouldBlock();
	//读锁是否应该被阻塞，留给子类实现
    abstract boolean writerShouldBlock();

    //尝试获取独占锁
    protected final boolean tryAcquire(int acquires) {
        Thread current = Thread.currentThread();
        int c = getState();
        //获取c的低16位，锁的重入次数
        int w = exclusiveCount(c);
        //c != 0，说明当前存在读锁，或写锁
        if (c != 0) {
            //1.如果w==0，说明当前写锁空闲
            //2.current != getExclusiveOwnerThread()，说明当前线程不是独占锁的持有者
            //由1、2可以知道，如果想获取写锁，要么当前c == 0，也就是没有读锁和写锁
            //要么当前存在独占锁且被当前线程持有
            if (w == 0 || current != getExclusiveOwnerThread())
                return false;
            //判断重入锁数量是否超过了最大值
            if (w + exclusiveCount(acquires) > MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            //更新重入锁数量
            setState(c + acquires);
            return true;
        }
        //1.到这一步说明当前不存在锁
        //2.判断写操作是否应该被阻塞(如果是非公平锁，读锁操作不需要阻塞，公平锁会判断当前
        //同步队列是否有等待的节点)
        //3.如果失败，通过cas获取修改重入数，成功代表获取锁成功
        if (writerShouldBlock() ||
            !compareAndSetState(c, c + acquires))
            return false;
        //获取锁成功后，设置锁的持有者为当前线程
        setExclusiveOwnerThread(current);
        return true;
    }
    
    //尝试释放锁
    protected final boolean tryRelease(int releases) {
        //判断当前线程是否为持有锁的线程
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        int nextc = getState() - releases;
        //如果独占模式重入数为0了，说明独占模式被释放
        boolean free = exclusiveCount(nextc) == 0;
        if (free)
            //将锁的持有者设置为null
            setExclusiveOwnerThread(null);
        //更新锁的state
        setState(nextc);
        return free;
    }

    //尝试释放共享锁
    protected final boolean tryReleaseShared(int unused) {
        Thread current = Thread.currentThread();
        //如果当前线程为第一个获取锁的线程
        if (firstReader == current) {
            //firstReaderHoldCount == 1，说明当前线程持有的锁为只有一个
            if (firstReaderHoldCount == 1)
                //将firstReader置为null
                firstReader = null;
            else
                //否则重入数-1
                firstReaderHoldCount--;
        } else {
            //获取缓存的最后一个获取读锁的计数器
            HoldCounter rh = cachedHoldCounter;
            //如果最计数器不存在，或者最后一次记录的不是当前线程的计数器
            if (rh == null || rh.tid != getThreadId(current))
                //获取当前线程的计数器
                rh = readHolds.get();
            //获取当前线程持有读锁的数量
            int count = rh.count;
            //count <= 1，说明当前线程持有读锁的重入数最多为1
            if (count <= 1) {
                //移除当前线程的读取器
                readHolds.remove();
                //count <= 0（说明当前线程根本没有获取到读锁就释放锁），抛出异常
                if (count <= 0)
                    throw unmatchedUnlockException();
            }
            //读锁数量-1
            --rh.count;
        }
        //自旋重试
        for (;;) {
            int c = getState();
            //读锁数-1，后剩余的锁数量
            int nextc = c - SHARED_UNIT;
            //cas修改读锁数
            if (compareAndSetState(c, nextc))
                //如果nextc == 0，说明已经释放了所有的读锁
                return nextc == 0;
        }
    }

    //没有加锁就释放锁，会引起异常
    private IllegalMonitorStateException unmatchedUnlockException() {
        return new IllegalMonitorStateException(
            "attempt to unlock read lock, not locked by current thread");
    }

    //尝试获取共享锁(写锁)
    protected final int tryAcquireShared(int unused) {
        Thread current = Thread.currentThread();
        int c = getState();
        //1.exclusiveCount(c) != 0，说明当前存在写锁
        //2.getExclusiveOwnerThread() != current,持有写锁的对象线程不是当前线程
        //由1、2可知，想要获取读锁，要么当前不存在写锁，要么写锁的持有者为当前线程
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
            //返回-1表示失败
            return -1;
        //获取共享锁数量
        int r = sharedCount(c);
        //1.readerShouldBlock(),判断读锁是否应该被阻塞
        //如果是非公平锁，同步队列存在写锁的时候阻塞
        //如果是公平锁，同步节点中存在节点的时候阻塞
        //也就是说，非公平锁中只有存在其他读锁的时候，才会阻塞
        //而公平锁中不管读写锁，都需要被阻塞
        //2.r < MAX_COUNT,并且读锁的数量没有达到最大值
        //3.compareAndSetState(c, c + SHARED_UNIT)，相当于高16位+1，修改锁状态
        if (!readerShouldBlock() &&
            r < MAX_COUNT &&
            compareAndSetState(c, c + SHARED_UNIT)) {
            //r == 0说明当前没有其他读锁
            if (r == 0) {
                //设置第一个获取读锁的为当前线程
                //firstReader是不会放到readHolds里的, 这样，
                //在读锁只有一个的情况下，就避免了查找readHolds。
                firstReader = current;
                //读锁数量为1
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                //如果第一个读锁的为当前用户，那么将读锁数量+1
                firstReaderHoldCount++;
            } else {
                //非firstReader读锁重入计数更新，
                //读锁重入计数缓存，基于ThreadLocal实现
                HoldCounter rh = cachedHoldCounter;
                //如果最后一次缓存的计数器不存在，或者不是当前线程的计数器
                if (rh == null || rh.tid != getThreadId(current))
                    //记录最后统计读锁的读取器
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    //设置当前的读取器
                    readHolds.set(rh);
                //将读锁的重入数+1
                rh.count++;
            }
            return 1;
        }
        //如果第一次，获取读锁失败，通过fullTryAcquireShared再次尝试获取读锁，
        //锁定降级的逻辑在此方法中
        return fullTryAcquireShared(current);
    }
    
	//获取读锁失败后重试
    final int fullTryAcquireShared(Thread current) {
        HoldCounter rh = null;
        //自旋，重试
        for (;;) {
            //获取锁状态
            int c = getState();
            //exclusiveCount(c) != 0，说明存在写锁
            if (exclusiveCount(c) != 0) {
                //如果存在写锁且写锁的持有线程不是当前线程，返回-1
                //此处的判断比较重要，如果用户获取写锁后，又获取读锁
                //如果次数不判断读锁是否为当前锁的话，那么当前会被阻塞
                //那么也就不能在释放写锁了，这个时候就会出现死锁
                //通过这里我们也知道，获取的写锁的线程，可以继续获取读锁
                //也就是我们所说的锁降级
                if (getExclusiveOwnerThread() != current)
                    return -1;
            //判断是否应该被阻塞
            } else if (readerShouldBlock()) {
                //如果应该被阻塞的话，说明存在写锁(公平模式下可能是因为存在读锁)
				//如果第一个获取读锁的，说明锁重入
                if (firstReader == current) {

                } else {
                    //第一次重试进入，说明有其他新线程获取读锁了
                    if (rh == null) {
                        //获取缓存的最后一次获取读锁的计数器
                        rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current)) {
                            //获取当前线程的计数器
                            rh = readHolds.get();
                            //如果rh.count == 0，说明该线程第一次获取读锁
                            if (rh.count == 0)
                                //移除当前线程的计数器
                                readHolds.remove();
                        }
                    }
                    //获取锁失败，跳出重试，后续的操作中会被加入到同步队列中
                    if (rh.count == 0)
                        return -1;
                }
            }
            //如果获取读锁的数量超过最大值，抛出异常
            if (sharedCount(c) == MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            //记录锁状态，c + SHARED_UNIT相当于高16位+1(右移后相当于读锁数+1)
            if (compareAndSetState(c, c + SHARED_UNIT)) {
                //如果分享锁数为0,那么设置第一个获取读锁的为当前线程
                if (sharedCount(c) == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                //重入数+1
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    if (rh == null)
                        rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    //读锁数+1
                    rh.count++;
                    //记录最后访问的线程读取器
                    cachedHoldCounter = rh; // cache for release
                }
                return 1;
            }
        }
    }

	//获取写锁
    final boolean tryWriteLock() {
        Thread current = Thread.currentThread();
        int c = getState();
        //当前存在读锁或者写锁
        if (c != 0) {
            //记录当前写锁的冲入次数
            int w = exclusiveCount(c);
            //1.如果w==0，说明当前没有写锁
            //2.current != getExclusiveOwnerThread()，说明当前锁的持有写锁线程不是当前线程
            //因此只有当前存在读锁，并且读锁的持有者为当前线程才会继续下去，否则返回false
            if (w == 0 || current != getExclusiveOwnerThread())
                return false;
            //重入数超过最大值，抛出异常
            if (w == MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
        }
        //通过cas修改state,占有锁
        if (!compareAndSetState(c, c + 1))
            return false;
        //设置独占锁的持有线程为当前线程
        setExclusiveOwnerThread(current);
        return true;
    }

    //获取读锁
    final boolean tryReadLock() {
        Thread current = Thread.currentThread();
        //自旋，重试
        for (;;) {
            int c = getState();
            //1.exclusiveCount(c) != 0，表示当前存在写锁
            //2.getExclusiveOwnerThread() != current表示写锁的线程不是当前线程
            //因此想要获取锁，要么当前写锁空闲，要么锁重入
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return false;
            //获取读锁数量
            int r = sharedCount(c);
            //如果读锁数量已经到达最大值，抛出异常
            if (r == MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            //修改读锁数量，如果失败，会进入下一次自旋重试
            if (compareAndSetState(c, c + SHARED_UNIT)) {
                //r == 0，说明当前不存在读锁
                if (r == 0) {
                    //记录第一个读锁和持有数量
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    //如果第一个读锁为当前线程，那么将读锁数量+1
                    firstReaderHoldCount++;
                } else {
                    //如果第一个获取读锁的线程不是当前线程
                    //
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return true;
            }
        }
    }

    //判断当前持有独占锁的线程是否为当前线程
    protected final boolean isHeldExclusively() {
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

	//创建一个等待队列节点
    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    //获取持有写锁的线程
    final Thread getOwner() {
        return ((exclusiveCount(getState()) == 0) ?
                null :
                getExclusiveOwnerThread());
    }

    //获取持有读锁的线程数量
    final int getReadLockCount() {
        return sharedCount(getState());
    }

    //是否为写锁
    final boolean isWriteLocked() {
        return exclusiveCount(getState()) != 0;
    }
	//获取写锁的重入数
    final int getWriteHoldCount() {
        return isHeldExclusively() ? exclusiveCount(getState()) : 0;
    }
    
	//获取读锁的数量
    final int getReadHoldCount() {
        if (getReadLockCount() == 0)
            return 0;
		
        Thread current = Thread.currentThread();
        //如果当前只有一个读线程，直接返回当前线程持有的读锁数
        if (firstReader == current)
            return firstReaderHoldCount;
		//如果缓存的是当前线程的计数器，那么直接返回count
        HoldCounter rh = cachedHoldCounter;
        if (rh != null && rh.tid == getThreadId(current))
            return rh.count;
		//通过readHolds获取当前线程的计数器，如果数量为0，移除当前线程计数器
        int count = readHolds.get().count;
        if (count == 0) readHolds.remove();
        return count;
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        readHolds = new ThreadLocalHoldCounter();
        setState(0); // reset to unlocked state
    }

    //获取锁状态
    final int getCount() { return getState(); }
}
```

### NonfairSync内部类

```java
//非公平锁
static final class NonfairSync extends Sync {

    //是否应该阻塞，非公平锁中默认写锁不需要阻塞
    final boolean writerShouldBlock() {
        return false; 
    }
    final boolean readerShouldBlock() {
        //该方法在AQS中，判断当前同步队列是否有独占节点在等待
        return apparentlyFirstQueuedIsExclusive();
    }
}

//AQS中的方法，判断同步队列中是否有独占节点在等待
final boolean apparentlyFirstQueuedIsExclusive() {
    Node h, s;
    return (h = head) != null &&
        (s = h.next)  != null &&
        !s.isShared()         &&
        s.thread != null;
}
```

### FairSync内部类

```java
//公平锁
static final class FairSync extends Sync {
    private static final long serialVersionUID = -2274990926593161451L;
    //公平锁中，需不需要阻塞写操作，看当前同步队列中是否有其他线程等待
    final boolean writerShouldBlock() {
        return hasQueuedPredecessors();
    }
    //公平锁中，需不需要阻塞读操作，看当前同步队列中是否有其他线程等待
    final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }
}
```

### ReadLock内部类

```java
public static class ReadLock implements Lock, java.io.Serializable {
    //sync继承自AQS
    private final Sync sync;

    protected ReadLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }

    //获取读锁
    public void lock() {
        //通过sync获取共享锁，会调用模板方法tryAcquireShared,
        //失败后会通过sync中的doAcquireShared方法生成一个记录当前线程的节点，
        //并加入到同步队列的末尾
        sync.acquireShared(1);
    }

    //获取读锁，响应中断
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    //尝试获取读锁，失败立即返回(使用此方法，失败不会创建同步节点，而是立即返回false)
    public boolean tryLock() {
        return sync.tryReadLock();
    }

    //尝试获取读锁，指定超时时间(同上)
    public boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    //释放锁
    public void unlock() {
        sync.releaseShared(1);
    }

    //ReadLock中，不允许创建等待队列节点
    public Condition newCondition() {
        throw new UnsupportedOperationException();
    }

    public String toString() {
        int r = sync.getReadLockCount();
        return super.toString() +
            "[Read locks = " + r + "]";
    }
}
```

### ReadLock内部类

```java
public static class WriteLock implements Lock, java.io.Serializable {

    private final Sync sync;

    protected WriteLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }

    //获取写锁
    public void lock() {
        sync.acquire(1);
    }

    //获取写锁，响应中断
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    //尝试获取锁
    public boolean tryLock( ) {
        return sync.tryWriteLock();
    }

    //尝试获取超时锁
    public boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

    //释放锁
    public void unlock() {
        sync.release(1);
    }

    //创建等待队列
    public Condition newCondition() {
        return sync.newCondition();
    }

    public String toString() {
        Thread o = sync.getOwner();
        return super.toString() + ((o == null) ?
                                   "[Unlocked]" :
                                   "[Locked by thread " + o.getName() + "]");
    }

    //判断当前线程是否是该锁的持有者
    public boolean isHeldByCurrentThread() {
        return sync.isHeldExclusively();
    }
	
    //获取写锁的重入数
    public int getHoldCount() {
            return sync.getWriteHoldCount();
    }
}
```

## 总结

介绍完`ReentrantReadWriteLock`中几个内部类，也就没啥方法了。`ReentrantReadWriteLock`获取写锁部分的代码还比较简单。基本上跟之前解析的重入锁`ReentrantLock`没多大区别。关键在于读锁的部分，引入了计数器、锁降级部分，逻辑上要复杂那么一点。下面简单做一个总结

获取写锁

1. `WriteLock`通过`lock()`方法调用获取写锁后，会调用`sync.acquire(1)`获取锁方法。`sync`继承自`AQS`，该方法在`AQS`中用于获取锁，`AQS`通过调用模板方法`tryAcquire()`又回到本类中。看过前面`ReentrantLock`解析的，这里应该比较熟悉了。
2. `tryAcquire()`方法，尝试获取锁。通过`exclusiveCount(c)`方法，获取写锁的数量(获取低16位，也就是写锁的重入数)，如果从c != 0说明写锁存在，判断持有写锁的是否为当前线程，如果是，则表示重入，那么写锁重入数+1，返回true，获取锁成功。如果不是当前线程持有的锁，直接返回false，获取锁失败。
3. 如果写锁不存在，通过`writerShouldBlock`判断写锁是否需要被阻塞。
4. 非公平锁情况下，写锁不会被阻塞，那么会执行`!compareAndSetState(c, c + acquires)`方法修改锁状态，修改成功，那么获取锁成功。修改失败，返回false，设置锁失败。
5. 公平情况下，会通过`FairSync`的方法最终调用`hasQueuedPredecessors`，判断当前同步队列是否存在节点。如果存在，说明前面有其他获取锁线程在等待，那么获取锁失败。如果同步队列中没有节点，那么跟上面一样，修改锁状态，看是否获取锁成功。
6. 上面使用`tryAcquire`方法获取锁，如果成功，那么`WriteLock`的`lock()`方法直接返回。如果失败，那么`AQS`的acquire方法中，会调用`addWaiter(Node.EXCLUSIVE), arg)`方法，生成一个记录当前线程的节点，并加入到同步队列的末尾。最后通过`acquireQueued`方法，以自旋的方式，获取锁，直到获取锁成功。或者线程被中断。

释放写锁

1. `WriteLock`通过`lock()`方法释放写锁，会调用`sync.release(1)`方法。跟上面获取锁一样。该方法也在`AQS`中，`release()`会调用模板方法`tryRelease(arg)`释放锁。
2. `tryRelease`方法会通过`isHeldExclusively`判断，当前线程是否为持有锁的线程。如果不是抛出IllegalMonitorStateException。
3. 判断`boolean free = exclusiveCount(nextc) == 0`，判断是否释放了所有的重入锁。如果是，设置持有写锁的持有者为null。设置新的state，返回free 。
4. 如果第3步返回true,也就是释放所有的重入锁。那么判断同步队列头节点不为null，且状态是否为-1(表示后驱节点需要被唤醒，不了解的可以去看一下前面关于 AQS的解析)，那么通过`unparkSuccessor`唤醒后驱节点，返回true。

获取读锁

1. `ReadLock`通过`lock()`方法获取读锁，会调用`sync.acquireShared(1)`。通过`tryAcquireShared`尝试获取共享锁，`exclusiveCount(c) != 0` ，判断当前是否存在独占锁(写锁)，如果存在，并且通过`getExclusiveOwnerThread() != current`判断，持有写锁的不是当前线程。会直接返回-1，表示尝试获取锁失败。
2. 如果当前不存在写锁，或者写锁的持有者是当前线程。那么通过`sharedCount(c)`获取读锁的数量(将state的高16位向右移16位，获取读锁的数量)。随后，通过`!readerShouldBlock()`判断，读锁是否需要阻塞。
3. 非公平锁情况下，会通过`NonfairSync`最终调用`apparentlyFirstQueuedIsExclusive`方法，判断当前同步队列中是否存在独占锁(写锁)，如果存在，那么应该被阻塞(我们知道存在写锁的情况下，是不能获取读锁的)，不会进入到`if`代码块中。
4. 公平锁情况下，会通过`FairSync`最终调用`hasQueuedPredecessors()`方法，用来判断当前同步队列中是否存在锁(包括读锁)。
5. 如果读锁没有被阻塞，判断读锁的数量是否小于最大值，随后通过cas修改锁状态(读锁+1)。成功获取读锁后，通过记录的读锁数判断，当前线程是否为第一次获取读锁线程。如果是则记录，如果不是，判断当前线程是否为第一个获取读锁的线程，如果是，那么记录首次获取锁的数量firstReaderHoldCount+1。
6. 如果不是，那么获取缓存的最后一次获取锁的计数器`cachedHoldCounter`。如果`cachedHoldCounter`为空，说明当前线程是第二个获取读锁的线程。如果`cachedHoldCounter`记录的线程id不是当前线程，更新`cachedHoldCounter`缓存。
7. 如果`cachedHoldCounter`存在，并且缓存的正是当前线程，如果`rh.count == 0`，说明这个线程原来获取读锁后被释放了，readHolds中没有`cachedHoldCounter`的记录了，那么通过`readHolds.set(rh)`重新记录到readHolds中，随后将读锁计数器+1
8. 如果读锁被阻塞，或者读锁的数量超过最大值，又或者更改state状态失败了(说明被其他线程抢了)，那么通过`fullTryAcquireShared(current)`重新获取读锁。
9. `fullTryAcquireShared`方法上来就是一个死循环，用于重试。
10. 通过`exclusiveCount(c) != 0`判断，是否存在写锁，如果存在，那么判断写锁是否被当前线程持有，如果不是，那么返回-1，跳出死循环，获取锁失败。
11. 通过判断写锁的持有者是否为当前线程，就是锁降级的体现。说明如果当前线程持有写锁，那么也能获取到读锁。
12. 下面的逻辑基本上跟`tryAcquireShared`中方法一致，获取锁失败后会返回-1。最后`sync.acquireShared`在`tryAcquireShared`返回-1后，会通过`doAcquireShared(arg)`方法生成一个记录当前线程的共享节点，并加入到同步队列中，等待被唤醒。
13. 设想一下，如果没有第11步的判断直接返回-1，那么该线程获取读锁被阻塞之后，原来的写锁就没法释放，就会产生死锁。

