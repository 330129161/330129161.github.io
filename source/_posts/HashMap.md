title: HashMap源码解析(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/book.png

toc: true 

tags:

  - 源码
  - java
  - 集合

categories: 

  - [源码,集合] 

date: 2019-12-25 20:41:00

---

## 概述

`HashMap`基于哈希算法实现，用于存储 key-value 键值对的数据结构，底层由数组+链表+红黑树(**jdk_1.8新增**)组成。`HashMap`相比于之前介绍的的`ArrayList`与`linkedList`结构要复杂得多。`ArrayList`底层由数据组成，查找容易，插入、删除的效率不高。`linkedList`由链表组成，插入、删除容易、查询的效率不高。而基于散列表的HashMap，在保证高效查询的情况下，还能保证插入、删除的效率。HashMap允许使用null值和null键，HashMap非线程安全。

## 结构特点

1. `HashMap`继承了`AbstractMap`，实现了`Map`接口。
2. 实现了`Cloneable`接口， 表示 `HashMap`支持克隆。
3. 实现了 `Serializable` 接口， 表示 `HashMap`支持序列化的功能 ，可用于网络传输。

![HashMap结构](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/HashMap.png)

## 重要属性

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    
    //默认容量大小（必须为2的幂次方） 
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 
	//最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30;
	//默认负载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
	//链表转红黑树长度阈值，当链表长度>8时，会转为红黑树
    static final int TREEIFY_THRESHOLD = 8;
	//红黑树还原链表数量阈值，当红黑树数量<6,会转为链表
    static final int UNTREEIFY_THRESHOLD = 6;
	//链表转红黑树长度容量阈值，当哈希表中容量>64,才会将链表转红黑树长度，否则直接扩容而不是转化为红黑树，为了避免扩容与转化后红黑树之间的冲突，这个值不能小于64.
    static final int MIN_TREEIFY_CAPACITY = 64;
    //存放数据的桶，每一个桶中都有一段链表或红黑树（桶长度必须为2的幂次方） 
    transient Node<K,V>[] table;
	//存储键值对形成的entrySet
    transient Set<Map.Entry<K,V>> entrySet;
    //元素数量
    transient int size;
    //修改次数(如果在entrySet中遍历时，出现modCount与预期值不一致，那么会抛出
    //ConcurrentModificationException异常，表示正在被多个线程同时操作，存在线程安全问题)
    transient int modCount;
	//扩容时的阈值 （负载因子 * 容量）
    int threshold;
	//负载因子
    final float loadFactor;

}
```

### Node节点

```java
//HashMap的内部单链表，用来储存桶中的元素，指定条件下会转为红黑树
static class Node<K,V> implements Map.Entry<K,V> {
    //当前node的hash值
    final int hash;
    //当前node的key
    final K key;
    //当前node的value
    V value;
    //下一个节点
    Node<K,V> next;
    
    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }
	
    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```

## 常用方法

### 无参构造方法

```java
public HashMap() {
    //负载因子设置为默认值
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```

### 带容量大小的构造方法

```java
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
//带容量大小，初始因子的默认构造方法（一般不使用）
public HashMap(int initialCapacity, float loadFactor) {
    //如果初始容量<0抛出异常
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    //如果初始容量>最大容量，则设置当前容量为最大容量
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    //校验负载因子合法性
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    /*
     * 计算扩容时的阈值
     * 例如传入的初始容量initialCapacity=7时，threshold = 8
     * initialCapacity = 9，threshold = 16
     */
    this.threshold = tableSizeFor(initialCapacity);
}
```

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

上面简单的说明了，该方法是如何通过|运算与位移,来找到最接近输入值的2的幂次方数。为什么最后要一直写到 `n |= n >>> 16`,int的范围在`-2^31 ~ 2^31-1`,因此最大2次幂数为`2^30`,也就是当前容量默认的最大值`MAXIMUM_CAPACITY`，代码`1+2+4+8+16=31`一共向右移了31位，是为了保证高位1以下的低位都会变为1。

### 带Map对象的构造方法

```java
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```
### putMapEntries方法

```java
//该方法会被HashMap的public HashMap(Map<? extends K, ? extends V> m)构造函数、clone方法，以及Map接口的putAll方法调用。
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    //获取传入的map长度
    int s = m.size();
    //传入的数据>0时，才有意义
    if (s > 0) {
        /* 
         * 如果当前table == null,说明当前是通过调用l构造函数、clone方法或者构造后还没有
         * 放入任何元素，此时需要设置对象的扩容阈值
         */
        if (table == null) { // pre-size
        	//通过传入的长度/加载因子，可以计算一个>=阈值的数，保证本次不会触发扩容
            float ft = ((float)s / loadFactor) + 1.0F;
            //如果大于最大容量，那么设置t为最大容量数
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            //如果t>阈值,那么需要重新设置阈值数，保证阈值为2的幂次方，如果t=阈值
            //那么就等到下次插入元素时，再进行扩容操作
            if (t > threshold)
                threshold = tableSizeFor(t);
        } //table != null,说明当前HashMap已经初始化过了，如果s>阈值，那么HashMap就需要扩容了
        else if (s > threshold) 
            resize(); //扩容
        // 将map中的所有元素添加至HashMap中
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            //往HashMap中添加元素
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

- 上面的方法有个很重要的一点，为什么此处`float ft = ((float)s / loadFactor) + 1.0F`需要 + 1，如果`((float)s / loadFactor)`算出来的是小数，此时如果向下取整，那么可能会导致容量不够大。
- 我们在计算`HashMap`阈值时（详见`resize()`函数中`newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE)`）向下取整（如果向上取整，那么当放入容器的元素，也就是`size>threshold`阈值时，可能就无法进行`resize()`扩容操作了）。反过来想，`float ft = ((float)s / loadFactor) `得到的参数，向上取整也就是顺理成章的事情了。
- 但是向上取整，也会带来一些问题。假如我们默认加载因子0.75，此时HashMap默认容量为16，那么阈值就是12，我们传入map的size为12，那么12 / 0.6 + 1 = 17，此时通过`tableSizeFor()`计算后，得到的容量为32，阈值为24，导致内存浪费。（但是为了稳定性，也只能牺牲一部分了）

### resize方法

```java
//扩容方法
final Node<K,V>[] resize() {
    //记录原来的table
    Node<K,V>[] oldTab = table;
    //记录原table长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    //记录扩容阈值
    int oldThr = threshold;
    int newCap, newThr = 0;
    //<1>.如果原table长度大于0
    if (oldCap > 0) {
        //<2>.如果长度>最大容量值
        if (oldCap >= MAXIMUM_CAPACITY) {
            //设置阈值为最大int数，之后也不会触发扩容了 
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //newCap = oldCap << 1，新容量设置为原来的容量的两倍，如果
        //新容量小于最大容量，判断原来容量是否大于默认容量16，否则设置新的
        //扩容阈值为原来的一倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            //设置新阈值为原来的一倍
            newThr = oldThr << 1; // double threshold
    }
    //<3>.如果原来的table长度=0，ldThr > 0说明是第一次带参数初始化，设置新容量为原来的threshold
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else { // zero initial threshold signifies using defaults
        //<4>.进入当前方法说明，当前HashMap通过无参构造方法构造的
        //设置新容量为默认值16
        newCap = DEFAULT_INITIAL_CAPACITY;
        //新阈值为16 * 0.75 = 12
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    //如果新阈值为0，说明程序运行到<3>处，也就是通过带参数初始化构建的HashMap
    if (newThr == 0) {
        //计算阈值（由于loadFactor的不稳定，得到的阈值可能为小数）
        float ft = (float)newCap * loadFactor;
        //如果小于最大容量值，向下取整，保证容量达到阈值后，进行扩容操作
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    //设置新阈值
    threshold = newThr;
    //创建newCap长度的node数组，也就是扩容
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    //如果原来的table有数据，那么将原来的数据复制到新的table中
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            //记录当前桶链表
            Node<K,V> e;
            //复制所有非空的桶
            if ((e = oldTab[j]) != null) {
                //将原桶置空，便于gc回收
                oldTab[j] = null;
                //如果当前链表没有下级元素，不存在链表转红黑树的情况，直接放置到新的table中
                if (e.next == null)
                    //<5>.通过e.hash & (newCap - 1)计算下标，放到指定的桶内
                    newTab[e.hash & (newCap - 1)] = e;
                //当前是否为树节点（红黑树）
                else if (e instanceof TreeNode)
                    //将节点分割到不同桶中，可能会触发树转链表操作
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                //如果是链表,且存在下级元素
                else { // preserve order
                    //记录原位置节点头
                    Node<K,V> loHead = null, loTail = null;
                    //记录
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        //记录下级元素
                        next = e.next;
                        //<6>如果(e.hash & oldCap) == 0，说明原来的元素位置没有变化，后面会解释
                        //会将链表中所有位置节点的元素放到头节点为loHead的链表中
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }//否则将元素放到头节点为hiHead的链表中
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    //如果loTail不为null，说明改链表中有元素
                    if (loTail != null) {
                        loTail.next = null;
                        //将当前链表放置到桶中
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        //将当前链表放置到桶中,当前获取的(下标j + 原来的容量)位置
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    //返回扩容后的新table
    return newTab;
}
```

**<5>处**，通过e.hash & (newCap - 1)计算index下标，为什么要通过这种方式来计算下标？我们知道table的容量设定为2的幂次方，而2的幂次方 - 1后，它的二进制低位都为1。这种情况下通过&运算，index的结果等同于HashCode后几位的值。因此只要我们输入的HashCode本身分布均匀，通过这种算法每次分布的index就是均匀的。

- 可能有人不明白，为什么index的结果等同于HashCode后几位的值的意思。我们现在就来举个例子。假如我们table的容量为默认值16，那么16 - 1 = 15，15的二进制位为00001111。此时hash值假设为00011110，00001111 & 10011110 = 00001110，因此无论hashcode的前几位如何变化，得到的下标值，只与newCap - 1二进制低位有关，且当前得到的index，不可能超过当前容量。

**<6>处**，如果(e.hash & oldCap) ，它得到的结果只会有两种情况，要么是 0，要么是 oldCap。例如当前`oldCap = 16`。hash分别为 12 ， 18 ， 33，49时，他的结果为 0，16，0 ，16 ,此时12 、33组成新的链表，index = 12(后续的都会假如到这个链表当中);而18，49组成新的index = 12 + 16;  其实这一步就是将原来链表中的数据，均匀分布到桶中，减少单个桶中的数据，保证table中数据分布均匀。同时避免对原来的数据进行resize操作，提高效率。（针对jdk1.7之后做的一个优化）

### put系列方法

```java
//map中添加元素，如果map中存在重复的key值，那么覆盖并返回原来的值
public V put(K key, V value) {
    //hash(key)计算key的hash值, onlyIfAbsent = false,默认如果key对应的值存在，那么覆盖
    return putVal(hash(key), key, value, false, true);
}

//向map中添加元素，如果map中存在重复的key值，那么不会放入值
@Override
public V putIfAbsent(K key, V value) {
    //hash(key)计算key的hash值, onlyIfAbsent = true,默认如果key对应的值存在，不会放入值
    return putVal(hash(key), key, value, true, true);
}

 /*
 * onlyIfAbsent：当前onlyIfAbsent为true时，不会改变链表中存在的value
 * evict：HashMap中没有用到，作用在LinkHashMap中，如果基于LinkHashMap实现LUR缓存的话
 * 该值就会用到，后续分析LinkHashMap的时候会讲到
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //如果table为null或者为空，需要初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        //初始化容量并返回长度，默认16
        n = (tab = resize()).length; 
    //<1>.通过hash计算当前元素放置的位置下表（前面说明过），
    //如果当前下标桶内没有元素，没有冲突，那么直接放置元素
    if ((p = tab[i = (n - 1) & hash]) == null)
        //向桶内插入节点，此时下个节点为null
        tab[i] = newNode(hash, key, value, null); 
    else {
        Node<K,V> e; K k;
        //如果当前已经存在key值，那么记录当前存在的节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //如果为树节点，那么假如到树节点中
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {//如果为链表，那么此时的p就为链表中的头节点，遍历链表，binCount记录链表长度
            for (int binCount = 0; ; ++binCount) {
                //如果当前下一个节点为null，说明当到链表的尾部了，且当前链表中没有已存在的节点
                if ((e = p.next) == null) {
                    //创建新节点，下一节点为null
                    p.next = newNode(hash, key, value, null);
                    //如果链表长度超过TREEIFY_THRESHOLD，那么尝试将当前链表转为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        //尝试将链表转为红黑树，如果桶的长度没有超过默认的
                        //MIN_TREEIFY_CAPACITY（64），那么会扩容而不是转树
                        treeifyBin(tab, hash);
                    break;
                }
                //如果在链表中发现节点已存在，那么不在循环，记录当前存在的节点
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    //找到存在的节点后跳出循环
                    break;
                p = e;
            }
        }
        //如果e != null，说明当前链表中已存在节点，那么替换掉value
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            //如果当前允许对存在的key对应的值更新，或者原来的值为null的时候。才会替换原来的值为新值
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            //返回旧的value，说明是更新操作
            return oldValue;
        }
    }
    //记录操作数
    ++modCount;
    //如果当前元素超过阈值，那么就需要扩容了
    if (++size > threshold)
        resize();
    // Callbacks to allow LinkedHashMap post-actions
    //空方法，留给子类LinkedHashMap去实现 
    afterNodeInsertion(evict);
    //如果链表中没有存在的值，返回null，说明时新增操作
    return null;
}
```

### hash方法

```java
//计算当前key的hash值
static final int hash(Object key) {
    int h;
    //<1>.hashcode向右无符号移动16位（右边补0），也就是取int类型的一半，
    //然后运用^（位不同那么结果为1，否则为0）计算得到hashcode
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

- **<1>处**，为什么要把hashcode向右无符号移动16位，再进行^运算呢？上面我们说过，我们获取桶的index下标是通过e.hash & (newCap - 1)计算，如果这个时候，我们假设容器长度为默认的16：

  | hashcode的值                 | 1001 0010 1011 0010 1110 0010 1010 0111     |
  | ---------------------------- | :------------------------------------------ |
  | **容器长度16 - 1**           | **0000 0000 0000 0000 0000 0000 0000 1111** |
  | **hashcode&（16 -1）运算后** | **0000 0000 0000 0000 0000 0000 0000 1111** |

  我们这个时候可以看到，hashcode前面28位都是没有参与运算的，位数越高参与度越低。如果通过这种方式，我们实际可能用到高位的情况就很少，那么我们在计算index下标时就会丢失高位的特性。假如两个hashcode相当接近的时候，可能就因为我们丢失高位的差异，导致产生一次hash碰撞。因此为尽可能减少发生碰撞的可能，将hashcode折中向右移16位，此时所有的高位都会移动到低位，通过^运算，此时hashcode的高位和低位都会参与计算，影响着hashcode的生成。而为什么会用^运算？如果使用&运算，那么计算的位会向0靠拢，使用|又会向1靠拢，而^可以保留hashcode的原始特性。

### get系列方法

```java
//通过key获取对应的value
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
//通过hash值，key获取对应的节点
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    //通过hash值计算桶的index，如果当前桶存在值，那么在桶中遍历查找对应的节点
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        //判断是否为头节点，如果是直接返回
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        //遍历链表（也可能是红黑树）
        if ((e = first.next) != null) {
            //如果是树节点，那么遍历树
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                //如果找到对应的key值，那么直接返回节点
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    //如果key值不存在，返回null
    return null;
}
//重写的map的方法，传入key和一个默认值，如果hashmap中找不对对应的key值，那么返回默认值
@Override
public V getOrDefault(Object key, V defaultValue) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
}
```

### remove系列方法

```java
//通过key移除节点
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}
//matchValue：是否需要匹配值，movable：LinkedHashMap中会用到
final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    //通过hash值计算桶的index，如果当前桶存在值，那么在桶中遍历查找对应的节点
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        //判断是否为头节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            //<1>.记录要移除的节点
            node = p;
        //如果不是头节点，那么遍历查找
        else if ((e = p.next) != null) {
            //如果为树节点，遍历红黑树查找并返回
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                //在链表中遍历查找
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        //如果找到，记录
                        node = e;
                        break;
                    }
                    //记录当前遍历的节点，如果找到存在的节点后，那么p节点就为要移除节点的上一节点
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        //根据hash值和key查找数据，且与value匹配(matchValue 是否需要匹配值)
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            //如果该节点为树节点
            if (node instanceof TreeNode)
                //去树中移除（可能触发树转链表操作）
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            //如果node == p，说明要移除的是头节点(对应操作<1>)
            else if (node == p)
                //替换头节点
                tab[index] = node.next;
            else
                //如果不为头节点，将当前节点从链表中截断
                //（例如：a -> b -> c 变成a -> c,b就是我们移除的节点）
                p.next = node.next;
            //记录操作数
            ++modCount;
            //map长度-1
            --size;
            //空方法，由linkedHashMap实现，一般没用的node节点，会将后驱节点置为null便于GC回收， 
            //而此时node的后驱节点并未清除，是为了node节点的完整，用于linkedHashMap的扩展
            afterNodeRemoval(node);
            //返回被移除的节点
            return node;
        }
    }
    //如果要移除的节点不存在，返回null
    return null;
}

```

## 总结

`HashMap`部分的源码解析就到这里。目前只分析了`HashMap`初始化、扩容、操作等核心代码。但没有红黑树部分代码的解析，由于考虑到红黑树代码部分较长，后面会针对红黑树单独出一期，所以就不再这里阐述了。如果各位小伙伴读完文章后，发现文章中有哪些错误或者不足之处，还请在评论区中留言。笔者看到也会尽快回复。