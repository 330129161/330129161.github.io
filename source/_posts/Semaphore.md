title:  【阻塞队列】--  Semaphore源码解析(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/book.png

toc: true 

tags:

  - 源码
  - java
  - 并发
  - juc
  - 锁

categories: 

  - [源码,并发] 

date: 2019-12-01 21:23:13

---

## 概述

​		上一篇我们解析了`CountDownLatch`，`Semaphore`、`CyclicBarrier`跟`CountDownLatch`一样，都是基于AQS实现的同步工具类。`Semaphore`用来控制线程的并发，初始时会指定一个许可数permits，线程执行前需要获取许可，获取到许可后许可数-1，线程会向下执行，如果没有可用许可，就会被阻塞。直到其他线程执行完成释放许可后，被阻塞线程才会继续尝试获取许可。<!--more-->也就是说`Semaphore`可以控制线程的并发量，如果一个线程执行完了，归还了许可，那么下一个线程才有机会获取许可，将同时执行的线程控制在设置的permits范围以内。

## 代码解析

```java
public class Semaphore implements java.io.Serializable {
    private static final long serialVersionUID = -3222578661600680210L;
	
    //基于AQS实现的共享锁
    private final Sync sync;
	
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;
		
        ///构造方法，设置设置许可数
        Sync(int permits) {
            setState(permits);
        }

        //获取当前剩余许可
        final int getPermits() {
            return getState();
        }

        //获取非公平共享锁
        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                //计算剩余许可
                int remaining = available - acquires;
                //如果remaining < 0，说明许可已经用完了，此时返回remaining == -1，表示获取失败
                //创建记录该线程的node节点加入到同步队列中，等待被唤醒
                //如果CAS设置成功，返回remaining的值，肯定>-1,代表成功
                //如果CAS设置失败，表示许可被其他线程抢到，重试
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
		
        //尝试释放共享
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                //next < current，说明releases<0
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                //更新许可
                if (compareAndSetState(current, next))
                    return true;
            }
        }

        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }

        //获取并返回立即可用的所有许可个数，并且将可用许可置0。
        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }

    //非公平锁
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
            super(permits);
        }

        //立马抢锁，许可有可能被新抢锁的线程获取
        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }

    //公平锁
    static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }

        //公平锁，会先检查同步队列中是否有其他的节点，如果存在的话，返回-1获取锁失败
        protected int tryAcquireShared(int acquires) {
            for (;;) {
                //同步队列中是否存在节点
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                //记录新的可用许可数
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }

    //构造函数，指定许可数，默认采用非公平锁
    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }

    //构造函数，指定许可数，是否使用公平锁
    public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }

    //获取锁，响应中断
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    //获取锁，不响应中断
    public void acquireUninterruptibly() {
        sync.acquireShared(1);
    }

    //尝试获取锁
    public boolean tryAcquire() {
        return sync.nonfairTryAcquireShared(1) >= 0;
    }

    //尝试获取锁超时
    public boolean tryAcquire(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    //释放锁
    public void release() {
        sync.releaseShared(1);
    }

    //获取锁，指定许可数
    public void acquire(int permits) throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireSharedInterruptibly(permits);
    }

    //获取锁，指定许可，响应中断
    public void acquireUninterruptibly(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireShared(permits);
    }

    //尝试获取锁，指定许可数
    public boolean tryAcquire(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        return sync.nonfairTryAcquireShared(permits) >= 0;
    }

    //尝试获取锁超时
    public boolean tryAcquire(int permits, long timeout, TimeUnit unit)
        throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        return sync.tryAcquireSharedNanos(permits, unit.toNanos(timeout));
    }

    //释放锁
    public void release(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.releaseShared(permits);
    }

    //获取可以用的许可数
    public int availablePermits() {
        return sync.getPermits();
    }

    //获取并返回立即可用的所有许可个数，并且将可用许可置0。
    public int drainPermits() {
        return sync.drainPermits();
    }

    protected void reducePermits(int reduction) {
        if (reduction < 0) throw new IllegalArgumentException();
        sync.reducePermits(reduction);
    }

    //当前是否为公平锁
    public boolean isFair() {
        return sync instanceof FairSync;
    }

    //当前同步队列中是否存在等待的线程
    public final boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }

    public final int getQueueLength() {
        return sync.getQueueLength();
    }

    protected Collection<Thread> getQueuedThreads() {
        return sync.getQueuedThreads();
    }

    public String toString() {
        return super.toString() + "[Permits = " + sync.getPermits() + "]";
    }

```
