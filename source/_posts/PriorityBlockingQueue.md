title: 【阻塞队列】-- PriorityBlockingQueue源码解析(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/cs/cs-24.jpg

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

​		`PriorityBlockingQueue`一个无界的带有优先级的动态阻塞队列，插入队列的元素默认情况下采用自然顺序升序排列，如果用户传入了Comparator，那么使用传入的Comparator进行排序。`PriorityBlockingQueue`内部使用一把锁用于消费，当容量到达阈值时，会自动扩容，扩容时由`volatile`修饰的`allocationSpinLock`作为cas自旋锁的标识（保证扩容操作不会阻塞take操作）。<!--more -->

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
    //是否需要重建堆(堆它是一个数组，不过满足一个特殊的性质)
    boolean heapify = true; // true if not known to be in heap order
    //是否需要筛选空值
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

`PriorityBlockingQueue`内部采用二叉堆来实现。二叉堆是一种特殊的堆，近似完全二叉树。二叉堆有两种堆：最大堆和最小堆。最大堆：父节点的键值总是大于或者等于任何一个子节点的键值；最小堆：父节点的键值总是小于或等于任何一个子节点的键值。而我们的`PriorityBlockingQueue`就是采用的最小堆。

二叉堆一般使用数组来表示，不过他是顺序结构存储而不是链式结构。看图：

![](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/%E4%BA%8C%E5%8F%89%E5%A0%86.jpg)

**二叉堆来实现优先队列的特点**：

1. 一般通过数组存储的逻辑二叉树结构
2. 要么为最大堆要么为最小堆
5. 父节点索引位 = （当前节点索引位 - 1 ）/ 1
6.  左子节点索引位 =  当前节点索引位 * 2 + 1
7. 右子节点索引位 =  当前节点索引位 * 2 + 2
8. 最后一个非叶子节点索引位 = 节点数/2  - 1；

**二叉堆的三个操作：**

1. 堆化：将一个完全无序的数组转化为一个符合二叉堆结构的数组
2. 元素插入：为了不破坏现有的二叉堆结构，插入都是从最后一个元素开始，然后对该节点进行上浮操作
3. 元素移除：删除节点时从堆顶删除，将最后一个节点置换到堆顶，然后对堆顶的元素进行下沉操作

通过上面的这些特点，以及最后的四个公式，再来看接下来的代码就要轻松许多。

```java
//heapify就是上面三个操作中的堆化操作
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
```

### 节点上浮操作

```java
//对节点进行上浮操作，将新插入到末尾的节点进行上浮操作
private static <T> void siftUpComparable(int k, T x, Object[] array) {
    Comparable<? super T> key = (Comparable<? super T>) x;
    //k>0说明还有非子叶节点存在
    while (k > 0) {
        //获取父节点的下标
        int parent = (k - 1) >>> 1;
        //记录父节点的值
        Object e = array[parent];
        //经过对比后，如果该节点的优先级已经大于父节点，那么不用进行升序了，break跳出
        if (key.compareTo((T) e) >= 0)
            break;
        //当前下表位置的节点指向父节点
        array[k] = e;
        //k置为parent，接下来通过while会继续向上找，如果此时已经到最顶端了那么
        //k就为0了，就会跳出结束while
        k = parent;
    }
    //交换最终的节点
    array[k] = key;
}

//跟siftUpComparable逻辑一致,不同点就是使用的是用户指定的Comparator比较器
private static <T> void siftUpUsingComparator(int k, T x, Object[] array,
                                              Comparator<? super T> cmp) {
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = array[parent];
        if (cmp.compare(x, (T) e) >= 0)
            break;
        array[k] = e;
        k = parent;
    }
    array[k] = x;
}
```

### 节点下沉操作

```java
//下沉操作，从队列中移除堆顶元素，把最后一个叶子节点替换到头部时，需要进行下沉操作
private static <T> void siftDownComparable(int k, T x, Object[] array,
                                               int n) {
    //如果传入的集合为空，那么就不需要下沉操作了
    if (n > 0) {
        Comparable<? super T> key = (Comparable<? super T>)x;
        //记录二叉树上最后一个非叶子节点的下标，如果当前操作的坐标小于half
        //说明k>half说明k现在记录的位置是叶子节点的位置，不用再下沉了
        int half = n >>> 1;           // loop while a non-leaf
        //k小于half说明可能还需要进行下沉操作
        while (k < half) {
            //获取左子节点下标
            int child = (k << 1) + 1; // assume left child is least
            //记录右子节点
            Object c = array[child]; 
            //获取右子节点下标
            int right = child + 1;
            //如果左边的子节点大于右边的子节点，那么就要用右边的右子节点来交换当前节点
            //（目的就是为了取一个最小值来交换）
            if (right < n &&
                ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                c = array[child = right];
            //如果当前节点的值已经小于最小的子节点，那么就不用再进行交换了
            if (key.compareTo((T) c) <= 0)
                break;
            //讲记录的最小的子节点来跟当前节点交换
            array[k] = c;
            //将k指向较小的子节点，以便于下次while循环
            //因为k跟子元素交换之后，子元素可能还有子元素，所以通过k来指向child
            //以便下次继续进行子节点的交换动作
            k = child;
        }
        //交换最终的节点
        array[k] = key;
    }
}

//逻辑跟上面的一样
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
            //通过上浮，调整堆结构
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
        //出队
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
        //出队，如果元素为空那么阻塞等待
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
        //记录头节点
        E result = (E) array[0];
        //记录尾节点
        E x = (E) array[n];
        //将尾节点位置置为null
        array[n] = null;
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            //将原本尾节点x从堆顶进行下沉，调整堆结构，此时原本的头节点result会被替换掉
            siftDownComparable(0, x, array, n);
        else
            siftDownUsingComparator(0, x, array, n, cmp);
        size = n;
        //返回被移除的头节点，也就是堆顶
        return result;
    }
}

//一次从队列中获取所有元素到指定集合中
public int drainTo(Collection<? super E> c) {
    return drainTo(c, Integer.MAX_VALUE);
}

//一次性获取指定元素个数到指定集合中，该方法不会阻塞
public int drainTo(Collection<? super E> c, int maxElements) {
    if (c == null)
        throw new NullPointerException();
    if (c == this)
        throw new IllegalArgumentException();
    if (maxElements <= 0)
        return 0;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //取小值，也就是如果传入的个数超过了队列中已存在的个数，那么以已有的数据为准
        int n = Math.min(size, maxElements);
        for (int i = 0; i < n; i++) {
            //将队头的元素添加到集合中
            c.add((E) queue[0]); // In this order, in case add() throws.
            //将队头元素从集合中移除，那么下一次上一步的操作也就是添加新的队头
            dequeue();
        }
        return n;
    } finally {
        lock.unlock();
    }
}
```

