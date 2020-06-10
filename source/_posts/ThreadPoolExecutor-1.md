title: 【线程池】-- ThreadPoolExecutor源码解析(上)(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/book.png

toc: true 

tags:

  - 源码
  - java
  - 线程池

categories: 

  - [源码,线程池] 

date: 2020-03-16 18:32:14

---

## 概述

线程池就是一个缓存的概念，将使用完的线程放入到线程池中管理，这样有下一个任务需要执行时，直接从线程池中获取线程执行就行，避免重复的执行线程创建、销毁操作，做到线程复用，从而提高线程的利用率，还能通过线程池来对执行任务的线程进行控制，避免线程被滥用。<!-- more -->java中通过`Executors`类提供了多种实现线程池的方式，以下列举常用的四种：

1. `newFixedThreadPool`： 创建一个可重用固定线程数的线程池 
2.  `newSingleThreadExecutor` :   创建一个使用单个线程的线程池。 
3. `newCachedThreadPool`： 创建一个可根据需要创建新线程的线程池
4. `newScheduledThreadPool`：  创建一个任务可延迟的线程池

以上创建线程池方法，都是以`ThreadPoolExecutor`为基础，等本章解析完，再来介绍上面几种创建线程池的用法。

## 结构

![](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/ThreadPoolExecutor.png)

## 基本属性

```java
public class ThreadPoolExecutor extends AbstractExecutorService {

    //使用高3位存储线程池工作状态，使用低29位保存工作线程数量
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    //29位，用于位于操作
    private static final int COUNT_BITS = Integer.SIZE - 3;
    //低29位用于保存线程的数量
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    //高3位：111：接受新任务并且继续处理阻塞队列中的任务(负数位移，高位补1)
    private static final int RUNNING    = -1 << COUNT_BITS;
    //高3位：000：不接受新任务但是会继续处理阻塞队列中的任务
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    //高3位：001：不接受新任务，不在执行阻塞队列中的任务，中断正在执行的任务
    private static final int STOP       =  1 << COUNT_BITS;
    //高3位：010：所有任务都已经完成，线程数都被回收，线程会转到TIDYING状态会继续执行模板方法
    private static final int TIDYING    =  2 << COUNT_BITS;
    //高3位：110：线程会转到TIDYING状态后，会执行模板方法，执行完毕后转为TERMINATED状态
    private static final int TERMINATED =  3 << COUNT_BITS;
    
    //用来保存等待被执行的任务的阻塞队列
    private final BlockingQueue<Runnable> workQueue;
    
    private final ReentrantLock mainLock = new ReentrantLock();

    //存储工作任务的集合，每次针对workers操作都需要加锁
    private final HashSet<Worker> workers = new HashSet<Worker>();

    private final Condition termination = mainLock.newCondition();

    //记录同时运行过的最大工作线程数
    private int largestPoolSize;

    //完成任务数
    private long completedTaskCount;
    
    //创建线程的工厂
    private volatile ThreadFactory threadFactory;

    //构造线程持时如果未指定拒绝策略，会使用该策略
    private volatile RejectedExecutionHandler handler;

    //存活时间
    private volatile long keepAliveTime;

    //是否允许核心线程超时，默认为false,只有非核心线程才会超时
    private volatile boolean allowCoreThreadTimeOut;

    //核心线程数
    private volatile int corePoolSize;

    //最大线程数，当前阻塞队列满了后，提交的任务
    private volatile int maximumPoolSize;
    
    //默认执行的拒绝策略
    private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();
    
    private static final RuntimePermission shutdownPerm =
        new RuntimePermission("modifyThread");
    
    private final AccessControlContext acc;
}
    
```

### Worker内部类

```java
//继承自AQS、实现了Runnable
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
    private static final long serialVersionUID = 6138294804551838833L;

    //每个worker有自己的内部线程，ThreadFactory创建失败时为null
    final Thread thread;
    //初始化任务，可能为null
    Runnable firstTask;
    //完成任务数
    volatile long completedTasks;

    //Worker构造方法，传入一个Runnable
    Worker(Runnable firstTask) {
        //<1>禁止线程在runWorker前被打断
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        //通过工厂创建线程，传入自身，
        //因此通过thread.start后会调用Worker的run方法
        this.thread = getThreadFactory().newThread(this);
    }

    //Runnable的run方法
    public void run() {
        //执行任务，传入Worker自身
        runWorker(this);
    }

    //锁是否被持有
    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    //尝试获取锁
    protected boolean tryAcquire(int unused) {
        //<2>.不允许重入
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    //释放锁
    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        //直接设置为0
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    //如果线程已经开始了，那么中断
    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```

`Worker`继承自`AQS`、实现了`Runnable`，通过使用`AQS`来实现锁，从上面代码，可以获取到的几条信息：

1. `Worker`创建时就已经上锁了
2. `Worker`锁不允许重入
3. `Worker`内部`thread`创建时，传入的是`Worker`自身

以下问题建议先看完后面解析，再回来看，或者你可以带着问题先往下看。

**为什么`Worker`在创建的时候就需要上锁呢?**

- 从`interruptIdleWorkers()`方法就知道，任务未开始或者正在执行的时候不应该被中断，通过锁的状态来表示任务的运行状态
- 在`runWorker()`之前，`Worker`的锁状态一直为-1，表示未开始。随后`runWorker()`方法会释放`Worker`的锁，将它状态变为-1，表示可以被中断。获取到任务后`runWorker()`又会获取锁，表示任务正在执行，不可被中断，并且不可并发执行，此时锁状态为1。
- 所以`Worker`锁有三种状态：-1表示`Worker`被初始化后还未执行，不可被中断；0表示`Worker`准备开始执行(可被中断)或者已经执行结束；1表示`Worker`正在执行，不可被中断。

**为什么`Worker`锁不允许重入?**

-  如果`Worker`为重入锁，那么`runWorker()`中反复执行阻塞队列中的任务的时候， 就有可能同一个线程同时执行多个任务。

**为什么`Worker`内部`thread`创建时，传入的是`Worker`自身？**

- 用户调用`execute()`方法,执行一个任务时，会传入一个`Runnable`，通过`addWorker()`方法后，会通过`Runnable`创建一个`Worker`，因此`Worker`内部既有一把锁，也有一个`Runnable`任务，执行`thread.start`时，实际上执行的是`Runnable`中的任务，而`Worker`又可以作为一把锁，保证执行的任务不会被中断

## 方法

### ctl相关方法

```java
// 根据ctl计算runState，使用&运算时，~CAPACITY表示只有高3位参与了计算
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 根据ctl计算workerCount，表示只有低29位参与了计算
private static int workerCountOf(int c)  { return c & CAPACITY; }
// 根据runState和workerCount计算clt值，rs的高3位与wc的低29通过|计算就是ctl的结果
private static int ctlOf(int rs, int wc) { return rs | wc; }

private static boolean runStateLessThan(int c, int s) {
    return c < s;
}

private static boolean runStateAtLeast(int c, int s) {
    return c >= s;
}

//判断是否为运行状态
private static boolean isRunning(int c) {
    return c < SHUTDOWN;
}

//工作线程数+1，失败返回false
private boolean compareAndIncrementWorkerCount(int expect) {
    return ctl.compareAndSet(expect, expect + 1);
}

//工作线程数-1，失败返回false
private boolean compareAndDecrementWorkerCount(int expect) {
    return ctl.compareAndSet(expect, expect - 1);
}

//工作线程数-1，直到成功为止
private void decrementWorkerCount() {
    do {} while (! compareAndDecrementWorkerCount(ctl.get()));
}

//判断线程池是否被关闭的方法
public boolean isShutdown() {
    return ! isRunning(ctl.get());
}
```

### 构造方法

```java
//该构造方法无需指定线程工厂，使用默认线程工厂defaultThreadFactory
//该构造方法无需指定拒绝策略，使用默认的defaultHandler
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), defaultHandler);
}

//该构造方法无需指定拒绝策略，使用默认的defaultHandler
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         threadFactory, defaultHandler);
}

//该构造方法无需指定线程工厂，使用默认线程工厂defaultThreadFactory
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          RejectedExecutionHandler handler) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), handler);
}

//corePoolSize：核心线程数
//maximumPoolSize：最大线程数
//keepAliveTime：线程存活时间
//unit：线程存活时间单位
//workQueue：线程等待队列
//threadFactory：线程工厂
//handler： 拒绝策略
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
        null :
    AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}

//默认线程工厂
public static ThreadFactory defaultThreadFactory() {
    //该类在Executors.class中
    return new DefaultThreadFactory();
}
```

### submit方法

`submit()`方法在父类`AbstractExecutorService.clss`中，跟`execute()`方法不同的是`submit()`方法有返回值。通过返回的`Future`对象调用它的`Future.get()`方法，可以获取到任务执行完成后的返回值。

本章主要通过`execute()`方法，来展开分析线程池的运作机制，因此`submit()`方法具体细节留到了下一章

```java
//Runnable没有返回值
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    //将Runnable包装成RunnableFuture，Runnable没有返回值，默认返回null
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    //最终还是执行的execute(Runnable command)方法
    execute(ftask);
    return ftask;
}

//Runnable没有返回值，可以指定一个result作为Runnable返回值，
//任务完成后可以通过Future.get()获取到result
public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task, result);
    execute(ftask);
    return ftask;
}

//Callable自带返回值，任务完成后可以通过Future.get()获取到Callable中的返回值
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

### execute方法

执行一个任务，大致分为三步：

1. 当工作线程数小于核心线程数时，会通过`addWorker(command, true)`新建一个核心线程执行任务，成功后直接返回
2. 如果核心线程没有空闲，且线程池还处于运行状态，那么通过`workQueue.offer(command)`将任务添加到阻塞队列 `BlockingQueue` 中等待执行，重新检查线程池状态后，如果此时线程池被关闭了，那么 `remove(command)`移除刚才添加的任务，如果未关闭，判断是否存在工作线程，如果不存在那么添加一个非核心线程执行任务,用于执行刚才`offer`到队列中的任务。
3. 通过`addWorker(command, false)`创建一个非核心线程执行任务，如果失败，说明线程达到最大线程数，那么执行拒绝策略

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
	//检查当前线程池状态
    int c = ctl.get();
    //<1>.如果工作线程小于核心线程数
    if (workerCountOf(c) < corePoolSize) {
        //创建一个核心线程来执行当前command，如果执行成功，直接返回
        //返回false，说明添加任务被拒绝了，或者command未执行
        if (addWorker(command, true))
            return;
        //检查线程池状态
        c = ctl.get();
    }
    //<2>.如果当前为运行状态，添加任务到阻塞队列中，如果失败说明阻塞队列已满
    if (isRunning(c) && workQueue.offer(command)) {
        //重新检查线程池状态，因为状态有可能被其他线程操作改变
        int recheck = ctl.get();
        //如果不是运行状态，被其他线程给中断了，移除刚刚添加到阻塞队列中的任务
        if (! isRunning(recheck) && remove(command))
            //移除成功后执行拒绝方法
            reject(command);
        //如果是运行状态但是没有线程来执行这个任务
        else if (workerCountOf(recheck) == 0)
            //创建一个非核心线程执行任务，传入的firstTask为null，
            //是因为刚才通过offer已经将任务添加到阻塞队列中了，直接去队列中获取任务就行了
            addWorker(null, false);
    }
    //<3>如果阻塞队列也满了，那么创建一个非核心线程执行任务
    //失败说明超过了最大线程数，那么执行拒绝策略
    else if (!addWorker(command, false))
        //详解看下面拒绝策略
        reject(command);
}

//将任务从队列中移除
public boolean remove(Runnable task) {
    boolean removed = workQueue.remove(task);
    //尝试终止线程池
    tryTerminate(); // In case SHUTDOWN and now empty
    return removed;
}
```

### addWorker方法

添加一个工作线程用于执行任务，`firstTask`表示要执行的任务(可以为null )，`core`表示是否使用核心线程来作为是否新建任务的条件。那么通过传入不同的参数，有以下四种情况，`execute`方法用到了下面的三种：

1. `addWorker(command, true)`：创建核心线程来执行`command`任务，如果核心线程没有空闲，那么返回`false`
2. `addWorker(null, false)`：创建非核心线程来执行阻塞队列`workQueue`中的任务，如果已经达到最大线程数，那么返回`false`
3. `addWorker(command, false)`：创建非核心线程来执行`command`任务，如果已经达到最大线程数，那么返回`false`
4. `addWorker(null, true)`，创建核心线程来执行阻塞队列`workQueue`中的任务，如果核心线程没有空闲，那么返回`false`

```java
private boolean addWorker(Runnable firstTask, boolean core) {
     retry:
     for (;;) {
         //检查线程池状态
         int c = ctl.get();
         //获取运行状态
         int rs = runStateOf(c);
         /**
          *	  1.rs >= SHUTDOWN 表示线程已关闭,不能接受新的任务
          *   2.如果rs != SHUTDOWN,说明线程状态为STOP, TIDYING或TERMINATED的时候
          *   这个时候不会接受新的任务，直接返回false
          *   3.firstTask != null(表示为新的任务),线程状态为SHUTDOWN，
          *   但是firstTask != null的时候，也不能接受任务
          *   4.线程状态为SHUTDOWN，并且firstTask == null
          *   但是等待线程为空的时候（说明等待队列中已经没有可执行的任务了），也不能接受新的任务
          *   5.总结下来就是，如果rs为SHUTDOWN，只有firstTask == null，并且等待线程不为空的时候
          *   才能接受本次添加工作任务的操作，这样也跟上面说的语义符合：SHUTDOWN不接受新任务但是
          *   会继续处理阻塞队列中的任务
         */
         if (rs >= SHUTDOWN &&
             ! (rs == SHUTDOWN &&
                firstTask == null &&
                ! workQueue.isEmpty()))
             return false;
		 //自旋重试
         for (;;) {
             //记录工作线程数
             int wc = workerCountOf(c);
             //1.wc >= CAPACITY，如果大于或者等于最大容量了，返回false
             //2.是创建核心线程，如果是，判断当前数量是否大于等于核心线程数，
             //否则判断是否大于等于最大线程数
             if (wc >= CAPACITY ||
                 wc >= (core ? corePoolSize : maximumPoolSize))
                 return false;
             //工作线程+1，失败说明工作线程的值被其他线程改变了，跳出当前循环
             if (compareAndIncrementWorkerCount(c))
                 break retry;
             //重新获取线程池状态，因为状态有可能被其他线程改变
             c = ctl.get();  // Re-read ctl
             //如果运行状态被改变了，重试
             if (runStateOf(c) != rs)
                 continue retry;
             // else CAS failed due to workerCount change; retry inner loop
         }
     }

    //任务是否开始
     boolean workerStarted = false;
    //是否成功添加了任务
     boolean workerAdded = false;
     Worker w = null;
     try {
         //创建一个新的任务线程
         w = new Worker(firstTask);
         //获取执行任务线程（由线程工厂创建，可能会失败）
         final Thread t = w.thread;
         //如果创建线程失败，会进入到finally中执行addWorkerFailed方法
         if (t != null) {
             final ReentrantLock mainLock = this.mainLock;
             //加锁，避免出现并发问题
             mainLock.lock();
             try {
                 // Recheck while holding lock.
                 // Back out on ThreadFactory failure or if
                 // shut down before lock acquired.
                 //检查运行状态
                 int rs = runStateOf(ctl.get());
				 //1.线程池状态为RUNNING
                 //2.线程池状态为SHUTDOWN，并且firstTask == null
                 //前面说了firstTask == null表示任务已经加入到阻塞队列中了，直接去取任务就行了
                 //说明线程池在SHUTDOWN状态下，也能执行阻塞队列中的任务
                 if (rs < SHUTDOWN ||
                     (rs == SHUTDOWN && firstTask == null)) {
                     //线程还没start情况下，如果t的状态为alive，说明线程状态异常
                     if (t.isAlive()) // precheck that t is startable
                         throw new IllegalThreadStateException();
                     //将要执行的任务记录到workers中，由于workers是一个HashSet，
                     //因此需要lock保证线程安全
                     workers.add(w);
                     int s = workers.size();
                     //largestPoolSize替换为较大值,记录线程池创建过的最大线程数
                     if (s > largestPoolSize)
                         largestPoolSize = s;
                     //标记创建线程成功
                     workerAdded = true;
                 }
             } finally {
                 mainLock.unlock();
             }
             //如果添加了任务，那么执行
             if (workerAdded) {
                 //最终会调用w的run方法，执行runWorker方法
                 t.start();
                 //标记任务已开始
                 workerStarted = true;
             }
         }
     } finally {
         //如果任务没有开始，说明任务没有创建成功
         //那么前面将任务线程数+1的操作就要减掉，如果任务线程已经加入到了workers中，也需要移除
         if (! workerStarted)
             //执行任务失败的操作
             addWorkerFailed(w);
     }
     return workerStarted;
 }
```

`addWorker`中方法分为两大步骤：外层的for循环，try代码块

**for循环结束的四种情况：**

1. 线程池状态为`SHUTDOWN`，并且`firstTask != null`，直接返回`false`，结束`addWorker`方法
2. 线程池状态为`STOP`、`TIDYING`或`TERMINATED`，直接返回`false`，结束`addWorker`方法
3. 添加到核心线程，如果核心线程没有空闲，那么直接返回false。如果添加到非核心线程，如果超过最大线程数，那么直接返回false
4. 线程池如果允许添加任务线程，则将工作线程数+1，如果成功跳出for循环，准备执行try代码块中的逻辑

通过上面三种情况也能看出，for循环的主要作用，确保当前线程池状态允许接收新的任务，如果可以就将工作线程数+1直到成功，否则直接结束`addWorker`方法。而只有两种情况还能执行下一步，要么线程池状态为`RUNNING`，要么线程池状态为`SHUTDOWN`并且`firstTask == null` 。

**try代码块：**

1. 创建一个任务线程`Worker`，如果`Worker`内部通过线程工厂创建失败，那么会直接执行最后的`finally`块中的 `addWorkerFailed(w)`方法。
2. `Worker`创建成功后，在两种情况下才可以执行任务，要么线程池状态为`RUNNING`，要么线程池状态为`SHUTDOWN`，且`firstTask == null`（表示执行阻塞队列中的任务），否则会执行最后的`finally`块中的 `addWorkerFailed(w)`方法。
3. 将要执行的任务记录到`workers`集合，由于`workers`是一个`HashSet`，因此需要`lock`加锁保证线程安全
4. 通过`Worker`中的`Thread.start()`启动执行任务，最终会调用`runWorker(Worker w)`传入当前的`Worker`
5. 任务创建失败后，也会进入到 `addWorkerFailed(w)`方法，将任务线程数-1，如果任务线程已经加入到了workers中，也需要移除。

### runWorker方法

```java
//执行任务
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    //记录Worker中的任务，可能为null
    Runnable task = w.firstTask;
    w.firstTask = null;
    //解锁将w的状态置为0(Worker初始化时会将state置为-1)，表示开始运行，允许被中断
    w.unlock(); // allow interrupts
    //是否出现异常
    boolean completedAbruptly = true;
    try {
        /**
         * task != null说明执行的是新的任务
         * (task = getTask()) != null，从阻塞队列中取出一个任务
         * 也就是说如果传入的task为null，并且从阻塞队列中获取不到任务的情况下，就会退出while循环
        */
        while (task != null || (task = getTask()) != null) {
            //加锁
            w.lock();
            /**
             *	1.如果当前线程池的状态>=STOP,并且未中断任务线程，那么就中断任务线程
             * 	2.检查当前线程是否被中断，再次检查线程池的状态，如果>=STOP，
             *	并且未中断任务线程，那么就中断任务线程
             *	从这一步可以看出，如果线程池的状态>=STOP,那么不会执行阻塞队列中的任务并且中断
             *	正在执行任务的线程
            */
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                //任务执行之前模板方法
                beforeExecute(wt, task);
                //记录异常
                Throwable thrown = null;
                try {
                    //执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    //任务执行后的模板方法
                    afterExecute(task, thrown);
                }
            } finally {
                //将task置为null，进入while循环，执行下一次任务
                //这就是线程池回收线程的关键
                task = null;
                //完整任务数+1
                w.completedTasks++;
                w.unlock();
            }
        }
        //将是否出现异常标记置为false
        completedAbruptly = false;
    } finally {
        //执行最终的退出方法
        processWorkerExit(w, completedAbruptly);
    }
}
```

1. 释放`Worker`中的锁，表示`Worker`的线程允许被中断
2. 判断传入的`Worker`中是否存在`task`，如果不存在，那么去通过`getTask()`方法去阻塞队列中获取，如果获取失败，直接执行`processWorkerExit(w, completedAbruptly)`退出方法
3. 执行任务前会判断线程池的状态，如果此时线程池的状态为`STOP`, `TIDYING`、`TERMINATED`，并且未中断任务线程，那么就中断任务线程。
4. 如果第一次检查线程池的状态为`RUNNING`或者`SHUTDOWN` ，那么检查当前线程是否被中断，如果被中断，那么再次检查线程池的状态，如果状态为`STOP`, `TIDYING`、`TERMINATED`，并且未中断任务线程，那么就中断任务线程。
5. 如果线程池状态始终为`RUNNING`或者`SHUTDOWN` ，接下来就要准备执行任务，执行任务前会先执行前置`beforeExecute(wt, task)`模板方法，然后通过`task.run()`执行任务，任务执行完毕后，会执行后置 `afterExecute(task, thrown)`方法，无论执行任务期间是否抛出异常，最终都会将task置为null，并且将完成任务数量+1，task置为null后又会进入到while循环，继续从队列中获取任务执行
6. 整个任务结束前，会执行`processWorkerExit(w, completedAbruptly)`退出方法。

通过该方法可以窥探出线程池复用线程的原理，也就是上面的第5步，将task置为null后，当前线程执行任务后不会立马退出，而是经过上面的while循环，继续去阻塞队列中获取未执行的任务，直到队列中不存在任务为止。

### getTask方法

```java
//从阻塞队列中取出任务
private Runnable getTask() {
    //上次轮询是否超时
    boolean timedOut = false; // Did the last poll() time out?
	//自旋重试
    for (;;) {
        int c = ctl.get();
        //检查线程池运行状态
        int rs = runStateOf(c);
        
        // Check if queue empty only if necessary.
        //仅在必要的时候检查队列是否为空
        //只有在状态为SHUTDOWN时才会检查队列是否为空
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            //如果线程池状态为STOP, TIDYING、TERMINATED，或者状态为
            //SHUTDOWN，但是队列为空的时候，就会执行退出
            //工作线程数-1
            decrementWorkerCount();
            //返回null
            return null;
        }
		//获取工作线程数量
        int wc = workerCountOf(c);
		
        // Are workers subject to culling?
        //当前是否线程是否允许超时
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
		//这几种情况下才可以获取任务
        //1.当前工作线程小于最大线程数并且没有经历过超时
        //2.上次虽然超时了，但是阻塞队列不为空
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            //工作线程数-1，设置失败后重试
            if (compareAndDecrementWorkerCount(c))
                //返回null
                return null;
            continue;
        }
		//到了这一步说明当前工作线程还未达到最大线程数，并且未经历过超时
        try {
            //如果设置了超时，那么超时获取队列中的任务，否则使用take()从队列中获取任务
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
            workQueue.take();
            //如果不为null,任务获取成功，返回任务
            if (r != null)
                return r;
            //标记任务获取超时，进入下一次循环
            timedOut = true;
        } catch (InterruptedException retry) {
            //如果线程被中断，那么将超时标记设为false
            //可能执行了shutdownNow方法
            timedOut = false;
        }
    }
}
```

获取任务中有三种情况会结束方法：

1. 如果线程池状态为`STOP`, `TIDYING`、`TERMINATED`，或者状态为`SHUTDOWN`，且队列为空的时候，将工作线程数-1，成功后会返回null。
2. 工作线程大于最大线程的，并且执行工作线程-1操作成功时，将工作线程数-1，成功后会返回null。
3. 上次获取任务超时并且还存在工作线程时，并且执行工作线程-1操作成功时，将工作线程数-1，成功后会返回null。
4. 上次获取任务超时并且阻塞队列为空的时候(上次获取任务失败，说明队列中已经没有任务可以获取了，再判断一次，如果没有的话直接退出)
5. 从阻塞队列中获取到任务，返回队列中的任务

从上面方法了解到：

1. 核心线程默认不会超时，除非设置`allowCoreThreadTimeOut=true`
2. 是否为核心线程只是一个逻辑并不存在具体的标记来划分，当前是否为核心线程，是根据当前活动线程数是否大于核心线程数来判定
3. 判定结果为非核心线程时，从阻塞队列中获取任务的操作会超时，超时后返回null，会回到`runWorker()`方法中会跳出while循环，而结束掉该线程。
4. 判定结果为核心线程时，从阻塞队列中获取任务的操作不会超时，知道获取到值为止，因此一般情况下核心线程不会结束，除非设置`allowCoreThreadTimeOut=true`
5. 由于判断是否为核心线程是有当前活动线程来决定的，因此任何线程都有可能成为核心线程，之前为核心线程获取到任务之后，下一次执行不一定就是核心线程。

### 拒绝策略

```java
//当达到最大线程数后，不会在接受新的任务，此时会通过此方法，执行拒绝策略
final void reject(Runnable command) {
    //通过调用RejectedExecutionHandler #rejectedExecution方法
    handler.rejectedExecution(command, this);
}

//线程池自带四种拒绝策略
public static class CallerRunsPolicy implements RejectedExecutionHandler {

    public CallerRunsPolicy() { }

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        //如果线程池还处于运行状态，那么使用调用者线程来执行任务
        if (!e.isShutdown()) {
            r.run();
        }
    }
}

//默认使用的拒绝策略
public static class AbortPolicy implements RejectedExecutionHandler {

    public AbortPolicy() { }
    //丢弃任务并抛出异常
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() +
                                             " rejected from " +
                                             e.toString());
    }
}

public static class DiscardPolicy implements RejectedExecutionHandler {
    public DiscardPolicy() { }
	//丢弃任务
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
}

public static class DiscardOldestPolicy implements RejectedExecutionHandler {

    public DiscardOldestPolicy() { }

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        //如果线程池还处于运行状态，丢弃阻塞队列尾部的一个任务，将该任务加入到队列尾部
        if (!e.isShutdown()) {
            e.getQueue().poll();
            e.execute(r);
        }
    }
}


```

线程持默认提供了四种拒绝策略：

1. CallerRunsPolicy：如果线程池还处于运行状态，那么会用调用者的线程执行任务，否则丢弃任务
2. AbortPolicy：丢弃任务并抛出RejectedExecutionException异常
3. DiscardPolicy：直接丢弃任务
4. DiscardOldestPolicy：如果线程池还处于运行状态，丢弃阻塞队列尾部的一个任务，将该任务加入到队列尾部

### tryTerminate方法

```java
//尝试终止线程池，回收线程，每次减少worker或者从队列中移除任务的时候都需要调用这个方法
final void tryTerminate() {
     //自旋重试
     for (;;) {
         //检查线程池状态
         int c = ctl.get();
         //1.如果线程池为RUNNING运行状态，直接返回
         //2.如果线程池状态为TIDYING、TERMINATED直接返回
         //3.如果线程状态为SHUTDOWN并且workQueue不为空，直接返回
         //也就是说1、2、3种情况，都不会继续执行终止线程池操作
         //只有线程池状态为STOP或者SHUTDOWN且workQueue为空的情况下，才会继续执行
         //终止线程池操作
         if (isRunning(c) ||
             runStateAtLeast(c, TIDYING) ||
             (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
             return;
         //当前线程数不为0时，才有资格终止线程池
         if (workerCountOf(c) != 0) { // Eligible to terminate
             //中断空闲的任务线程
             interruptIdleWorkers(ONLY_ONE);
             return;
         }
		 
         final ReentrantLock mainLock = this.mainLock;
         //加锁
         mainLock.lock();
         try {
             //将线程池运行状态设置为TIDYING，工作线程数置为0
             if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                 try {
                     //执行模板方法
                     terminated();
                 } finally {
                     //模板方法执行完成后，将线程池状态设置为TERMINATED
                     ctl.set(ctlOf(TERMINATED, 0));
                     //awaitTermination()方法中，会调用termination.awit方法
                     termination.signalAll();
                 }
                 return;
             }
         } finally {
             mainLock.unlock();
         }
         // else retry on failed CAS
     }
 }
```

尝试终止线程池，从这里可以看出线程池状态的变化

- `SHUTDOWN`   一> `TIDYING`: `workQueue`为空，且工作线程数为0
- `STOP `一> `TIDYING`：工作线程数为0
- `TIDYING` 一> `TERMINATED`: 执行完`terminated()`后

### interruptIdleWorkers方法

```java
//中断空闲的线程(也就是等待从阻塞队列中获取任务的线程)
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        //遍历workers集合中的所有任务线程
        for (Worker w : workers) {
            Thread t = w.thread;
            //如果任务的线程未被中断，那么尝试通过tryLock获取锁
            //线程如果还未开始或者正在执行，不能获取锁
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    //中断线程
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    //解除锁
                    w.unlock();
                }
            }
            //如果只有一个中断完成之后，跳出循环
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

### addWorkerFailed方法

```java
//任务创建失败后，还原任务创建前执行的其他操作
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    //加锁，避免多个线程并发中断
    mainLock.lock();
    try {
        //如果任务线程存在，从待执行的任务集合中去除
        if (w != null)
            workers.remove(w);
        //工作线程数-1
        decrementWorkerCount();
        //尝试中断线程池
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```

### processWorkerExit方法

```java
//调用runWorker方法完成后，会执行此方法，表示退出 completedAbruptly表示是否出现过异常
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    //如果程序退出前出现过异常，那么将当前工作线程数-1
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    //加锁，workers非线程安全
    mainLock.lock();
    try {
        //统计完成的任务数
        completedTaskCount += w.completedTasks;
        //将任务线程从workers集合中移除
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
	//尝试终止线程池
    tryTerminate();
	//检查线程池状态
    int c = ctl.get();
    //如果线程池状态为RUNNING或者SHUTDOWN
    if (runStateLessThan(c, STOP)) {
        //如果执行的任务没出现异常
        if (!completedAbruptly) {
            //如果允许核心线程失效，那么为0，否则为核心线程数
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            //如果允许核心线程失效，并且阻塞队列不为空，设置min为1
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            //如果工作线程有一个就返回
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        //否则创建一个任务线程，去执行阻塞队列中还未执行的任务
        addWorker(null, false);
    }
}
```

### shutdown方法

停止线程池，不会接受新任务，但是会继续处理阻塞队列中的任务，中断没被执行的任务

```java
//关闭线程池
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    //获取锁
    mainLock.lock();
    try {
        //检查是否允许执行shutdown
        checkShutdownAccess();
        //将线程池状态设置为SHUTDOWN
        advanceRunState(SHUTDOWN);
        //中断所有空闲的线程
        interruptIdleWorkers();
        //钩子方法，ScheduledThreadPoolExecutor中重写了
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    //尝试终止线程池
    tryTerminate();
}

//检查是否允许执行shutdown
private void checkShutdownAccess() {
    SecurityManager security = System.getSecurityManager();
    if (security != null) {
        security.checkPermission(shutdownPerm);
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
                security.checkAccess(w.thread);
        } finally {
            mainLock.unlock();
        }
    }
}

//空方法，留给子类实现
void onShutdown() {
}
```

### shutdownNow方法

立刻停止线程池，不接受新任务，中断正在执行的任务，不在执行阻塞队列中的任务，并返回阻塞队列中未执行的任务

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        //检查是否允许执行shutdown
        checkShutdownAccess();
        //将线程池状态设置为STOP
        advanceRunState(STOP);
        //中断所有线程
        interruptWorkers();
        //获取阻塞队列中未执行的任务
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    //返回阻塞队列中未执行的任务
    return tasks;
}

private List<Runnable> drainQueue() {
    BlockingQueue<Runnable> q = workQueue;
    ArrayList<Runnable> taskList = new ArrayList<Runnable>();
    //将阻塞队列中的任务添加到taskList中
    q.drainTo(taskList);
    //确保所有的任务都已经移动到taskList中
    if (!q.isEmpty()) {
        for (Runnable r : q.toArray(new Runnable[0])) {
            //从队列中移除
            if (q.remove(r))
                //添加到taskList中
                taskList.add(r);
        }
    }
    return taskList;
}
```

### interruptWorkers方法

```java
//中断所有已启动的线程
private void interruptWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers)
            w.interruptIfStarted();
    } finally {
        mainLock.unlock();
    }
}
```

## 总结

### 线程池状态图

![](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%8A%B6%E6%80%81.png)

