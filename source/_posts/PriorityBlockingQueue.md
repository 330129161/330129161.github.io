title: 【阻塞队列】-- PriorityBlockingQueue源码解析(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/book.png

toc: true 

tags:

  - 源码
  - java
  - 阻塞队列

categories: 

  - [源码,阻塞队列] 

date: 2020-03-15 19:09:09

---

## 概述

​		`PriorityBlockingQueue`一个无界的带有优先级的动态阻塞队列，插入队列的元素默认情况下采用自然顺序升序排列，如果用户传入了Comparator，那么使用传入的Comparator进行排序。`PriorityBlockingQueue`内部使用一把锁用于消费，当容量到达阈值时，会自动扩容，扩容时由`volatile`修饰的`allocationSpinLock`作为cas自旋锁的标识（保证扩容操作不会阻塞take操作）。

## 结构

![](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/PriorityBlockingQueue.png)

## 属性

```java
public class PriorityBlockingQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {
    private static final long serialVersionUID = 5595510919245408276L;
	
    //队列默认容量
    private static final int DEFAULT_INITIAL_CAPACITY = 11;

    //队列最大容量
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    //存储元素的数组
    private transient Object[] queue;

    //队列元素个数
    private transient int size;

    //比较器
    private transient Comparator<? super E> comparator;

    //锁用于构建消费的条件锁
    private final ReentrantLock lock;

    //用于消费的条件锁
    private final Condition notEmpty;

    //扩容时，作为cas自旋锁的标识
    private transient volatile int allocationSpinLock;

	//优先级队列
    private PriorityQueue<E> q;
}
```

## 方法

### 构造方法

```java
//无参构造方法，默认容量为DEFAULT_INITIAL_CAPACITY=11
public PriorityBlockingQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}

//带初始化容量的构造方法
public PriorityBlockingQueue(int initialCapacity) {
    this(initialCapacity, null);
}

//带初始化容量与比较器的构造方法
public PriorityBlockingQueue(int initialCapacity,
                             Comparator<? super E> comparator) {
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.lock = new ReentrantLock();
    this.notEmpty = lock.newCondition();
    this.comparator = comparator;
    this.queue = new Object[initialCapacity];
}

//带集合的构造方法
public PriorityBlockingQueue(Collection<? extends E> c) {
    this.lock = new ReentrantLock();
    this.notEmpty = lock.newCondition();
    //是否重建堆(堆它是一个数组，不过满足一个特殊的性质)
    boolean heapify = true; // true if not known to be in heap order
    //是否筛选空值
    boolean screen = true;  // true if must screen for nulls
    //如果传入的是一个有序集合，那么使用集合本身的比较器
    if (c instanceof SortedSet<?>) {
        SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
        this.comparator = (Comparator<? super E>) ss.comparator();
        heapify = false;
    }
    //如果传入集合是PriorityBlockingQueue类型，则不进行堆有序化
    else if (c instanceof PriorityBlockingQueue<?>) {
        PriorityBlockingQueue<? extends E> pq =
            (PriorityBlockingQueue<? extends E>) c;
        //使用阻塞队列的比较器
        this.comparator = (Comparator<? super E>) pq.comparator();
        screen = false;
        if (pq.getClass() == PriorityBlockingQueue.class) // exact match
            heapify = false;
    }
    Object[] a = c.toArray();
    //记录数组长度
    int n = a.length;
    // If c.toArray incorrectly doesn't return Object[], copy it.
    //<1>.如果c.toArray不正确地返回Object[]，请复制它
    if (a.getClass() != Object[].class)
        a = Arrays.copyOf(a, n, Object[].class);
    //如果传入的集合类型不为有序集合或者PriorityBlockingQueue，那么要校验是否传入null值
    //如果传入的集合中存在null，那么抛出NullPointerException
    // (n == 1 || this.comparator != null) 有两种情况
    //1.
    if (screen && (n == 1 || this.comparator != null)) {
        for (int i = 0; i < n; ++i)
            if (a[i] == null)
                throw new NullPointerException();
    }
    //讲a赋值给队列内部储存元素的数组
    this.queue = a;
    //设置size为n
    this.size = n;
    if (heapify)
        //是否堆化
        heapify();
}
```

### heapify方法

`PriorityBlockingQueue`内部采用二叉堆来实现，属于一个最小堆。

> 二叉堆是一种特殊的堆，二叉堆是完全二元树（二叉树）或者是近似完全二元树（二叉树）。二叉堆有两种：[最大堆](https://baike.baidu.com/item/最大堆/4633143)和[最小堆](https://baike.baidu.com/item/最小堆/9139372)。最大堆：[父结点](https://baike.baidu.com/item/父结点/9796346)的键值总是大于或等于任何一个子[节点](https://baike.baidu.com/item/节点/865052)的键值；最小堆：父结点的键值总是小于或等于任何一个子节点的键值。
>
> 二叉堆一般用[数组](https://baike.baidu.com/item/数组/3794097)来表示。如果根节点在数组中的位置是1，第*n*个位置的子节点分别在2*n*和 2*n*+1。因此，第1个位置的子节点在2和3，第2个位置的子节点在4和5。以此类推。这种基于1的数组存储方式便于寻找父节点和子节点。

与二叉树不同的是二叉堆的数据是顺序存储，而不是链式存储

```java
private void heapify() {
    Object[] array = queue;
    int n = size;
    //找到最后一个非叶子节点，也就是最后一个节点的父节点
    int half = (n >>> 1) - 1;
    Comparator<? super E> cmp = comparator;
    //如果比较器为null,那么使用自然排序
    if (cmp == null) {
        //从half开始向前数，对所有非叶子节点进行下沉操作
        for (int i = half; i >= 0; i--)
            siftDownComparable(i, (E) array[i], array, n);
    }
    else {
        for (int i = half; i >= 0; i--)
            siftDownUsingComparator(i, (E) array[i], array, n, cmp);
    }
}

//k：表示当前要操作的非叶子节点的坐标 5 
//x:表示当前要操作的非叶子节点的元素 6 
//array:表示当前队列的数组
//n：表示当前队列的元素个数 12
private static <T> void siftDownComparable(int k, T x, Object[] array,
                                               int n) {
    //如果传入的集合为空，那么就不需要进行堆化操作
    if (n > 0) {
        Comparable<? super T> key = (Comparable<? super T>)x;
        //half为当前元素个数/2
        int half = n >>> 1;           // loop while a non-leaf 6
        //k如果小于父节点的情况下才有效
        while (k < half) {
            int child = (k << 1) + 1; // assume left child is least
            Object c = array[child];
            int right = child + 1;
            if (right < n &&
                ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                c = array[child = right];
            if (key.compareTo((T) c) <= 0)
                break;
            array[k] = c;
            k = child;
        }
        array[k] = key;
    }
}

private static <T> void siftDownUsingComparator(int k, T x, Object[] array,
                                                    int n,
                                                    Comparator<? super T> cmp) {
    if (n > 0) {
        int half = n >>> 1;
        while (k < half) {
            int child = (k << 1) + 1;
            Object c = array[child];
            int right = child + 1;
            if (right < n && cmp.compare((T) c, (T) array[right]) > 0)
                c = array[child = right];
            if (cmp.compare(x, (T) c) <= 0)
                break;
            array[k] = c;
            k = child;
        }
        array[k] = x;
    }
}
```

### 生产方法

```java
//添加并返回元素到队列头，由offer方法实现，成功后返回true
public boolean add(E e) {
    return offer(e);
}
//添加元素到队列头
public boolean offer(E e) {
    //队列中不允许存储空值
    if (e == null)
        throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    lock.lock();
    int n, cap;
    Object[] array;
    //如果当前元素的个数已经>=当前容量，那么就需要扩容
    while ((n = size) >= (cap = (array = queue).length))
        //扩容方法
        tryGrow(array, cap);
    try {
        Comparator<? super E> cmp = comparator;
        //如果对比器不存在，那么使用自然排序
        if (cmp == null)
            siftUpComparable(n, e, array);
        else
            siftUpUsingComparator(n, e, array, cmp);
        //元素数+1
        size = n + 1;
        //唤醒一个消费的线程
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
    return true;
}

//添加元素到队列头，由offer方法实现，成功后返回true
//生产方法永远不会被阻塞
public void put(E e) {
    offer(e); // never need to block
}

//添加元素到队列头，由offer方法实现，成功后返回true
//生产方法永远不会被阻塞，timeout和unit参数在本队列不生效，因为生产是不会阻塞的
public boolean offer(E e, long timeout, TimeUnit unit) {
    return offer(e); // never need to block
}
```

### tryGrow方法

```java
private void tryGrow(Object[] array, int oldCap) {
    //释放锁，保证扩容操作不会阻塞消费
    lock.unlock(); // must release and then re-acquire main lock
    Object[] newArray = null;
    //释放锁后，扩容会出现竞争
    //allocationSpinLock == 0表示当前没有正在扩容的线程
    //通过cas修改allocationSpinLockOffset为1，表示开始扩容
    if (allocationSpinLock == 0 &&
        UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                 0, 1)) {
        try {
            //如果旧的容量<64,那么扩容为原来的2倍+2(容量越小，则增长的越快)，如果原来的容量
            //大于或者等于64，那么扩容为原来的1.5倍
            int newCap = oldCap + ((oldCap < 64) ?
                                   (oldCap + 2) : // grow faster if small
                                   (oldCap >> 1));
            //如果扩容后的容量>最大容量
            if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                int minCap = oldCap + 1;
                //minCap < 0，说明minCap整数已经溢出了
                //minCap > MAX_ARRAY_SIZE说明已经超出最大限制了
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                    //抛出异常
                    throw new OutOfMemoryError();
                //设置新容量为最大容量MAX_ARRAY_SIZE
                newCap = MAX_ARRAY_SIZE;
            }
            //newCap > oldCap说明扩容的容量还未超过最大容量，
            //queue == array说明是第一次扩容
            //只有满足这个两个条件，才有必要扩容创建新的数组
            if (newCap > oldCap && queue == array)
                newArray = new Object[newCap];
        } finally {
            //重置 allocationSpinLock = 0，表示扩容结束
            allocationSpinLock = 0;
        }
    }
    //newArray == null表示扩容操作被其他线程抢先执行了
    if (newArray == null) // back off if another thread is allocating
        //让出cpu资源
        Thread.yield();
    //获取锁
    lock.lock();
    if (newArray != null && queue == array) {
        //如果newArray != null && queue == array，说明queue没有发生变化那么讲新的数组指向queue
        queue = newArray;
        //讲原来的array中的值，复制到新的数组中
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}
```

### 消费方法

```java
//移除队头元素并返回，如果队列为空返回null
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return dequeue();
    } finally {
        lock.unlock();
    }
}

//移除队头元素并返回，如果队列为空阻塞，响应中断
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {
        while ( (result = dequeue()) == null)
            notEmpty.await();
    } finally {
        lock.unlock();
    }
    return result;
}

//移除队头元素并返回，如果队列为空超时阻塞，响应中断
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {
        while ( (result = dequeue()) == null && nanos > 0)
            nanos = notEmpty.awaitNanos(nanos);
    } finally {
        lock.unlock();
    }
    return result;
}

//获取队头元素，如果队列为空返回null
public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (size == 0) ? null : (E) queue[0];
    } finally {
        lock.unlock();
    }
}

//元素出队并返回
private E dequeue() {
    int n = size - 1;
    if (n < 0)
        return null;
    else {
        Object[] array = queue;
        E result = (E) array[0];
        E x = (E) array[n];
        array[n] = null;
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            siftDownComparable(0, x, array, n);
        else
            siftDownUsingComparator(0, x, array, n, cmp);
        size = n;
        return result;
    }
}
```

