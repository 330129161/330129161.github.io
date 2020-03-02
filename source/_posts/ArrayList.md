title: ArrayList源码解析(jdk1.8)

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

 `ArrayList`是我们日常开发中比较常见的一个容器类。它底层由动态数组实现， 所以和数组一样，可以根据索引对容器对象所包含的元素，进行快速随机的查询操作 。`Arraylist`在中间位置插入或者删除元素时，需要对数据进行复制、移动、代价比较高。因此，它适合随机查找和遍历，不适合插入和删除。

## 结构特点

1.  `ArrayList` 继承了`AbstractList`，实现了`List`。提供了相关的添加、删除、修改、遍历等功能。
2.  实现了`RandmoAccess`接口，即提供了随机访问功能 ， 这样`ArrayList`使用`for`循环遍历元素要比使用迭代器遍历元素要快 。
3.  实现了`Cloneable`接口， 表示 `ArrayList` 支持克隆。
4. 实现了 `Serializable` 接口， 表示 `ArrayList` 支持序列化的功能 ，可用于网络传输。
5. `ArrayList`非线程安全，因此只适用于单线程中。如果在多线程，可使用 `CopyOnWriteArrayList`和`Vector`。 `Vector`方法和`ArrayList`基本相同，不过在修改方法上，都使用`synchronized`修饰。 而`CopyOnWriteArrayList` 采用写时拷贝策略，对其进行修改操作和元素迭代，都是在低层创建一个拷贝数组上进行，兼顾了线程安全的同时，又提高了并发性，性能比`Vector`有不少提高 。因此，**多线程情况下推荐使用`CopyOnWriteArrayList`**。

![ArrayList结构](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/ArrayList.png)

## 重要属性

```java
 public class ArrayList<E> extends AbstractList<E>        
     implements List<E>,RandomAccess, Cloneable, java.io.Serializable
 {
     //默认容量大小
     private static final int DEFAULT_CAPACITY = 10;
     //空数组，用于其他构造方式扩容，按照1.5倍扩容
     private static final Object[] EMPTY_ELEMENTDATA = {};
     //空数组，用于无参构造方法初始化，首次扩容为10
     private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
     //底层储存元素的数组
     transient Object[] elementData;
     //实际元素个数
     private int size;
     //最大数组容量
     private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
     
 }
```
`EMPTY_ELEMENTDATA`与`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`相比，扩容策略有所不同

## 常用方法

### 无参数构造方法

```java
public ArrayList() {
    // 将底层数组指向DEFAULTCAPACITY_EMPTY_ELEMENTDATA
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

###  带容量大小的构造函数

根据传入的`initialCapacity`创建`ArrayList`数组

```java
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        //创建一个initialCapacity大小的数组，底层数组指向此数组
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

平时用的比较少的一个方法，通过集合类来生成`ArrayLIst`

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

> 为什么说`c.toArray`返回的类型 不一定是一个`Object[]`。`collection.toArray()`理论上应该返回`Object[]`。然而使用`Arrays.asList`得到的`list`，其`toArray`方法返回的数组却不一定是`Object[]`类型的，而是返回它本来的类型。

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
    //同<1>处
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    /**
    * <2>.将数组elementData从下标index开始的元素，长度为size - index（即index到size之间的元素）复制
    * 到数组elementData以index+1开始的位置，再将element放在数组index的位置。
    */
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
<2>处`arraycopy()`方法：

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

### 扩容

```java
//确保容器的当前容量足够容纳新增的元素
private void ensureCapacityInternal(int minCapacity) {
    //<1>.当前实例是否通过无参构造方法生成，且为第一次扩容
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        //<2>.对比当前最大容量与默认的容量10对比，取大值
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    //确保是否需要扩容
    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    //<3>.增加数组修改次数
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

<1>处，如果`elementData`指向`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`，则表示当前实例是通过无参构造方法创建的，且当前是第一次扩容。

<2>处，将需要的最小容量与默认容量对比，取大值，此时容量默认为10。**`ArrayList`调用无参构造方式时，并没有直接将容量初始化为10，而是通过懒加载的形式 ，在第一次调用add时设置**。

<3>，`AbstractList`包含一个`modCount`变量，它的初始值是0，**当集合中的内容每被修改时（调用`add`， `remove`等方法），`modCount`加1** ,`modCount`的作用是什么，看看官方的说明：

> This field is used by the iterator and list iterator implementation returned by the iterator and listIterator methods. If the value of this field changes unexpectedly, the iterator (or list iterator) will throw a ConcurrentModificationException in response to the next, remove, previous, set or add operations. This provides *fail-fast* behavior, rather than non-deterministic behavior in the face of concurrent modification during iteration.

大概意思就是说：在使用迭代器遍历的时候，用来检查列表中的元素是否发生变化，主要在多线程环境下需要使用，防止一个线程正在迭代遍历，另一个线程修改了这个列表的结构。前面也说过`ArrayList`是非线程安全的。

关于`ArrayList`的`add()`方法就分析到这里了， `addAll(int index, Collection<? extends E> c)`方法思路与上面两个方法基本一致，这里就不分析了。

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
        //通过遍历元素的方式，移除匹配元素
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

- 针对`removeAll`方法的情况，此时`complement = false`

<1>. 看到`elementData[w++] = elementData[r]` 不要懵，跟着往下看。如果容器中存在该元素，并且`complement = false`时，那么不做处理。如果容器中不存在该元素，并且`complement = false`时，会把容器w位置的元素替换为r位置的元素，最后得到的结果就是，相同的元素全部被替换了。而不相同的元素，会慢慢跟着w的坐标往左靠... 

<2>.如果发生了异常。那么将r后面未处理的元素，加入到w的后面，也就是没有修改集合。

<3>.最后根据w的值来删除掉末尾多余的元素。

- 原数组：`{1，2，3，4，5，6，7，8，9，10}`


- c: `{1，3，4}`


我们来看看程序中 `elementData[w++] = elementData[r]` 的执行情况：

| 次数 | c中是否存在 | 是否处理 | 处理前r和w当前值 |                       处理后的值                        | 处理后r和w的值 |
| :--: | :---------: | :------: | :--------------: | :-----------------------------------------------------: | :------------: |
|  1   |    存在1    |  不处理  |  r = 0 ，w = 0   |                                                         | r = 1 ，w = 0  |
|  2   |   不存在2   |   处理   |  r = 1 ，w = 0   |  elementData[0] = 2 >> {2，2，3，4，5，6，7，8，9，10}  | r = 2 ，w = 1  |
|  3   |    存在3    |  不处理  |  r = 2 ，w = 1   |                                                         | r = 3 ，w = 1  |
|  4   |    存在4    |  不处理  |  r = 3 ，w = 1   |                                                         | r = 4 ，w = 1  |
|  5   |   不存在5   |   处理   |  r = 4， w = 1   |  elementData[1] = 5 >> {2，5，3，4，5，6，7，8，9，10}  | r = 5 ，w = 2  |
|  6   |   不存在6   |   处理   |  r = 5， w = 2   |  elementData[2] = 6 >> {2，5，6，4，5，6，7，8，9，10}  | r = 6 ，w = 3  |
|  7   |   不存在7   |   处理   |  r = 6 ，w = 3   |  elementData[3] = 7 >> {2，5，6，7，5，6，7，8，9，10}  | r = 7 ，w = 4  |
|  8   |   不存在8   |   处理   |   r =7 ，w = 4   |  elementData[4] = 8 >> {2，5，6，7，8，6，7，8，9，10}  | r = 8 ，w = 5  |
|  9   |   不存在9   |   处理   |  r = 8 ，w = 5   |  elementData[5] = 9 >> {2，5，6，7，8，9，7，8，9，10}  | r = 9 ，w = 6  |
|  10  |  不存在10   |   处理   |  r = 9 ，w = 6   | elementData[6] = 10 >> {2，5，6，7，8，9，10，8，9，10} | r = 10 ，w = 7 |

**最终结果：`{2，5，6，7，8，9，10，8，9，10}`**

- 针对`retainAll`方法的情况，此时`complement = true`

<1>.如果容器中存在该元素，并且`complement = true`时，那么不做处理。如果容器中不存在该元素，并且`complement = true`时，会把容器w位置的元素替换为r位置的元素，最后得到的结果就是，相同的元素全部被替换了。而不相等的元素，会跟着w的坐标往左靠... 

- 原数组：`{1，2，3，4，5，6，7，8，9，10}`


- c : `{1，3，4}`

`elementData[w++] = elementData[r]` 的执行情况：

|  1   | c中是否存在 | 是否处理 | 处理前r和w当前值 |                      处理后的值                       | 处理后r和w的值 |
| :--: | :---------: | :------: | :--------------: | :---------------------------------------------------: | :------------: |
|  2   |    存在1    |   处理   |  r = 0 ，w = 0   | elementData[0] = 1 >> {1，2，3，4，5，6，7，8，9，10} | r = 1 ，w = 1  |
|  3   |   不存在2   |  不处理  |  r = 1 ，w = 1   |                                                       | r = 2 ，w = 1  |
|  4   |    存在3    |   处理   |  r = 2 ，w = 1   | elementData[1] = 1 >> {1，3，3，4，5，6，7，8，9，10} | r = 3 ，w = 2  |
|  5   |    存在4    |   处理   |  r = 3 ，w = 2   | elementData[2] = 1 >> {1，3，4，4，5，6，7，8，9，10} | r = 4 ，w = 3  |
|  6   |   不存在5   |  不处理  |  r = 4 ，w = 3   |                                                       | r = 5 ，w = 3  |

接下来因为都不存在，所以后面都不会再处理了，**最终结果：`{1，3，4，4，5，6，7，8，9，10}`**

<3>.同上处理方法

### 其他常用方法

```java
//通过下标获取元素
public E get(int index) {
    //检查下标是否越界
    rangeCheck(index);
    return elementData(index);
}

//通过元素寻找下标位置
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}

public Object[] toArray() {
    return Arrays.copyOf(elementData, size);
}

//将容器底层数组的元素复制到a中
public <T> T[] toArray(T[] a) {
    if (a.length < size)
        // Make a new array of a's runtime type, but my contents:
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());
    //将整个底层数组的元素，复制到a中
    System.arraycopy(elementData, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}

public E set(int index, E element) {
    //检查下标是否越界
    rangeCheck(index);
	//记录旧元素
    E oldValue = elementData(index);
    //替换为指定元素
    elementData[index] = element;
    return oldValue;
}

@Override
public void forEach(Consumer<? super E> action) {
    //传入的函数不能为空
    Objects.requireNonNull(action);
    //记录期望值
    final int expectedModCount = modCount;
    @SuppressWarnings("unchecked")
    final E[] elementData = (E[]) this.elementData;
    final int size = this.size;
    //遍历执行，函数方法
    for (int i=0; modCount == expectedModCount && i < size; i++) {
        action.accept(elementData[i]);
    }
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}

```

### iterator迭代器

 `AbstractList` 也提供了一个 `Itr` 的实现，但是 `ArrayList` 为了更好的性能，所以自己实现了 。

```java
//ArrayList的内部类，实现了java.util.Iterator 接口
 private class Itr implements Iterator<E> {
     //下一个要访问元素的下标
     int cursor;       // index of next element to return
     //返回的最后一个元素的索引（如果没有返回-1）
     int lastRet = -1; // index of last element returned; -1 if no such
     //期望修改次数
     int expectedModCount = modCount;

     //是否存在下一个元素
     public boolean hasNext() {
         //如果当前cursor == size说明已经在数组尾部了，无法继续迭代
         return cursor != size;
     }

     //获取下一个元素
     @SuppressWarnings("unchecked")
     public E next() {
         //<1>.校验数组是否发生了变化
         checkForComodification();
         //记录下一个要访问的元素坐标
         int i = cursor;
         //如果下一个元素的坐标，大于或者等于容器中元素的个数，说明获取不到元素了，抛出异常
         if (i >= size)
             throw new NoSuchElementException();
         Object[] elementData = ArrayList.this.elementData;
         //如果i大于容器的总长度，说明可能发生并发操作，改变了elementData数组，抛出异常
         if (i >= elementData.length)
             throw new ConcurrentModificationException();
         //更新下一个要访问元素的坐标
         cursor = i + 1;
         //返回本次获取的元素，并将上一个访问元素的下标置为当前获取元素的坐标
         return (E) elementData[lastRet = i];
     }

     //移除元素
     public void remove() {
         //参数不合法，抛出异常
         if (lastRet < 0)
             throw new IllegalStateException();
         //同上
         checkForComodification();

         try {
             //调用当前容器的remove方法，移除当前元素
             ArrayList.this.remove(lastRet);
             //元素移除后，元素的位置会被下一个元素取代，那么下一次操作的元素的位置，就是当前移除元素的位置
             cursor = lastRet;
             //移除元素时，设置为 -1 ，表示最后访问的元素不存在了
             lastRet = -1;
             /** 
             * <2>.上面调用ArrayList.remove方法会更改modCount，这里需要同步当前的期望值，
             * 否则下一次调用该remove的方法时，就会出现expectedModCount不一致的情况，
             * 从而抛出ConcurrentModificationException异常
             */
             expectedModCount = modCount;
         } catch (IndexOutOfBoundsException ex) {
             /** 
             * <3>.如果ArrayList.this.remove(lastRet)出现下标越界的情况，
             * 说明elementData数组的被修改，抛出ConcurrentModificationException异常
             */
             throw new ConcurrentModificationException();
         }
     }

     //对每个元素执行某个操作
     @Override
     @SuppressWarnings("unchecked")
     public void forEachRemaining(Consumer<? super E> consumer) {
         //传入的元素不能为null
         Objects.requireNonNull(consumer);
         final int size = ArrayList.this.size;
         int i = cursor;
         if (i >= size) {
             return;
         }
         final Object[] elementData = ArrayList.this.elementData;
         //如果超过elementData元素长度，说明数组可能被修改，抛出异常
         if (i >= elementData.length) {
             throw new ConcurrentModificationException();
         }
         while (i != size && modCount == expectedModCount) {
             consumer.accept((E) elementData[i++]);
         }
         // update once at end of iteration to reduce heap write traffic
         cursor = i;
         lastRet = i - 1;
         checkForComodification();
     }

     //迭代期间，如果修改次数与预期值不等，则抛出异常
     final void checkForComodification() {
         if (modCount != expectedModCount)
             throw new ConcurrentModificationException();
     }
 }

//ListItr继承自Itr，实现了ListIterator接口
private class ListItr extends Itr implements ListIterator<E> {
    ListItr(int index) {
        super();
        cursor = index;
    }

    //上一个元素是否存在
    public boolean hasPrevious() {
        return cursor != 0;
    }

    //下一个元素的位置
    public int nextIndex() {
        return cursor;
    }

    //上一个元素的位置
    public int previousIndex() {
        return cursor - 1;
    }

    //上一个元素
    @SuppressWarnings("unchecked")
    public E previous() {
        //校验数组是否发生了变化
        checkForComodification();
        //得到上一个元素位置
        int i = cursor - 1;
        //如果小于0抛出异常
        if (i < 0)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        //如果超过elementData元素长度，说明数组可能被修改，抛出异常
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        //将当前指针前移
        cursor = i;
        return (E) elementData[lastRet = i];
    }

    //设置当前坐标元素
    public void set(E e) {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            ArrayList.this.set(lastRet, e);
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }

    //新增元素
    public void add(E e) {
        checkForComodification();

        try {
            //记录下一个操作的元素
            int i = cursor;
            //插入元素
            ArrayList.this.add(i, e);
            //更新下一个操作的元素
            cursor = i + 1;
            //移除元素时，设置为 -1 ，表示最后访问的元素不存在了
            lastRet = -1;
            //更新预期值
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            //跟<2>一个逻辑
            throw new ConcurrentModificationException();
        }
    }
}


```

- 上面列出了两种迭代器，分别为`Iterator`与`ListIterator`，`ListIterator`继承自`Iterator`。


#### <a name="iteratorDiff">`ListIterator`中对比`Iterator`增加的方法</a>

1. 增加了`nextIndex()`和`previousIndex()`方法，可以获取当前索引的位置。
2.  添加`hasPrevious()`和`previous()`方法，可以通过遍历寻找上一个元素，实现反向遍历
3.  增加了`set()`方法，可以实现元素的修改
4.  增加了`add()`方法，可以向集合中，添加元素

#### 创建迭代器的几种方式

```java
//无参构造方法创建iterator
public Iterator<E> iterator() {
    return new Itr();
}

//无参构造方法创建listIterator
public ListIterator<E> listIterator() {
    return new ListItr(0);
}

//创建listIterator，指定下一个要操作的下标
public ListIterator<E> listIterator(int index) {
    if (index < 0 || index > size)
        throw new IndexOutOfBoundsException("Index: "+index);
    return new ListItr(index);
}

```

- 经常用到过`ArrayList`的读者，可能知道它在遍历的时候，是不能通过 `ArrayList`的`remove` 方法来进行移除元素的操作，因为程序可能会抛出异常。为什么会发生这种情况，我们可以通过以上的源码分析一下。


- 前面我们已经分析过`ArrayList`的`remove`方法，该方法通过匹配后进入`fastRemove`方法：

```java
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

- 执行方法会将`ArrayList`的操作记录数+1。此时,我们通过 `iterator` 迭代，调用`iterator`的`next`方法，


```java
public E next() {
	//<1>.校验数组是否发生了变化
	checkForComodification();
}

final void checkForComodification() {
    if (modCount != expectedModCount)
    	throw new ConcurrentModificationException();
}
```



- 该方法第一步就是调用`checkForComodification`()方法，检测当前的预期值`expectedModCount`与当前集合的修改次数是否一直，来判断当前数组是否被更改。如果此时我们调用`ArrayList`的`remove`方法来移除元素，那么在下一次调用next的时候，就会因为**`modCount != expectedModCount`** ，而抛出异常。

- 同样的情况，如果我们通过先创建一个`iterator` ，此时`iterator` 的`expectedModCount`会初始化为`modCount`，然后通过`forEach`循环中来`remove`元素，那么`modCount`的值发生改变，而 `iterator` 的`expectedModCount` 中的值没有变。那么我们接下来通过`iterator` 遍历，调用`next`的时候，程序就会抛出异常。所以无论我们通过以上两种方式，都会破坏`ArrayList`结构，让`ArrayList`变得不安全，最后在使用`iterator`的`next`的时候抛出异常。

- 我们来看看Itr中自带的`remove`方法:


```java
public void remove() {
    //... 省略其他操作
    checkForComodification();
    //...
    try {
        //....
            ArrayList.this.remove(lastRet);
        //...
            expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```

- 刚开始也会通过`checkForComodification()`方法检查，数组是否改变，接着`ArrayList.this.remove(lastRet)`调用外部`ArrayList`的`remove`方法，接着最重要的一步`expectedModCount = modCount`，更新`expectedModCount` 的值，保证预期值与操作数一致。与外部`ArrayList`的`remove`方法相比，`iterator` 的`remove`的方法增加了这一步，也就是这一步，保证了iterator 遍历时的操作安全。因此，在循环中操作Arraylist删除元素，最安全的方法就是调用内部`iterator`的`remove`方法。




ArrayLIst的源码分析到这里就结束了！🎉🎉撒花。如果各位小伙伴读完后，发现文章中有哪些错误或者不足之处，烦请各位在评论区告知笔者，将不胜感激！

