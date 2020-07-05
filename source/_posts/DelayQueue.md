title: 【阻塞队列】-- DelayQueue源码解析(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/cs/cs-17.jpg

toc: true 

tags:

  - 源码
  - java
  - 阻塞队列

categories: 

  - [源码,阻塞队列] 

date: 2020-03-16 18:32:14

---

## 概述

​	`DelayQueue`是一个基于`PriorityQueue`优先级队列实现的有序的无界阻塞队列。放入队列的元素必须实现Delayed接口，其中的对象只能在其到期时才能从队列中取走。<!-- more -->

## 结构

![](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/DelayQueue.png)

1. `DelayQueue`实现了BlockingQueue，具有阻塞队列的特征
2. DelayQueue还实现了`Delayed`接口，实现了 compareTo和getDelay方法，`compareTo`用于`PriorityQueue`中对比时间排序， getDelay用于获取剩余时间。

## 属性

```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {

    private final transient ReentrantLock lock = new ReentrantLock();
    
    //PriorityQueue作为内部实现优先级
    private final PriorityQueue<E> q = new PriorityQueue<E>();

    //用于优化内部阻塞通知的线程
    private Thread leader = null;

    //用于控制的锁
    private final Condition available = lock.newCondition();
}
```

## 方法

在上一章[PriorityBlockingQueue源码解析中]("https://www.yingu.site/2020/03/15/PriorityBlockingQueue/")已经介绍过优先阻塞队列了，本章的`PriorityQueue`跟`PriorityBlockingQueue`中元素的插入删除操作基本一致，这里就不在介绍了，想了解的可以去看看上一章。

### 构造方法

```java
public DelayQueue() {}

//通过集合初始化的构造方法
public DelayQueue(Collection<? extends E> c) {
    this.addAll(c);
}

//父类AbstractQueue.class中的方法
public boolean addAll(Collection<? extends E> c) {
    if (c == null)
        throw new NullPointerException();
    if (c == this)
        throw new IllegalArgumentException();
    boolean modified = false;
    for (E e : c)
        if (add(e))
            modified = true;
    return modified;
}
```

### 生产方法

```java
//添加元素，交给offer
public boolean add(E e) {
    return offer(e);
}

public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //通过PriorityQueue来实现插入
        q.offer(e);
        //如果队首元素是刚插入的元素，则设置leader为null，并且唤醒一个阻塞的线程
        if (q.peek() == e) {
            leader = null;
            available.signal();
        }
        return true;
    } finally {
        lock.unlock();
    }
}

//添加元素，交给offer
public void put(E e) {
    offer(e);
}

//添加元素，交给offer
public boolean offer(E e, long timeout, TimeUnit unit) {
    return offer(e);
}
```

### 消费方法

```java
//移除并返回元素，获取不到元素时不回阻塞，直接返回null
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //查看头部元素
        E first = q.peek();
        //如果不存在或者还未到时间，那么返回null
        //first.getDelay方法，获取剩余时间
        if (first == null || first.getDelay(NANOSECONDS) > 0)
            return null;
        else
            //否则返回
            return q.poll();
    } finally {
        lock.unlock();
    }
}

//移除并返回元素，响应中断，获取不到元素时阻塞
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            //查看元素
            E first = q.peek();
            //如果不存在，那么阻塞
            if (first == null)
                available.await();
            else {
                //获取元素的剩余时间
                long delay = first.getDelay(NANOSECONDS);
                //如果元素时间到了，那么返回
                if (delay <= 0)
                    return q.poll();
                first = null; // don't retain ref while waiting
                //如果已经有等待的线程
                if (leader != null)
                    //那么等待被唤醒
                    available.await();
                else {
                    //如果没有，那么设置当前线程为正在等待的线程
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        //等待剩余时间
                        available.awaitNanos(delay);
                    } finally {
                        //将leader置为null
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        //如果最终等待的线程不存在，队列不为空，说明没有其他线程再等待，那么通知一个后续的线程
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock();
    }
}

//移除并返回元素，响应中断，获取不到元素时阻塞超时
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            //检查头部元素
            E first = q.peek();
            if (first == null) {
                //如果头部不存在，并且已经超时了，那么返回null
                if (nanos <= 0)
                    return null;
                else
                    //否则超时阻塞
                    nanos = available.awaitNanos(nanos);
            } else {
                //获取延迟的剩余时间
                long delay = first.getDelay(NANOSECONDS);
                //时间已到返回元素
                if (delay <= 0)
                    return q.poll();
                //如果等待超时返回null
                if (nanos <= 0)
                    return null;
                //释放引用
                first = null; // don't retain ref while waiting
                //如果超时的时间小于剩余时间或者没有再等待的线程
                if (nanos < delay || leader != null)
                    //阻塞超时
                    nanos = available.awaitNanos(nanos);
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        //等待超时，唤醒后返回timeLeft，表示剩余时间
                        long timeLeft = available.awaitNanos(delay);
                        //delay - timeLeft表示阻塞已用的时间，最终计算nanos还需要等待的
                        //剩余时间
                        nanos -= delay - timeLeft;
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        //如果最终等待的线程不存在，队列不为空，说明没有其他线程再等待，那么通知一个后续的线程
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock();
    }
}

//获取头部元素
public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return q.peek();
    } finally {
        lock.unlock();
    }
}
```

