title: 【阻塞队列】-- LinkedBlockingDeque源码解析(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/cs/cs-21.jpg

toc: true 

tags:

  - 源码
  - java
  - 阻塞队列

categories: 

  - [源码,阻塞队列] 

date: 2020-03-15 15:23:21

---

## 概述

​			`LinkedBlockingDeque`与`LinkedBlockingQueue`不同，它是一个由链表组成的无界双端阻塞队列，生产消费用一把锁，头尾都可以生产消费元素，由于多了一个生产插入的入口，因此多线程情况下，入队的效率会提升一倍。<!-- more -->

## 结构

![](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/LinkedBlockingDeque.png)

- `LinkedBlockingDeque`继承`AbstractQueued`,实现了`BlockingDeque`，`BlockingDeque`还继承了`BlockingQueue`,也就是`LinkedBlockingDeque`是具备队列的所有基本特性的一个双端队列。

## 属性

```java
public class LinkedBlockingDeque<E>
    extends AbstractQueue<E>
    implements BlockingDeque<E>, java.io.Serializable {
    
    //用户储存元素的节点
	static final class Node<E> {

        E item;

        //指向上一个节点
        Node<E> prev;
		//指向下一个节点
        Node<E> next;
		
        Node(E x) {
            item = x;
        }
    }

    //头节点
    transient Node<E> first;

    //尾节点
    transient Node<E> last;

    //元素个数
    private transient int count;

    //容量
    private final int capacity;

    final ReentrantLock lock = new ReentrantLock();

    //控制消费的条件锁
    private final Condition notEmpty = lock.newCondition();
	//控制生产的条件锁
    private final Condition notFull = lock.newCondition();
}
```



## 方法

### 构造方法

```java
//初始化，默认容量为Integer.MAX_VALUE
public LinkedBlockingDeque() {
    this(Integer.MAX_VALUE);
}

//初始化，指定容量
public LinkedBlockingDeque(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
}

//通过集合初始化
public LinkedBlockingDeque(Collection<? extends E> c) {
    this(Integer.MAX_VALUE);
    final ReentrantLock lock = this.lock;
    lock.lock(); // Never contended, but necessary for visibility
    try {
        for (E e : c) {
            //传入的元素，不能为null
            if (e == null)
                throw new NullPointerException();
            //将元素插入到队尾，如果传入的集合长度超过默认的容量Integer.MAX_VALUE，抛出异常
            if (!linkLast(new Node<E>(e)))
                throw new IllegalStateException("Deque full");
        }
    } finally {
        lock.unlock();
    }
}
```

### 生产方法

```java
//添加元素到队尾，队列满的情况下抛出异常，成功添加返回true
public boolean add(E e) {
    addLast(e);
    return true;
}

//添加元素到队尾，队列满的情况下返回false，成功添加返回true
public boolean offer(E e) {
    return offerLast(e);
}

//添加元素到队尾，队列满的情况下会阻塞，响应中断
public void put(E e) throws InterruptedException {
    putLast(e);
}

//添加元素到队尾，队列满的情况下会超时阻塞，响应中断
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    return offerLast(e, timeout, unit);
}

//添加元素到队列头
public void addFirst(E e) {
    if (!offerFirst(e))
        throw new IllegalStateException("Deque full");
}

//添加元素到队列尾
public void addLast(E e) {
    if (!offerLast(e))
        throw new IllegalStateException("Deque full");
}

//通过linkFirst实现元素的插入
public boolean offerFirst(E e) {
    if (e == null) throw new NullPointerException();
    Node<E> node = new Node<E>(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return linkFirst(node);
    } finally {
        lock.unlock();
    }
}

//通过linkLast实现元素的插入
public boolean offerLast(E e) {
    if (e == null) throw new NullPointerException();
    Node<E> node = new Node<E>(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return linkLast(node);
    } finally {
        lock.unlock();
    }
}

//向队列头添加元素，响应中断
public void putFirst(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    Node<E> node = new Node<E>(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //如果向队列头添加元素失败，说明队列满了，那么等待
        while (!linkFirst(node))
            notFull.await();
    } finally {
        lock.unlock();
    }
}

//向队列尾添加元素，响应中断
public void putLast(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    Node<E> node = new Node<E>(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //如果向队列尾添加元素失败，说明队列满了，那么等待
        while (!linkLast(node))
            notFull.await();
    } finally {
        lock.unlock();
    }
}

//向队列头添加元素，响应中断，超时阻塞
public boolean offerFirst(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (e == null) throw new NullPointerException();
    Node<E> node = new Node<E>(e);
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        //如果向队列头添加元素失败，说明队列满了，那么超时等待
        while (!linkFirst(node)) {
            //已超时，返回false
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);
        }
        return true;
    } finally {
        lock.unlock();
    }
}

//向队列尾部添加元素，响应中断，超时阻塞
public boolean offerLast(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (e == null) throw new NullPointerException();
    Node<E> node = new Node<E>(e);
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
         //如果向队列尾添加元素失败，说明队列满了，那么超时等待
        while (!linkLast(node)) {
            //已超时，返回false
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);
        }
        return true;
    } finally {
        lock.unlock();
    }
}


//将元素插入到队列头
private boolean linkFirst(Node<E> node) {
    // assert lock.isHeldByCurrentThread();
    //必须要先获取锁
    //如果队列已经满了，直接返回false
    if (count >= capacity)
        return false;
    //记录头节点
    Node<E> f = first;
    //插入到头节点
    node.next = f;
    //更新头节点
    first = node;
    //如果尾节点为null,那么设置为尾节点
    if (last == null)
        last = node;
    else
        //将旧的头节点指向新的头节点node
        f.prev = node;
    //元素数量+1
    ++count;
    //唤醒一个消费的线程
    notEmpty.signal();
    return true;
}

//将元素插入到队列尾
private boolean linkLast(Node<E> node) {
    // assert lock.isHeldByCurrentThread();
    //必须要先获取锁
    //如果队列已经满了，直接返回false
    if (count >= capacity)
        return false;
    //记录尾节点
    Node<E> l = last;
    //将插入节点的prev指向尾节点
    node.prev = l;
    //将node置为新的尾节点
    last = node;
    //如果头为null,那么将node设置为新的头
    if (first == null)
        first = node;
    else
        //否则将原来的尾节点的next指向新的尾节点node
        l.next = node;
    //元素数量+1
    ++count;
    //唤醒一个消费的线程
    notEmpty.signal();
    return true;
}
```

### 消费方法

```java
//移除队头元素并返回
public E remove() {
    return removeFirst();
}

//
public E poll() {
    return pollFirst();
}

public E take() throws InterruptedException {
    return takeFirst();
}

public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    return pollFirst(timeout, unit);
}

//将头节点从链表中断开并返回，如果不存在返回null
private E unlinkFirst() {
    // assert lock.isHeldByCurrentThread();
    Node<E> f = first;
    if (f == null)
        return null;
    Node<E> n = f.next;
    E item = f.item;
    f.item = null;
    f.next = f; // help GC
    first = n;
    if (n == null)
        last = null;
    else
        n.prev = null;
    --count;
    notFull.signal();
    return item;
}

//将尾节点从链表中断开并返回，如果不存在返回null
private E unlinkLast() {
    // assert lock.isHeldByCurrentThread();
    Node<E> l = last;
    if (l == null)
        return null;
    Node<E> p = l.prev;
    E item = l.item;
    l.item = null;
    l.prev = l; // help GC
    last = p;
    if (p == null)
        first = null;
    else
        p.next = null;
    --count;
    notFull.signal();
    return item;
}

//移除头节点并返回，如果队列为空，抛出异常
public E removeFirst() {
    E x = pollFirst();
    if (x == null) throw new NoSuchElementException();
    return x;
}

//移除队尾点并返回，如果队列为空，抛出异常
public E removeLast() {
    E x = pollLast();
    if (x == null) throw new NoSuchElementException();
    return x;
}

//移除队头，通过unlinkFirst方法实现，如果队列为空，返回null
public E pollFirst() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return unlinkFirst();
    } finally {
        lock.unlock();
    }
}

//移除队尾，通过unlinkLast方法实现，如果队列为空，返回null
public E pollLast() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return unlinkLast();
    } finally {
        lock.unlock();
    }
}

//消费队头，如果队列为空，那么阻塞
public E takeFirst() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        E x;
        while ( (x = unlinkFirst()) == null)
            notEmpty.await();
        return x;
    } finally {
        lock.unlock();
    }
}

//消费队尾，如果队列为空，那么阻塞
public E takeLast() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        E x;
        while ( (x = unlinkLast()) == null)
            notEmpty.await();
        return x;
    } finally {
        lock.unlock();
    }
}

//移除队头，通过unlinkLast方法实现，如果队列为空，超时阻塞
public E pollFirst(long timeout, TimeUnit unit)
    throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        E x;
        while ( (x = unlinkFirst()) == null) {
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        return x;
    } finally {
        lock.unlock();
    }
}

//移除队尾，通过unlinkLast方法实现，如果队列为空，超时阻塞
public E pollLast(long timeout, TimeUnit unit)
    throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        E x;
        while ( (x = unlinkLast()) == null) {
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        return x;
    } finally {
        lock.unlock();
    }
}

//获取队头，如果队列为空，那么抛出异常
public E getFirst() {
    E x = peekFirst();
    if (x == null) throw new NoSuchElementException();
    return x;
}

//获取队尾，如果队列为空，那么抛出异常
public E getLast() {
    E x = peekLast();
    if (x == null) throw new NoSuchElementException();
    return x;
}

//返回队头，如果队列为空，返回null
public E peekFirst() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (first == null) ? null : first.item;
    } finally {
        lock.unlock();
    }
}

//返回队尾，如果队列为空，返回null
public E peekLast() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (last == null) ? null : last.item;
    } finally {
        lock.unlock();
    }
}
```

