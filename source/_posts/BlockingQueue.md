title: 【阻塞队列】-- BlockingQueue总结(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/cs/cs-12.jpg

toc: true 

tags:

  - 源码
  - java
  - 阻塞队列

categories: 

  - [源码,阻塞队列] 

date: 2020-03-13 19:11:21

---

## 概述

​			`BlockingQueue`即是我们所说的阻塞队列，它是一个接口，继承自Queue接口。后面我们要讲的阻塞队列，都实现自该接口。包括：<!-- more -->

- [ArrayBlockingQueue]("https://www.yingu.site/2020/03/13/ArrayBlockingQueue/")
- [LinkedBlockingQueue]("https://www.yingu.site/2020/03/14/LinkedBlockingQueue/")
- [LinkedBlockingDeque]("https://www.yingu.site/2020/03/15/LinkedBlockingDeque/")(实现了`BlockingDeque`，`BlockingDeque`继承自`BlockingQueue`)
- [PriorityBlockingQueue]("https://www.yingu.site/2020/03/15/PriorityBlockingQueue/")
- ...

## 方法

​		`BlockingQueue` 具有不同的方法用于插入、移除以及对队列中的元素进行检查。如果请求的操作不能得到立即执行的话，每个方法的表现也不同。而通过`BlockingQueue` 实现的阻塞队列，也保留了这些特性。

### 生产方法

```java
//向队列中添加元素，如果队列已满直接抛出IllegalStateException异常
boolean add(E e);

//向队列中添加元素，如果队列已满返回false
boolean offer(E e);

//向队列中添加元素，队列已满会阻塞等待，响应中断
void put(E e) throws InterruptedException;

//向队列中添加元素，队列已满会超时阻塞等待，响应中断
boolean offer(E e, long timeout, TimeUnit unit)
throws InterruptedException;
```

### 消费方法

```java
//获取并移除头部元素，如果队列为空会阻塞，响应中断
E take() throws InterruptedException;

//获取并移除头部元素，如果队列为空返回null
E poll();

//获取并移除头部元素，如果队列为空超时阻塞，超时后返回null，并且可以响应中断
E poll(long timeout, TimeUnit unit)
throws InterruptedException;

//获取头部元素，如果队列为空返回null
E peek();

//一次从队列中获取所有元素到指定集合中
int drainTo(Collection<? super E> c);

//一次性获取指定元素个数到指定集合中，该方法不会阻塞
int drainTo(Collection<? super E> c, int maxElements);
```

