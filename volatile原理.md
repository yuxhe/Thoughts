volatile是怎样实现了？比如一个很简单的Java代码：

instance = new Instancce() //instance是volatile变量

在生成汇编代码时会在volatile修饰的共享变量进行写操作的时候会多出Lock前缀的指令（具体的大家可以使用一些工具去看一下，这里我就只把结果说出来）。我们想这个Lock指令肯定有神奇的地方，那么Lock前缀的指令在多核处理器下会发现什么事情了？主要有这两个方面的影响：

将当前处理器缓存行的数据写回系统内存；
这个写回内存的操作会使得其他CPU里缓存了该内存地址的数据无效

为了提高处理速度，处理器不直接和内存进行通信，而是先将系统内存的数据读到内部缓存（L1，L2或其他）后再进行操作，但操作完不知道何时会写到内存。如果对声明了volatile的变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的数据写回到系统内存。但是，就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题。所以，在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。因此，经过分析我们可以得出如下结论：

Lock前缀的指令会引起处理器缓存写回内存；
一个处理器的缓存回写到内存会导致其他处理器的缓存失效；
当处理器发现本地缓存失效后，就会从内存中重读该变量数据，即可以获取当前最新值。

这样针对volatile变量通过这样的机制就使得每个线程都能获得该变量的最新值。

JMM
关于定义的理解这是一个仁者见仁智者见智的事情。出现线程安全的问题一般是因为主内存和工作内存数据不一致性和重排序导致的，而解决线程安全的问题最重要的就是理解这两种问题是怎么来的，那么，理解它们的核心在于理解java内存模型（JMM）。

一般在多个线程条件下，为了性能优化，还会涉及到编译器的指令重排和处理器的指令重排。我们知道CPU的处理速度和主存的读写速度不是一个量级的，为了平衡这种巨大的差距，每个CPU都会有缓存。因此，共享变量会先放在主存中，每个线程都有属于自己的工作内存，并且会把位于主存中的共享变量拷贝到自己的工作内存，之后的读写操作均使用位于工作内存的变量副本，并在某个时刻将工作内存的变量副本写回到主存中去。JMM就从抽象层次定义了这种方式，并且JMM决定了一个线程对共享变量的写入何时对其他线程是可见的。

![](https://user-images.githubusercontent.com/12162133/65619318-86504d00-dff2-11e9-90dd-b6d3fdb842f4.png)
从横向去看看，线程A和线程B就好像通过共享变量在进行隐式通信。这其中有很有意思的问题，如果线程A更新后数据并没有及时写回到主存，而此时线程B读到的是过期的数据，这就出现了“脏读”现象。可以通过同步机制（控制不同线程间操作发生的相对顺序）来解决或者通过volatile关键字使得每次volatile变量都能够强制刷新到主存，从而对每个线程都是可见的。

重排序：
在不改变执行结果的情况下，尽可能提高并行度。

![](https://user-images.githubusercontent.com/12162133/65619481-cca5ac00-dff2-11e9-858c-da7e4519ce81.png)

happens-before 规则：
1）如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。

程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。
监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。
volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。
传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。
start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。
join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。
程序中断规则：对线程interrupted()方法的调用先行于被中断线程的代码检测到中断时间的发生。
对象finalize规则：一个对象的初始化完成（构造函数执行结束）先行于发生它的finalize()方法的开始。

-------------------------------------------------------
Exception
Java所有异常都继承Exception类，从Exception本质上可以分为两大类，检查异常和非检查异常。

CheckedException
检查异常，出现这种异常，编译器要求必须处理的异常，开发者必须捕获或者抛出，也就是开发者经常写的try-catch-finally或throws，出RuntimeException以及子类之外，其他的Exception子类都属于检查异常，例如我们常见的IOException、SQLException等。

UncheckedException
包括RuntimeException和Error，非检查异常，也称为运行时异常，运行时的异常的特点是Java编译器不会检查它，这类异常不要求开发者在开发过程中捕获或者抛出。
---------------------------------------------------

显示的Condition对象
正如Lock是一种广义的内置锁，Condition也是一种广义的内置队列，它和wait()和notify()方法的作用大致相同的，但是wait()和notify()方法是和synchronized关键字合作使用的，而Condition是与重入锁相关联的，通过Lock接口的Condition newCondition()方法可以生成一个与当前重入锁绑定的Condition实例。利用Condition对象，我们就可以让线程在合适的时间内等待，或者在某一个时刻得到通知，继续执行。


![](https://user-images.githubusercontent.com/12162133/41202963-9705fe3c-6d03-11e8-941a-1c150522569a.png)

内置队列存在一些缺陷，每个内置锁只能存在一个相关联的条件队列，因而在像BoundedBuffer这种类中，多个线程可能在同一个条件队列上等待不同的条件副词，并且在常见的加锁模式下公开条件队列对象。Lock和Condition是一种更灵活性的选择。
一个Condition和一个Lock关联，就像一个条件队列和一个内置锁相关联，要创建一个Condition对象，可以用相关联的Lock上调用Lock.newCondition方法。正如Lock比内置锁提供更丰富的功能，Conditiont同样比内置队列提供更丰富的功能，在每个锁上可存在多个等待、条件等待可以是可中断或不可中断的、基于限时的等待，以及公平或公平的队列操作。

ReentrantLock lock = new ReentrantLock();
Condition condition = lock.newCondition();
对于每个Lock，可以有任意数量的Condition对象，Condition对象继承了Lock对象的公平性，对于公平的锁，线程会按照FIFO顺序从Condition.await中释放。在Condition对象中，与wait、notify、notifyAll相对应的方法是await、singal、singalAll。

/**
 * 暂停此线程直至以下四种情况发生
 *（1）此Condition被singnal
 *（2）此Condition被signalAll
 *（3）Thread.interrupt()
 *（4）伪wakeup
 * 以上情况，都能恢复当前线程继续执行
 */
void await() throws InterruptedException;

/**
 * 不响应中断
 */
void awaitUninterruptibly();

ConditionObject
条件队列
“条件队列”这个名字来源于：它使得一组线程（称之为等待线程的集合），能够通过某种方式等待特定的条件变成真。传统的队列是一个一个元素，而与之不同的是条件队列中的元素是一个正在等待相关条件的线程。
正如每个Java对象都可以作为锁，每个对象同样可以作为一个条件队列，并且Object中的wait、notify、notifyAll构成了内部条件队列的API。对象的内置锁与其内部条件队列是相互关联的，要调用对象中X条件队列的任何一个方法，必须持有对象X上的锁。
Object.wait会自动释放锁，并请求操作系统挂起当前的线程，从而使其他线程能够获得这个锁并修改对象的状态。当被挂起的线程醒来时，它将在返回之前重写获取锁。
BoundBuffer中使用了wait和notifyAll来实现了一个有界缓存，代码如下

使用条件队列
条件队列使构建高效以及高可响应性的状态依赖类变得更容易，但同时也很容易被不正确地使用。

条件谓词
要想正确地使用条件队列，关键是找出对象在哪个条件谓词上等待。条件谓词使某个操作成为状态依赖操作的前提条件。在有界缓存中，只有当缓存bu为空的时候，take方法才能执行，否则必须等待，对于take方法来说，它的条件谓词就是“缓存不为空”。同样，对于put方法的谓词是“缓存不能满”。
在条件等待中存在一种重要的三元关系，包括加锁、wait方法和一个条件谓词。在条件谓词中包含多个状态变量，而状态变量是由一个锁来维护。因此在测试条件谓词之前必须持有这个锁。锁对象和条件列对象必须是同一个对象。锁、条件谓词、条件队列之间构成了三元关系。

----------------------------------------------------------------
Lock
Lock接口定义了一组抽象的加锁操作。与内置加锁机制不同的是，Lock提供了一种无条件的、可轮询的、定时以及可中断的锁获取操作，所有加锁和解锁的方法都是显示的。


![](https://user-images.githubusercontent.com/12162133/41201685-decb2e28-6cee-11e8-8304-9689e863c19a.png)

ReentrantLock
实现了Lock接口，并提供了与synchronized相同的互斥性和内存可见性，在获取ReentrantLock时，有着进入同步块相同的内存语义。重入锁完全可以代替synchronized关键字，在JDK1.5的早起版本中，重入锁的性能远远好于synchronized，但从JDK1.6开始，JDK在synchronized上做了大量的优化，使得两者性能差距不大。

重入锁的特性
重入性
lock.lock();
try{
  i++;
}finally{
  lock.unlock();
  lock.unlock();
}
在这种情况下，一个线程连续两次获取同一把锁，这是允许的，如果不允许这样操作，那么同一个线程在第二次获取锁的话将会和自己发生死锁。值得注意的是，如果同一个线程多次获得锁，那么在释放锁的时候，也必须是相同的释放次数，如果释放次数多了，会出现异常java.lang.IllegalMonitorStateException异常，反之，那么相当于线程还持有锁，因此，其他线程将无法进入临界区了。

中断响应
对于synchronized，如果一个线程在等地锁，那么结果只有两种情况，要么它获得锁继续执行，要么它就保持等待。而使用重入锁，则提供另外一种可能，那就是线程可以被中断。这种情况对处理死锁是有一定帮助的。lock.lockInterruptibly()方法可以对中断响应的锁申请操作，即在等待锁的过程中，可以响应中断，Thread的interrupt()方法可以

锁申请等待限时
除了等待外部通知外，要避免死锁还有另外一种方法，那就是限时等待
lock.tryLock(5, TimeUnit.SECONDS);
表示这个现场在这个锁申请请求中，最多等待5秒。如果超过5秒还没有得到锁，就会返回false，如果成功得到锁，则会返回true。
lock.tryLock();
也可以不带参数直接运行，如果被其他线程占用，则立即返回false。

公平锁
在锁申请队列中，一般系统只会从这个锁的等待队列中随机选择一个，因此不能保证其公平性。而公平的锁，则不是这样，就会按照时间的先后顺序运行。它不会产生饥饿现象，允许我们对其公平性进行设置
public ReentrantLock(boolean fair);
当参数为true时，表示锁是公平的，公平锁看起来很优美，但是实现公平锁必然要求一个有序的队列，因此公平锁的实现成本比较高，性能相对比较低下。

重入锁的原理
可重入锁是如何实现的，这要从ReentrantLock的一个内部类Sync，父类是AbstractQueuedSynchronizer，以下简称AQS

-------------------------------------------------------------------------
什么是AQS
java.util.concurrent.locks中包含很多Lock的实现类，其内部实现依赖AQS，队列同步器AQS是用来构建锁或其他同步组件的基础框架。

![](https://user-images.githubusercontent.com/12162133/40922071-c510b34a-6843-11e8-9ef9-9b75f4bdc719.png)


它维护了一个volatile int state和一个FIFO线程等待队列，volatile代表着共享资源，FIFO队列提供多线程竞争资源被阻塞时会进入该队列。

两种资源共享方式
Exclusive：独占模式，只有一个线程能执行（如：ReentrantLock）
Share：共享模式，多个线程可同时执行（如：CountDownLatch）
不同的自定义同步器竞争共享资源的方式也不同，自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可。至于具体的线程等待队列，AQS已经实现好了。自定义同步器实现时主要实现以下几种方法：
isHeldExclusively()：该线程是否正在独占资源，只有用到condition才需要实现它
tryAcquire(int)：独占方式，尝试获取资源，成功返回true，失败返回false
tryRelease(int)：独占方式，尝试释放资源，成功返回true，失败返回false
tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。
源码
state
private volatile int state;
AQS中通过一个volatile的state变量表示同步状态，其子类通过继承同步器并需要它的方法管理它的状态。-

独占模式下，以ReentrantLock为例，state初始化为0，表示未锁定状态，A线程lock时，会调用tryAcquire()独占该锁并将state加1，此后其他线程再tryAcquire()时就会失败，就如等待队列（FIFO），直到A线程释放调用tryRelease()将state减1，当state恢复为0时，其他线程才有机会获取该锁。当然，A线程自己可以重复获取锁，state会累加，这就是可重入的概念。但是要注意，获取多少次就需要释放多少次，这样才能保证state恢复为0。
共享模式下，以CountDownLatch为例，任务分为N个线程执行，state会初始化为N。这N个线程是并行执行的，每个子线程执行任务完成，通过CAS将state减1。等待所有的线程都执行完后，state变为0。
管理方式可以通过acquire和release方法来操作状态。在多线程中对其状态的管理必须是原子性，同步器提供三个方法对状态进行操作状态：

protected final int getState() {
  return state;
}
protected final void setState(int newState) {
  state = newState;
}
protected final boolean compareAndSetState(int expect, int update) {
  // See below for intrinsics setup to support this
  return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
Node
static final class Node {
        /**
         * 表示节点的状态，其中包含的状态有：
         * 1:CANCELLED 表示当前线程被取消
         * -1:SIGNAL 表示当前节点的下一个节点包含的线程需要运行，也就是unpark
         * -2:CONDITION 表示当前节点在等待condition，也就是在condition队列中
         * -3:PROPAGATE 表示当前场景下后面的节点acquireShared能够得到执行
         * 0:表示当前节点在sync队列中，等待着获取锁执行
         **/
        volatile int waitStatus;
        // 前一个节点
        volatile Node prev;
        // 后一个节点
        volatile Node next;
        // 入队列时的当前线程
        volatile Thread thread;
        Node nextWaiter;
}
acquire
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
}
独占模式下线程获取共享资源的入口。如果获取到资源，线程将直接返回，否则进入等待队列，直到获取到资源为止，且整个过程忽略中断的影响，流程如下：
（1）tryAcquire：由AQS的实现类实现该方法，尝试直接获取该资源，如果成功则直接返回
（2）addWaiter：将该线程加入到等待队列的尾部，并标记为独占模式
（3）acquireQueued：线程在等待队列中获取资源，一直等获取到资源为止。如果整个等待过程中被中断，则返回为true，否则返回false
（4）selfInterrupt：线程被中断过，它是不响应的，只是获取资源后再进行自我中断，补上中断
4. tryAcquire

protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
}
此方法尝试去获取独占模式下的资源，这里需要自定义同步器去实现功能，如果成功则返回true，否则返回false。
5. addWaiter

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
变量waitStatus则表示当前被封装成Node节点的等待状态，AQS在判断状态时，大于0表示取消状态，小于0表示有效状态。
6. enq

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
CAS策略将node节点加入到队尾。
7. acquireQueued

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
前面的方法已经说明当线程获取资源失败的时，已经被放入等待队列的尾部了，acquireQueued就是在等待队列中排队拿号，直到拿到号为止。
通过自旋，如果该节点的前一个节点是HEAD节点，那么该节点便有资格获取资源，如果能拿到资源便将原先HEAD节点置为空。
8. shouldParkAfterFailedAcquire

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
此方法主要是检查状态，如果前一个节点是SIGNAL，表示当前节点需要运行，那么就不能park，需要再次尝试获取；如果前一个节点的状态大于0，则表示前一节点线程已放弃执行，那就一直往前找，直到找到一个正常的节点，排在它的后面；如果前一个节点正常，则设置前一节点为SIGNAL。
9. parkAndCheckInterrupt

private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
设置当前线程等待，park()会让当前线程进入waiting状态，有两种途径可以唤醒线程：（1）被unpark()；（2）被interrupted()。

总结：上述代码流程图如下：

![](https://user-images.githubusercontent.com/12162133/41192675-0966e070-6c34-11e8-84b9-fbc1d9c70760.png)


release
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
上述方法讲述的是获取资源，此方法讲述的是独占模式下线程释放共享资源的顶层入口，释放资源，如果state等于0，它会唤醒等待队列里面的其他线程来获取资源。它调用tryRealease方法来释放资源，它是根据tryRelease的返回值判断该线程是否已经完成资源的释放。
11. tryRelease

protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
和tryAcquire方法一样，这个方法需要自定义同步器去实现。正常来说，都会成功，因为这是独占模式，该线程释放资源，说明该线程已经拿到了资源，直接减掉相应的量。如果彻底释放掉资源，则返回为true，否则返回为false。
12. unparkSuccessor

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
此方法用于唤醒等待队列中的下一个线程，如果当前节点小于0，则设置为0。查找到下一个节点，然后执行LockSupport.unpark()方法唤醒下一个等待节点的线程。
---------------------------------------------------------------------
Semaphore

信号量
允许多个线程同时访问，信号量为多个线程提供了更为强大的控制方法。广义上说，信号量是对锁的扩展，无论是内部锁synchronized还是重入锁ReentrantLock，一次都只允许一个线程访问一个资源，而信号量却可以提供多个线程，同时访问某个资源。

应用场景
信号量可以用作流量控制，特别公共资源有限的应用场景，比如数据库连接。

构造方法
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}

在构造信号量方法时，必须要指定信号量的准入数，即同时能够申请多少个许可，当每个线程每次只申请一个许可时，这就相当于指定了同时多个线程可以访问某个资源了。

主要方法
public void acquire(int permits) throws InterruptedException
public void acquireUninterruptibly(int permits) 
public boolean tryAcquire(int permits)
public boolean tryAcquire(int permits, long timeout, TimeUnit unit) throws InterruptedException
public void release(int permits)
acquire方法尝试获取一个准入的许可。若无法获得，则线程会等待，直到有线程释放一个许可或者当前线程被中断；acquireUninterruptibly不响应中断；tryAcquire尝试获取一个许可，成功返回true，否则返回false；release用于线程访问资源结束后，释放一个许可。

源码解析
创建一个N许可的信号量
Sync(int permits) {
    setState(permits);
}
Semaphore内部是基于个AQS实现的，可以看到调用setState方法，其实是设置AQS中的state。Sync类继承了AQS这个抽象类。
2. AQS实现类
非公平同步器：
static final class NonfairSync extends Sync {
private static final long serialVersionUID = -2694183684443567898L;

NonfairSync(int permits) {
    super(permits);
}

protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}
}
公平同步器：
static final class FairSync extends Sync {
private static final long serialVersionUID = 2014338818796000944L;

FairSync(int permits) {
    super(permits);
}

protected int tryAcquireShared(int acquires) {
    for (;;) {
        if (hasQueuedPredecessors())
            return -1;
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
}
根据以上代码可以看出，公平同步器首先会判断当前同步列中有没有线程在等待，如果有，进入到等待队列排队执行。
3.acquire

public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}

static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -2694183684443567898L;

    NonfairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
}

public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}

final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}

private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
先构造非公平同步器，如果线程被中断了，则抛出异常，AQS子类需要实现AQS共享模式，NonfairSync需实现tryAcquireShared方法，，然后调用nonfairTryAcquireShared方法，方法实现：如果剩余许可数量，计算获取后还剩下许可数量，如果许可不够用或者CAS设置许可剩余许可数量成功则返回剩余许可值。返回剩余许可值，如果小于0，则调用doAcquireSharedInterruptibly(AQS内部方法)，将线程加入到等待队列中。
4. release

public void release(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.releaseShared(permits);
}

public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
AQS子类实现共享模式tryReleaseShared方法。首先，获取当前许可数量，计算回收后的数量，如果回收后的数量小于当前数量，则抛出异常，否则，则CAS改变许可数量成功。

**总结：**Semaphore是信号量，用于管理一组资源，其内部是基于AQS共享模式，AQS的状态表示许可证的数量，在许可证数量不够时，线程将会被挂起。

-------------------------------------------------------------------------------
LockSupport

阻塞工具类
是一个非常方便的线程阻塞工具类，它可以在线程任意位置让线程阻塞。和Thread.suspend()相比，它弥补了由于resume()在前发生，导致线程无法继续执行的情况。和Object.wait()相比，它不需要先获得某个对象的锁，也不会抛出InterruptedException异常。

主要方法
public static void park(Object blocker)
public static void parkNanos(Object blocker, long nanos) 
public static void parkUntil(Object blocker, long deadline)
park方法阻塞当前线程，parkNanos、parkUntil可以显示限时等待。这是因为LockSupport类使用类似信号量的机制。它为每一个线程准备了一个许可，如果许可可用，那么park函数会立即返回，并且消费这个许可（设置许可不可用），就会阻塞，而unpark函数则使得一个许可变为可用（但是和信号量不同的是，许可不可累加可用，你不可能拥有超过一个许可，它永远只有一个）。
我们探究 park() 内部实现的源码，发现内部都调用了 setBlocker() 方法，具体代码如下：

private static void setBlocker(Thread t, Object arg) {
    // Even though volatile, hotspot doesn't need a write barrier here.
    UNSAFE.putObject(t, parkBlockerOffset, arg);
}
-----------------------------------------------------
ReadWriteLock

读写锁
ReadWriteLock是JDK1.5中提供的读写分离锁，读写分离锁可以有效的减少锁竞争。以提升系统性能。

读	写
读	非阻塞	阻塞
写	阻塞	阻塞
如果在系统中，读的次数远远超过写的次数，则读写锁可以发挥很大的作用。
ReadWriteLock
public interface ReadWriteLock {
    /**
     * Returns the lock used for reading.
     *
     * @return the lock used for reading
     */
    Lock readLock();

    /**
     * Returns the lock used for writing.
     *
     * @return the lock used for writing
     */
    Lock writeLock();
}
该接口有两个方法，读锁和写锁。

ReentrantReadWriteLock
实现了ReadWriteLock接口，该接口提供readLock()方法获取读锁，writeLock()方法获取写锁。

public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}
public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
默认构造方法为非公平模式，开发者也可以指定fail为true设置为公平模式。而公平模式由内部类FailSync和NonFailSync实现，这两个类都继承Sync，Sync继承自AQS，这和ReentrantLock内部实现一致。

abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 6317671515068378041L;

    /*
     * Read vs write count extraction constants and functions.
     * Lock state is logically divided into two unsigned shorts:
     * The lower one representing the exclusive (writer) lock hold count,
     * and the upper the shared (reader) hold count.
     */

    static final int SHARED_SHIFT   = 16;
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

    /** Returns the number of shared holds represented in count  */
    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
    /** Returns the number of exclusive holds represented in count  */
    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

    abstract boolean readerShouldBlock();
    abstract boolean writerShouldBlock();
    ……
}
Sync中的两个方法readerShouldBlock、writerShouldBlock是抽象的，子类必须继承。表示读是否需要阻塞、写是否需要阻塞。

static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -8159625535654395037L;
    final boolean writerShouldBlock() {
        return false; // writers can always barge
    }
    final boolean readerShouldBlock() {
        /* As a heuristic to avoid indefinite writer starvation,
         * block if the thread that momentarily appears to be head
         * of queue, if one exists, is a waiting writer.  This is
         * only a probabilistic effect since a new reader will not
         * block if there is a waiting writer behind other enabled
         * readers that have not yet drained from the queue.
         */
        return apparentlyFirstQueuedIsExclusive();
    }
}
非公平模式下，writeShouldBlock直接返回false，说明不需要阻塞；而readShouldBlock调用了apparentlyFirstQueuedIsExclusive方法，该方法在当前线程是写锁占用线程的时候，返回true，否则返回false，说明如果当前有一个写线程在写，那么该线程应该阻塞。继承AQS类都需要使用state代表某种资源，ReentrantReadWriteLock中的state代表了读锁的数量和写锁的持有与否，如下图

public void lock() {
    sync.acquireShared(1);
}
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    int r = sharedCount(c);
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}
读锁，使用的是AQS模式，调用子类继承AQS的tryAcquireShared方法，如果tryAcquireShared方法小于0，则将当前线程加入到等待队列中。tryAcquireShared方法首先会获取共享资源state，exclusiveCount方法不为0表示当前有写线程，并且写线程不是当前线程，则返回-1。
------------------------------------------------------------------------------------

**什么是生产者-消费者模式**

生产者是生产数据的线程，消费者是消费数据的线程。在多线程场景中，如果生产者处理速度快，而消费者处理速度慢，那么生产者必须等消费者处理完，才能继续生产数据。同理，如果消费者处理速度大于生产者处理速度，则消费者必须等待生产者。为了解决生产消费能力不均衡问题，才有了生产者-消费者模式

生产者-消费者模式是一个经典的多线程设计模式，通过一个容器解决生产者和消费者的强耦合能力。生产者和消费者彼此之间不直接通讯，而通过阻塞队列进行通讯，阻塞队列就相当于一个缓冲区，平衡了生产者和消费者的处理能力。

image

BlockingQueue
BlockingQueue是一个接口，它的主要实现如下图：

image

ArrayBlockingQueue
是基于数组实现的，它的内部元素是一个对象数组：

final Object[] items;
向队列中压入元素可以使用offer()方法和put()方法。对于offer()方法，如果当前队列已经满了，它会立即返回为false。put()方法也是将元素压入队列末尾，如果队列满了，它会一直等待，直到队列中有空闲的位置。
从队列中弹出元素可以使用poll()方法和take()方法。它们都从队列的头部获得一个元素，不同之处在于，如果队列为空poll()方法会立即为null，而taker()方法会等待，直到队列中有可用元素。
因此，put()和take()方法才是体现Blocking的关键，因此定义如下字段：

/** Main lock guarding all access */
final ReentrantLock lock;
/** Condition for waiting takes */
private final Condition notEmpty;
/** Condition for waiting puts */
private final Condition notFull;
当执行take()方法操作时，如果队列为空，则让当前线程等待notEmpty()上。新元素入队时，则进行一次notEmpty上的通知：

public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            notEmpty.await();
        return dequeue();
    } finally {
        lock.unlock();
    }
}

private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length)
        takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal();
    return x;
}
take()方法取数据，尝试获取锁，如果此时锁被其他线程占用，那么当前线程就处于WAITING状态（支持中断），如果没有其他线程占用，则进入临界区，如果队列为空，则让当前线程等待在notEmpty()上，如果不为空，则进行一次出队操作；出队操作则取出队列中元素，然后唤醒通知notFull。

public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length)
            notFull.await();
        enqueue(e);
    } finally {
        lock.unlock();
    }
}

private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length)
        putIndex = 0;
    count++;
    notEmpty.signal();
}
首先，判断入队元素是否为空，进入临界区，如果队列满时，则需要通知等待在notFull上的线程，如果不为空或者唤醒，则入队操作。
--------------------------------------------------------------------------------
**数据库事务**
事务的特性
原子性：数据库事务是不可分割的单位，即要么都做，要么都不做
一致性：一致性指事务将数据库从一种状态转变为下一种一致的状态
隔离性：即该事务提交前对其他事务不可见，通常使用锁来实现
持久性：事务一旦提交，其结果就是永久性的。即使发生宕机等故障，数据库也能将数据恢复
事务的一致性基本可以理解为事务对数据完整性约束的遵循。这些约束包括主键约束、外键约束或一些用户自定义约束。事务执行前后都是合法的数据状态，不会违背任何数据的完整性。

Master Thread
Master Thread是一个非常核心的后台线程，主要负责将缓冲池中的数据异步刷新到磁盘上，保证数据的一致性，包括脏页的刷新、合并插入缓冲（INSERT BUFFER）、UNDO页的回收等。
IO Thread
在InnoDB存储引擎中大量使用AIO（Async IO）来处理写IO请求，这样可以极大提高数据库的性能。而IO Thread的工作主要是负责这些IO请求的回调（call back）处理。InnoDB工有4个IO Thread，分别是write、read、insert buffer和log IO thread。

--------------------------------------------------------------
**MySQL日志&锁**

日志
MySQL的日志有很多种，常见的有错误日志、慢查询日志、查询日志、二进制日志、redo日志、undo日志，事务的ACID四大特性，事务的隔离性由锁来实现；事务的原子性、持久性由redo log实现；事务的一致性由undo log实现。

二进制日志
二进制日志（binary log），记录了对MySQL数据库执行更改的所有操作包括DDL、DML(不包括SELECT、SHOW)。

mysql>INSERT INTO test(id,status) values (3,4);
QUERY OK,0 rows affected(0.00 sec)
mysql>SHOW BINARY LOGS;        --查看当前服务器二进制文件及大小
+------------------------------------+-----------+
| Log_name                           | File_size |
+------------------------------------+-----------+
| hostname-bin.000001 |       668 |
+------------------------------------+-----------+
1 row in set (0.06 sec)
mysql>SHOW MASTER STATUS;      --查看当前二进制文件位置
+------------------------------------+----------+--------------+------------------+-------------------+
| File                               | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------------------------+----------+--------------+------------------+-------------------+
| hostname-bin.000001|      668 |              |                  |                   |
+------------------------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
mysql>SHOW BINLOG EVENTS IN 'hostname.000001';      --查看所选二进制文件内容
+------------------------------------+-----+----------------+-----------+-------------+---------------------------------------+
| Log_name                           | Pos | Event_type     | Server_id | End_log_pos | Info                                  |
+------------------------------------+-----+----------------+-----------+-------------+---------------------------------------+
| hostname-bin.000001|   4 | Format_desc    |         1 |         123 | Server ver: 5.7.21-log, Binlog ver: 4 |
| hostname-bin.000001 | 123 | Previous_gtids |         1 |         154 |                                       |
| hostname-bin.000001| 154 | Anonymous_Gtid |         1 |         219 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'  |
| hostname-bin.000001| 219 | Query          |         1 |         289 | BEGIN                                 |
| hostname-bin.000001| 289 | Table_map      |         1 |         335 | table_id: 110 (mq.test)               |
| hostname-bin.000001| 335 | Write_rows     |         1 |         380 | table_id: 110 flags: STMT_END_F       |
| hostname-bin.000001| 380 | Xid            |         1 |         411 | COMMIT /* xid=130 */                  |
| hostname-bin.000001| 411 | Anonymous_Gtid |         1 |         476 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'  |
| hostname-bin.000001| 476 | Query          |         1 |         546 | BEGIN                                 |
| hostname-bin.000001| 546 | Table_map      |         1 |         592 | table_id: 110 (mq.test)               |
| hostname-bin.000001| 592 | Write_rows     |         1 |         637 | table_id: 110 flags: STMT_END_F       |
| hostname-bin.000001| 637 | Xid            |         1 |         668 | COMMIT /* xid=590 */                  |
+------------------------------------+-----+----------------+-----------+-------------+---------------------------------------+
12 rows in set (0.11 sec)
上述操作对数据库即使没有更改，但是通过命令可以查看在二进制日志中是有记录的。二进制日志主要有一下作用：

恢复：某些数据恢复需要二进制日志，例如：如果一个数据库全备文件恢复后，需要二进制日志进行point-in-time的恢复
复制：通过复制和执行二进制日志使一台远程MySQL数据库（一般称为slave或standby）与一台MySQL数据库进行同步
审计：用户可以通过二进制日志中的信息来进行审计，判断是否有对数据库进行注入的攻击
如何开启二进制文件：

#find / -name 'my.cnf'
/etc/my.cnf
#vi /etc/my.cnf
[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
innodb_buffer_pool_size = 3200M
#
# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
log_bin
server-id=1
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

#service mysqld stop
#service mysqld start
在[mysqld]块，设置log-bin[=name]可以启动二进制日志，如果不指定name，则默认二进制文件名为主机名，后缀名为二进制日志的序列号，还需要设置一个server_id=1，在复制中，为主库和备库提供一个独立的ID。datadir目录用于存储文件，包括二进制日志文件：

host-name-bin.000001
host-name-bin.index
host-name-bin.000001存储为二进制日志文件，host-name-bin.index存储为二进制索引文件，用来生产产生二进制日志文件序号。开启二进制日志文件会影响数据库性能，根据MySQL官方手册开启二进制日志文件会使性能降低1%。当存储引擎使用事务时，所有未提交的二进制日志会记录到一个缓存中，该事务提交后才会将二进制日志写入到二进制文件中去。
二进制日志并不是每次写的时候同步到磁盘，因此，当数据库发生宕机的时候，可能会有一部分数据没有写入二进制文件中，参数sync_binlog=[N]表示每次写缓冲多少写入到磁盘，如果N为1，则表示同步到磁盘，但是设置为1时候，InnoDB引擎使用事务，在一个事务COMMIT之前，已经写入到磁盘文件中，但是提交还没有发生，并且此时发生宕机，那么在MySql数据库发生启动时，由于COMMIT没有发生，会被回滚掉，但是二进制文件已经记录了这条消息。这个问题可以通过参数innodb_support_xa设置为1解决。
binlog_format参数也很重要，在MySQL5.1版本之前，没有这个参数，所有二进制文件都是基于SQL语句（statement）级别的，MySQL5.1开始引入binlog_format参数，可以设置如下格式：

STATEMENT:逻辑SQL语句
ROW:记录表的行更改情况
MIXED:默认采用STATEMENT格式进行二进制文件日志记录，但是有些情况下会使用ROW格式，可能情况有：
（1）存储引擎为NDB，对表DML操作都会以ROW格式
（2）使用了UUID()、USER()、CURRENT_USER()、FOUND_ROWS()、ROW_COUNT()等不确定函数
（3）使用了INSERT DELAY语句
（4）使用了用户自定义函数(UDF)
（5）使用了临时表
但是不能忽略的一点是ROW格式可能需要更大的容量，二进制的日志文件是二进制的，所有查看具体的SQL操作需要通过MySQL提供的工具mysqlbinlog，如下：
# mysqlbinlog --start-postion=100 host-name-bin.000001
如果是ROW格式，便不可读，需要加上-v或-vv查看；是STATEMENT格式，则可读。

redo log
MySQL日志类型分为逻辑日志和物理日志

逻辑日志：存储了逻辑SQL修改语句
物理日志：存储了数据被修改的值
重做日志文件，其属于物理日志的特性。当实例或介质失败时，重做日志就能派上用场了，当数据库掉电时，InnoDB存储引擎就会使用重做日志恢复到掉电前的时刻，以此来保证数据的完整性。在InnoDB存储引擎下数据目录会有两个名为ib_logfile0 和ib_logfile1的文件，参数innodb_log_files_in_group指定了日志文件组中重做日志文件的数量，默认为2。重做日志文件的大小对于InnoDB存储引擎的性能有着非常大的影响，设置太大，在恢复时需要很长的时间；另一方面，设置太小，可能导致一个事务的日志需要多次切换重做日志文件。此外，重做日志文件太小容易造成频繁地发生async checkpoint，导致性能抖动。
写入重做日志文件不是直接写，而是写入一个重做日志缓冲(redo log buffer)，然后按照一定的顺序写入日志文件，参数innodb_flush_log_at_trx_commit的有效值为0、1、2。

0：并不将事务的重做日志写入磁盘上的日志文件，而是等待主线程每秒刷新
1：执行COMMIT时，将重做日志缓冲同步写到磁盘，即伴有fsync的调用
2：表示将重做日志异步写入到磁盘，即写到文件系统的缓冲中。因此，不能完全保证在执行COMMIT时肯定会写入重做日志文件。
undo log
重做日志记录了事务的行为，可以很好的通过对其页进行“重做”操作，但是事务有时需要回滚操作，这是和undo就派上了用场。
上述redo log存放在重做日志文件中，undo log存放在数据库内部一个的一个特殊段（segment）中。

--------------------------------------------------------
通道 Channel
通过它可以读取和写入数据，它就像自来水管一样，网络数据通过 Channel 来读取和写入。通道与流的不同之处在于通道是双向的，因为 Channel 是全双工的，特别是 UNIX 网络编程模型中，底层操作系统的通道是全双工的，同时支持读写。实际上 Channel 可以分为两类：分别是用于网络读写的 SelectableChannel 和用于文件操作的 FileChannel。
多路复用器 Selector
通常来讲，Selector 会不断地轮询注册在其上的 Channel，如果某个 Channel 上面有新的 TCP 连接接入、读和写事件，这个 Channel 就处于就绪状态，会被 Selector 轮询出来，然后通过 SelecttionKey 可以获取就绪的 Channel 集合，进行后续的 IO 操作。
一个多路复用器 Selector 可以同时轮询多个 Channel，由于 JDK 使用了**epoll**代理传统的 select 实现，所以它并没有最大连接句柄的限制。这也就意味着只需要一个线程负责 Selector 的轮询，就可以接入成千上万的客户端。

---------------------------------------------------------------------------
mmap 加速内核与用户空间的信息传递。epoll 是通过内核与用户空间 mmap 同一块内存，避免了无谓的内存拷贝
-----------------------------------------------------------------------------------------
而epoll是这样做的：

1、调用epoll_creat函数建立一个epoll对象（一颗红黑树，一个准备就绪list链表）。

2、调用epoll_ctl函数把socket放到红黑树上，给内核中断处理程序注册一个回调函数，告诉内核，如果这个句柄的中断到了，就把这个socket放到准备就绪list链表里。

3、调用epoll_wait到准备就绪list链表中处理socket，并把数据返回给用户。

Note:不需要把全部的连接处理一遍，只需要去list链表里处理socket。
--------------------------------------------------------------------

一个epoll模式的代码大概的样子是：
while true {
active_stream[] = epoll_wait(epollfd)
for i in active_stream[] {
read or write till
}
}

------------------------------------
epoll管理fd使用了红黑树结构

可见不论是epoll fd本身还是epitem都是基于内核内存分配创建，没有mmap之说，网上关于epoll与mmap有关的文章完全是无稽之谈

可见结合上面几行，这里就是核心的注册事件唤醒回调，并没有像select那样每次主动做事件轮询

可以认为epoll可以监听的fd数量非常大
第35行，将当前进程挂在等待队列睡眠，当相应待监听事件就绪时会有回调ep_poll_callback唤醒。
第45行，唤醒时调用ep_events_available检查就绪链表（实现参见第82行），不像select每次都需要轮询，这里epoll只需要检查下就绪链表是否为空（时间复杂度O(1)）。
第74行，ep_send_events调用ep_scan_ready_list执行回调ep_read_events_proc遍历就绪链表并传入用户空间以及回调执行期间发生的事件通过eventpoll的ovflist（回调执行前置空）将就绪fd去重插入就绪链表以便下一次epoll_wait调用处理，这里还涉及ET和LT模式下的不同处理，暂时不作分析
------------------------------------------------
多路是指网络连接，复用指的是同一个线程

同步非阻塞（NIO）
服务器端当accept一个请求后，加入fds集合，每次轮询一遍fds集合recv(非阻塞)数据，没有数据则立即返回错误，每次轮询所有fd（包括没有发生读写事件的fd）会很**浪费cpu**

轮询策略 死循环
select
poll
epoll
------------------------------------------------------------
VM Thread：某些操作需要等待 JVM 到达 安全点（Safe Point），即堆区没有变化。比如：GC 操作、线程 Dump、线程挂起 这些操作都在 VM Thread 中进行

---------------------------------------------------------
垃圾收集 Garbage Collection
GC目的:
那些内存需要回收
什么时候回收
如何回收
判断对象死亡
引用计数算法(java中不用)
可达性分析算法
GC Roots:
虚拟机栈中本地变量表中的引用对象(reference)
方法区中类静态属性引用的对象
方法区中常量引用的对象
本地方法栈(JNI)引用对象(native)
引用种类
强 strong 1,弱 weak 3,软 soft 2,虚 phantom 4
垃圾收集算法
种类 标记-清除算法 Mark-Sweep
标记-整理算法 Mark-Compact
复制算法
分代收集算法
新生代 young generation-->复制算法 Minor GC
老生代 tenured generation-->标记 清理\整理 算法
Major GC
算法实现
枚举根节点->停顿
安全点/区域
发生GC时,要求所有线程处在安全域内
垃圾收集器
并行:多条GC线程同时工作,用户线程处于等待状态
并发:用户线程与GC线程同时执行(但不一定是并行的,可能会交替执行)
serial(串行) 暂停其他所有工作线程
serial new 复制算法 新生代
serial old 标记-整理算法 老年代
parnew 新生代
多线程版serial
parallel(并行)
目标:达到一个可控的吞吐量
吞吐量= 用户代码运行时间/(用户代码运行时间 + GC时间)
parallel scavenge(打扫) 新生代
parallel old 老年代
CMS concunrrent mark sweep(打扫) 老年代
目标:获取最短回收停顿时间
G1 garbage first
其他收集器区别: 将整个java堆划分成多个大小相等的区 域(Region)
region包含有 新/老代
内存分配
young generation
Eden space 8(比例)
当Eden区内存不够的时候就会触发MinorGC，对新生代区进行一次垃圾回收。
survivor space 2
form space 1
上一次GC的幸存者，作为这一次GC的被扫描者。
to space 1
保留了一次MinorGC过程中的幸存者。
gc:MinorGC
MinorGC的过程：
MinorGC采用复制算法。
首先，把Eden和ServivorFrom区域中存活的对象复制到ServicorTo区域（如果有对象的年龄以及达到了老年的标准，则赋值到老年代区），同时把这些对象的年龄+1（如果ServicorTo不够位置了就放到老年区）；
然后，清空Eden和ServicorFrom中的对象；最后，ServicorTo和ServicorFrom互换，原ServicorTo成为下一次GC时的ServicorFrom区。
tenured generation
gc:MajorGC
进入条件
survivor对象每经历一次GC,年龄+1,默认年龄15时,对象进入老年代
大对象(很长的字符串/数组)直接进入老年代
permanent generation PermGen PermGen space的全称是Permanent Generation space,是指内存的永久保存区域
主要存放Class和Meta（元数据）的信息,Class在被加载的时候被放入永久区域.它和存放实例的区域不同,GC不会在主程序运行期对永久区域进行清理。所这也导致了永久代的区域会随着加载的Class的增多而胀满，最终抛出OOM异常。
在Java8中，永久代已经被移除，被一个称为“元数据区”（元空间）的区域所取代。
元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。因此，默认情况下，元空间的大小仅受本地内存限制。类的元数据放入 native memory, 字符串池和类的静态变量放入java堆中. 这样可以加载多少类的元数据就不再由MaxPermSize控制, 而由系统的实际可用空间来控制.

------------------------------------------------------



