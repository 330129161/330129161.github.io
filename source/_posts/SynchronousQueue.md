title: 【阻塞队列】-- SynchronousQueue源码解析(jdk1.8)

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/book.png

toc: true 

tags:

  - 源码
  - java
  - 阻塞队列

categories: 

  - [源码,阻塞队列] 

date: 2020-03-18 18:32:14

---

## 概述

​		`SynchronousQueue`是一个没有数据缓冲的阻塞队列，生产者线程对其的插入操作`put`必须等待消费者的移除操作`take`，反过来也一样。`SynchronousQueue`中因为不存储元素，所以peek方法永远返回null。 `SynchronousQueue`支持公平策略。`SynchronousQueue`看似是阻塞队列中最简单的一种，却是几个解析中最复杂的一个。<!-- more -->

## 结构

![](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/SynchronousQueue.png)

1. `SynchronousQueue`实现自`BlockingQueue`,因此是一个阻塞队列。
2. `SynchronousQueue`有一个`Transferer`内部类，`TransferStack`与`TransferQueue`都继承自`Transferer`，当构建一个非公平队列时，会使用`TransferStack`以后进先出的顺序访问队列元素，当构建一个公平队列时，则会使用`TransferQueue`以先进先出的顺序访问队列元素，保证公平。

## 属性

```java
public class SynchronousQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {
    private static final long serialVersionUID = -3223113410248163686L;

    //内部类
    abstract static class Transferer<E> {
        abstract E transfer(E e, boolean timed, long nanos);
    }

    //获取处理器个数，用于判断后续自旋的条件
    static final int NCPUS = Runtime.getRuntime().availableProcessors();

    //指定超时时的自旋次数,如果处理器个数<2，那么自旋次数为0否则为32
    //处理器只有1个的时候，不需要自旋
    static final int maxTimedSpins = (NCPUS < 2) ? 0 : 32;

    //未指定超时的自旋次数，默认为maxTimedSpins * 16，最大为512
    static final int maxUntimedSpins = maxTimedSpins * 16;
	
    //超时时间间隔阈值，当超时时间大于此阈值的时候，才有必要阻塞
    //不然可能阻塞一秒马上超时，不如在这一秒之内直接让他快速重试，避免阻塞带来的效率问题
    //也就是优化了这一步，AQS中也有一样的设计
    static final long spinForTimeoutThreshold = 1000L;
    
    private transient volatile Transferer<E> transferer;
}
```

### TransferStack类

```java
//非公平情况下使用栈，后进先出
static final class TransferStack<E> extends Transferer<E> {
    //消费模式
    static final int REQUEST    = 0;
    //填充模式
    static final int DATA       = 1;
    //匹配中
    static final int FULFILLING = 2;

    //是否正在匹配
    static boolean isFulfilling(int m) { return (m & FULFILLING) != 0; }

    static final class SNode {
        //下一个等待的节点
        volatile SNode next;        // next node in stack
        //匹配的节点
        volatile SNode match;       // the node matched to this
        //等待在当前节点上的线程，匹配上之后会释放该线程
        volatile Thread waiter;     // to control park/unpark
        Object item;                // data; or null for REQUESTs
        int mode;
        SNode(Object item) {
            this.item = item;
        }
		
        //如果传入的节点是当前节点的下一个节点，那么替换它为指定节点
        boolean casNext(SNode cmp, SNode val) {
            return cmp == next &&
                UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
        }

        //尝试讲一个节点与当前匹配，成功后会释放当前节点阻塞的线程
        boolean tryMatch(SNode s) {
            if (match == null &&
                UNSAFE.compareAndSwapObject(this, matchOffset, null, s)) {
                Thread w = waiter;
                if (w != null) {    // waiters need at most one unpark
                    waiter = null;
                    LockSupport.unpark(w);
                }
                return true;
            }
            //匹配上了返回true
            return match == s;
        }

		//尝试取消
        void tryCancel() {
            UNSAFE.compareAndSwapObject(this, matchOffset, null, this);
        }

        //当前节点是否被取消，当节点的match指向自身的时候，说明被取消了
        boolean isCancelled() {
            return match == this;
        }

        // Unsafe mechanics
        private static final sun.misc.Unsafe UNSAFE;
        private static final long matchOffset;
        private static final long nextOffset;

        //以下静态方法都是根据UNSAFE类获取节点的偏移量，用于CAS更新
        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> k = SNode.class;
                matchOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("match"));
                nextOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("next"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }

    //当前链表头，每次都是从头匹配
    volatile SNode head;

    //设置头
    boolean casHead(SNode h, SNode nh) {
        return h == head &&
            UNSAFE.compareAndSwapObject(this, headOffset, h, nh);
    }

    //创建一个节点，如果s为null，那么e作为s中的item
    static SNode snode(SNode s, Object e, SNode next, int mode) {
        if (s == null) s = new SNode(e);
        s.mode = mode;
        s.next = next;
        return s;
    }

    //插入或者移除元素，移除时e默认为null
    @SuppressWarnings("unchecked")
    E transfer(E e, boolean timed, long nanos) {

        SNode s = null; // constructed/reused as needed
        //判断当前模式，e为null时为REQUEST模式，否则为DATA模式
        int mode = (e == null) ? REQUEST : DATA;
		//自旋重试
        for (;;) {
            //记录头结点
            SNode h = head;
            //如果头结点为null或者本次模式与头结点模式相同
            if (h == null || h.mode == mode) {  // empty or same-mode
                //如果配置了超时，判断是否已超时
                if (timed && nanos <= 0) {      // can't wait
                    //如果超时，头结点不为null并且没有被取消，那么设置下一个节点为新的头结点
                    if (h != null && h.isCancelled())
                        casHead(h, h.next);     // pop cancelled node
                    else
                        //否则返回null
                        return null;
                //如果没有设置超时，或者还未到超时时间，那么创建一个对象设置为新的头部
                } else if (casHead(h, s = snode(s, e, h, mode))) {
                    //等待被填充，填充完成后返回匹配节点
                    SNode m = awaitFulfill(s, timed, nanos);
                    //如果匹配节点与等待的节点相同，说明被取消了，需要清理掉节点
                    if (m == s) {               // wait was cancelled
                        clean(s);
                        return null;
                    }
                    //到这里说明匹配成功了，如果当前头部不为null，并且h.next == s
                    //这里是等待匹配的，因此匹配节点应该是头结点，
                    //头结点后面的节点就是我们被匹配的节点
                    if ((h = head) != null && h.next == s)
                        //设置新的头结点，也就是s后面的节点
                        casHead(h, s.next);     // help s's fulfiller
                    //如果为消费模式，那么返回匹配的元素
                    return (E) ((mode == REQUEST) ? m.item : s.item);
                }
             //如果头结点还没有进行匹配
            } else if (!isFulfilling(h.mode)) { // try to fulfill
                //如果头结点被取消，那么设置下一个元素为头结点
                if (h.isCancelled())            // already cancelled
                    casHead(h, h.next);         // pop and retry
                //设置新的头结点，FULFILLING|mode得到的值要么为2，要么为3
                //通过isFulfilling得到的都是正在匹配
                else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                    for (;;) { // loop until matched or waiters disappear
                        //记录s的下一个节点m,也就是s要匹配的节点
                        SNode m = s.next;       // m is s's match
                        //如果m=null,说明被其他节点抢先给匹配了,
                        //并且已经没有其他在等待匹配的节点了
                        if (m == null) {        // all waiters are gone
                            //将头结点置为null，下一次自旋会重新生成，接着往下看
                            casHead(s, null);   // pop fulfill node
                            //将s置为null,进行下一次自旋
                            s = null;           // use new node next time
                            break;              // restart main loop
                        }
                        //记录m的下一个节点
                        SNode mn = m.next;
                        //进行匹配，tryMatch方法中匹配成功后会释放m中阻塞的线程
                        if (m.tryMatch(s)) {
                            //匹配成功后，mn应该置为新的头结点
                            casHead(s, mn);     // pop both s and m
                            //根据模式返回匹配的元素
                            return (E) ((mode == REQUEST) ? m.item : s.item);
                        } else                  // lost match
                            //如果匹配失败，说明m被其他节点匹配上了
                            //将m从栈中移除，s将对下一个节点进行匹配（也就是mn）
                            s.casNext(m, mn);   // help unlink
                    }
                }
                //如果头结点正在匹配中
            } else {                            // help a fulfiller
                //记录头结点的下一个节点，也就是跟头结点进行匹配的节点m
                SNode m = h.next;               // m is h's match
                //如果m为null
                if (m == null)                  // waiter is gone
                    //说明后面没有等待节点了，那么将头结点置为null,继续下一次自旋
                    casHead(h, null);           // pop fulfilling node
                else {
                    //记录m的被匹配的下一个节点
                    SNode mn = m.next;
                    //帮助m与h进行匹配
                    if (m.tryMatch(h))          // help match
                        //成功后设置新的头结点为mn
                        casHead(h, mn);         // pop both h and m
                    else                        // lost match
                        //如果匹配失败，说明m被其他节点匹配上了
                        //将m从栈中移除，h将对下一个节点进行匹配（也就是mn）
                        h.casNext(m, mn);       // help unlink
                }
            }
        }
    }


    //等待被填充
    SNode awaitFulfill(SNode s, boolean timed, long nanos) {
        //记录超时时间
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        Thread w = Thread.currentThread();
        //判断当前自旋次数，如果设置了超时则用maxTimedSpins，否则使用maxUntimedSpins
        int spins = (shouldSpin(s) ?
                     (timed ? maxTimedSpins : maxUntimedSpins) : 0);
        for (;;) {
            //如果当前线程被中断，那么设置该节点为取消
            if (w.isInterrupted())
                s.tryCancel();
            //记录匹配的节点
            SNode m = s.match;
            //<1>.如果不为null，直接返回
            if (m != null)
                return m;
            //如果配置了超时
            if (timed) {
                nanos = deadline - System.nanoTime();
                //检查是否超时，如果超时会取消当前节点，会将节点的match指向自身
                //然后下次自旋，上面<1>处就会直接返回
                if (nanos <= 0L) {
                    s.tryCancel();
                    continue;
                }
            }
         	//检查自旋次数是否已经用完
            if (spins > 0)
                //判断是否应该自旋
                spins = shouldSpin(s) ? (spins-1) : 0;
            //记录节点上等待的线程
            else if (s.waiter == null)
                s.waiter = w; // establish waiter so can park next iter
            //如果没有配置超时，那么直接阻塞
            else if (!timed)
                LockSupport.park(this);
            //如果配置超时了，那么超时阻塞
            else if (nanos > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanos);
        }
    }

    //如果当前节点为头部，或者头部为null，又或者当前正在匹配中，那么还需要自旋
    //这种时候说明很快就能匹配上了，不需要阻塞
    boolean shouldSpin(SNode s) {
        SNode h = head;
        return (h == s || h == null || isFulfilling(h.mode));
    }

    //节点被取消后需要清除
    void clean(SNode s) {
        //将节点的item、waiter置为null
        s.item = null;   // forget item
        s.waiter = null; // forget thread
		//记录要移除节点的下一个节点past
        SNode past = s.next;
        //<1>如果past存在并且被取消，那么找到past后面正常的节点
        if (past != null && past.isCancelled())
            //那么继续向下，直到找到一个正常的节点
            past = past.next;

        //从头节点开始清理
        //如果头节点不为null，并且不是当前找到的最后一个正常的节点，
        //并且头部节点被取消了，那么设置p的下一个节点为头部
        //此时新的头节点还是被取消之后，这步还会继续进行，直到头部被置为null，
        //或者置为一个正常的的节点，上面的<1>处，是从past后面开始找，这里是从head从后面开始找
        SNode p;
        while ((p = head) != null && p != past && p.isCancelled())
            casHead(p, p.next);
		//如果P不为null，说明经过了上一步的while改变了p的值，并且p为一个正常的节点
      	//如果p != past，说明p与past之间还有其他节点，针对这中间节点再做一次清理
        //直到清理完整个队列或者past位置为止
        while (p != null && p != past) {
            SNode n = p.next;
            if (n != null && n.isCancelled())
                p.casNext(n, n.next);
            else
                p = n;
        }
    }

    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long headOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = TransferStack.class;
            headOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("head"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

### TransferQueue类

```java
//公平情况下使用队列，先进先出
static final class TransferQueue<E> extends Transferer<E> {

    static final class QNode {
        //指向队列中的下一个节点
        volatile QNode next;          // next node in queue
        //存储元素
        volatile Object item;         // CAS'ed to or from null
        //节点中的等待线程
        volatile Thread waiter;       // to control park/unpark
        //是否为插入操作
        final boolean isData;

        //节点构造方法
        QNode(Object item, boolean isData) {
            this.item = item;
            this.isData = isData;
        }

        //通过cas设置下一个节点
        boolean casNext(QNode cmp, QNode val) {
            return next == cmp &&
                UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
        }

        //通过cas设置节点的item
        boolean casItem(Object cmp, Object val) {
            return item == cmp &&
                UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
        }

       //尝试取消，节点中的item指向自身，说明被取消
        void tryCancel(Object cmp) {
            UNSAFE.compareAndSwapObject(this, itemOffset, cmp, this);
        }

        //判断是否已经取消
        boolean isCancelled() {
            return item == this;
        }

		// 节点是否从已经从队列中移除
        boolean isOffList() {
            return next == this;
        }

        // Unsafe mechanics
        private static final sun.misc.Unsafe UNSAFE;
        private static final long itemOffset;
        private static final long nextOffset;

        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> k = QNode.class;
                itemOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("item"));
                nextOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("next"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }

    //队列头节点
    transient volatile QNode head;
    //队列尾节点
    transient volatile QNode tail;
	//指向一个被取消但是还没有从队列移除的节点
    transient volatile QNode cleanMe;

    TransferQueue() {
        //默认初始化一个哨兵节点
        QNode h = new QNode(null, false); // initialize to dummy node.
        head = h;
        tail = h;
    }

	//替换头节点
    void advanceHead(QNode h, QNode nh) {
        if (h == head &&
            UNSAFE.compareAndSwapObject(this, headOffset, h, nh))
            h.next = h; // forget old next
    }

	//替换尾节点
    void advanceTail(QNode t, QNode nt) {
        if (tail == t)
            UNSAFE.compareAndSwapObject(this, tailOffset, t, nt);
    }


    boolean casCleanMe(QNode cmp, QNode val) {
        return cleanMe == cmp &&
            UNSAFE.compareAndSwapObject(this, cleanMeOffset, cmp, val);
    }

    @SuppressWarnings("unchecked")
    E transfer(E e, boolean timed, long nanos) {

        QNode s = null; // constructed/reused as needed
        //是否是填充数据
        boolean isData = (e != null);

        for (;;) {
            QNode t = tail;
            QNode h = head;
            if (t == null || h == null)         // saw uninitialized value
                continue;                       // spin
			//如果队列没有其他元素或者模式相同
            if (h == t || t.isData == isData) { // empty or same-mode
                //记录下一个节点
                QNode tn = t.next;
                //如果t != tail，说明可能有其他节点插入了队列的尾部，重试
                if (t != tail)            		// inconsistent read
                    continue;
                //如果t != tail，说明可能有其他节点插入了队列的尾部
                //通过cas更新新的尾部
                if (tn != null) {               // lagging tail
                    advanceTail(t, tn);
                    continue;
                }
                //超时
                if (timed && nanos <= 0)        // can't wait
                    return null;
                //如果s为null，创建一个新节点
                if (s == null)
                    s = new QNode(e, isData);
                //将s添加到队列中，如果失败那么重试
                if (!t.casNext(null, s))        // failed to link in
                    continue;
				//更新尾节点，即使失败，其他节点也会进行这步操作
                advanceTail(t, s);              // swing tail and wait
                //等待匹配
                Object x = awaitFulfill(s, e, timed, nanos);
                //匹配完成，如果x == s说明，被取消了
                if (x == s) {                   // wait was cancelled
                    clean(t, s);
                    return null;
                }
				//如果节点还未从队列中移除
                if (!s.isOffList()) {           // not already unlinked
                    //尝试将s置为头从队列中移除
                    advanceHead(t, s);          // unlink if head
                    //x!=null说明当前是插入操作
                    if (x != null)              // and forget fields
                        //说明节点被移除
                        s.item = s;
                    s.waiter = null;
                }
                //如果是插入操作，那么返回插入的元素，移除操作，返回匹配的元素
                return (x != null) ? (E)x : e;

            } else {                            // complementary-mode
                //如果队列存在元素并且与当前模式不同，那么就直接去队列中匹配
                QNode m = h.next;               // node to fulfill
                //只要存在其中的一种情况，就说明队列被改动过，那么重试
                if (t != tail || m == null || h != head)
                    continue;                   // inconsistent read
				//记录要匹配的元素
                Object x = m.item;
                //如果isData == (x != null) 它们模式相同，说明已经被匹配了，重试
                //x == m说明被取消了，重试
                //将m的item设置为e表示匹配，如果失败重试
                if (isData == (x != null) ||    // m already fulfilled
                    x == m ||                   // m cancelled
                    !m.casItem(x, e)) {         // lost CAS
                    //出队并重试
                    advanceHead(h, m);          // dequeue and retry
                    continue;
                }
				//匹配成功
                advanceHead(h, m);              // successfully fulfilled
                LockSupport.unpark(m.waiter);
                return (x != null) ? (E)x : e;
            }
        }
    }

    //等待匹配跟TransferStack中一样
    Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
        /* Same idea as TransferStack.awaitFulfill */
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        Thread w = Thread.currentThread();
        int spins = ((head.next == s) ?
                     (timed ? maxTimedSpins : maxUntimedSpins) : 0);
        for (;;) {
            if (w.isInterrupted())
                s.tryCancel(e);
            Object x = s.item;
            if (x != e)
                return x;
            if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    s.tryCancel(e);
                    continue;
                }
            }
            if (spins > 0)
                --spins;
            else if (s.waiter == null)
                s.waiter = w;
            else if (!timed)
                LockSupport.park(this);
            else if (nanos > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanos);
        }
    }

    //节点被取消后，需要移除，pred为最开始记录尾节点，s是取消的节点
    void clean(QNode pred, QNode s) {
        s.waiter = null; // forget thread
		//如果已经断开，那么提前返回
        while (pred.next == s) { // Return early if already unlinked
            QNode h = head;
            QNode hn = h.next;   // Absorb cancelled first node as head
            //从头部向后找，如果下一个节点不为null，并且取消了，那么替换头节点
            //直到队列没有其他节点，或者找到一个正常的节点为止
            if (hn != null && hn.isCancelled()) {
                advanceHead(h, hn);
                continue;
            }
            //记录尾节点
            QNode t = tail;      // Ensure consistent read for tail
            //如果头节点等于尾节点，那么返回
            if (t == h)
                return;
            QNode tn = t.next;
            //如果t不为尾部节点，说明有新的节点进入到队列中，那么重试
            if (t != tail)
                continue;
            //如果tn != null，说明该节点进入到队列中，但是还没有置为尾节点
            //也就是确保新的节点能保证置换为尾节点，虽然其他地方也会执行这部操作
            if (tn != null) {
                advanceTail(t, tn);
                continue;
            }
            //如果s不是尾节点，那么将该节点从队列中移除
            if (s != t) {        // If not tail, try to unsplice
                QNode sn = s.next;
                if (sn == s || pred.casNext(s, sn))
                    return;
            }
            //dp表示被取消但是还未从队列中移除的节点
            QNode dp = cleanMe;
            if (dp != null) {    // Try unlinking previous cancelled node
                QNode d = dp.next;
                QNode dn;
                if (d == null ||               // d is gone or
                    d == dp ||                 // d is off list or
                    !d.isCancelled() ||        // d not cancelled or
                    (d != t &&                 // d not tail and
                     (dn = d.next) != null &&  //   has successor
                     dn != d &&                //   that is on list
                     dp.casNext(d, dn)))       // d unspliced
                    casCleanMe(dp, null);
                if (dp == pred)
                    return;      // s is already saved node
                //移除取消的节点
            } else if (casCleanMe(null, pred))
                return;          // Postpone cleaning s
        }
    }

    private static final sun.misc.Unsafe UNSAFE;
    private static final long headOffset;
    private static final long tailOffset;
    private static final long cleanMeOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = TransferQueue.class;
            headOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("head"));
            tailOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("tail"));
            cleanMeOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("cleanMe"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

## 方法

### 构造方法

```java
//构造方法，默认使用非公平模式
public SynchronousQueue() {
    this(false);
}

//构造方法，可以通过fair指定是否位公平模式
public SynchronousQueue(boolean fair) {
    transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
}
```

### 生产方法

```java
//插入元素，响应中断
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    //返回null，表示被取消
    if (transferer.transfer(e, false, 0) == null) {
        Thread.interrupted();
        throw new InterruptedException();
    }
}

//插入元素，超时阻塞，响应中断
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (e == null) throw new NullPointerException();
    if (transferer.transfer(e, true, unit.toNanos(timeout)) != null)
        return true;
    if (!Thread.interrupted())
        return false;
    throw new InterruptedException();
}

//插入元素，返回成功或者失败
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    return transferer.transfer(e, true, 0) != null;
}


```

### 消费方法

```java
//获取元素，响应中断
public E take() throws InterruptedException {
    E e = transferer.transfer(null, false, 0);
    //如果返回为null，说明被取消
    if (e != null)
        return e;
    Thread.interrupted();
    throw new InterruptedException();
} 

//获取元素，超时阻塞，响应中断
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E e = transferer.transfer(null, true, unit.toNanos(timeout));
    if (e != null || !Thread.interrupted())
        return e;
    throw new InterruptedException();
}

//获取元素，如果被取消返回null
public E poll() {
    return transferer.transfer(null, true, 0);
}

//获取元素，由于SynchronousQueue中不存储元素，因此永远返回null
public E peek() {
    return null;
}

//将队列的元素放入指定集合中
public int drainTo(Collection<? super E> c) {
    if (c == null)
        throw new NullPointerException();
    if (c == this)
        throw new IllegalArgumentException();
    int n = 0;
    for (E e; (e = poll()) != null;) {
        c.add(e);
        ++n;
    }
    return n;
}

//将指定个数的队列元素放入指定集合中
public int drainTo(Collection<? super E> c, int maxElements) {
    if (c == null)
        throw new NullPointerException();
    if (c == this)
        throw new IllegalArgumentException();
    int n = 0;
    for (E e; n < maxElements && (e = poll()) != null;) {
        c.add(e);
        ++n;
    }
    return n;
}
```

