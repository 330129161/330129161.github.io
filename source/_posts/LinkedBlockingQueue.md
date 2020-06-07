title: 【阻塞队列】--  LinkedBlockingQueue源码解析(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/book.png

toc: true 

tags:

  - 源码
  - java
  - 阻塞队列

categories: 

  - [源码,阻塞队列] 

date: 2020-03-14 16:45:22

---

## 概述

​		`LinkedBlockingQueue`是由链表组成的单向无界阻塞(严格意义上来并不是无界限) 队列。与`ArrayBlockingQueue` 不同的是，`LinkedBlockingQueue`生产消费各用一把锁，生产用的是putLock，消费是takeLock，目的就是为了增大吞吐量，但是因为每个节点都是一个对象，所以比较耗费内存。<!-- more -->`LinkedBlockingQueue`默认大小为Integer.MAX_VALUE，如果出现生产大于消费的情况，导致队列中存放着大量未被消费的元素，那么有可能出现OOM。

## 结构

![](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/LinkedBlockingQueue.png)

`LinkedBlockingQueue`继承了抽象队列，并且实现了阻塞队列，因此它具备队列的所有基本特性。

## 属性

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
    
    //用来存放元素的节点
    static class Node<E> {
        E item;
		//指向下一个节点
        Node<E> next;

        Node(E x) { item = x; }
    }

    //队列的容量
    private final int capacity;

    //队列的元素个数，线程安全
    private final AtomicInteger count = new AtomicInteger();

    //节点头（头节点元素值为null,不存放实际数据)
    transient Node<E> head;

    //节点尾
    private transient Node<E> last;

    //消费锁
    private final ReentrantLock takeLock = new ReentrantLock();
	//控制消费的条件锁
    private final Condition notEmpty = takeLock.newCondition();

    //生产锁
    private final ReentrantLock putLock = new ReentrantLock();
	//控制生产的条件锁
    private final Condition notFull = putLock.newCondition();
}
```

## 方法

### 构造方法

```java
//初始化，默认容量为Integer.MAX_VALUE
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}

//初始化，指定容量
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    //初始化头尾节点，内部元素为空
    last = head = new Node<E>(null);
}

//通过集合初始化
public LinkedBlockingQueue(Collection<? extends E> c) {
    this(Integer.MAX_VALUE);
    final ReentrantLock putLock = this.putLock;
    //从未竞争，但对于可见性而言却是必需的，跟ArrayBlockingQueue中一样，为什么这里要加上锁呢？
    putLock.lock(); // Never contended, but necessary for visibility
    try {
        int n = 0;
        for (E e : c) {
            if (e == null)
                throw new NullPointerException();
            if (n == capacity)
                //如果传入的集合长度超过默认的容量Integer.MAX_VALUE，抛出异常
                throw new IllegalStateException("Queue full");
            //加入到链表尾部
            enqueue(new Node<E>(e));
            ++n;
        }
        //元素数+1
        count.set(n);
    } finally {
        putLock.unlock();
    }
}
```

### 生产方法

```java
//向队列中添加元素响应中断
public void put(E e) throws InterruptedException {
    //队列不能插入元素为null
    if (e == null) throw new NullPointerException();
    int c = -1;
    //创建一个新的节点用于存储元素
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    //<1>.为啥要用AtomicInteger来记录count
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        //如果超过了容量，那么等待
        while (count.get() == capacity) {
            notFull.await();
        }
        //加入到尾部
        enqueue(node);
        //将容量数+1，但是注意 count.getAndIncrement()结果返回的是count未+1前的结果
        c = count.getAndIncrement();
        //如果链表长度没有超过限制容量，那么唤醒其中一个生产线程
        //<2>.为什么要在这里唤醒一个其他的生产线程
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    //如果c==0，代表之前队列是空的，现在有数据了，那么要释放其他等待的消费线程
    //<3>为什么容量为0的时候才唤醒消费线程，之前不是空的时候，就不能释放么
    if (c == 0)
        signalNotEmpty();
}

//向队列中添加元素响应中断，超时阻塞
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (e == null) throw new NullPointerException();
    //计算超时时间
    long nanos = unit.toNanos(timeout);
    int c = -1;
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        //如果队列满了阻塞
        while (count.get() == capacity) {
            //如果超时了直接返回false
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);
        }
        //添加元素到队列尾部
        enqueue(new Node<E>(e));
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return true;
}

////向队列中添加元素响应中断，获取锁后，如果队列满，立即返回false
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    final AtomicInteger count = this.count;
    if (count.get() == capacity)
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        //如果队列还没满
        if (count.get() < capacity) {
            enqueue(node);
            //元素数+1，返回旧的count值
            c = count.getAndIncrement();
            //如果队列至少还有一个空余，那么唤醒其中一个生产线程
            if (c + 1 < capacity)
                notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return c >= 0;
}

//释放队列不为空的信号，唤醒其中一个消费的线程
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}

//添加元素到链表尾部
private void enqueue(Node<E> node) {
    // assert putLock.isHeldByCurrentThread();
    // assert last.next == null;
    last = last.next = node;
}
```

- <1>处，为什么我们上面要用到AtomicInteger来更新count,以上面的put方法为例，由于生产消费不是使用的同一把锁，因此使用put方法生产元素的时候，其他生产的线程会被阻塞，但是消费的线程不会阻塞。将count进行自增操作时候，可能会出现不一致的情况。因此必须使用AtomicInteger来保证它的线程安全。
- <2>处，为什么要通过c + 1 < capacity这里判断，来唤醒生产线程，因为notFul这里可能阻塞着多个生产线程，而取元素的时候，只有是队列是满的时候，才会通过signalNotFull()方法唤醒一个生产线程。
- 那么，为什么队列满的时候才唤醒notFull呢，因为唤醒是需要加上putLock的，为了减少锁的次数，所以在这里索性就检测一下，未满就释放其他notFull上面的线程。
- <3>处，为什么容量为0的时候才唤醒消费线程，之前不是空的时候，就不能释放么。其实道理跟<2>处，一样都是为了减少锁的次数，才选择到临界条件时，通过获取对方锁来唤醒线程的操作。

### 消费方法

```java
//从队列中消费消息，响应中断
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        //如果队列为空，那么阻塞等待被唤醒
        while (count.get() == 0) {
            notEmpty.await();
        }
        //将头节点从队列中移除，并返回新的头节点
        x = dequeue();
        //元素数-1
        c = count.getAndDecrement();
        //如果队列中至少还有一个元素，那么唤醒别的消费线程
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    //如果元素数等于容量，也就是说之前队列是满的，现在未满，那么唤醒一个生产线程
    if (c == capacity)
        signalNotFull();
    return x;
}

//从队列中消费消息，响应中断，超时阻塞
//逻辑基本跟上面的一样
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E x = null;
    int c = -1;
    long nanos = unit.toNanos(timeout);
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) {
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}

//从队列中消费消息，如果发现队列为空，立即返回null,否则将节点从队列中移除并返回
public E poll() {
    final AtomicInteger count = this.count;
    if (count.get() == 0)
        return null;
    E x = null;
    int c = -1;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        if (count.get() > 0) {
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        }
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}

//从队列中获取最新的元素，
public E peek() {
    if (count.get() == 0)
        return null;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        Node<E> first = head.next;
        if (first == null)
            return null;
        else
            return first.item;
    } finally {
        takeLock.unlock();
    }
}

//<1>将节点从链表中移除并返回
private E dequeue() {
    // assert takeLock.isHeldByCurrentThread();
    // assert head.item == null;
    //头节点的item为null
    Node<E> h = head;
    //
    Node<E> first = h.next;
    //引用自身，帮助GC回收
    h.next = h; // help GC
    head = first;
    E x = first.item;
    first.item = null;
    return x;
}
//获取生产锁，唤醒其他生产线程
private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        notFull.signal();
    } finally {
        putLock.unlock();
    }
}
```

- <1>处。队列中头结点和尾结点一开始总是指向一个哨兵的结点，它不持有实际数据，当队列中有数据时，头结点仍然指向这个哨兵，尾结点指向有效数据的最后一个结点。这样做的好处在于，与计数器 count 结合后，对队头、队尾的访问可以独立进行，而不需要判断头结点与尾结点的关系。

