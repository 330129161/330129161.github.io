title: 【阻塞队列】-- ArrayBlockingQueue源码解析(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/book.png

toc: true 

tags:

  - 源码
  - java
  - 阻塞队列

categories: 

  - [源码,阻塞队列] 

date: 2020-03-13 20:09:32

---

## 概述

​			`ArrayBlockingQueue`是由数组组成的一个单向有界阻塞队列。`ArrayBlockingQueue`内部只有一把锁`ReentrantLock`，通过`ReentrantLock`的`Condition`来控制内部的生产与消费。`ArrayBlockingQueue`创建时必须指定容量，当队列满后会阻塞生产的线程，队列空时会阻塞消费的线程。<!-- more -->

## 结构特点

![](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/ArrayBlockingQueue.png)

1. `ArrayBlockingQueue`继承了抽象队列，并且实现了阻塞队列，因此它具备队列的所有基本特性。

## 重要属性

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

	//用于存储内部的元素
    final Object[] items;

	//下一个消费坐标
    int takeIndex;
    
	//下一个生产坐标
    int putIndex;
    
	//当前队列中的元素数
    int count;
    
	//重入锁
    final ReentrantLock lock;
    
	//队列为空的时候，会阻塞消费的线程
    private final Condition notEmpty;
    
	//队列已满的时候，会阻塞消费的线程
    private final Condition notFull;

	//内部实现的迭代器，用于迭代items数组
    transient Itrs itrs = null;
}
```

## 常用方法

### 构造方法

```java
//带容量的构造方法，默认使用非公平锁
public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
}

//可以指定容量与是否公平锁的构造方法
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}

//可以指定容量与是否公平锁的构造方法，构造后默认向队列中填充元素
public ArrayBlockingQueue(int capacity, boolean fair,
                          Collection<? extends E> c) {
    this(capacity, fair);

    final ReentrantLock lock = this.lock;
    lock.lock(); <1>.// Lock only for visibility, not mutual exclusion
    try {
        int i = 0;
        try {
            //遍历集合插入到队列中
            for (E e : c) {
                checkNotNull(e);
                items[i++] = e;
            }
        } catch (ArrayIndexOutOfBoundsException ex) {
            //如果传入的集合长度大于我们设置的容量值，那么会抛出异常
            throw new IllegalArgumentException();
        }
        //记录当前队列中元素个数
        count = i;
        //如果队列已经满了，那么下一个生产的数据就存放在items数组的0下标处，也就是从头开始
        //如果没满，那么i就是下一次生产数据插入的位置
        putIndex = (i == capacity) ? 0 : i;
    } finally {
        lock.unlock();
    }
}
```

- 上面第三个构造方法中，<1>处，lock.lock()方法处加了一行注释`Lock only for visibility, not mutual exclusion`，这句话的意思是说，这个锁的操作并不是为了互斥操作，而是保证其可见性。当我们把集合中的数据全部插入队列中之后，我们会修改相应的count以及putIndex的数值，但是如果我们没有加锁，那么在集合插入完成前count以及putIndex没有完成初始化操作的时候如果有其他线程进行了插入等操作的话，会造成数据同步问题从而使得数据不准确。

### 生产方法

```java
//向队列中添加元素，如果队列已满直接抛出异常
public boolean add(E e) {
    //调用的父类AbstractQueue.class的add方法
    return super.add(e);
}

//父类AbstractQueue.class的add方法
public boolean add(E e) {
    //会间接调用offer方法
    if (offer(e))
        return true;
    else
        //如果失败，那么抛出队列已满异常
        throw new IllegalStateException("Queue full");
}

//向队列中添加元素，获取锁后立即得到插入结果，不会阻塞
public boolean offer(E e) {
    //队列中不允许添加null元素
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //如果容量满了，立即返回false
        if (count == items.length)
            return false;
        else {
            //向队列添加元素
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}

//向队列中添加元素，队列满后会阻塞，响应中断
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    //上锁，响应中断
    lock.lockInterruptibly();
    try {
        //如果队列满了，立刻阻塞，等待被释放
        while (count == items.length)
            notFull.await();
        //释放后，添加元素到队列中
        enqueue(e);
    } finally {
        lock.unlock();
    }
}

//向队列中添加元素，队列满后超时阻塞，响应中断
public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

     checkNotNull(e);
    //计算超时时间
     long nanos = unit.toNanos(timeout);
     final ReentrantLock lock = this.lock;
     lock.lockInterruptibly();
     try {
         while (count == items.length) {
             if (nanos <= 0)
                 //超时直接返回false
                 return false;
             //如果队列满了，超时阻塞
             nanos = notFull.awaitNanos(nanos);
         }
         enqueue(e);
         return true;
     } finally {
         lock.unlock();
     }
 }

//检查插入的元素是否为null
private static void checkNotNull(Object v) {
    if (v == null)
        throw new NullPointerException();
}

//入队
private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    //将元素放入当前putIndex下标
    items[putIndex] = x;
    //++putIndex记录下一次下表位置，如果到队尾了，那么重置下标为0
    if (++putIndex == items.length)
        putIndex = 0;
    //元素数+1
    count++;
    //有元素入队之后，释放队列不为空信号
    notEmpty.signal();
}
```

### 消费方法

```java
//消费数据，获取锁后立即返回状态
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //如果队列为空，返回null
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}

//消费数据，如果队列为空会阻塞
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            //如果队列为空，直到队列有元素后，被唤醒
            notEmpty.await();
        //出队
        return dequeue();
    } finally {
        lock.unlock();
    }
}

//消费数据，如果队列为空会超时阻塞
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0) {
            //超时后，直接返回null
            if (nanos <= 0)
                return null;
            //等待超时
            nanos = notEmpty.awaitNanos(nanos);
        }
        return dequeue();
    } finally {
        lock.unlock();
    }
}

//获取锁定后立刻返回元素，当队列为空时，返回null
public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return itemAt(takeIndex); // null when queue is empty
    } finally {
        lock.unlock();
    }
}

//出队
private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    //记录要出对的元素
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    //计算下一次消费的坐标，如果到队列的尾部了，那么从头开始
    if (++takeIndex == items.length)
        takeIndex = 0;
    //元素数-1
    count--;
    //判断当前迭代器是否为空，如果不为空，那么更新迭代器数据
    if (itrs != null)
        itrs.elementDequeued();
    //有元素入队之后，释放队列未满信号
    notFull.signal();
    return x;
}

void unlink(Node<E> x) {
    // assert lock.isHeldByCurrentThread();
    Node<E> p = x.prev;
    Node<E> n = x.next;
    if (p == null) {
        unlinkFirst();
    } else if (n == null) {
        unlinkLast();
    } else {
        p.next = n;
        n.prev = p;
        x.item = null;
        // Don't mess with x's links.  They may still be in use by
        // an iterator.
        --count;
        notFull.signal();
    }
}
```

阻塞队列全部讲完之后，会专门出一章讲阻塞队列中的迭代器。

## 总结

​		以上就是`ArrayBlockingQueue`解析的全部内容了。ArrayBlockingQueue通过一把重入锁创建两条等待队列，分别用于生产与消费的情况，通过putIndex与takeIndex用于记录下一个元素入队与出队处的下标。