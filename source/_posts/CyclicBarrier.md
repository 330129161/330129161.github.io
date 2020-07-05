title: 【阻塞队列】-- CyclicBarrier源码解析(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/cs/cs-16.jpg

toc: true 

tags:

  - 源码
  - java
  - 并发
  - juc
  - 锁

categories: 

  - [源码,并发] 

date: 2019-12-25 20:41:01

---

## 概述

​		`CyclicBarrier`与`CountDownLatch`类似，它们都是阻塞一组线程直到某个事件的发生。`CyclicBarrier`与`CountDownLatch`的关键区别在于，`CyclicBarrier`中的所有的线程必须同时达到屏蔽点才能继续执行，如果其中一个线程被中断，那么所有的等待的线程都会立刻被唤醒，并且抛出异常。而`CountDownLatch`中线程之间不会收到干扰。`CyclicBarrier`可以复用，每次打破屏障后，都会生成一个新的屏障，供下次使用，而`CountDownLatch`用一次之后就无效了。

<!--more-->

## 解析

```java
public class CyclicBarrier {

    //内部类，表示屏障
    private static class Generation {
        //当前屏障是否被打破，屏障被打破后，等待的线程才会执行
        boolean broken = false;
    }
	
    private final ReentrantLock lock = new ReentrantLock();
	
    private final Condition trip = lock.newCondition();

    //屏障拦截的线程数
    private final int parties;

    //栅栏打破后，执行的线程
    private final Runnable barrierCommand;

    //初始一个新的屏障
    private Generation generation = new Generation();

    private int count;

    //屏障被打破后，会调用此方法，重置屏障，以便下次复用
    private void nextGeneration() {
        //唤醒所有等待的线程
        trip.signalAll();
        //初始化拦截的线程数
        count = parties;
        //初始化新的屏障
        generation = new Generation();
    }

    //打破屏障
    private void breakBarrier() {
        //屏障标识设置为true，标识被打破
        generation.broken = true;
        //初始化拦截的线程数
        count = parties;
        //唤醒所有等待的线程
        trip.signalAll();
    }

    //线程等待，timed是否超时，超时时间
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;
			//如果屏障已经被打破，那么抛出异常
            if (g.broken)
                throw new BrokenBarrierException();
			//如果线程被中断，那么打破当前屏障，其他被拦截到屏障前的线程将得到释放
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }
			//当前拦截数-1
            int index = --count;
            //如果拦截数为0，说明达到了突破屏障的条件
            if (index == 0) {  // tripped
                //标记执行状态
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    //如果预设的打破屏障后的方法存在，那么执行
                    if (command != null)
                        command.run();
                    ranAction = true;
                    //重置屏障
                    nextGeneration();
                    return 0;
                } finally {
                    //栅栏打破后，执行的线程如果出现异常，或者屏障重置失败，那么打破屏障
                    //打破屏障后，所有等待的线程都将被唤醒，并抛出BrokenBarrierException异常
                    if (!ranAction)
                        breakBarrier();
                }
            }
			//如果当前还没有达到打破屏障的条件，并且线程未被中断
            for (;;) {
                try {
                    //是否锁超时
                    if (!timed)
                        //如果没启用，那么直接调用条件锁的等待方法，响应中断
                        trip.await();
                    else if (nanos > 0L)
                        //如果启用，那么直接调用条件锁的超时等待方法，响应中断
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    //如果等待的途中，被中断
                    //判断当前屏障是被重置，并且未被打破
                    if (g == generation && ! g.broken) {
                        //那么打破屏障
                        breakBarrier();
                        throw ie;
                    } else {
                        //如果屏障已经被其他线程重置了，或者被打破了，那么响应中断
                        Thread.currentThread().interrupt();
                    }
                }
				//判断屏障是否被打破，如果被打破，那么抛出异常
                if (g.broken)
                    throw new BrokenBarrierException();
				//如果屏障被重置，说明已经达到突破屏障的条件了，返回index,执行线程
                if (g != generation)
                    return index;
				//如果当前启动锁超时，那么检测当前是否等待超时，如果等待超时
                //那么打破屏障
                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            //无论是否成功，最终都会释放锁
            lock.unlock();
        }
    }

    //栅栏的初始方法，执行一个等待屏障的线程数parties，以及打破屏障后执行的一个Runnable
    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }

    public CyclicBarrier(int parties) {
        this(parties, null);
    }

    public int getParties() {
        return parties;
    }

    //等待阻塞，直到屏障被打破
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }

    //等待阻塞，直到屏障被打破或者超时等待
    public int await(long timeout, TimeUnit unit)
        throws InterruptedException,
               BrokenBarrierException,
               TimeoutException {
        return dowait(true, unit.toNanos(timeout));
    }

    //当前屏障是否被打破
    public boolean isBroken() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return generation.broken;
        } finally {
            lock.unlock();
        }
    }

    //重置屏障
    public void reset() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //跳过当前的屏障，之前的等待的线程，会抛出异常
            breakBarrier();   // break the current generation
            //打开一个新的屏障
            nextGeneration(); // start a new generation
        } finally {
            lock.unlock();
        }
    }

    //获取等待的线程数
    public int getNumberWaiting() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return parties - count;
        } finally {
            lock.unlock();
        }
    }
}
```

