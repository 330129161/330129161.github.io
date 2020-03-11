title: LinkedList源码解析(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/book.png

toc: true 

tags:

  - 源码
  - java
  - 集合

categories: 

  - [源码,集合] 

date: 2019-12-28 20:41:00

---

## 概述

本章我们进行`LinkedList`的分析，同上篇介绍的[ArrayList](https://www.yingu.site/2019/12/25/ArrayList/)一样，`LinkedList`也是`List`的实现类。不过`ArrayList`是基于数组实现的，而`LinkedList`是基于链表实现的。`LinkedList`比较适合进行增加、删除的操作，因为只需要改变链表中节点的指向。而对于获取元素，`LinkedList`则没那么容易。与`ArrayList`不同，只需要传入坐标位置，就能根据底层数组，获取到指定位置的元素。`LinkedList`需要通过遍历列表的方式来匹配元素，因此效率比较低。最后，LinkedList是线程不安全的。 

## 结构特点

1.  `LinkedList`继承了`AbstractList` 、`AbstractSequentialList` ，实现了`List`。提供了相关的添加、删除、修改、遍历等功能。`AbstractSequentialList` 不像 `AbstractList` 可以实现随机访问。`AbstractSequentialList` 只支持次序访问。如果想访问某个元素， 必须从链头开始顺着指针才能找到。
2. `LinkedList`实现了`Deque`，因此`LinkedList`既是一个列表的同时，也是一个双端队列。提供了相关出队、入队等功能。
3. 实现了`Cloneable`接口， 表示 `LinkedList`支持克隆。
4. 实现了 `Serializable` 接口， 表示 `LinkedList`支持序列化的功能 ，可用于网络传输。

![LinkedList结构](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/LinkedList.png)

## 重要属性

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    //实际元素个数
    transient int size = 0;
    //头节点
    transient Node<E> first;
    //尾节点
    transient Node<E> last;
    
    //内部类node，用于储存元素的节点
    private static class Node<E> {
        //当前节点元素
        E item;
        //上一个节点
        Node<E> next;
        //下一个节点
        Node<E> prev;
		//创建节点，指定上一个和下一个节点
        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
}
```

## 常用方法

### 构造方法

`LinkedList`是由链表组成的无界队列，不需要指定容器，也不需要扩容。

```java
//无参构造方法，用于创建LinkedList实例
public LinkedList() {
}

//通过传入的Collection集合，创建拥有指定节点的LinkedList实例
public LinkedList(Collection<? extends E> c) {
    //使用无参构造方法创建实例
    this();
    //将所有元素添加到LinkedList中
    addAll(c);
}
```

### 添加元素方法

#### <a name="addPerfix">add方法</a>

```java
//将指定的集合中的所有元素添加到LinkedList中
public boolean addAll(Collection<? extends E> c) {
    //将所有的元素添加到LinkedList的末尾
    return addAll(size, c);
}

//将指定集合所有的元素添加到LinkedList的指定位置后
public boolean addAll(int index, Collection<? extends E> c) {
    checkPositionIndex(index);
	//将传入的集合转化为数组
    Object[] a = c.toArray();
    int numNew = a.length;
    //如果传入的元素为空，返回失败
    if (numNew == 0)
        return false;
	
    //pred:插入元素位置的上一个元素 succ：被插入的当前元素
    Node<E> pred, succ;
    if (index == size) {
        //<1>.index==size，说明插入的位置在LinkedList的末尾，此时，index下没有元素
        succ = null;
        //当前index的上一个元素，也就是末尾元素
        pred = last;
    } else {
        //获取下标获取当前元素
        succ = node(index);
        //记录上一个元素
        pred = succ.prev;
    }
	//遍历插入
    for (Object o : a) {
        @SuppressWarnings("unchecked") E e = (E) o;
        //为当前传入的元素创建节点，指定上一个节点，下一节点为null
        Node<E> newNode = new Node<>(pred, e, null);
        if (pred == null)
            //如果上一个元素为null，说明当前LinkedList中不存在元素，指定头结点为当前节点
            first = newNode;
        else
            //否则指定下一个节点为当前节点
            pred.next = newNode;
        //对于下一个要插入的元素来说，上一个节点，就是当前节点
        pred = newNode;
    }
	
    if (succ == null) {
        //如果succ == null，根据<1>处，说明当前是向末尾插入元素，
        //那么最后一个插入的元素，就是新的尾节点
        last = pred;
    } else {
        //否则，最后插入的元素的下一个，就是被插入的元素
        pred.next = succ;
        //同理，被插入的元素的上一个就是最后一个插入的元素
        succ.prev = pred;
    }
	//节点数增加numNew个元素
    size += numNew;
    //操作数+1
    modCount++;
    return true;
}

//向LinkedList末尾添加元素
public boolean add(E e) {
    //向尾部添加节点
    linkLast(e);
    return true;
}

//向指定的位置加入元素
public void add(int index, E element) {
    //校验传入参数
    checkPositionIndex(index);
    if (index == size)
        //index == size，向尾部添加元素
        linkLast(element);
    else
        //否则就是在指定节点前，添加元素
        linkBefore(element, node(index));
}

//检查当前的传入的坐标是否越界
private void checkPositionIndex(int index) {
    if (!isPositionIndex(index))
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

//检查当前坐标是否在元素列表范围内
private boolean isPositionIndex(int index) {
    return index >= 0 && index <= size;
}

//向头部添加元素，用于兼容Deque中的方法
public void push(E e) {
    addFirst(e);
}

//向头部添加元素，用于兼容Deque中的方法
public void addFirst(E e) {
    linkFirst(e);
}
//向尾部添加元素，用于兼容Deque中的方法
public void addLast(E e) {
    linkLast(e);
}
```

- 上述方法中，存在着许多相似的方法，但它们的应用场景有所不同，后面[offer方法](#offerPerfix)中会提出。

#### link方法

```java
//将节点插入到尾部
void linkLast(E e) {
    //记录尾节点
    final Node<E> l = last;
    //创建节点，指定上一节点为当前的尾节点，下一节点为null
    final Node<E> newNode = new Node<>(l, e, null);
    //替换尾节点为当前插入的节点
    last = newNode;
    if (l == null)
        //如果记录的尾节点为null，说明集合中不存在元素，那么首尾为同一节点
        first = newNode;
    else
        //否则,记录的尾节点的下一个节点为新的尾节点
        l.next = newNode;
    //节点数+1
    size++;
    //操作数+1
    modCount++;
}

//将节点插入到头部
private void linkFirst(E e) {
    //记头尾节点
    final Node<E> f = first;
    //创建节点，指定上一节点为null，下一节点为记录的头结点
    final Node<E> newNode = new Node<>(null, e, f);
    //替换头节点为当前插入的节点
    first = newNode;
    if (f == null)
        //如果记录的头节点为null，说明集合中不存在元素，那么首尾为同一节点
        last = newNode;
    else
        //否则,记录的头节点的上一个节点为新的尾节点
        f.prev = newNode;
    //节点数+1
    size++;
    //操作数+1
    modCount++;
}

//将元素插入到指定节点前
void linkBefore(E e, Node<E> succ) {
    //记录被插入节点的上一个节点
    final Node<E> pred = succ.prev;
    //创建节点，指定上一节点为pred，下一节点为succ
    final Node<E> newNode = new Node<>(pred, e, succ);
    //指定被插入节点的上一节点为新建节点
    succ.prev = newNode;
    if (pred == null)
        //如果pred == null，说明插入到了头部，那么设置新的头结点为新节点
        first = newNode;
    else
        //否则，指定pred的下一个节点为新创建的节点
        pred.next = newNode;
    size++;
    modCount++;
}
```

- 从link方法前缀就可以看出，上面的方法是作为一个链表时的操作。

#### <a name="offerPerfix">offer方法</a>（实现Deque中的方法）

```java
//添加元素到末尾
public boolean offer(E e) {
    return add(e);
}

//添加元素到头部
public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}

//添加元素到末尾
public boolean offerLast(E e) {
    addLast(e);
    return true;
}
```

- 看到这里有些读者可能会疑惑，上面的三个方法，看起来作用跟[add的方法](#addPerfix)类似，那么它们的区别是什么，为什么要这么做。
- `LinkedList`实现了`List`和`Deque`接口，兼容了`Deque`中的方法。因此，**`LinkedList`即是一个列表，也是一个队列**。把`LinkedList`当做一个`List`时，通过调用`add`方法压入/获取对象 。而把`LinkedList`当做一个`Deque`的时候，通过调用 `offer`方法，实现队列的入队出队操作。因此上面几个方法，作用看似差不多，但是应用场景不同。

### node节点方法

重要属性中已经介绍了Node节点的相关信息，接下来看看Node的方法:

```java
//获取指定坐标的节点
Node<E> node(int index) {
        // assert isElementIndex(index);
		//<1>.如果当前坐标位置,小于当前元素大小的一半，那么就从头部开始遍历节点，直到找到指定节点
    	//位运算符>>, size >> 1相当于size/2
        if (index < (size >> 1)) {
            //记录头部
            Node<E> x = first;
            //从头部，依次遍历
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            //记录尾部
            Node<E> x = last;
            //从尾部，依次遍历
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

- <1>处,也就是说，如果当前坐标，在链表的中间点前，也就是比较靠近头部，那么从头开始查找，如果当前靠近尾部，那么从尾部开始查找，**最近原则**。

### 获取元素

```java
//获取下标元素
public E get(int index) {
    //<1>.检查传入的index合法性
    checkElementIndex(index);
    //返回指定位置节点的元素
    return node(index).item;
}

//获取头元素
public E element() {
    return getFirst();
}

//获取头元素,如果不存在，抛出异常
public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }

//获取头元素,如果不存在，抛出异常
public E getLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return l.item;
    }

//获取头元素，如果队列为空，返回null
public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}

//获取头元素，并移除。如果队列为空，返回null
public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}
```

-  虽然`LinkedList`没有禁止添加`null`元素，但是一般情况下`Queue`的实现类都不允许添加`null`元素。因为`poll`这种方法在异常的时候返回也是`null`，如果有`null`元素，就很难分辨这些函数返回`null`到底是出现错误了还是在正常运行。 

<1>处，检查元素下标是否在范围内

```java
private void checkElementIndex(int index) {
    if (!isElementIndex(index))
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

private void checkPositionIndex(int index) {
    if (!isPositionIndex(index))
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

### 移除元素

```java
//出队
public E pop() {
    //返回被移除的头节点
    return removeFirst();
}

//移除头节点并返回元素
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    //移除返回头节点
    return unlinkFirst(f);
}

//移除尾节点并返回元素
public E removeLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    //移除返回尾节点
    return unlinkLast(l);
}

//从指定节点位置截断链表，保留后段链表，返回指定节点元素，调用此方法传入的f == first都为头节点
//这也是方法叫做unlinkFirst的原因吧
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    //当前节点元素
    final E element = f.item;
    //记录下一个节点
    final Node<E> next = f.next;
    //当前节点元素,和指向置为null，便于gc回收
    f.item = null;
    f.next = null; // help GC
    //当下一节点置为头节点
    first = next;
    if (next == null)
        //如果next == null，说明移除的节点为尾节点，将last置为null，相当于删除了整个链表节点
        last = null;
    else
        //否则，next成头节点后，不存在上一个节点，因此置为null
        next.prev = null;
    //只移除了头节点，节点数量-1
    size--;
    //操作数+1
    modCount++;
    return element;
}

//从指定节点位置截断链表，保留前段链表，返回指定节点元素，调用此方法传入的f默认 f == last，传入的都是尾节点
//移除尾节点，并返回元素
private E unlinkLast(Node<E> l) {
    // assert l == last && l != null;
    //当前节点元素
    final E element = l.item;
    //记录上一个节点
    final Node<E> prev = l.prev;
    l.item = null;
    l.prev = null; // help GC
    //当上一节点置为尾节点
    last = prev;
    if (prev == null)
        first = null;
    else
        prev.next = null;
    //只移除了尾节点，节点数量-1
    size--;
    modCount++;
    return element;
}

//移除指定节点，并返回节点元素
E unlink(Node<E> x) {
    // assert x != null;
    //记录当前节点元素
    final E element = x.item;
    //记录下一个节点
    final Node<E> next = x.next;
    //记录上一个节点
    final Node<E> prev = x.prev;

    if (prev == null) {
        //如果上一个节点为null，说明当前节点为头节点，被移除之后，下一个节点就是头节点
        first = next;
    } else {
        //否则将上一个节点的指向下一个
        prev.next = next;
        //无效的节点的引用置为null
        x.prev = null;
    }
	
    if (next == null) {
        //如果下一个节点为null，说明当前节点为尾节点，那么上一个节点就是尾节点了
        last = prev;
    } else {
        //否则下一个节点的前驱节点就是上一个节点了
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}

//移除并返回指定下标节点
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}

//移除指定节点，如果不存在返回fasle
public boolean remove(Object o) {
    if (o == null) {
        //遍历为元素为null的节点
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                //移除素为null节点，返回true
                unlink(x);
                return true;
            }
        }
    } else {
        //遍历为元素为o的节点
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                //移除元素为o节点，返回true
                unlink(x);
                return true;
            }
        }
    }
    return false;
}

//移除元素节点，从链表头开始
public boolean removeFirstOccurrence(Object o) {
    return remove(o);
}

//移除元素节点，从链表尾开始
public boolean removeLastOccurrence(Object o) {
    if (o == null) {
        for (Node<E> x = last; x != null; x = x.prev) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = last; x != null; x = x.prev) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```

### iterator迭代器

跟`ArrayList` 一样，`LinkedList`也实现了自己的迭代器，关于`ListIterator`在[ArrayList源码分析](https://www.yingu.site/2019/12/25/ArrayList/)中也有说明。

```java
private class ListItr implements ListIterator<E> {
    //上次访问的节点
    private Node<E> lastReturned;
    //下一个要迭代的节点
    private Node<E> next;
    //下一个要迭代的位置
    private int nextIndex;
    //期望值
    private int expectedModCount = modCount;

    //唯一的构造方法，创建一个迭代器从index位置开始迭代
    ListItr(int index) {
        // assert isPositionIndex(index);
        //<1>.如index == size，那么说明从尾部开始，下一个迭代的节点为null，否则获取下一个要迭代的节点
        next = (index == size) ? null : node(index);
        //设置下一个要迭代的位置
        nextIndex = index;
    }

    //是否存在下一个结点
    public boolean hasNext() {
        return nextIndex < size;
    }

    //获取下一个元素
    public E next() {
        //<1>.校验链表是否发生了变化
        checkForComodification();
        //如果不存在下一个元素，则抛出异常
        if (!hasNext())
            throw new NoSuchElementException();
		//上次访问的节点设置为当前操作的节点
        lastReturned = next;
        //替换next为下一个要迭代的节点
        next = next.next;
        //下一个要迭代的位置 + 1，向后
        nextIndex++;
        //返回当前操作节点的元素
        return lastReturned.item;
    }

    //是否存在上一个节点
    public boolean hasPrevious() {
        return nextIndex > 0;
    }

    //获取上一个元素
    public E previous() {
        //<2>.校验链表是否发生了变化
        checkForComodification();
        //如果不存在上一个元素，则抛出异常
        if (!hasPrevious())
            throw new NoSuchElementException();
		//设置上次访问的节点和最后一个迭代的节点
        //<3>.如果next == null，说明<1>处，创建迭代器时index == size，此时下一个要迭代的就是尾元素
        lastReturned = next = (next == null) ? last : next.prev;
        //下一个要迭代的位置 - 1，向前
        nextIndex--;
        //返回上次访问的节点的元素
        return lastReturned.item;
    }

    //获取下一个要迭代的元素位置
    public int nextIndex() {
        return nextIndex;
    }

    //获取上一个要迭代的元素位置
    public int previousIndex() {
        return nextIndex - 1;
    }

    //移除元素
    public void remove() {
        //校验链表是否发生了变化
        checkForComodification();
        //如果上次访问的节点为null，说明当前没有迭代过元素，会抛出异常
        if (lastReturned == null)
            throw new IllegalStateException();
		//记录下一个元素
        Node<E> lastNext = lastReturned.next;
        //调用外部的LinkedList的unlink方法，移除上次访问的节点
        unlink(lastReturned);
        if (next == lastReturned)
            /**
           	* 如果next == lastReturned，看<3>处，说明此时元素是向前迭代
            * 那么next此时，就应该是当前移除元素的下一个节点，
            * 而下一次，迭代的节点，就是它的前驱节点,
            * 不明白可以看看<3>处
            */
            next = lastNext;
        else
             /**
             * 调用next方法中，nextIndex会+1，此时移除节点后，
             * 让nextIndex保持在当前元素准备移除前的位置
             */
            nextIndex--;
        //设置上次访问的节点为null
        lastReturned = null;
        //因为上面unlink方法操作数会+1，此时期望值也+1，保持同步，
        expectedModCount++;
    }

    //设置最后迭代的元素值
    public void set(E e) {
        if (lastReturned == null)
            throw new IllegalStateException();
        checkForComodification();
        lastReturned.item = e;
    }

    //从下一个处理节点后插入新节点
    public void add(E e) {
        checkForComodification();
        //设置上次访问的节点为null
        lastReturned = null;
        if (next == null)
            //如果next = null，<1>处，说明构造方法指定元素迭代位置在末尾，从末尾后添加元素
            linkLast(e);
        else	
            //否则，从指定元素处添加元素
            linkBefore(e, next);
        nextIndex++;
        expectedModCount++;
    }

    //对每个元素执行某个操作
    public void forEachRemaining(Consumer<? super E> action) {
        //传入的函数不能为null
        Objects.requireNonNull(action);
        //如果nextIndex>= size 或者，期望值与操作值不同，说明链表可能被修改，抛出异常
        while (modCount == expectedModCount && nextIndex < size) {
            //执行操作
            action.accept(next.item);
            //记录上次访问的节点
            lastReturned = next;
            //更新下一次访问的节点
            next = next.next;
            //向后迭代
            nextIndex++;
        }
        //检查链表是否被修改
        checkForComodification();
    }

    //跟ArrayList中迭代器一样，LinkedList也是非线程安全
    //迭代期间，如果修改次数与预期值不等，则抛出异常
    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

- `LinkedList`和`ArrayList`迭代器中一样，都有一个预期值`expectedModCount`，同时他们作用也一样，检查内部结构是否被改变，同样`LinkedList`中，想要遍历移除节点，最安全的方式就是通过迭代器，使用它自己的`remove()`方法。因为它有一个`expectedModCount++`  让期望值与操作值保持一致。
- 同`ArrayList`中的`remove()`方法一样,`LinkedList`中的`unlink()`、`remove()`方法都是将`modCount+1`。如果创建`LinkedList`的迭代器以后，调用`LinkedList`的`remove()`方法，那么调用迭代器的`next()`方法，就会因为`modCount != expectedModCount`而抛出异常。

#### 迭代器的创建

```java
//返回以正向顺序在此双端队列的元素上进行迭代的迭代器。元素将从第一个（头部）到最后一个（尾部）的顺序返回。
public ListIterator<E> listIterator(int index) {
    //检查传入的参数是否合法
    checkPositionIndex(index);
    return new ListItr(index);
}

//返回以逆向顺序在此双端队列的元素上进行迭代的迭代器。元素将从最后一个（尾部）到第一个（头部）的顺序返回。
public Iterator<E> descendingIterator() {
    return new DescendingIterator();
}

//通过代理ListItr来倒置下一节点的访问顺序
private class DescendingIterator implements Iterator<E> {
    private final ListItr itr = new ListItr(size());
    public boolean hasNext() {
        return itr.hasPrevious();
    }
    public E next() {
        return itr.previous();
    }
    public void remove() {
        itr.remove();
    }
}
```

- 作为双端队列，有一个反向遍历的方法，也很正常。实际上，通过`listIterator`创建的迭代器，通过判断上一个元素是否存在，以及获取上一个元素，已经可以实现队列的反向遍历了。这里加入一个`DescendingIterator`的内部类，可以在不修改业务逻辑的情况下。通过更换迭代器，实现反向遍历。

### 总结

`LinkedList`的分析讲到这里，也就已经结束了。上面分析的方法，也不是`LinkedList`的全部方法，还有部分方法没有讲到的。例如，`toArray`、`clone`、序列化等方法，因为这些方法也不是`LinkedList`特有的，同时在`LinkedList`中，跟其他方法的关联度较小，所以就在这里偷了个懒。之后有时间，也会将所有未讲解的方法补全。如果各位小伙伴读完文章后，发现文章中有哪些错误或者不足之处，还请在评论区中留言。笔者看到也会尽快回复。