title: CountDownLatch源码解析(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/cs/cs-15.jpg

toc: true 

tags:

  - 源码
  - java
  - 并发
  - juc
  - 锁

categories: 

  - [源码,并发] 

date: 2019-11-23 22:33:54

---

## 概述

​		`CountDownLatch`是基于AQS实现的一个同步工具类。它允许一个线程一直等待，直到其他线程执行完成后再执行。`CountDownLatch`源码比较简单，基于AQS实现共享锁的等待 ，初始化时只需要设置一个初始值，后续针对锁的状态进行控制，最后根据锁的状态来释放等待线程即可。`CountDownLatch`跟`CyclicBarrier`不同，8.19是不可复用的，`CountDownLatch`释放等待线程后，就不能再次使用了。看此源码之前，建议先看[AQS源码解析]("https://www.yingu.site/2019/07/26/AbstractQueuedSynchronizer/")。

<!--more-->

## 代码解析

以下就是`CountDownLatch`的全部代码：

```java
public class CountDownLatch {
	
    //基于AQS实现的共享锁
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        //构造方法，设置一个初始count，
        //当count==0时，释放被await方法阻塞的线程
        Sync(int count) {
            setState(count);
        }

        //获取锁的state状态，也就是count
        int getCount() {
            return getState();
        }

        //尝试获取共享锁
        protected int tryAcquireShared(int acquires) {
            //如果state已经为0了，返回1，后续的await操作不会被阻塞，也就是说
            //CountDownLatch已经失效了，如果返回-1，获取锁失败，那么会创建记录该线程
            //的node节点加入到同步队列中，等待被唤醒
            return (getState() == 0) ? 1 : -1;
        }

        //尝试释放共享锁
        protected boolean tryReleaseShared(int releases) {
            for (;;) {
                int c = getState();
                //如果state为0，直接返回false，表示锁未完全释放
                if (c == 0)
                    return false;
                int nextc = c-1;
                //否则，将state-1
                if (compareAndSetState(c, nextc))
                    //如果nextc已经为0，说明当前锁已经完全释放了，返回成功
                    //通过共享锁的传播性，最终所有被await阻塞的线程，都会被陆续唤醒
                    return nextc == 0;
            }
        }
    }

    private final Sync sync;

    public CountDownLatch(int count) {
        //初始值不能小于0
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

    //阻塞调用线程
    public void await() throws InterruptedException {
        //调用AQS的方法，响应中断
        sync.acquireSharedInterruptibly(1);
    }

    //超时阻塞调用线程
    public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    //每一次调用countDown时，state都会-1，直到state==0为止
    public void countDown() {
        //释放共享锁，state会-1，state=0时，会释放await等待的线程。
        sync.releaseShared(1);
    }

	//获取当前数量，也就是还需要执行多少次countDown后才会释放await等待线程的数量
    public long getCount() {
        return sync.getCount();
    }
    
    public String toString() {
        return super.toString() + "[Count = " + sync.getCount() + "]";
    }

}
```

## 总结

​		上面就是针对`CountDownLatch`的全部解析了，可以看出来代码量很少，实现的逻辑也很简单(主要的代码都在AQS中)。`CountDownLatch`就像是一个计数器，初始化时执行一个计数值，每一次调用countDown时，就将值-1，当值到0时，释放等待线程。

