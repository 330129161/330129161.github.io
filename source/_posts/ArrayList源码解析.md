title: ArrayList源码解析(jdk1.8)

author: yingu

thumbnail: http://q3ti54das.bkt.clouddn.com/static/20200109/uIs3Bzu9Fq1t.png?imageslim

tags:
  - 源码
  - java
  - 集合
categories: 
  - [源码,集合]
date: 2019-12-25 20:41:00
---

# ArrayList源码解析(jdk1.8)

## 一、概述

 ArrayList是我们日常开发中比较常见的一个容器类。它底层由动态数组实现， 所以和数组一样，可以根据索引对容器对象所包含的元素，进行快速随机的查询操作 。Arraylist在中间位置插入或者删除元素时，需要对数据进行复制、移动、代价比较高。因此，它适合随机查找和遍历，不适合插入和删除，同时因为ArrayList内部没有对并发做出处理，所以**Array是非线程安全的**。

## 二、结构特点

1.  ArrayList 继承了AbstractList，实现了List。它是一个数组队列，提供了相关的添加、删除、修改、遍历等功能。 
2.  ArrayList 实现了RandmoAccess接口，即提供了随机访问功能 
3.  ArrayList 实现了Cloneable接口，即可以被克隆
4. ArrayList 实现了 Serializable 接口，即可被序列化，可用于网络传输
5. ArrayList非线程安全，因此只适用于单线程中。如果在多线程，可使用 CopyOnWriteArrayList 代替， 不推荐Vector。

![结构](http://q3ti54das.bkt.clouddn.com/static/20200109/uIs3Bzu9Fq1t.png?imageslim)

## 三、重要属性

```java
 public class ArrayList<E> extends AbstractList<E>        
     implements List<E>,RandomAccess, Cloneable, java.io.Serializable
 {
     //默认容量大小
     private static final int DEFAULT_CAPACITY = 10;
     //当添加新的元素时，如果该数组不够，会创建新数组，并将原数组的元素拷贝到新数组。之后，将该变量指向新数组
     private static final Object[] EMPTY_ELEMENTDATA = {};
     //使用无参构造方法时，使用该属性进行对象的初始化
     private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
     //元素数据
     transient Object[] elementData;
     //实际元素大小
     private int size;
     //最大数组容量
     private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
     
 }
```

## 四、常用方法

### 无参数构造方法

调用无参构造方法，该方法会指向一个空的的ArrayList对象，**注意，此时的对象没有容量。**

```java
public ArrayList() {
    // 无参构造使用 DEFAULTCAPACITY_EMPTY_ELEMENTDATA 进行初始化
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

###  带容量大小的构造函数

方法通过传入参数来构造一个空的 List，并指定其容量为`initialCapacity`

```java
public ArrayList(int initialCapacity) {
    //指定的容量>0，则创建一个指定大小的数组
    //指定的容量=0，使用 EMPTY_ELEMENTDATA 进行初始化
    //否则抛出异常
    if (initialCapacity > 0) {
   		this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
    	this.elementData = EMPTY_ELEMENTDATA;
    } else {
    	throw new IllegalArgumentException("Illegal Capacity: "+
    		initialCapacity);
    }
}
```

###  带Collection对象的构造函数

平时用的比较少的一个方法，通过集合类来生成ArrayLIst

```java
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    //<1>获取集合中的数据
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        //<2>c.toArray返回的类型 不一定是一个Object[]
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        //<3>如果传入的集合为空，那么初始化一个空的数据
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

- <2>处，为什么说c.toArray返回的类型 不一定是一个Object[]。collection.toArray()理论上应该返回Object[]。然而使用Arrays.asList得到的list，其toArray方法返回的数组却不一定是Object[]类型的，而是返回它本来的类型。

### add添加单个元素

```java
public boolean add(E e) {
    //<1>检查是否需要扩容
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}

public void add(int index, E element) {
    //<2>校验位置是否在数组范围内
    rangeCheckForAdd(index);
    //<3>检查是否需要扩容
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    elementData[index] = element;
    size++;
}
```

<1>检查当前集合的容量是否需要扩容

```java
private void ensureCapacityInternal(int minCapacity) {
    //如果当前的集合容量为空，设置当前容器大小为默认值10
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    ensureExplicitCapacity(minCapacity);
}
```



## 四、应用场景

## 五、常见问题

