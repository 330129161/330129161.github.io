title: ConcurrentHashMap源码解析(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/book.png

toc: true 

tags:

  - 源码
  - java
  - 集合
  - 并发

categories: 

  - [源码,集合] 

date: 2019-12-25 20:41:00

---

## 概述

`ConcurrentHashMap`是`HashMap`线程安全的版本。jdk1.8以前`ConcurrentHashMap`采用`Segment`分段锁技术，`Segment`继承自`ReentrantLock`，通过对`Segment`加锁实现并发操作。<!--more--> 而jdk1.8中抛弃了分段锁，使用`CAS+synchronized`,通过锁定桶中的头节点，使用cas自旋的方式，进行并发操作。好处在于，通过锁定node节点减少锁的粒度，提高并发效率。相较于`HashTable`中，锁定整个`HashTable`对象的方式，`ConcurrentHash`的优势就很明显了。也正是由于降低了锁的粒度，使得代码的实现变得更加的复杂。

## 结构特点

1. 继承自`AbstractMap`,实现了`ConcurrentMap`接口
2. 实现了 `Serializable` 接口， 表示 `ConcurrentHash`支持序列化的功能 ，可用于网络传输。

![ConcurrentHashMap结构图](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/ConcurrentHashMap.png)

## 重要属性

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {
    //最大容量
    private static final int MAXIMUM_CAPACITY = 1 << 30;
	//默认容量
    private static final int DEFAULT_CAPACITY = 16;
	//转化为数组允许的最大容量
    static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
	//并发级别，遗留下来的，为兼容以前的版本
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
    //默认负载因子
    private static final float LOAD_FACTOR = 0.75f;
    //链表转红黑树长度阈值，当链表长度>8时，会转为红黑树
    static final int TREEIFY_THRESHOLD = 8;
	//红黑树还原链表数量阈值，当红黑树数量<6,会转为链表
    static final int UNTREEIFY_THRESHOLD = 6;
    //链表转红黑树长度容量阈值，当哈希表中容量>64,才会将链表转红黑树长度，
    //否则直接扩容而不是转化为红黑树，为了避免扩容与转化后红黑树之间的冲突，这个值不能小于64.
    static final int MIN_TREEIFY_CAPACITY = 64;
	//单个线程最小处理的桶数，扩容时，通过cpu处理器数量，来为每个处理器平均分配需要迁移的桶数
    //如果桶数量较小，分配个桶个数<16时，默认每个线程处理16个桶，单核默认处理16个桶
    private static final int MIN_TRANSFER_STRIDE = 16;
	//表示扩容标记
    private static int RESIZE_STAMP_BITS = 16;
    // 2^15-1，帮助扩容的最大线程数
    private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
	//并行扩容线程数
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
    //forwardingNode的hash值,占位符表示正在转移
    static final int MOVED = -1;
    //TreeBin的hash值，当为红黑树节点时，进入特殊处理
    static final int TREEBIN = -2; 
    //ReservationNode的hash值，占位符
    static final int RESERVED = -3; 
    //0x7FFFFFFF是一个用16进制表示的整型,是整型里面的最大值，转化为二进制为 
    // 0111 1111 1111 1111 1111 1111 1111 1111，第一位表示正负符号，0表示正，1表示负
    //在此类中参与hash计算时，可以保证hash为正数
    static final int HASH_BITS = 0x7fffffff; 
    //可用的处理器的数量
    static final int NCPU = Runtime.getRuntime().availableProcessors();
    //存放数据的table
    transient volatile Node<K,V>[] table;
	// 扩容后的新的table数组，为原table的2倍，只有在扩容时才有用
    private transient volatile Node<K,V>[] nextTable;
	//ConcurrentHashMap中元素个数，但不一定是当前真实的元素个数，基于cas无锁定更新
    private transient volatile long baseCount;

	 /*
	 * sizeCtl是一个用于同步多个线程的共享变量，用来控制table的初始化和扩容的操作，不同的值有不同的含义
	 * sizeCtl=0：表示没有指定初始容量。
	 * sizeCtl=-1,标记作用，告知其他线程，正在初始化。-N代表有N-1个线程正在 进行扩容
	 * sizeCtl>0：如果table没有被初始化，表示接下来的真正的初始化操作中使用的容量，
	 * table初始化之后，sizeCtl为扩容阈值。
	 */
    private transient volatile int sizeCtl;

	//表示迁移时的下标，初始为扩容前的table的长度
    private transient volatile int transferIndex;

	//通过cas实现的锁，用于counterCells计数时的锁定操作，0无锁，1获得锁
    private transient volatile int cellsBusy;

	//计数器
    private transient volatile CounterCell[] counterCells;

 
    private transient KeySetView<K,V> keySet;
    private transient ValuesView<K,V> values;
    private transient EntrySetView<K,V> entrySet;
    
    
}
```

## node节点

```java
//ConcurrentHashMap内部node节点，用于储存桶内数据
static class Node<K,V> implements Map.Entry<K,V> {
    //hash值
    final int hash;
    //key值
    final K key;
    //使用get获取val不需要加锁，是因为val通过volatile修饰，可以保证可见性
    volatile V val; 
    //后驱节点，通过volatile修饰，可以保证数组扩容时的可见性
    volatile Node<K,V> next;

    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }

    public final K getKey()       { return key; }
    public final V getValue()     { return val; }
    public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
    public final String toString(){ return key + "=" + val; }
    //不允许更新value
    public final V setValue(V value) {
        throw new UnsupportedOperationException();
    }
    //判断是否同一节点
    public final boolean equals(Object o) {
        Object k, v, u; Map.Entry<?,?> e;
        return ((o instanceof Map.Entry) &&
                (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                (v = e.getValue()) != null &&
                (k == key || k.equals(key)) &&
                (v == (u = val) || v.equals(u)));
    }
    //通过hash与key值，在当前链表中找到对应节点
    Node<K,V> find(int h, Object k) {
        Node<K,V> e = this;
        if (k != null) {
            do {
                K ek;
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
            } while ((e = e.next) != null);
        }
        return null;
    }
}
```

## ForwardingNode节点

扩容时存放的节点类型，代表此处已完成扩容。它包含一个`nextTable`指针，用于指向下一个桶。 

```java
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        //hash值默认为-1，key, value, next都为null
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }

    //从nextTable中查询节点
    Node<K,V> find(int h, Object k) {
        // loop to avoid arbitrarily deep recursion on forwarding nodes
        outer: for (Node<K,V>[] tab = nextTable;;) {
            Node<K,V> e; int n;
            //如果在当前桶中找不到元素，返回null
            if (k == null || tab == null || (n = tab.length) == 0 ||
                (e = tabAt(tab, (n - 1) & h)) == null)
                return null;
            for (;;) {
                int eh; K ek;
                //找到元素，直接返回
                if ((eh = e.hash) == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                //如果当前元素的hash值<0
                if (eh < 0) {
                    //如果当前为ForwardingNode节点
                    if (e instanceof ForwardingNode) {
                        //将tab指向nextTable，结束自此循环，去nextTable中查找节点（当前要查找的节点已经迁移到nextTable中了）
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        continue outer;
                    }
                    else
                        //去树节点中查找
                        return e.find(h, k);
                }
                //如果到尾部，还找不到匹配的元素，直接返回null
                if ((e = e.next) == null)
                    return null;
            }
        }
    }
}
```



## 常用方法

### 构造方法

```java
//无参构造方法
public ConcurrentHashMap() {
}	
//指定容量大小
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    //sizeCtl为初始容量
    this.sizeCtl = cap;
}
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    //sizeCtl为默认初始容量
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}
public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
    //校验参数合法性
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    //初始容量最小为1
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    //通过传入的长度/加载因子，可以计算一个>=阈值的数，保证本次不会触发扩容
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    //计算初始容量
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

### tableSizeFor方法

```java
/*
 * 计算扩容时的阈值，通过位运算，来计算最接近且大于当前输入值的2的幂次方数
 * 例如传入的初始容量initialCapacity=7时，返回 8
 * initialCapacity = 9，返回 16
 */
static final int tableSizeFor(int cap) {
    //假如传入的cap为61，那么 n = 60
    int n = cap - 1;
    // n |= n >>> 1 相当于n = n | n >>> 1,向右边移动1位 
    // 0111100移动一位后，n = 0111100 | 011110, n = 0111110
    n |= n >>> 1;
    //0111110移动两位后，n = 0111110 || 0011110，n = 0111111
    //此时所有的低位都为1，无论怎么位移这个数通过|运算后这个值，都不会再改变了
    //此时111111转化为十进制为63
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    //返回n + 1 = 64
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

上面简单的说明了，该方法是如何通过|运算与位移,来找到最接近输入值的2的幂次方数。为什么最后要一直写到 `n |= n >>> 16`,int的范围在`-2^31 ~ 2^31-1`,因此最大2次幂数为`2^30`,也就是当前容量默认的最大值`MAXIMUM_CAPACITY`，代码`1 + 2 + 4 + 8 + 16 = 31`一共向右移了31位，是为了保证高位1以下的低位都会变为1。

### spread方法

```java
static final int spread(int h) {
    //右移16位与原hashcode进行^操作，保证在不破坏hashcode的特性下，让高低位都参与计算
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

- `spread()`同`HashMap`中`hash()`方法，使用这种方式主要是为了减少hash冲突。([HashMap源码解析](https://www.yingu.site/2019/12/25/HashMap/))

### put系列方法

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

 /*
 * onlyIfAbsent：当前onlyIfAbsent为true时，不会改变链表中存在的value
 */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    //获取key的hash值
    int hash = spread(key.hashCode());
    //链表长度
    int binCount = 0;
    //死循环table（并发重试补偿，因为节点可能因为存在并发，而操作失败）
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //如果table未初始化
        if (tab == null || (n = tab.length) == 0)
            //初始化table
            tab = initTable();
        //如果当前位置没有元素
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //新建node节点，通过cas方法设置到当前位置，可能失败（存在并发）
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                //如果设置成功，跳出死循环
                break; // no lock when adding to empty bin
        }
        //如果f.hash为-1，说明当前f是ForwardingNode节点，正在被其他线程扩容，帮助其扩容
        else if ((fh = f.hash) == MOVED)
            //帮助扩容
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            //通过synchronized锁定当前位置头节点
            synchronized (f) {
                //如果当前位置的头元素为当前锁定元素（二次验证，因为可能被其他线程改变）
                if (tabAt(tab, i) == f) {
                    //如果当前为链表
                    if (fh >= 0) {
                        //记录binCount链表长度
                        binCount = 1;
                        //遍历此条链表，binCount每次+1
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            //如果新增的key值，在链表中存在
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                //根据是否置换策略，选择是否更新value值
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            //记录上次遍历的节点
                            Node<K,V> pred = e;
                            //如果==null,说明已经遍历到链尾了
                            if ((e = e.next) == null) {
                                //替换当前新增节点为尾节点
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    //如果当前节点为红黑树
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        //像树中添加元素
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                              value)) != null) {
                            oldVal = p.val;
                            //根据是否置换策略，选择是否更新value值
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            //如果binCount不为0，说明操作过链表
            if (binCount != 0) {
                //如果链表长度大于默认的阈值
                if (binCount >= TREEIFY_THRESHOLD)
                    //链表转红黑树
                    treeifyBin(tab, i);
                if (oldVal != null)
                    //如果oldVal != null，说明当前key值在链表中已存在，
                    //那么直接返回旧值（当然，也不做用做后续长度+1的操作了）
                    return oldVal;
                break;
            }
        }
    }
    //当前map长度+1
    addCount(1L, binCount);
    return null;
}

public void putAll(Map<? extends K, ? extends V> m) {
    //尝试扩容
    tryPresize(m.size());
    for (Map.Entry<? extends K, ? extends V> e : m.entrySet())
        //遍历添加到map中
        putVal(e.getKey(), e.getValue(), false);
}

```
### initTable方法

```java
//初始化table
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    //
    while ((tab = table) == null || tab.length == 0) {
        //说明当前tab正在被其他线程初始化
        if ((sc = sizeCtl) < 0)
            //退出线程竞争，避免当前线程一直占用cpu资源(初始化操作，只能由一个线程进行)
            Thread.yield(); // lost initialization race; just spin
        //将SIZECTL设置为-1，表示table正在被初始化
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                //二次验证
                if ((tab = table) == null || tab.length == 0) {
                    //sc大于0的情况下，表示扩容量，如果没有指定容量大小，那么为默认容量
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    //记录扩容阈值
                    sc = n - (n >>> 2);
                }
            } finally {
                //如果当前线程初始化成功，设置扩容阈值，失败将sizeCtl还原成初始状态，这样其他线程有机会进行初始化操作，只有某个线程完成初始化操作后，其他线程才会退出while循环
                sizeCtl = sc;
            }
            break;
        }
    }
    //返回新table
    return tab;
}
```

### tryPresize方法

```java
//尝试扩容
private final void tryPresize(int size) {
    //如果超过了容量的一半，那么直接设置为最大容量，否则选择大于且最接近当前值的2的幂次方
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
    tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    //确保没有其他线程在进行扩容操作的时候执行
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        //如果table没有被初始化
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            //cas修改sizeCtl为-1，表示table正在进行扩容
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    ///确认其他线程没有对table修改
                    if (table == tab) {
                  		//初始化table
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        //sc = 0.75*n,负载因子在ConcurrentHashMap中默认为0.75
                        sc = n - (n >>> 2);
                    }
                } finally {
                    //设置容量为当前扩容值
                    sizeCtl = sc;
                }
            }
        }
        //如果扩容大小没有达到阈值，或者超过最大容量
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        //tab == table说明还没开始迁移节点
        else if (tab == table) {
            //通过容量n，计算rs。通过对rs的移位处理，可以求出新的sizeCtl值
            int rs = resizeStamp(n);
            //sc<0,说明正在被其他线程扩容
            if (sc < 0) {
                Node<K,V>[] nt;
                //如果sc前 16 位如果不等于标识符，则表示标识符已经被改变
            	//第一个线程扩容时，设置(rs << RESIZE_STAMP_SHIFT) + 2，如果sc == rs + 1，说明第一个			//线程已经完成扩容了（完成的时候会-1）
            	//如果达到了最大帮助线程的数量，那么当前不参与扩容
            	//
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                //帮助扩容，扩容线程数+1
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            //如果当前线程为扩容的第一个线程，设置(rs << RESIZE_STAMP_SHIFT) + 2)
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```

**<1>处**，如果当前线程是扩容的第一个线程，那么会将`sizeCtl`以`CAS`的方式设置为`(rs << RESIZE_STAMP_SHIFT) + 2`，通过`resizeStamp`获取到的rs，左移16位 + 2后，返回`1000 0000 0001 1011 0000 0000 0000 0010`。它的高16位由`risizeCtl(n)`的结果组成，如果有n个线程加入扩容，低16位的值为`1+n`。由于此时sizeCtl的符号位为1，所以处于扩容状态`sizeCtl`的值总是负数。

### resizeStamp方法

```java
//根据传入桶的长度，生成一个扩容戳
static final int resizeStamp(int n) {
    //numberOfLeadingZeros：该方法的作用是返回无符号整型i的最高非零位前面的0的个数，包括符号位(最高位)在内
    //1 << (RESIZE_STAMP_BITS - 1)：把1左移(RESIZE_STAMP_BITS - 1)位，也就是15位
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```

- 如果传入的桶长度为16，`Integer.numberOfLeadingZeros(n)  = 27`(16转化为二进制后，1前面有27个0)，转化为二进制为：0000 0000 0000 0000 0000 0000 0001 1011  ，异或运算`0000 0000 0000 0000 0000 0000 0001 1011 || 0000 0000 0000 0000 1000 0000 0000 0000  =  0000 0000 0000 0000 1000 0000 0001 1011`

### helpTransfer方法

```java
//帮助扩容，当前线程不是第一个扩容的线程的时候，才会进入此方法
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    //如果当前tab存在并且，扩容未完成，并且指向的下一个桶已经初始化
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        //获取标识符
        int rs = resizeStamp(tab.length);
        //如果nextTab没有被其他线程修改，sizeCtl<0扩容还在进行
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            //同上面介绍的tryPresize方法中一样
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            //帮助扩容，扩容线程数+1
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}

```

### transfer方法

```java
//核心方法，也是ConcurrentHashMap的精华部分
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    //stride为每个cpu所需要处理的桶个数，stride单核下为n，多核情况下，如果桶较少，那么默认一个线程处理16个
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range 细分范围
	//如果新的桶还没有初始化，第一个扩容的线程会进入此方法
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            //设置新的桶为原来的一倍
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            //扩容失败，设置sizeCtl为最大扩容量
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        //nextTable指向新tab
        nextTable = nextTab;
        //transferIndex初始化为，原tab长度
        transferIndex = n;
    }
    //新tab的长度
    int nextn = nextTab.length;
    // 创建一个 ForwardingNode 节点，用于占位。
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // advance 指的是做完了一个位置的迁移工作，寻找下一个需要迁移的位置
    boolean advance = true;
    //完成状态，代表所有迁移已经完成
    boolean finishing = false; // to ensure sweep before committing nextTab
    //i数组下标，bound表示迁移任务的边界值，从后向前
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        //如果advance为true，说明可以进行下一个位置的迁移了
        while (advance) {
            int nextIndex, nextBound;
            //--i,表示下一个要处理的桶下标，如果--i>=bound表示当前线程还未完成所有迁移工作，或者通过<4>处，已经将已经将i设置为n了，说明当前任务已经完成了
            //将advance置为false，那么会继续进行下一个桶的迁移任务
            //如果finishing为true（说明下面<4>处，已经将i设置为n了，）,说明所有任务已经迁移完成了，将advance置为false，那么此时i=n,会进入到<5>处，完成迁移操作
            if (--i >= bound || finishing)
                advance = false;
            //第一个线程扩容时，会将transferIndex置为原table的长度，也就是需要扩容桶的最大位置，如果
            //transferIndex<=0,说明所有的位置都有对应的线程去处理了
            else if ((nextIndex = transferIndex) <= 0) {
                //<1>.进入到下面的for循环中，判断所有迁移任务是否完成
                i = -1;
                advance = false;
            }
            //分配迁移区间,通过cas将transferIndex置为下一次迁移的最大下标
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                //此时bound代表，线程负责桶区间当前最小下标
                bound = nextBound;
                //此时i代表，线程负责桶区间当前最大下标
                i = nextIndex - 1;
                //分配区间完成，退出while循环
                advance = false;
            }
        }
        //<5>.i<0（满足<1>处,i = -1）说明所有区间已经分配完了，i >= n（满足<2>处，i=n),说明所有的迁移已经完成了
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                //如果迁移完成，将nextTable置为null
                nextTable = null;
                //table指向扩容后的新tab
                table = nextTab;
                //n时原来table长度，左移动一位后为2n，n向右移动一位为n/2,2n - 0.5n = 0.75n
                //所以sizeCtl为新数组长度的0.75，此时的sizeCtl也就是扩容阈值了
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            //通过cas将当前SIZECTL - 1，代表当前线程完成迁移，迁移线程数-1
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    //当前迁移完成，退出(因为其他区间都已经分配了)
                    return;
                //<4>.如果(sc - 2) == resizeStamp(n) << RESIZE_STAMP_SHIFT表示所有的迁移，已经完成了，之后会回到上面的finishing，完成整个迁移操作
                finishing = advance = true;
                //<2>将i置为n,再次检查整个table
                i = n; // recheck before commit
            }
        }
        //获取原来tab位置的元素，如果为null,那么设置为ForwardingNode节点占位
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 如果f存在且该位置处是一个 ForwardingNode，代表已经被其他线程处理过了
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            //开始迁移，锁定节点，避免其他线程putVal操作当前链表
            synchronized (f) {
                //二次校验
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        //节点的hash值要么为0，要么为n
                        int runBit = fh & n;
                        //最后遍历的节点
                        Node<K,V> lastRun = f;
                        //遍历链表
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            //第一次，runBit = 0，b = n，记录高位,设置runBit = n
                            //第二次，runBit = n, b = n,不记录
                            //第三次，runBit = n, b = 0, 设置低位，设置runBit = 0
                            if (b != runBit) {
                                runBit = b;
                                //更新最后遍历节点
                                lastRun = p;
                            }
                        }
                        //如果最后更新的runBit==0，设置低位节点，此时后面的节点都为低位，且存在链接关						系，不用再处理
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        //如果最后更新的runBit!=0，设置高位节点，此时后面的节点都为低位，且存在链接关						系，不用再处理
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        //如果此时lastRun为高位，那么lastRun后面的都为低位
                        //如果此时lastRun为低位，那么lastRun后面的都为高位
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                //将当前节点链接到低位的头节点（从后向前）
                                //如果原来ln为2->4，当前节点为1，那么ln为1->2->4
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                //将当前节点链接到高位的头节点（从后向前）
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        //低位的链表在当前桶位置
                        setTabAt(nextTab, i, ln);
                        //高位的链表在当前桶i+n位置（跟HashMap中一样，扩容后，元素要么在当前位置，要么						在当前位置+n位置）
                        setTabAt(nextTab, i + n, hn);
                        //fwd占位，代表当前位置已经被处理
                        setTabAt(tab, i, fwd);
                        //处理完成当前位置，继续处理下一个位置
                        advance = true;
                    }
                    //如果为树节点，跟上面一样
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        //如果元素迁移后，树长度不满足要求，则转化为链表
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                        (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                        (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        //同上面链表迁移的操作
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

### tabAt系列方法

```java
//获取table上下标为i的头结点，通过cas同步
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
//基于CAS尝试更新table上下标为i的结点的值为v，设置成功说明获取到锁
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
//setTabAt用于设置table上下标为i的结点为v
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
```
### addCount方法

```java
//x代表要添加元素的个数，check表示是否需要进行扩容检查，如果>=0,需要检查
//因此通过putVal进入此方法,每次添加元素之后都会进行检查
private final void addCount(long x, int check) {
     CounterCell[] as; long b, s;
     //如果计数器存在，说明当前map被其他线程操作过，直接统计baseCount不一定准确，使用counterCells统计
    //计数器不存在，且修改baseCount失败（说明并发冲突)，通过fullAddCount统计数量
     if ((as = counterCells) != null ||
         !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
         CounterCell a; long v; int m;
         boolean uncontended = true;
         //如果as==null，说明上面修改baseCount失败，存在并发
         //<1>.如果as!=null,存在值，并且当前线程获取到的Probe位置有值，且通过cas设置单元值成功，那么通过sumCount()统计总数
         //否则进入到fullAddCount方法中
         if (as == null || (m = as.length - 1) < 0 ||
             (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
             !(uncontended =
               U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) 
             //只有as!=null,存在值，并且当前线程获取到的Probe位置有值的情况下，uncontended才会为false（存在竞争）,否则为true，表示未竞争
             fullAddCount(x, uncontended);
             return;
         }
    	//如果check<=1
        if (check <= 1)
             return;
        s = sumCount();
    }
	//扩容检查
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        //此时s为统计后的map中元素的总数
        //如果s>=扩容阈值，并且tab存在，且容量小于默认的最大容量，那么接下来的操作跟tryPresize中一样
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
               	//如果有其他线程正在扩容，帮助扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            //如果当前为第一个扩容的线程
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            //帮助扩容完成后，统计元素总数，如果仍然大于扩容阈值，可能进行下一次扩容
            s = sumCount();
        }
    }
}
```

- **<1>处**，`ThreadLocalRandom.getProbe()`，`ThreadLocalRandom`是一个线程私有的伪随机数生成器，每个线程的`probe`都是不同的，通过`ThreadLocalRandom.getProbe() & m`每次都能找当当前线程对应的`CounterCell`，可以认为每个线程的`probe`就是它在`CounterCell`数组中的`hashcode`。

### size方法

```java
//获取ConcurrentHashMap中size的方法，通过统计多个CounterCell计数器的value来计算最终的size
public int size() {
    //统计总数
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}
```

### sumCount方法

```java
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    //baseCount为当前size
    long sum = baseCount;
	//如果计数器存在，那么统计计数器中的所有值
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```



```java
private final void fullAddCount(long x, boolean wasUncontended) {
    //当前线程的probe
    int h;
    //判断当前有没有初始化过ThreadLocalRandom，没有则初始化
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();      // force initialization
        h = ThreadLocalRandom.getProbe();
        //进入该方法中表示不存在竞争
        wasUncontended = true;
    }
    //当前CounterCell中是否存在冲突，默认不冲突
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
        //如果计数器存在且不为空
        if ((as = counterCells) != null && (n = as.length) > 0) {
            //如果没有找到当前线程对应的计数器
            if ((a = as[(n - 1) & h]) == null) {
                //当前无锁
                if (cellsBusy == 0) {            // Try to attach new Cell
                    //创建一个value为x的计数器
                    CounterCell r = new CounterCell(x); // Optimistic create
                    //通过cas，改变CELLSBUSY为1，获取锁
                    if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            //如果当前线程计数器不存在，为当前线程创建计数器
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            //释放锁
                            cellsBusy = 0;
                        }
                        //表示增加的count已经成功设置到CounterCell中了，结束方法
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                //当前CounterCell中没有出现冲突
                collide = false;
            }
            //addCount中，通过cas设置单元值失败uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))失败，自旋重试
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            //addCount中没有机会进行的cas操作，as == null || (m = as.length - 1) < 0 ||(a = as[ThreadLocalRandom.getProbe() & m]) == null，在此处执行
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
            //如果计数器被其他线程改变或者计数器个数超过处理器数量
            else if (counterCells != as || n >= NCPU)
                //设置当前线程循环失败，不进行扩容
                collide = false;            // At max size or stale
            else if (!collide)
                //设置当前CounterCell存在冲突，如果下循环的时候有机会对counterCells扩容
                collide = true;
            //如果当前线程存在计数器，获取锁，collide = true为true时，才会进入此方法，代表着冲突频繁
            else if (cellsBusy == 0 &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    //确保没有被其他线程操作过
                    if (counterCells == as) {// Expand table unless stale
                        //将CounterCell容量增大为原来的2倍，减少冲突
                        CounterCell[] rs = new CounterCell[n << 1];
                        for (int i = 0; i < n; ++i)
                            //复制到新CounterCell中
                            rs[i] = as[i];
                        //更换为新的counterCells
                        counterCells = rs;
                    }
                } finally {
                    //释放锁
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            h = ThreadLocalRandom.advanceProbe(h);
        }
        //抢锁，如果counterCells没有被改变，且为null,那么初始化counterCells
        else if (cellsBusy == 0 && counterCells == as &&
                 U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
            //初始化完成标时
            boolean init = false;
            try {                           // Initialize table
                //二次检查
                if (counterCells == as) {
                    //初始化一个长度为2的CounterCell数组
                    CounterCell[] rs = new CounterCell[2];
                    //取模计算下标，要么在0要么在1
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    //初始化完成
                    init = true;
                }
            } finally {
                //释放锁
                cellsBusy = 0;
            }
            //如果初始化成功，结束当前方法
            if (init)
                break;
        }
        //如果counterCells被别的线程初始化了，那么继续更新baseCount的值，如果设置成功，结束当前方法
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
```



### get方法

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    //通过key的hashcode计算下标
    int h = spread(key.hashCode());
    //如果key对应的value值存在
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        //如果头节点就是当前要查找的值，直接返回
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        //如果当前hash<0,说明当前为ForwardingNode节点或者树节点，使用它们自带的find方法查找
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        //遍历节点
        while ((e = e.next) != null) {
            //找到指定节点，直接返回
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    //如果没有找到，返回null
    return null;
}
```

### remove系列方法

```java
//通过key移除元素
public V remove(Object key) {
    //删除节点
    return replaceNode(key, null, null);
}

/**
* 实现四种公共删除/替换方法:
* 用v替换节点值，条件是匹配cv非空。如果结果值为空，则删除。
* v代表要替换的值
*/
final V replaceNode(Object key, V value, Object cv) {
    //获取元素hash值
    int hash = spread(key.hashCode());
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //如果table存在，并且hash计算的下标桶不存在，那么直接结束方法(可能被其他用户删除了)
        if (tab == null || (n = tab.length) == 0 ||
            (f = tabAt(tab, i = (n - 1) & hash)) == null)
            break;
        //如果当前桶位置为ForwardingNode节点，那么帮助扩容
        else if ((fh = f.hash) == MOVED)
            //扩容完成后返回新的tab，回到for循环再次执行删除
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            boolean validated = false;
            //锁定头节点
            synchronized (f) {
                //二次校验
                if (tabAt(tab, i) == f) {
                    //如果fh>=0，说明为链表结构
                    if (fh >= 0) {
                        validated = true;
                        //遍历链表
                        for (Node<K,V> e = f, pred = null;;) {
                            K ek;
                            //如果找到对应的value值
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                V ev = e.val;
                                //判断cv==null,或者cv符合节点value指向删除操作
                                if (cv == null || cv == ev ||
                                    (ev != null && cv.equals(ev))) {
                                    //记录被覆盖的值
                                    oldVal = ev;
                                    //如果value！=null，说明当前只是替换操作
                                    if (value != null)
                                        //替换原来的值
                                        e.val = value;
                                    else if (pred != null)
                                        //从链表中截断当前节点，也就是删除。
                                        pred.next = e.next;
                                    else
                                        //如果pred==null，说明要删除的为当前链表的头节点
                                        //那么将下一个节点置为当前链表的头节点
                                        setTabAt(tab, i, e.next);
                                }
                                break;
                            }
                            pred = e;
                            //如果遍历到链表末尾还没找到元素，跳出循环，结束方法
                            if ((e = e.next) == null)
                                break;
                        }
                    }
                    else if (f instanceof TreeBin) {
                        validated = true;
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> r, p;
                        if ((r = t.root) != null &&
                            (p = r.findTreeNode(hash, key, null)) != null) {
                            V pv = p.val;
                            if (cv == null || cv == pv ||
                                (pv != null && cv.equals(pv))) {
                                oldVal = pv;
                                if (value != null)
                                    p.val = value;
                                else if (t.removeTreeNode(p))
                                    setTabAt(tab, i, untreeify(t.first));
                            }
                        }
                    }
                }
            }
            //是否对节点进行了操作
            if (validated) {
                //如果oldVal != null，说明上面方法对节点进行了处理
                if (oldVal != null) {
                    //如果是删除节点动作
                    if (value == null)
                        //将table元素个数-1
                        addCount(-1L, -1);
                    //返回旧值
                    return oldVal;
                }
                break;
            }
        }
    }
    //没找到返回null
    return null;
}

```

## 总结

`ConcurrentHashMap`源码部分篇幅较长，且难以理解，容易看了前面忘了后面。此处对`ConcurrentHashMap`中最核心的`putVal`方法进行一个总结。

1. 通过`int hash = spread(key.hashCode())`获取`hash`。如果当前`table`没有初始化，那么通过`initTable`方法初始化
2. 通过hash计算下标,如果当前桶中没有元素,创建一个节点,通过cas设置到当前桶
3. 如果当前桶节点`hash = -1`,说明当前`map`正在扩容,`helpTransfer(tab, f)`方式帮助扩容,完成扩容后返回tab
4. 如果当前桶存在元素且没有扩容,锁定头节点,判断当前桶类是链表还是红黑树,通过遍历链表或红黑树的方式,通过cas方式将节点设置到链表(期间如果链表长度>8,会触发转红黑树操作)或红黑树中.
5. 如果添加元素在链表或红黑树中存在,那么覆盖掉原节点的value,如果不存在那么添加一个新的节点,并且通过addCount(1L, binCount)方法,将元素数量+1



