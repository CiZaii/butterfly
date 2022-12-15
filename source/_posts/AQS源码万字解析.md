---
title: AQS源码万字解析
date: '2022/12/14 21:25'
updated: '2022/12/14 21:25'
tags: JUC
categories: AQS
sticky: 1
cover: 'https://zangzang.oss-cn-beijing.aliyuncs.com/img/20221214214344.png'
abbrlink: 9b957ada
---
# AQS源码

最近研究了一下AQS的源码这里写一篇文章讲一下AQS到底是干什么的怎么工作的

## AbstractQueuedSynchronizer

AbstractQueuedSynchronizer这个类大家应该都听说过，他是一个用于编写并发编程的框架，可以在他的基础上对一些方法进行重写实现不同的策略

![image-20221213112527308](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221213112527308.png)

可以看到我们这个类是一个抽象类，但是他里边并没有任何一个抽象方法，而是留有很多这种以protected关键字修饰的方法

![image-20221213112838142](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221213112838142.png)

那么可能会有疑问,为什么在抽象类中一个抽象方法都没有,而是好多这种默认方法呢，因为为了子类更好的实现定制化如果子类不去实现的话直接就会抛出异常，而不是像抽象方法一样必须重写。

然后我们看一下里边的Node节点是怎样的

```java
static final class Node {
    /** Marker to indicate a node is waiting in shared mode */
    static final Node SHARED = new Node();
    /** Marker to indicate a node is waiting in exclusive mode */
    static final Node EXCLUSIVE = null;

    /** waitStatus value to indicate thread has cancelled */
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking */
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition */
    static final int CONDITION = -2;
    /**
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate
     */
    static final int PROPAGATE = -3;

    /**
     * Status field, taking on only the values:
     *   SIGNAL:     The successor of this node is (or will soon be)
     *               blocked (via park), so the current node must
     *               unpark its successor when it releases or
     *               cancels. To avoid races, acquire methods must
     *               first indicate they need a signal,
     *               then retry the atomic acquire, and then,
     *               on failure, block.
     *   CANCELLED:  This node is cancelled due to timeout or interrupt.
     *               Nodes never leave this state. In particular,
     *               a thread with cancelled node never again blocks.
     *   CONDITION:  This node is currently on a condition queue.
     *               It will not be used as a sync queue node
     *               until transferred, at which time the status
     *               will be set to 0. (Use of this value here has
     *               nothing to do with the other uses of the
     *               field, but simplifies mechanics.)
     *   PROPAGATE:  A releaseShared should be propagated to other
     *               nodes. This is set (for head node only) in
     *               doReleaseShared to ensure propagation
     *               continues, even if other operations have
     *               since intervened.
     *   0:          None of the above
     *
     * The values are arranged numerically to simplify use.
     * Non-negative values mean that a node doesn't need to
     * signal. So, most code doesn't need to check for particular
     * values, just for sign.
     *
     * The field is initialized to 0 for normal sync nodes, and
     * CONDITION for condition nodes.  It is modified using CAS
     * (or when possible, unconditional volatile writes).
     */
    volatile int waitStatus;

    /**
     * Link to predecessor node that current node/thread relies on
     * for checking waitStatus. Assigned during enqueuing, and nulled
     * out (for sake of GC) only upon dequeuing.  Also, upon
     * cancellation of a predecessor, we short-circuit while
     * finding a non-cancelled one, which will always exist
     * because the head node is never cancelled: A node becomes
     * head only as a result of successful acquire. A
     * cancelled thread never succeeds in acquiring, and a thread only
     * cancels itself, not any other node.
     */
    volatile Node prev;

    /**
     * Link to the successor node that the current node/thread
     * unparks upon release. Assigned during enqueuing, adjusted
     * when bypassing cancelled predecessors, and nulled out (for
     * sake of GC) when dequeued.  The enq operation does not
     * assign next field of a predecessor until after attachment,
     * so seeing a null next field does not necessarily mean that
     * node is at end of queue. However, if a next field appears
     * to be null, we can scan prev's from the tail to
     * double-check.  The next field of cancelled nodes is set to
     * point to the node itself instead of null, to make life
     * easier for isOnSyncQueue.
     */
    volatile Node next;

    /**
     * The thread that enqueued this node.  Initialized on
     * construction and nulled out after use.
     */
    volatile Thread thread;

    /**
     * Link to next node waiting on condition, or the special
     * value SHARED.  Because condition queues are accessed only
     * when holding in exclusive mode, we just need a simple
     * linked queue to hold nodes while they are waiting on
     * conditions. They are then transferred to the queue to
     * re-acquire. And because conditions can only be exclusive,
     * we save a field by using special value to indicate shared
     * mode.
     */
    Node nextWaiter;

    /**
     * Returns true if node is waiting in shared mode.
     */
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    /**
     * Returns previous node, or throws NullPointerException if null.
     * Use when predecessor cannot be null.  The null check could
     * be elided, but is present to help the VM.
     *
     * @return the predecessor of this node
     */
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    // Used to establish initial head or SHARED marker
    }

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

>waitStatus这个属性是一个当前的节点状态，可能有以下状态
>```java
>// 为1的时候说明这个任务可能因为中断或者其他原因取消了
>static final int CANCELLED =  1;
>// 代表下一个节点需要被唤醒
>static final int SIGNAL    = -1;
>// 用于条件队列
>static final int CONDITION = -2;
>// 用于共享锁
>static final int PROPAGATE = -3;
>```

## 从ReentrantLock的非公平锁独占锁来看AQS的原理

以lock和unlock来看

ReentrantLock中有这样一个静态内部类NonfairSync也就是非公平锁的实现

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

## lock详解

我们来看一下lock中的语句,一开始他通过cas操作将标识state从0更改为1，如果更改成功的话将当前线程设置为有访问权限线程当前拥有权限，如果cas失败的话会进入else语句执行acquire(1),这个方法很重点

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
// 里边包含了三个方法分别是tryAcquire(arg),addWaiter(Node.EXCLUSIVE),acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
```

从第一个方法开始看,tryAcquire直接调用了nonfairTryAcquire，所以直接看nonfairTryAcquire

```java
/**
 * Performs non-fair tryLock.  tryAcquire is implemented in
 * subclasses, but both need nonfair try for trylock method.
 */
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

这个方法总体来说就是通过cas的方式去尝试获得锁，获取不到的话也不会自旋而是进入addWaiter(Node.EXCLUSIVE)方法，我们来看一下

1. 获得我们当前的线程然后拿到当前的状态，

2. 如果当前资源的状态为0的话使用cas的方法将状态从0设为1，如果获取到的话将当前拥有锁的线程设置为当前的线程

3. 如果不是的话会再去判断当前的线程是不是和我们资源的拥有者线程是一个线程如果是的话会将当前的state的值+1并且重新设置到其中，

```java
   if (nextc < 0) // overflow
       throw new Error("Maximum lock count exceeded");
   // 我们都知道int类型的数据取值范围是-32768~32767
   // 所以在我们达到阈值的时候回抛出异常
```

>如果都不满足返回false
>上面我说的是理想状态下，如果不是理想状态下呢，我们可以看一下，假如线程A进来的时候判断了状态码为0,可是此时cas按理说应该是成功的,可是在这个时候线程B来的时候状态码也为0，而且线程B的cas成功了，那么我们A就失败了此时再进行如下的判断这里我就省略了，他如果是递归调用可能会进入else if 分支可是如果不是的话就直接返回false


下面我们回到acquire方法如果tryAcquire方法返回了true说明设置成功了那么我们直接返回，执行代码就行，如果返回了false我们就会执行后边的方法

```java
if (!tryAcquire(arg) &&
    acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
```

```java
acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
// 这个方法会先执行assWaiter方法，参数为当前的模式独占模式和1，这个方法在尝试获得锁之后如果没有获得锁的话会将他加入到队列中
```

### addWaiter(Node.EXCLUSIVE)

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

我们来一行一行的看

首先会创建一个节点，节点值有当前线程信息和当前使用的模式

然后获得尾节点判断尾节点是不是为null，我们直接看为null的情况下，因为里边有一段逻辑和不为null的时候是一样的所以这里直接讲enq(node)里边的逻辑

#### enq(node)

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

这个里边的逻辑其实也挺简单的我们看一下啊这是个死循环相当于一个自旋，

>理想状态
>
>在理想状态下首先同样会拿到当前队列的尾节点然后判断是否为null，如果为null的话说明此时还不存在尾节点，则创建一个节点当队列的头节点，里边的数据是null，然后cas的方式将其设为头节点然后尾节点也是它，因为我们这个分支里边并没有return语句所以还会重新进入循环，此时尾节点已经不是null了，那么我们进入else里边的分支，首先
>
>   ```java
>   node.prev = t;
>   // 这句话会将当前节点的上一个节点设为头节点，然后通过cas的方式去将当前的节点设置为队列的尾节点
>   // 如果设置成功的话将当前队列的尾节点的下一个节点设置成当前节点然后将其返回
>   ```
>
>不理想状态(并发)
>
>在不理想状态的时候，模拟一下场景加入我们当前的线程A进入了for循环也判断了当前的头节点为null，但是他在执行compareAndSetHead(new Node())的时候刚new出自己的节点来之后，就被线程B抢先将队列的头节点设置null，然后队列的尾节点为线程B的节点数据成了线程B的数据，那么此时线程A去cas设置头节点就会失败，此时线程A会重新执行if判断此时的头节点已经有了，我们就会将B线程的数据节点的前置节点设置成当前的队尾节点也就是线程A节点然后通过cas的方式去将队列的尾节点设置为线程B的节点数据，并且将队尾节点(线程A节点)的后置节点设置为线程B节点，最终返回

### acquireQueued(addWaiter(Node.EXCLUSIVE), arg)

上边讲完了addWaiter下面讲一下acquireQueued方法，上一个方法将我们的节点添加到了同步队列中之后会将其返回,然后进入当前的acquireQueued方法中我们来点进去看一下

![image-20221214105350190](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221214105350190.png)

```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

>这个方法理解起来还是有点难度的，我们先看一下这两个标志位
>
> boolean failed = true; // 在获取锁的过程中是否获取失败
>
>boolean interrupted = false; // 在获取锁的过程中是否发生了中断

接下来我们看一下for循环里边的逻辑

>1. 首先会拿到当前节点的前置节点判断当前的前置节点是不是头节点，如果是头节点的话去重新抢占锁，注意这里是只有他的的前置节点是头节点的时候才有资格再次尝试去获得锁。
    >
    >   1. 我们先看获取锁成功的时候，如果获取锁成功的话我们就会将当前的节点设置为头节点，我们来看一下这个设置头节点的代码是怎么做的。
           >
           >      ```java
>      private void setHead(Node node) {
>      	// 首先将我们的当前节点设为头节点
>          head = node;
>          // 然后将当前节点的线程数据置为null
>          node.thread = null;
>          // 将当前节点的前驱节点置为null
>          node.prev = null;
>      }
>      ```
>   2. 接下来将我们的前驱节点的后继节点设为null
       >
       > ```markdown
>      这里的意思也就是，因为我们上一步已经将当前节点的前置节点设为了null所以现在已经不需要之前的头节点了，所以我们需要将原始头节点的的后继节点设为null，此时头节点就没有任何的引用了我们的GC会将他进行回收
>      ```
>
>   3. 然后将是否获取失败标志位设为false，然后返回false
>
>2. 我们现在看了当前节点的前置节点是头节点并且抢占锁成功的状态，我们接下来看一下如果当前节点不是头节点或者是头节点但是抢占锁失败的情况下是如何处理的，也就是
> ```java
> if (shouldParkAfterFailedAcquire(p, node) &&
>                          parkAndCheckInterrupt())
>                          interrupted = true;
> // 这里的处理我们挨个方法来讲首先来看一下shouldParkAfterFailedAcquire(p, node)
> ```
           >
           >      1. shouldParkAfterFailedAcquire(p, node)
                     >
                     >         ```java
>         private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
>                 int ws = pred.waitStatus;
>                 if (ws == Node.SIGNAL)
>                     /*
>                      * This node has already set status asking a release
>                      * to signal it, so it can safely park.
>                      */
>                     return true;
>                 if (ws > 0) {
>                     // 这里我在下边没有讲到这里就是如果大于0就说明这个任务可能因为中断或者其他原因取消了，我们就将当前的节点的前驱节点置为前驱节点的前驱，其实目的就是跳过有问题的节点，找到我们上一个可以唤醒下一节点的节点
>                     do {
>                         node.prev = pred = pred.prev;
>                     } while (pred.waitStatus > 0);
>                     pred.next = node;
>                 } else {
>                     /*
>                      * waitStatus must be 0 or PROPAGATE.  Indicate that we
>                      * need a signal, but don't park yet.  Caller will need to
>                      * retry to make sure it cannot acquire before parking.
>                      */
>                     compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
>                 }
>                 return false;
>             }
>         ```
                     >
                     >         >这个方法呢他的作用就是检查并重新获取失败节点的状态，我们可以看一下详细的代码，以为我们使用的非公平独占锁来进行讲解所以他的waitStatus只可能为-1，1，还有初始值0，注意有两个参数，参数一是我们的前驱节点，参数二是我们的当前的节点，当我们队列中添加了节点之后int ws = pred.waitStatus;获得前驱节点默认值肯定是0，然后进来进行判断走进else语句中执行compareAndSetWaitStatus(pred, ws, Node.SIGNAL);将前驱节点的状态值改为Node.SIGNAL，也就是-1，然后返回false此时就不去执行parkAndCheckInterrupt()方法了，而是重新进入for循环，此时如果还不满足第一个if的条件的话还是进入我们的
                     >         >
                     >         >```java
       if (shouldParkAfterFailedAcquire(p, node) &&
>         >    parkAndCheckInterrupt())
>         >```
                     >         >
                     >         >此时在进入shouldParkAfterFailedAcquire(p, node)之后我们的前驱节点的标志位就是-1了所以进入parkAndCheckInterrupt()方法中
                     >         >
                     >         >```java
        private final boolean parkAndCheckInterrupt() {
>         >        LockSupport.park(this);
>         >        return Thread.interrupted();
>         >    }
>         >```
                     >         >
                     >         >这个方法就是将当前的线程挂起
>
>   2. 我为什么没有说finally里边的cancelAcquire(node);方法呢因为他在我们的非公平锁独占模式下发生的概率很小，可以看一下上边的代码，如果我们想要进入cancelAcquire(node);方法首先failer必须为true，但是呢一旦进入了
       >
       >      ```java
>      if (p == head && tryAcquire(arg)) {
>          setHead(node);
>          p.next = null; // help GC
>          failed = false;
>          return interrupted;
>      }
>      // 只有进入这段代码之后才会进入finally但是一旦进入finally我们的faile的值已经变成false了不会执行，还有一种情况就是抛出异常但是只有node.predecessor();这段代码会抛出异常所以概率很小很小，下面我单独讲一下这个方法的内部实现就不说出现的场景了
>      ```

### cancelAcquire(node)

这个方法简单来说就是取消正在进行的获取尝试。

```java
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;

    node.thread = null;

    // Skip cancelled predecessors
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary.
    Node predNext = pred.next;

    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference from other threads.
    node.waitStatus = Node.CANCELLED;

    // If we are the tail, remove ourselves.
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```

>同样来一步一步的分析，我们aqs是如何将Node节点的状态标记为CANCELLED的
>
>首先进入方法后会先判断当前节点是否为null也就是是否是有意义的，如果没有意义直接结束
>
>然后当前节点的关联线程设置为null，然后获取到我们的前驱节点
>
>通过while循环将我们waitStatus的值大于0的过滤掉也就是跳过有问题的节点，找到我们上一个可以唤醒下一节点的节点。并且把当前的节点状态设置为Node.CANCELLED;此时出现if语句根据不同情况进行不同的处理
>
>情况一:
>
> 1. 如果当前节点是尾节点的话，将从后往前找，找到第一个状态为非取消状态的节点设置为尾节点
     >
     >    	1. 如果设置成功的话将当前尾节点的后继节点设为null
     >    	2. 如果设置失败的话将进入else语句
>
> 2. 如果当前节点不是尾节点的话也进入eles语句
     >
     >    1. 进入else语句之后声明一个ws变量再次进入if语句判断这里也分为不同的情况下面挨个说一下
             >
             >       1. 首先这一段
                        >
                        >          ```java
>          pred != head 
>          ```
                        >
                        >          >如果当前节点不是头节点的后继节点
>
>       2. 然后中间的
           >
           >          ```java
>          ((ws = pred.waitStatus) == Node.SIGNAL || (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL)))
>          ```
           >
           >          >1. 判断当前节点的前驱节点状态是不是SIGNAL ，如果不是的话则把前驱节点设置为SIGNAL 看看是否成功
>
>       3. 
> ```java
>          pred.thread != null
>          ```
           >
           >          > 如果上边中间条件任意成立一个的话再判断当前节点的线程是否为null
>
>    2. 此时如果上述条件都满足则把当前节点的前驱节点的后继指针指向当前节点的后继节点
>
>    3. 如果上述条件都不满足的话也就是当前节点是头节点的后继节点或者不满足上边的条件那就调用unparkSuccessor(node);环形当前线程的后继节点

以上就是所有的加锁流程了

## unlock详解

我们ReentrantLock中的解锁并没有区分公平锁和非公平锁，我们来根据源码一步一步的来看

```java
public void unlock() {
	sync.release(1);
}
```

进入release方法

```java
public final boolean release(int arg) {
	if (tryRelease(arg)) {
		Node h = head;
		if (h != null && h.waitStatus != 0)
			unparkSuccessor(h);
		return true;
	}
	return false;
}
```

>会先进入tryRelease(arg)方法

![image-20221214202126150](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221214202126150.png)

很明显这个方法是由子类去实现的我们下面来看一下

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

>引入我们的ReentrantLock加锁过程中支持可重入锁，首先会减去可重入的次数，然后判断一下当前持有锁的线程是不是当前线程如果不是的话直接抛出IllegalMonitorStateException();异常
>
>1. 接下来定义一个free，如果我们tryRelease将当前持有的线程全部释放掉的话则返回true，否则返回false
>2. 如果已经全部释放掉了做一下处理将free设置为true然后设置当前的锁没有线程拥有它然后返回true

然后我们回到release()方法，如果此时的tryRelease(arg)返回了ture说明了该锁没有被任何线程持有然后我们可以进行if语句里边的操作

```java
// 获取头节点		
Node h = head;
		// 头节点不为空并且头节点的waitStatus不是初始化节点情况，解除线程挂起状态
		if (h != null && h.waitStatus != 0)
			unparkSuccessor(h);
		return true;
```

>我说一下这里为什么是`h != null && h.waitStatus != 0`
>
>1. h == null Head还没初始化。初始情况下，head == null，第一个节点入队，Head会被初始化一个虚拟节点。所以说，这里如果还没来得及入队，就会出现head == null 的情况。
>
>2. h != null && waitStatus == 0 表明后继节点对应的线程仍在运行中，不需要唤醒。
>
>3. h != null && waitStatus < 0 表明后继节点可能被阻塞了，需要唤醒。

### unparkSuccessor();

```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

>首先进入方法后会会的头节点的状态，如果小于0则设置为0，也就是初始状态
>
>然后获取当前节点的下一个节点
>
>if语句的意思就是如果当前节点的下一个节点被*cancelled*掉了就找到队列最开始的非*cancelled*的节点，但是里边的for循环是从队列的尾部开始找的，而不是从一开始就找，是从队尾到队首拿到队列的第一个*waitStatus<0*的节点
>
>>这里我说一下为什么是从后往前找这里要回顾一下我们的addWaiter方法
>>
>>```java
>>private Node addWaiter(Node mode) {
>>    Node node = new Node(Thread.currentThread(), mode);
>>    // Try the fast path of enq; backup to full enq on failure
>>    Node pred = tail;
>>    if (pred != null) {
>>        node.prev = pred;
>>        if (compareAndSetTail(pred, node)) {
>>            pred.next = node;
>>            return node;
>>        }
>>    }
>>    enq(node);
>>    return node;
>>}
>>```
>>
>>这个节点入队列的操作并不是原子的，所以不排除有这种情况在入队过程中执行到了pred.next = node的时候，此时还没有执行这条代码，但是这时候调用了unparkSuccessor，到达这里的时候就没有办法从前往后找了因为这里相当于链表的断链了所以需要从后往前找

### 线程被唤醒后的操作

在我们的中断线程恢复之后会做什么操作呢

```java
private final boolean parkAndCheckInterrupt() {
	LockSupport.park(this);
	return Thread.interrupted();
}
// 首先回到这里执行return Thread.interrupted();
// 这里的作用是返回当前执行线程的中断状态，并清除
```

然后回到我们的acquireQueued()方法中

```java
if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
// 这里parkAndCheckInterrupt返回True或者False的时候，interrupted的值不同，但都会执行下次循环。如果这个时候获取锁成功，就会把当前interrupted返回。
```

### selfInterrupt

能力有限当前方法的作用后续会补充