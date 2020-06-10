title: 【线程池】-- ThreadPoolExecutor源码解析(下)(jdk1.8)

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

​		上一章我们主要通过`execute()`方法，来展开分析线程池的运作机制，这一章我们就来看看线程池中`submit()`这个具有返回值的方法是如何是实现的，又是如何获取返回值的。<!-- more -->

## 结构

### AbstractExecutorService类

`ThreadPoolExecutor`继承自`AbstractExecutorService`类，而`submit()`方法就在父类`AbstractExecutorService`中

```java
public abstract class AbstractExecutorService implements ExecutorService {

    //通过传入的Runnable,value来构造一个RunnableFuture，
    //runnable作为线程执行的任务，value作为返回值
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        //将Runnable包装成一个FutureTask返回
        return new FutureTask<T>(runnable, value);
    }
	
    //由于Callable本身就有返回值，因此不用指定
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        //将Callable包装成一个FutureTask返回
        return new FutureTask<T>(callable);
    }

    //提交一个任务返回一个Future，通过Future.get()方法可以获取返回值null
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }

     //提交一个任务返回一个Future，通过Future.get()方法可以获取返回值result
    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
    }

    //提交一个任务返回一个Future，通过Future.get()方法可以获取Callable中的返回值
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
}
```

​				`Callable`是一个接口，里面声明了一个具有返回值的`call()`方法，类似`Runnable`中的`run()`方法，不同的是`call()`方法具有返回值，并且可以抛出异常。

### Callable接口

```
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

从`AbstractExecutorService`中的`submit()`方法可以看到，无论当你传入一个`Runnable`还是`Callable`，最终都会包装成一个`FutureTask`返回，我们看看`FutureTask`里面的结构：

### FutureTask类

#### FutureTask类结构

![](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/FutureTask.png)

从`FutureTask`结构图看到，`FutureTask`实现了`RunnableFuture`接口，而`RunnableFuture`又继承了`Runnable`和`Future`接口，因此`FutureTask`也具有`RunnableFuture`的特点，在成功执行`run()`方法后，可以通过`Future`访问执行结果。

```java
public class FutureTask<V> implements RunnableFuture<V> {

    private volatile int state;
    //任务新建正在执行中
    private static final int NEW          = 0;
    //任务即将完成
    private static final int COMPLETING   = 1;
    //任务正常执行完成
    private static final int NORMAL       = 2;
    //任务异常
    private static final int EXCEPTIONAL  = 3;
    //任务线程被取消
    private static final int CANCELLED    = 4;
    //任务线程将被中断
    private static final int INTERRUPTING = 5;
    //任务线程已被中断
    private static final int INTERRUPTED  = 6;

    //存储传入的任务
    private Callable<V> callable;
    //保存get方法返回的结果，也有可能保存的是一个异常
    //这里没有使用volatile修饰，意欲何为
    private Object outcome; // non-volatile, protected by state reads/writes
    /** The thread running the callable; CASed during run() */
    //执行任务的线程
    private volatile Thread runner;
    /** Treiber stack of waiting threads */
    //等待节点，多个节点组成一个链表
    private volatile WaitNode waiters;

    @SuppressWarnings("unchecked")
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        //如果正常执行，返回任务结果
        if (s == NORMAL)
            return (V)x;
        //如果任务被取消，抛出任务CancellationException异常
        if (s >= CANCELLED)
            throw new CancellationException();
        //抛出任务执行ExecutionException异常
        throw new ExecutionException((Throwable)x);
    }

    //通过Callable来构造一个FutureTask
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        //初始状态为NEW
        this.state = NEW;       // ensure visibility of callable
    }

    //通过FutureTask来构造一个FutureTask,result作为方法执行后的返回值
    public FutureTask(Runnable runnable, V result) {
        //
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }
    
    //Executors.class中的方法，通过RunnableAdapter将Runnable适配为一个Callable
    public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
    }
    
    //Executors.class的内部类，RunnableAdapter实现了Callable接口
    //通过传入Runnable构造一个Callable，当使用Callable.call时，实际上
    //执行的是Runnable的run方法
    static final class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
    }
    
	//检查任务是否被取消
    public boolean isCancelled() {
        return state >= CANCELLED;
    }

    //检查任务是否执行完毕
    public boolean isDone() {
        return state != NEW;
    }

    //取消任务，mayInterruptIfRunning为false表示不允许在线程运行时中断
    public boolean cancel(boolean mayInterruptIfRunning) {
        //如果任务状态为NEW，如果允许中断运行中的线程，那么将任务状态设置为INTERRUPTING
        //否则设置为CANCELLED，如果设置失败，直接返回false
        //如果任务状态不为NEW，直接返回false
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            //如果允许在线程运行时候取消
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                        //中断线程，中断后会唤醒waiters上等待获取该任务结果的线程
                        t.interrupt();
                } finally { // final state
                    //将线程状态设置为INTERRUPTED
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            //执行任务完成方法
            finishCompletion();
        }
        //取消成功
        return true;
    }

	//获取任务执行完成之后的返回值，响应中断
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        //如果任务还未完成
        if (s <= COMPLETING)
            //那么阻塞等待
            s = awaitDone(false, 0L);
        return report(s);
    }

	//超时获取任务执行完成之后的返回值，响应中断
    public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        //如果任务还未完成，那么阻塞等待
        if (s <= COMPLETING &&
            (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
        return report(s);
    }


    //空方法
    protected void done() { }

	//通过cas设置任务的状态为即将完成状态，成功后记录返回值
    //将任务最终状态置为NORMAL
    //执行finishCompletion()方法完成任务
    protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            //记录返回值
            outcome = v;
            //设置任务最终状态为正常执行
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            //完成任务
            finishCompletion();
        }
    }

    //通过cas设置任务的状态为即将完成状态，设置成功后
    //记录异常作为返回值，设置任务最终状态为EXCEPTIONAL
    //执行finishCompletion()方法完成任务
    protected void setException(Throwable t) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            //记录返回值为异常
            outcome = t;
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }

    //执行任务
    public void run() {
        //如果任务状态不为NEW，或者执行任务的不是当前线程
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            //获取任务
            Callable<V> c = callable;
            //任务状态为NEW的时候才执行
            if (c != null && state == NEW) {
                V result;
                //是否正常执行
                boolean ran;
                try {
                    //执行任务
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    //如果发生异常，设置异常
                    setException(ex);
                }
                if (ran)
                    //正常执行后设置
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            //执行任务的线程置为null
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            //如果线程被中断
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

    //通过该方法可以执行多次任务，任务执行完成后不会修改线程状态，并且不会设置返回值
    protected boolean runAndReset() {
        //如果状态不为NEW或者更改任务线程为当前线程失败，那么直接返回false
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return false;
        boolean ran = false;
        int s = state;
        try {
            Callable<V> c = callable;
            if (c != null && s == NEW) {
                try {
                    //不设置返回结果
                    c.call(); // don't set result
                    ran = true;
                } catch (Throwable ex) {
                    setException(ex);
                }
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            s = state;
            //如果任务即将被中断
            if (s >= INTERRUPTING)
                //等待任务变为中断状态
                handlePossibleCancellationInterrupt(s);
        }
        return ran && s == NEW;
    }

    //等待任务变为中断状态
    private void handlePossibleCancellationInterrupt(int s) {
        // It is possible for our interrupter to stall before getting a
        // chance to interrupt us.  Let's spin-wait patiently.
        if (s == INTERRUPTING)
            while (state == INTERRUPTING)
                Thread.yield(); // wait out pending interrupt

    }

    //等待节点
    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }

    //完成任务操作
    private void finishCompletion() {
        // assert state > COMPLETING;
        //如果有线程在等待任务结果
        for (WaitNode q; (q = waiters) != null;) {
            //将链表设置为null,然后释放等待节点中的线程
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();
		
        callable = null;        // to reduce footprint
    }

    //等待完成
    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        //计算超时时间，如果未设置超时deadline为0
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            //如果当前线程被中断，那么从等待节点中移除
            if (Thread.interrupted()) {
                removeWaiter(q);
                //抛出中断异常
                throw new InterruptedException();
            }

            int s = state;
            //如果任务已经完成
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                //返回任务状态
                return s;
            }
            //如果任务即将完成，那么让给cpu资源
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            //如果等待链表不存在，设置等待节点
            else if (q == null)
                q = new WaitNode();
            //将q放置到waiters前面
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            //如果设置超时了
            else if (timed) {
                nanos = deadline - System.nanoTime();
                //判断是否超时，如果超时移除等待节点，返回任务状态
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                //超时阻塞
                LockSupport.parkNanos(this, nanos);
            }
            //如果未设置超时，那么直接阻塞
            else
                LockSupport.park(this);
        }
    }

    //移除等待节点
    private void removeWaiter(WaitNode node) {
        if (node != null) {
            node.thread = null;
            retry:
            for (;;) {          // restart on removeWaiter race
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                    if (q.thread != null)
                        pred = q;
                    else if (pred != null) {
                        pred.next = s;
                        if (pred.thread == null) // check for race
                            continue retry;
                    }
                    else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                          q, s))
                        continue retry;
                }
                break;
            }
        }
    }

    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long stateOffset;
    private static final long runnerOffset;
    private static final long waitersOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = FutureTask.class;
            stateOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("state"));
            runnerOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("runner"));
            waitersOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("waiters"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }

}
```

## 总结

现在我们基本知道了，调用`submit()`方法来执行、获取返回值的一个过程，我们来回顾一下：

1. 通过`submit()`来提交一个任务，无论提交的是`Runnable`还是`Callable`，最终都会包装为一个`FutureTask`对象，`FutureTask`内部有一个`Callable`用于存储任务，`Callable`的`call()`会返回一个执行结果，`FutureTask`通过`get()`方法可以获取到`Callable`任务正常执行完成后的一个结果
2. `Callable`的`call()`方法有结果，而我们`Runnable`的`run()`方法没有返回结果, 因此当我们通过`submit`传入一个`Runnable`时，会通过`FutureTask`的构造方法使用`Executors.callable(runnable, result)`将传入的`Runnable`适配`Callable`，创建一个`RunnableAdapter`适配器，并且以`result`作为它的返回值。
3. `FutureTask`通过`get()`方法到任务正常执行完成后的一个结果，如果任务此时还未完成，那么会调用`awaitDone()`方法创建一个`WaitNode`等待节点，并将它加入到以`WaitNode`组成的waiters链表的头部。
4. 当任务执行正常执行完成后，会移除并释放在`waiters`上等待的节点线程，线程被唤醒后，如果发现结束，那么`report`判断是否正常结束，如果正常返回`result`结果，否则抛出异常。如果任务执行中被中断，那么移除waiters上等待该结果的节点，抛出异常。