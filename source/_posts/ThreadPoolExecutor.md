title: 【线程池】-- ThreadPoolExecutor源码解析(jdk1.8)

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

<!-- more -->

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
    //高3位：010：所有任务都已经完成，线程数都被回收，线程会转到TIDYING状态会继续执行钩子方法
    private static final int TIDYING    =  2 << COUNT_BITS;
    //高3位：110：钩子方法执行完毕
    private static final int TERMINATED =  3 << COUNT_BITS;
    
    //线程等待队列
    private final BlockingQueue<Runnable> workQueue;
    
    private final ReentrantLock mainLock = new ReentrantLock();

    //存储工作任务
    private final HashSet<Worker> workers = new HashSet<Worker>();

    private final Condition termination = mainLock.newCondition();

    private int largestPoolSize;

    private long completedTaskCount;
    
    //创建线程的工厂
    private volatile ThreadFactory threadFactory;

    //构造线程持时如果未指定拒绝策略，会使用该策略
    private volatile RejectedExecutionHandler handler;

    //存活时间
    private volatile long keepAliveTime;

    private volatile boolean allowCoreThreadTimeOut;

    //核心线程数
    private volatile int corePoolSize;

    //最大线程数
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
//继承自AQS、实现了Runnable，因此Worker本身既是一个Runnable，也是一个同步器
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
        //禁止线程在启动前被打断
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        //通过工厂创建线程
        this.thread = getThreadFactory().newThread(this);
    }

    //Runnable的run方法
    public void run() {
        //执行任务，传入this
        runWorker(this);
    }

    //锁是否被持有
    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    //尝试获取锁
    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    //释放锁
    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

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

## 方法

### 操作ctl方法

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
//拒绝策略
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

### execute方法

```java
//执行一个Runnable
//分三步：
//	1. 
//	2.
//	3.
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
	//检查当前线程池状态
    int c = ctl.get();
    //如果工作线程小于核心线程数
    if (workerCountOf(c) < corePoolSize) {
        //添加一个工作任务来执行当前command，如果执行成功，直接返回
        //返回false，说明添加任务被拒绝了，或者command未执行
        if (addWorker(command, true))
            return;
        //检查线程池状态
        c = ctl.get();
    }
    //如果当前为运行状态，那么添加Runnable到工作队列中
    if (isRunning(c) && workQueue.offer(command)) {
        //重新检查线程池状态
        int recheck = ctl.get();
        //如果不是运行状态，那么移除Runnable
        if (! isRunning(recheck) && remove(command))
            //移除成功后执行拒绝方法
            reject(command);
        //如果是运行状态或者移除Runnable失败
        else if (workerCountOf(recheck) == 0)
            //添加一个任务
            addWorker(null, false);
    }
    //
    else if (!addWorker(command, false))
        reject(command);
}

//将Runnable任务从工作队列中移除
public boolean remove(Runnable task) {
    boolean removed = workQueue.remove(task);
    tryTerminate(); // In case SHUTDOWN and now empty
    return removed;
}
```

### addWorker方法

```java
//添加一个工作任务，firstTask为null时表示执行一个
private boolean addWorker(Runnable firstTask, boolean core) {
     retry:
     for (;;) {
         //检查线程池状态
         int c = ctl.get();
         //获取运行状态
         int rs = runStateOf(c);
         //1.rs >= SHUTDOWN 表示线程已关闭
         //2.如果rs != SHUTDOWN,说明线程状态为STOP, TIDYING或TERMINATED的时候
         //这个时候不会接受新的任务，直接返回false
         //3.firstTask != null(表示为新的任务),线程状态为SHUTDOWN，
         //但是firstTask != null的时候，也不能接受任务
         //4.线程状态为SHUTDOWN，并且firstTask == null
         //但是等待线程为空的时候（说明等待队列中已经没有可执行的任务了），也不能接受新的任务
         //5.总结下来就是，如果rs为SHUTDOWN，只有firstTask == null，并且等待线程不为空的时候
         //才能接受本次添加工作任务的操作，这样也跟上面说的语义符合：不接受新任务但是
         //会继续处理阻塞队列中的任务
         if (rs >= SHUTDOWN &&
             ! (rs == SHUTDOWN &&
                firstTask == null &&
                ! workQueue.isEmpty()))
             return false;
		 //自旋重试
         for (;;) {
             //记录工作线程数
             int wc = workerCountOf(c);
             //1.wc >= CAPACITY，如果大于或者等于最大容量了，那么肯定时不行的，返回false
             //2.是否为核心线程，如果时核心线程，判断当前数量是否大于核心线程数，
             //否则判断是否大于最大线程数
             if (wc >= CAPACITY ||
                 wc >= (core ? corePoolSize : maximumPoolSize))
                 return false;
             //工作线程+1，如果成功跳出外层循环，失败说明工作线程的值被其他线程改变了
             if (compareAndIncrementWorkerCount(c))
                 break retry;
             //重新获取线程池状态，因为状态有可能被其他线程改变
             c = ctl.get();  // Re-read ctl
             //如果运行状态被改变了，跳出外层循环
             if (runStateOf(c) != rs)
                 continue retry;
             // else CAS failed due to workerCount change; retry inner loop
         }
     }

    //任务是否已经开始
     boolean workerStarted = false;
    //是否添加了任务
     boolean workerAdded = false;
     Worker w = null;
     try {
         //创建一个新的任务
         w = new Worker(firstTask);
         //记录新的任务线程
         final Thread t = w.thread;
         if (t != null) {
             final ReentrantLock mainLock = this.mainLock;
             //获取锁
             mainLock.lock();
             try {
                 // Recheck while holding lock.
                 // Back out on ThreadFactory failure or if
                 // shut down before lock acquired.
                 //记录运行状态
                 int rs = runStateOf(ctl.get());
				 //1.rs < SHUTDOWN，说明状态为RUNNING
                 if (rs < SHUTDOWN ||
                     (rs == SHUTDOWN && firstTask == null)) {
                     //线程是否存活
                     if (t.isAlive()) // precheck that t is startable
                         throw new IllegalThreadStateException();
                     //将任务添加到任务集合中
                     workers.add(w);
                     int s = workers.size();
                     //largestPoolSize替换为较大值
                     if (s > largestPoolSize)
                         largestPoolSize = s;
                     //将添加任务标志设置true
                     workerAdded = true;
                 }
             } finally {
                 mainLock.unlock();
             }
             //如果添加了任务，那么执行
             if (workerAdded) {
                 t.start();
                 //将开始任务标志设置true
                 workerStarted = true;
             }
         }
     } finally {
         //如果任务未开始，未添加任务，或者任务执行失败了
         if (! workerStarted)
             //执行任务失败的操作
             addWorkerFailed(w);
     }
     return workerStarted;
 }

 private void addWorkerFailed(Worker w) {
     final ReentrantLock mainLock = this.mainLock;
     mainLock.lock();
     try {
         //如果任务存在
         if (w != null)
             //从任务集合中移除
             workers.remove(w);
         //将任务数-1
         decrementWorkerCount();
         tryTerminate();
     } finally {
         mainLock.unlock();
     }
 }
```



### runWorker方法

```java
//执行一个任务
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    //提取任务
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

### tryTerminate方法

```java
 final void tryTerminate() {
     for (;;) {
         int c = ctl.get();
         if (isRunning(c) ||
             runStateAtLeast(c, TIDYING) ||
             (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
             return;
         if (workerCountOf(c) != 0) { // Eligible to terminate
             interruptIdleWorkers(ONLY_ONE);
             return;
         }

         final ReentrantLock mainLock = this.mainLock;
         mainLock.lock();
         try {
             if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                 try {
                     terminated();
                 } finally {
                     ctl.set(ctlOf(TERMINATED, 0));
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