title: LinkedHashMap源码解析(jdk1.8)

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

`LinkedHashMap`是`Map` 接口的哈希表和链接列表实现，具有可预知的迭代顺序。此实现提供所有可选的映射操作，并允许使用null值和null键。由于`LinkedHashMap`中很多方法直接继承自HashMap，因此在看本章之前，建议先看看[HashMap的源码解析](https://www.yingu.site/2019/12/25/HashMap/)。<!--more--> 

## 结构特点

1. `LinkedHashMap`继承自`HashMap`,实现了`Map`的接口。与`HashMap`不同之处在于，`LinkedHashMap`通过维护着一条双向链表，解决了`HashMap` 不能随时保持遍历顺序和插入顺序一致的问题，并且`LinkedHashMap`提供了特殊的构造方法来创建链接哈希映射。
2. 实现了`Cloneable`接口， 表示 `LinkedHashMap`支持克隆。
3. 实现了 `Serializable` 接口， 表示 `LinkedHashMap`支持序列化的功能 ，可用于网络传输。

![LinkedHashMap类图](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/LinkedHashMap.png)

## 重要属性

```java
//LinkedHashMap中通过Entry维护着元素的前后驱节点
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        //通过HashMap中的node创建
        super(hash, key, value, next);
    }
}
//链表头
transient LinkedHashMap.Entry<K,V> head;
//链表尾
transient LinkedHashMap.Entry<K,V> tail;
//如果为true，则表示访问有序（新访问的数据会被移至到链尾）。如果为false,表示插入有序。通过此参数，可以
//使用LinkedHashMap设计LRU缓存;
final boolean accessOrder;

```

## 常用方法

### 构造方法

```java
//无参构造方法，默认accessOrder为false
public LinkedHashMap() {
    //调用父类构造方法
    super();
    accessOrder = false;
}

//带初始容量的构造方法，默认accessOrder为false
public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}

//带初始容量、负载因子的构造方法，默认accessOrder为false
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}

//带map对象的构造方法，默认accessOrder为false
public LinkedHashMap(Map<? extends K, ? extends V> m) {
    super();
    accessOrder = false;
    //将元素放置到map中
    putMapEntries(m, false);
}

//使用此方法，指定accessOrder为true，可以针对访问顺序调整链表顺序
public LinkedHashMap(int initialCapacity,
                     float loadFactor,
                     boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}
```

### Node节点

```java
//LinkedHashMap中的节点，e为null
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    //创建Entry
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    //将改节点插入到链表尾部
    linkNodeLast(p);
    return p;
}

private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    //记录尾节点
    LinkedHashMap.Entry<K,V> last = tail;
    //将元素置为尾节点
    tail = p;
    //如果尾节点为空，说明该链表中没有元素
    if (last == null)
        //将元素置为头节点
        head = p;
    else {
        //否则将该元素置为尾节点
        p.before = last;
        last.after = p;
    }
}
```

### HashMap预留的后置方法

```java
void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
void afterNodeRemoval(Node<K,V> p) { }
```

前面我们解析`HashMap`的时候，就发现`HashMap`在执行一些方法后，就会执行上面空方法，而这些空方法就是留给`LinkedHashMap`扩展实现的。LinkedHashMap正是通过重写这三个方法来保证链表的插入、删除的有序性。

```java
//移除指定节点的后置方法，将节点从当前LinkedHashMap的双向链表中去除
void afterNodeRemoval(Node<K,V> e) { // unlink
    //记录节点，b前驱节点，a后驱节点
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    //当前节点前后驱节点置为null，便于gc回收
    p.before = p.after = null;
    //如果b为null,说明被移除的是头节点，那么记录后驱节点a为头节点
    if (b == null)
        head = a;
    else
        //否则将前驱节点b的后驱节点指向后驱节点a
        b.after = a;
    //如果a为null，说明被移除的是尾节点，那么记录前驱节点为头节点
    if (a == null)
        tail = b;
    else
        //否则将后驱节点a的前驱节点指向前驱节点b
        a.before = b;
}

//添加节点的后置方法，可能移除当前双向链表中最老的节点，evict为true
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;，
    //如果evict为true,当前双向链表中存在元素，且允许移除最老的节点
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        //移除头节点
        removeNode(hash(key), key, null, false, true);
    }
}

// 返回false, 所以LinkedHashMap不会删除最少使用的节点，子类可以通过覆盖此方法实现不同策略的缓存
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}


//访问节点的后置方法，将节点移动到双向链表的尾部
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    //accessOrder 为 true，表示访问过后的节点，移动到链表的尾部
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        //操作数+1
        ++modCount;
    }
}
```

### get方法

```java
//通过key找寻对应的值
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    //accessOrder为true，将访问的节点移动到链表尾部
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
//通过key找寻对应的值，如果没找到，返回默认值
public V getOrDefault(Object key, V defaultValue) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return defaultValue;
    //accessOrder为true，将访问的节点移动到链表尾部
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```

### 总结

`LinkedHashMap`的源码解析就到这里了。从上面的一些方法中就可以看出来`LinkedHashMap`的特点：`LinkedHashMap` 通过 `Entry` 维护了一条双向链表，实现了散列数据结构的有序遍历。同时内部支持通过访问顺序，来调整双向链表的节点顺序，可以通过此特性，实现`LRU`算法。