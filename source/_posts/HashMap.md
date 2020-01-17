title: ArrayList源码解析(jdk1.8)

author: yingu

thumbnail: http://q3ti54das.bkt.clouddn.com/static/20200115/KD21H6R83yJu.jpg?imageslim

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

## 结构特点

![HashMap结构](http://q3ti54das.bkt.clouddn.com/static/20200117/YlyyrtqXRKnH.png?imageslim)

## 重要属性

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 

    static final int MAXIMUM_CAPACITY = 1 << 30;

    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    static final int TREEIFY_THRESHOLD = 8;

    static final int UNTREEIFY_THRESHOLD = 6;

    static final int MIN_TREEIFY_CAPACITY = 64;
    
    transient Node<K,V>[] table;

    transient Set<Map.Entry<K,V>> entrySet;

    transient int size;

    transient int modCount;

    int threshold;

    final float loadFactor;

}
```



## 常用方法

## 总结