title: ReentrantLock源码解析(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/cs/cs-25.jpg

toc: true 

tags:

  - 源码
  - java
  - 并发

categories: 

  - [源码,并发] 

date: 2019-09-24 20:41:01

---

## 概述

`ReentrantLock`是一个基于AQS([AQS源码解析]("https://www.yingu.site/2019/07/26/AbstractQueuedSynchronizer/"))实现的可重入的独占锁，提供了公平锁与非公平锁的获取。与synchronized关键词实现的独占锁不同，`ReentrantLock`锁的粒度更细、功能更加丰富、使用也更灵活。不过`ReentrantLock`需要手动控制锁的释放。<!-- more -->

## 结构特点

1. `ReentrantLock`实现了`Lock`接口，提供了获取锁，释放锁，创建`Condition`等功能。
2. `ReentrantLock`实现了 `Serializable` 接口， 表示 `ReentrantLock`支持序列化的功能 ，可用于网络传输。

![ReentrantLock结构图](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/ReentrantLock.png)

## 重要属性

`ReentrantLock`中`Sync`通过继承`AQS`实现同步器的同时，实现了`AQS`中获取锁、释放锁的模板方法。

而`NonfairSync`与`FairSync`通过继承自`Sync`，分别实现了非公平锁与公平锁的获取。

```java
public class ReentrantLock implements Lock, java.io.Serializable {
	//继承自AQS做为内部的同步器
    private final Sync sync;
}
```

### Sync内部类

```java
//Sync继承自AQS做为内部的同步器,实现了AQS中获取锁、释放锁的模板方法
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;

    //模板方法、留给子类实现，本类中分别由内部类NonfairSync、FairSync实现，通过lock接口分别获取
    //非公平锁与公平锁
    abstract void lock();

    //获取非公平锁
    final boolean nonfairTryAcquire(int acquires) {
        //记录当前线程
        final Thread current = Thread.currentThread();
        //获取锁状态
        int c = getState();
        if (c == 0) {
            //如果当前锁未被其他线程占有，通过cas设置state,占有锁
            if (compareAndSetState(0, acquires)) {
                //占有锁成功后，设置当前锁的拥有者未当前线程
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //判断当前锁是为当前线程持有，如果是那么可重入
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            //更新state
            setState(nextc);
            return true;
        }
        //如果锁已经被其他线程占有，返回false
        return false;
    }

    //释放锁
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        //如果当前持有锁的线程不是当前线程，那么抛出IllegalMonitorStateException
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        //是否完全释放
        boolean free = false;
        //c == 0，表示完成释放
        if (c == 0) {
            free = true;
            //将当前持有锁线程置为null
            setExclusiveOwnerThread(null);
        }
        //更改state值
        setState(c);
        //返回free标记
        return free;
    }

    //当前锁的持有者是否为当前线程
    protected final boolean isHeldExclusively() {
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

    //创建一个等待队列Condition(创建的方法在Aqs中)
    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    //获取当前持有锁的线程
    final Thread getOwner() {
        return getState() == 0 ? null : getExclusiveOwnerThread();
    }

    //获取持有锁的数量，如果当前线程持有锁，返回state，否则为0
    final int getHoldCount() {
        return isHeldExclusively() ? getState() : 0;
    }

    //判断是否已经被锁定
    final boolean isLocked() {
        return getState() != 0;
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
    }
}
```

### NonfairSync内部类

```java
//NonfairSync继承自Sync，实现了lock中非公平锁的功能
static final class NonfairSync extends Sync {
	//获取非公平锁
    final void lock() {
        //快速获取锁，通过cas修改state，标记获取锁
        if (compareAndSetState(0, 1))
            //获取锁成功后，将持有锁的线程置为当前线程
            setExclusiveOwnerThread(Thread.currentThread());
        else
            //调用acquire获取锁后，进入到aqs的acquire方法中，执行tryAcquire操作获取锁
            //如果失败，将创建一个记录当前线程的节点，加入到同步队列中
            acquire(1);
    }

    //获取锁
    protected final boolean tryAcquire(int acquires) {
        //返回一个非公平锁
        return nonfairTryAcquire(acquires);
    }
}
```

### FairSync内部类

```java
//FairSync继承自Sync，实现了lock中公平锁的功能
static final class FairSync extends Sync {

    //获取公平锁
    final void lock() {
        //与NonfairSync中lock方法不同，该lock不会立即通过cas修改state的方式快速获取锁
        //而是先通过acquire方法，尝试获取锁
        acquire(1);
    }
	
    //尝试获取锁
    protected final boolean tryAcquire(int acquires) {
        //记录当前线程
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            //1.如果当前锁未被其他线程持有，hasQueuedPredecessors()判断当前同步队列中是否有等待的节点。			//2.如果当前同步队列中没有等待的节点，那么通过cas修改state获取锁
            //3.如果当前同步节点中存在等待的节点，那么该方法会返回false，会被加入到同步队列末尾
            //4.第三步不清楚的地方，可以看看aqs源码解析中的acquire方法
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                //获取锁成功后，设置当前占有的线程
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //判断当前锁是为当前线程持有，如果是那么可重入，下面的操作，跟上面非公平锁中一致
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

## 常用方法

### 构造方法

```java
//创建ReentrantLock锁，默认是非公平锁
public ReentrantLock() {
	sync = new NonfairSync();
}

//通过fair标记来创建一个公平锁，或者非公平锁
public ReentrantLock(boolean fair) {
    //fair为true时，创建公平锁，false创建非公平锁
    sync = fair ? new FairSync() : new NonfairSync();
}
```

### 基本方法

```java
//获取锁
public void lock() {
	sync.lock();
}
//获取锁，响应中断
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}

//尝试获取非公平锁
public boolean tryLock() {
    return sync.nonfairTryAcquire(1);
}

//获取超时锁，响应中断
public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}

//释放锁
public void unlock() {
    sync.release(1);
}

//创建一个等待队列
public Condition newCondition() {
    return sync.newCondition();
}

//获取持有锁的数量（重入次数）
public int getHoldCount() {
    return sync.getHoldCount();
}

//判断锁是否被当前线程持有
public boolean isHeldByCurrentThread() {
    return sync.isHeldExclusively();
}

//当前是否已经被锁定
public boolean isLocked() {
    return sync.isLocked();
}

//判断是否为公平锁
public final boolean isFair() {
    return sync instanceof FairSync;
}

//获取持有锁的线程
protected Thread getOwner() {
    return sync.getOwner();
}

//同步队列中是否有节点在等待
public final boolean hasQueuedThreads() {
    return sync.hasQueuedThreads();
}

//判断传入的线程是否在同步队列中
public final boolean hasQueuedThread(Thread thread) {
    return sync.isQueued(thread);
}

//获取同步队列长度
public final int getQueueLength() {
    return sync.getQueueLength();
}

//获取同步队列中所有等待的线程
protected Collection<Thread> getQueuedThreads() {
    return sync.getQueuedThreads();
}

//查询是否与指定condition有关的等待线程。
public boolean hasWaiters(Condition condition) {
    if (condition == null)
        throw new NullPointerException();
    if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
        throw new IllegalArgumentException("not owner");
    return sync.hasWaiters((AbstractQueuedSynchronizer.ConditionObject)condition);
}

//获取指定等待condition有关的等待线程长度
public int getWaitQueueLength(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitQueueLength((AbstractQueuedSynchronizer.ConditionObject)condition);
}

//获取指定等待condition有关的所有等待线程
protected Collection<Thread> getWaitingThreads(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitingThreads((AbstractQueuedSynchronizer.ConditionObject)condition);
    }
```

### 总结

`ReentrantLock`中同步委托给AQS，而加锁和解锁是通过改变AbstractQueuedSynchronizer的state属性。`ReentrantLock`方法比较少，也简单。下面针对非公平锁和公平锁做一个总结。

非公平锁：
	1.线程获取锁时，检查该锁是否被占用，返回被获取的次数
	2.如果线程未被占用获取，获取锁，利用CAS进行State的设置 ，设置当前获取锁次数为1，设置占用该锁的线程
	3.如果线程被占用，检查被占用的线程是否 是当前线程
	4.如果是当前线程，获取锁次数并且+1

公平锁：
	1.上面第二步，同时会检查 当前节点是否拥有前驱节点，如果有前驱节点，证明有线程比当前线程更早请求自愿，根据公平性，当前线程获取资源失败




​	

