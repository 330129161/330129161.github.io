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
2.  实现了RandmoAccess接口，即提供了随机访问功能 ， 这样ArrayList使用for循环遍历元素要比使用迭代器遍历元素要快 。
3.  实现了Cloneable接口， 表示 ArrayList 支持克隆。
4. 实现了 Serializable 接口， 表示 ArrayList 支持序列化的功能 ，可用于网络传输。
5. ArrayList非线程安全，因此只适用于单线程中。如果在多线程，可使用 CopyOnWriteArrayList和Vector。 Vector方法和ArrayList基本相同，不过在修改方法上，都使用synchronized修饰。 而CopyOnWriteArrayList 采用写时拷贝策略，对其进行修改操作和元素迭代，都是在低层创建一个拷贝数组上进行，兼顾了线程安全的同时，又提高了并发性，性能比Vector有不少提高 。因此，多线程情况下推荐使用CopyOnWriteArrayList。

![结构](http://q3ti54das.bkt.clouddn.com/static/20200109/uIs3Bzu9Fq1t.png?imageslim)

## 三、重要属性

```java
 public class ArrayList<E> extends AbstractList<E>        
     implements List<E>,RandomAccess, Cloneable, java.io.Serializable
 {
     //默认容量大小
     private static final int DEFAULT_CAPACITY = 10;
     //空数组
     private static final Object[] EMPTY_ELEMENTDATA = {};
     //空数组，用于无参构造方法初始化
     private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
     //底层储存元素的数组
     transient Object[] elementData;
     //实际元素个数
     private int size;
     //最大数组容量
     private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
     
 }
```

## 四、常用方法

### 无参数构造方法

```java
public ArrayList() {
    // 将底层数组指向DEFAULTCAPACITY_EMPTY_ELEMENTDATA
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

###  带容量大小的构造函数

创建一个容量大小为`initialCapacity`的ArrayList实例

```java
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        //将容器底层数组的容量初始化为initialCapacity
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        //如果传入的参数为0，将底层数组指向EMPTY_ELEMENTDATA
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
     if ((size = elementData.length) != 0) {
         // c.toArray might (incorrectly) not return Object[] (see 6260652)
         //c.toArray返回的类型不一定是一个Object[]
         if (elementData.getClass() != Object[].class)
             //创建一个size大小的新数组，将原本的elementData数据复制到数组中，最后elementData指向新数组，完成扩容。
             elementData = Arrays.copyOf(elementData, size, Object[].class);
     } else {
         // replace with empty array.
         //将底层数组指向EMPTY_ELEMENTDATA
         this.elementData = EMPTY_ELEMENTDATA;
     }
 }
```

> 为什么说c.toArray返回的类型 不一定是一个Object[]。collection.toArray()理论上应该返回Object[]。然而使用Arrays.asList得到的list，其toArray方法返回的数组却不一定是Object[]类型的，而是返回它本来的类型。

### add添加方法

```java
public boolean add(E e) {
    //<1>.确保容器的当前容量足够容纳新增的元素
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
//在指定位置添加元素
public void add(int index, E element) {
    //校验位置是否在数组范围内
    rangeCheckForAdd(index);
    //同<1>
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //<2>.将数组elementData从下标index开始的元素，长度为size - index（即index到size之间的元素）复制到数组elementData以index+1开始的位置，再将element放在数组index的位置。
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    elementData[index] = element;
    size++;
}

private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```
<2>.arraycopy方法：

> public static void arraycopy(Object src, int srcPos, Object dest, int destPos, int length)
>
> `src`:源数组 
>
> `srcPos`:原数组开始位置  
>
> `dest`:目标数组 
>
> `desPos`:目标数组位置  
>
> `length`: 需要复制的元素个数 
>
> 复制指定的源数组`src`的数组，在指定的位置`srcPos`开始，到目标数组`dest`的指定位置`destPos`。一共需要复制`length`的元素个数。

```java
private void ensureCapacityInternal(int minCapacity) {
    //<3>.当前实例是否通过无参构造方法生成，且为第一次扩容
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        //<4>.对比当前最大容量与默认的容量10对比，取大值
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    //确保是否需要扩容
    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    //<6>.增加数组修改次数
    modCount++;
    // overflow-conscious code
    //如果当前所需要的容量，大于当前的容量那么就需要扩容
    if (minCapacity - elementData.length > 0)
        //扩容
        grow(minCapacity);
}

private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    //容器的新长度将设置为原来的1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    //如果扩容后的长度小于当前所需的长度，那么设置当前容量的长度为所需长度
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    //如果扩容后的长度大于的长度，判断当前长度是否超过限制的最大长度
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```



<3>.如果elementData指向DEFAULTCAPACITY_EMPTY_ELEMENTDATA，则表示当前实例是通过无参构造方法创建的，且当前是第一次扩容。

<4>.将需要的最小容量与默认容量对比，取大值，此时容量默认为10。ArrayList调用无参构造方式时，并没有直接将容量初始化为10，而是通过懒加载的形式 ，通过第一次调用add时设置的。

<5>. AbstractList包含一个modCount变量，它的初始值是0，**当集合中的内容每被修改一次时**（调用add()， remove()等方法）**，modCount加1** ,modCount的作用是什么，看看官方的说明：

> This field is used by the iterator and list iterator implementation returned by the iterator and listIterator methods. If the value of this field changes unexpectedly, the iterator (or list iterator) will throw a ConcurrentModificationException in response to the next, remove, previous, set or add operations. This provides *fail-fast* behavior, rather than non-deterministic behavior in the face of concurrent modification during iteration.

大概意思就是说：在使用迭代器遍历的时候，用来检查列表中的元素是否发生结构性变化（列表元素数量发生改变）了，主要在多线程环境下需要使用，防止一个线程正在迭代遍历，另一个线程修改了这个列表的结构。前面也说过ArrayList是非线程安全的。

关于ArrayList的add方法就分析到这里了， `addAll(int index, Collection<? extends E> c)`方法思路与上面两个方法基本一致，这里就不分析了。

### remove移除方法

```java
//通过下标移除方法
public E remove(int index) {
    //校验参数合法性
    rangeCheck(index);
	//增加数组修改次数
    modCount++;
    //记录要移除的下标元素
    E oldValue = elementData(index);
	//如果numMoved > 0,说明要移除的不是数组末尾的元素，因此需要移动整个数组
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    //将原本最末尾的元素置为空，便于GC回收
    elementData[--size] = null; // clear to let GC do its work
	//返回被移除的元素
    return oldValue;
}

private void rangeCheck(int index) {
    //如果传入的下与或者等于当前元素个数，抛出下标越界异常
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

//通过指定元素移除
public boolean remove(Object o) {
    if (o == null) {
        //通过遍历元素的方式，移除所有元素
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                //快速移除
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

//快速移除，与通过下标移除方法相比，少了一个参数校验的步骤
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}

//范围移除
 protected void removeRange(int fromIndex, int toIndex) {
     modCount++;
     //获取移除的元素
     int numMoved = size - toIndex;
     System.arraycopy(elementData, toIndex, elementData, fromIndex,
                      numMoved);

     // clear to let GC do its work
     //记录移除后元素的个数
     int newSize = size - (toIndex-fromIndex);
     //将新元素末尾位置到旧元素末尾位置之间的元素设置为null,便于GC回收
     for (int i = newSize; i < size; i++) {
         elementData[i] = null;
     }
     //重新设置容器中的元素个数
     size = newSize;
 }

// 删除c中存在的元素
public boolean removeAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return batchRemove(c, false);
}

//删除c中不存在的元素
public boolean retainAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return batchRemove(c, true);
}

//批量移除元素，complement ：是否移除在集合中存在的元素。retainAll默认为true，也就是做交集。如果为false，也就是做差集。
private boolean batchRemove(Collection<?> c, boolean complement) {
    final Object[] elementData = this.elementData;
    //r素遍历的位置 w写入覆盖的位置
    int r = 0, w = 0;
    //判断方法是否执行成功的标记
    boolean modified = false;
    try {
        //遍历所有元素
        for (; r < size; r++)
           	// <1>.complement为false，相同的元素会被覆盖，不同的元素会往左边移动
            // complement为true，相同的元素往左边移动，同时覆盖掉最前面不同的元素
            if (c.contains(elementData[r]) == complement)
                elementData[w++] = elementData[r];
    } finally {
        // Preserve behavioral compatibility with AbstractCollection,
        // even if c.contains() throws.
        //<2>正常情况下遍历完 r == size，r != size，说明发生了异常。
        if (r != size) {
            System.arraycopy(elementData, r,
                             elementData, w,
                             size - r);
            
            w += size - r;
        }
        // <3>.成功删除了元素，将后面空间置空
        if (w != size) {
            // clear to let GC do its work
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w;
            size = w;
            modified = true;
        }
    }
    return modified;
}
```

##### 针对removeAll方法的情况，此时complement = false

<1>. 看到elementData[w++] = elementData[r] 不要懵，跟这往下看。如果容器中存在该元素，并且complement = false时，那么不做处理。如果容器中不存在该元素，并且complement = false时，会把容器w位置的元素替换为r位置的元素，最后得到的结果就是，相同的元素全部被替换了。而不相同的元素，会慢慢跟着w的坐标往左靠... 

<2>.如果发生了异常。那么将r后面未处理的元素，加入到w的后面，也就是没有修改集合。

<3>.最后根据w的值来删除掉末尾多余的元素，妈耶！有点东西的。  

原数组：{1，2，3，4，5，6，7，8，9，10}

c: {1，3，4}

1. 存在1 不处理  r = 0 w = 0  原数组：{1，2，3，4，5，6，7，8，9，10}
2. 不存在 2 处理  r = 1 w = 0  原数组：elementData[0] = 2  ===> {2，2，3，4，5，6，7，8，9，10}  处理后 w = 1 

3.  存在3 不处理  r = 2 w = 1  原数组：{2，2，3，4，5，6，7，8，9，10}
4. 存在4 不处理  r = 3 w = 1  原数组：{2，2，3，4，5，6，7，8，9，10}
5. 不存在 5 处理  r = 4 w = 1 原数组：elementData[1] = 5  ===>{2，5，3，4，5，6，7，8，9，10} 处理后 w = 2
6. 不存在 5 处理  r = 5 w = 2 原数组：elementData[2] = 6  ===>{2，5，6，4，5，6，7，8，9，10} 处理后 w = 3
7. 不存在 5 处理  r = 6 w = 3 原数组：elementData[3] = 7  ===>{2，5，6，7，5，6，7，8，9，10} 处理后 w = 4
8. 不存在 5 处理  r = 7 w = 4 原数组：elementData[4] = 8  ===>{2，5，6，7，8，6，7，8，9，10} 处理后 w = 5
9. 不存在 5 处理  r = 8 w = 5 原数组：elementData[5] = 9  ===>{2，5，6，7，8，9，7，8，9，10} 处理后 w = 6
10. 不存在 5 处理  r = 9 w = 6 原数组：elementData[6] = 10  ===>{2，5，6，7，8，9，10，8，9，10} 处理后 w = 7

最终结果：{2，5，6，7，8，9，10，8，9，10}

##### 针对retainAll方法的情况，此时

<1>.如果容器中存在该元素，并且complement = true时，那么不做处理。如果容器中不存在该元素，并且complement = true时，会把容器w位置的元素替换为r位置的元素，最后得到的结果就是，相同的元素全部被替换了。而不相等的元素，会慢慢跟着w的坐标往左靠... 

原数组：{1，2，3，4，5，6，7，8，9，10}

c: {1，3，4}

1. 存在1 处理  r = 0 w = 0  原数组：elementData[0] = 2  ===>{1，2，3，4，5，6，7，8，9，10} 处理后 w = 1
2. 不存在2 不处理 r = 1w = 1  原数组：{1，2，3，4，5，6，7，8，9，10}
3. 存在3 处理r = 2 w = 1 原数组：elementData[1] = 3  ===>{1，3，3，4，5，6，7，8，9，10} 处理后 w = 2
4. 存在4 处理r = 3 w = 2 原数组：elementData[2] =3  ===>{1，3，4，4，5，6，7，8，9，10}  处理后 w = 3 
5. 接下来因为都不存在，所以后面都不会再处理了

最终结果：{1，3，4，4，5，6，7，8，9，10}

<3>.同上处理方法

### get获取元素

```java
//通过下标获取元素
public E get(int index) {
    //检查下表是否越界
    rangeCheck(index);
    return elementData(index);
}
```

get方法通过下标获取元素，数组中我们常用的方法，就不多作介绍

### iterator迭代器

​		



## 四、应用场景

## 五、常见问题

