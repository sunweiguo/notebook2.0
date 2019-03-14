title: 从ReentrantLock引出AQS的原理
tag: java多线程
---

如果对并发编程稍微熟悉的话，就不会对ReentrantLock陌生，也可能对一些组件比如CountDownLatch,FutureTask以及Semaphore等同步组件耳闻过，他们都是JUC包下的类或者工具，他们都有一个共同的基础：AQS，即AbstractQueuedSynchronizer，从今天开始，让我们记住它，并且尝试去理解它。

<!--more-->

## 一、ReentrantLock

首先我们先来看看`ReentrantLock`这个可重入锁的性质和使用，因为它往往会在面试中被面试官拿来同`synchronized`相比较。如果这种基本的比较都不知道的话，那就没有后续深入的探讨了，面试可能也会结束了。

它的用法极其简单，如下：

![image](http://bloghello.oursnail.cn/thread7-1.jpg)

他们两兄弟的区别是：

- `synchronized`是关键字，`ReentrantLock`是一个类
- `ReentrantLock`可以对获取锁的等待时间进行设置，避免死锁
- `ReentrantLock`可以获取各种锁的信息
- `ReentrantLock`可以灵活地实现多路通知
- 机制：`synchronized`操作`MarkWord`，`lock`调用`Unsafe`类的`park()`方法
- `ReentrantLock`可以设置锁的公平性
- `ReentrantLock`调用`lock()`之后必须调用`unlock()`释放锁
- 性能上`ReentrantLock`未必就比`synchronized`高，他们都是可重入的

可以看出，`ReentrantLock`更加灵活，可以更加细腻度操作锁，而`synchronized`看起来则相对比较笨拙，但是笨拙的是简单的，不存在忘记释放锁的问题。可谓存在即合理嘛！

针对上文中提到的`Unsafe`类，其中最经典的一个方法是：`compareAndSwapXXX`这类`CAS`方法，它其实是JAVA留的一个后门，它可以直接操作内存，因此如果普通开发者拿来用的话，可能会出现各种问题，因此被成为不安全的类。

好了，关于区别已经说的差不多了，下面我们就要来真格的了，首先来翻翻源码。**前方高能预警，请非战斗人员紧急撤离现场，老司机要开车了。**


首先呢，我们来看看`lock()`方法的实现是：


```java
public void lock() {
    sync.lock();
}
```

这里多了一个东西叫`Sync`，`Sync`为`ReentrantLock`里面的一个内部类，它继承`AQS`，它有两个子类：公平锁`FairSync`和非公平锁`NonfairSync`。

`ReentrantLock`里面大部分的功能都是委托给`Sync`来实现的，同时`Sync`内部定义了`lock()`抽象方法由其子类去实现，默认实现了`nonfairTryAcquire(int acquires)`方法，可以看出它是非公平锁的默认实现方式。

![image](http://bloghello.oursnail.cn/thread7-2.jpg)

几乎每一个方法都是通过`sync.xxx`来实现的，而`Sync`这个内部类在`AQS`的基础上增加一些东西而已，所以本质上都是基于`AQS`来实现的。

不仅仅是这个，JUC包基本都是以`AQS`为基础构成，因此`AQS`可以理解为JUC的一个实现框架。既然`AQS`这么重要，下面有必要挖地三尺掘出它的原理。

## 二、AQS简介

java的内置锁一直都是备受争议的，在JDK 1.6之前，`synchronized`这个重量级锁性能一直都是较为低下，虽然在1.6后，进行大量的锁优化策略,但是与`Lock`相比`synchronized`还是存在一些缺陷的：虽然`synchronized`提供了便捷性的隐式获取锁释放锁机制（基于JVM机制），但是它却缺少了获取锁与释放锁的可操作性，可中断、超时获取锁，且它为独占式在高并发场景下性能大打折扣。


AQS：`AbstractQueuedSynchronizer`，即队列同步器。它是构建锁或者其他同步组件的基础框架（如`ReentrantLock`、`ReentrantReadWriteLock`、
`Semaphore`等），JUC并发包的作者（`Doug Lea`）期望它能够成为实现大部分同步需求的基础。它是JUC并发包中的核心基础组件。


AQS解决了在实现同步器时涉及当的大量细节问题，例如获取同步状态、FIFO同步队列。基于AQS来构建同步器可以带来很多好处。它不仅能够极大地减少实现工作，而且也不必处理在多个位置上发生的竞争问题。

AQS的主要使用方式是继承，子类通过继承同步器并实现它的抽象方法来管理同步状态。

AQS使用一个`int`类型的成员变量`state`来表示同步状态，当`state>0`时表示已经获取了锁，当`state = 0`时表示释放了锁。它提供了三个方法（`getState()`、`setState(int newState)`、`compareAndSetState(int expect,int update)`）来对同步状态`state`进行操作，当然AQS可以确保对`state`的操作是安全的。


AQS通过内置的`FIFO`同步队列来完成资源获取线程的排队工作，如果当前线程获取同步状态失败（锁）时，AQS则会将当前线程以及等待状态等信息构造成一个节点（Node）并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，则会把节点中的线程唤醒，使其再次尝试获取同步状态。

## 三、CLH同步队列

`CLH`同步队列是一个`FIFO`双向队列，AQS依赖它来完成同步状态的管理，当前线程如果获取同步状态失败时，AQS则会将当前线程已经等待状态等信息构造成一个节点（Node）并将其加入到CLH同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点唤醒（公平锁），使其再次尝试获取同步状态。

在`CLH`同步队列中，一个节点表示一个线程，它保存着线程的引用（`thread`）、状态（`waitStatus`）、前驱节点（`prev`）、后继节点（`next`），`CLH`同步队列结构图如下：

![image](http://bloghello.oursnail.cn/thread7-9.jpg)

举例理解：假设目前有三个线程`Thread1`、`Thread2`、`Thread3`同时去竞争锁，如果结果是`Thread1`获取了锁，`Thread2`和`Thread3`进入了等待队列，那么他们的样子如下：

![image](http://bloghello.oursnail.cn/thread7-10.jpg)

AQS的等待队列基于一个双向链表实现的，`HEAD`节点不关联线程，后面两个节点分别关联`Thread2`和`Thread3`，他们将会按照先后顺序被串联在这个队列上。这个时候如果后面再有线程进来的话将会被当做队列的`TAIL`。

## 四、入列


```java
private Node addWaiter(Node mode) {
    //新建Node
    Node node = new Node(Thread.currentThread(), mode);
    //快速尝试添加尾节点
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        //CAS设置尾节点
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

`addWaiter(Node node)`先通过快速尝试设置尾节点，如果失败，则调用`enq(Node node)`方法设置尾节点:


```java
private Node enq(final Node node) {
    //多次尝试，直到成功为止 
    for (;;) {
        Node t = tail;
        //tail不存在，设置为首节点
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            //设置为尾节点 
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

其实就很明了了，首先是尝试快速用`CAS`设置当前的节点为尾节点，但是可能存在并发问题设置不成功，下面用死循环的方式不断地尝试添加节点并且设置为尾节点，直到成功。

过程如下：

![image](http://bloghello.oursnail.cn/thread7-4.jpg)

## 五、出列

CLH同步队列遵循`FIFO`，首节点的线程释放同步状态后，将会唤醒它的后继节点（`next`），而后继节点将会在获取同步状态成功时将自己设置为首节点，这个过程非常简单，head执行该节点并断开原首节点的`next`和当前节点的`prev`即可，注意在这个过程是不需要使用CAS来保证的，因为只有一个线程能够成功获取到同步状态。

![image](http://bloghello.oursnail.cn/thread7-5.jpg)

其实这里按照源码的解释，是将第一个获取到同步状态的`node`作为新的`head`，然后将原来的`head`置空。


## 六、同步状态的获取与释放

在前面提到过，AQS是构建Java同步组件的基础，我们期待它能够成为实现大部分同步需求的基础。AQS的设计模式采用的模板方法模式，子类通过继承的方式，实现它的抽象方法来管理同步状态，对于子类而言它并没有太多的活要做，AQS提供了大量的模板方法来实现同步，主要是分为三类：独占式获取和释放同步状态、共享式获取和释放同步状态、查询同步队列中的等待线程情况。自定义子类使用AQS提供的模板方法就可以实现自己的同步语义。

**下面具体来解释一下独占式和共享式的含义**

在具体分析之前，我们先解释两种同步的方式，独占模式和共享模式：

- 独占模式：资源是独占的，一次只能一个线程获取。
- 共享模式：同时可以被多个线程获取，具体的资源的个数可以通过参数指定。

如果我们自己实现一个同步器的框架，我们怎么设计呢？下面可能是我们想到的比较通用的设计方案（独占模式）:

- 定义一个变量`int state=0`，使用这个变量表示被获取的资源的数量。
- 线程在获取资源前要先检查`state`的状态，如果为0，则修改为1，表示获取资源成功，否则表示资源已经被其他线程占用，此时线程要堵塞以等待其他线程释放资源。
- 为了能使得资源释放后找到那些为了等待资源而堵塞的线程，我们把这些线程保存在FIFO队列中。
- 当占有资源的线程释放掉资源后，可以从队列中唤醒一个堵塞的线程，由于此时资源已经释放，因此这个被唤醒的线程可以获取资源并且执行。

这个`state`变量到底是什么呢？

- 当`AQS`的子类实现独占功能时，如`ReentrantLock`，资源是否可以被访问被定义为：只要`AQS`的`state`变量不为0，并且持有锁的线程不是当前线程，那么代表资源不可访问。此时，`state`是用来表示当前线程获取锁的可重入次数；
- 当`AQS`的子类实现共享功能时，如`CountDownLatch`，资源是否可以被访问被定义为：只要`AQS`的`state`变量不为0，那么代表资源不可以为访问。此时，`state`用来表示当前计数器的值。

## 七、独占式-独占式同步状态获取

独占式，同一时刻仅有一个线程持有同步状态。

独占式同步状态获取`acquire(int arg)`方法为AQS提供的模板方法，该方法为独占式获取同步状态，但是该方法对中断不敏感，也就是说由于线程获取同步状态失败加入到CLH同步队列中，后续对线程进行中断操作时，线程不会从同步队列中移除。代码如下：


```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        {
            selfInterrupt();
        }
}
```

- `tryAcquire`：去尝试获取锁，获取成功则设置锁状态并返回true，否则返回false。该方法自定义同步组件自己实现(`ReentrantLock`中实现公平锁和非公平锁就是分别重写了这个方法实现的，下面看`ReentrantLock`的原理的时候就明白了)，该方法必须要保证线程安全的获取同步状态。
- `addWaiter`：如果`tryAcquire`返回`FALSE`（获取同步状态失败），则调用该方法将当前线程加入到CLH同步队列尾部。
- `acquireQueued`：当前线程会根据公平性原则来进行阻塞等待（自旋）,直到获取锁为止；并且返回当前线程在等待过程中有没有中断过。
- `selfInterrupt`：产生一个中断。

对这里的`acquireQueued`有疑惑，下面来看看它做了什么。`acquireQueued`方法为一个自旋的过程，也就是说当前线程（`Node`）进入同步队列后，就会先进入一个自旋的过程，每个节点都会自省地观察，当条件满足，获取到同步状态后，就可以从这个自旋过程中退出。


```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //获取当前节点node的前驱结点p
            final Node p = node.predecessor();
            //如果p确实是head，那说明当前节点node是可用的第一个线程
            //即为当前队列的第一个线程，则最先处理它
            //当前线程则尝试获取同步状态
            if (p == head && tryAcquire(arg)) {
                //从这里可以看出，更新当前节点为头节点
                //将原来头节点的next引用置空以供JVM回收
                //具体见出列小标题下的示意图
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //如果前驱节点不是头节点就继续阻塞继续等待呗
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

从上面代码中可以看到，当前线程会一直尝试获取同步状态，当然前提是只有其前驱节点为头结点才能够尝试获取同步状态，理由：

- 保持FIFO同步队列原则。
- 头节点释放同步状态后，将会唤醒其后继节点，后继节点被唤醒后需要检查自己是否为头节点。

对这个的理解简单来说就是：

> 在AQS中维护着一个FIFO的同步队列，当线程获取同步状态失败后，则会加入到这个CLH同步队列的队尾并一直保持着自旋。在CLH同步队列中的线程在自旋时会判断其前驱节点是否为首节点，如果当前节点的前驱节点就是头节点，则表明当前节点是当前队列中的第一个可用线程，则让其不断尝试获取同步状态，如果获取到，则退出CLH同步队列。当线程执行完逻辑后，会释放同步状态，释放后会唤醒其后继节点。

继续，我们看到，如果发现前驱节点并不是`head`，那么就说明是比较靠后的节点了，这个时候，很有可能需要一段时间之后才会用到它，所以根本不需要再参与自旋浪费CPU的性能了，即下面一个if:


```java
if (shouldParkAfterFailedAcquire(p, node) &&
    parkAndCheckInterrupt())
    interrupted = true;
```
通过这段代码我们可以看到，在获取同步状态失败后，线程并不是立马进行阻塞，需要检查该线程的状态，检查状态的方法为 `shouldParkAfterFailedAcquire(Node pred, Node node)` 方法，该方法主要靠前驱节点判断当前线程是否应该被阻塞，代码如下：


```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //前驱节点
    int ws = pred.waitStatus;
    //状态为signal，表示当前线程处于等待状态，直接放回true
    if (ws == Node.SIGNAL)
        return true;
    //前驱节点状态 > 0 ，则为Cancelled,表明该节点已经超时或者被中断了，需要从同步队列中取消
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    }
    //前驱节点状态为Condition、propagate
    else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

这段代码主要检查当前线程是否需要被阻塞，具体规则如下：

- 如果当前线程的前驱节点状态为`SINNAL`，则表明当前线程需要被阻塞，调用`unpark()`方法唤醒，直接返回true，当前线程阻塞
- 如果当前线程的前驱节点状态为`CANCELLED（ws > 0）`，则表明该线程的前驱节点已经等待超时或者被中断了，则需要从CLH队列中将该前驱节点删除掉，直到回溯到前驱节点状态 <= 0 ，返回false
- 如果前驱节点非`SINNAL`，非`CANCELLED`，则通过CAS的方式将其前驱节点设置为`SINNAL`，返回false

针对`pred.waitStatus`的几种状态：


```java
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
```

如果 `shouldParkAfterFailedAcquire(Node pred, Node node)` 方法返回`true`，则调用`parkAndCheckInterrupt()`方法阻塞当前线程：


```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

`parkAndCheckInterrupt()` 方法主要是把当前线程挂起，从而阻塞住线程的调用栈，同时返回当前线程的中断状态。其内部则是调用`LockSupport`工具类的`park()`方法来阻塞该方法。

那么，此时，当第一个线程已经执行完毕，释放锁了，就需要唤醒队列中后继节点：


```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            //唤醒后继节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
调用`unparkSuccessor(Node node)`唤醒后继节点：


```java
private void unparkSuccessor(Node node) {
    //当前节点状态
    int ws = node.waitStatus;
    //当前状态 < 0 则设置为 0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    //当前节点的后继节点
    Node s = node.next;
    //后继节点为null或者其状态 > 0 (超时或者被中断了)
    if (s == null || s.waitStatus > 0) {
        s = null;
        //从tail节点来找可用节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    //唤醒后继节点
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

可能会存在当前线程的后继节点为`null`，超时、被中断的情况，如果遇到这种情况了，则需要跳过该节点，但是为何是从`tail`尾节点开始，而不是从`node.next`开始呢？原因在于`node.next`仍然可能会存在`null`或者取消了，所以采用`tail`回溯办法找第一个可用的线程。最后调用`LockSupport`的`unpark(Thread thread)`方法唤醒该线程。

从上面我可以看到，当需要阻塞或者唤醒一个线程的时候，AQS都是使用LockSupport这个工具类来完成的。

`LockSupport`定义了一系列以`park`开头的方法来阻塞当前线程，`unpark(Thread thread)`方法来唤醒一个被阻塞的线程。这些方法的实现都是通过`Unsafe`类调用`native`方法来实现的。

好了，至此就完完全全地搞明白了独占式同步状态获取`acquire(int arg)`方法的原理，特别是其中节点如何进出、队列第一个节点如何尝试获取同步状态、如何阻塞后继线程以及如何唤醒。

## 八、独占式获取响应中断

`AQS`提供了`acquire(int arg)`方法以供独占式获取同步状态，但是该方法对中断不响应，对线程进行中断操作后，该线程会依然位于`CLH`同步队列中等待着获取同步状态。为了响应中断，`AQS`提供了`acquireInterruptibly(int arg)`方法，该方法在等待获取同步状态时，如果当前线程被中断了，会立刻响应中断抛出异常`InterruptedException`。

具体原理就不深究了，其实源码跟上面个相差不大，只是不再是使用`interrupted`标志，而是直接抛出`InterruptedException`异常。再深究这博客没法继续写啦。

## 九、独占式超时获取

`AQS`除了提供上面两个方法外，还提供了一个增强版的方法：`tryAcquireNanos(int arg,long nanos)`。该方法为`acquireInterruptibly`方法的进一步增强，它除了响应中断外，还有超时控制。即如果当前线程没有在指定时间内获取同步状态，则会返回`false`，否则返回`true`。


针对超时控制，程序首先记录唤醒时间`deadline` :`deadline` = `System.nanoTime()` +` nanosTimeout`（时间间隔）。

如果获取同步状态失败，则需要计算出需要休眠的时间间隔`nanosTimeout` = `deadline` - `System.nanoTime()`，如果`nanosTimeout` <= 0 表示已经超时了，返回`false`;

如果大于`spinForTimeoutThreshold(1000L)`则需要休眠`nanosTimeout` ;

如果`nanosTimeout` <= `spinForTimeoutThreshold` ，就不需要休眠了，直接进入快速自旋的过程。原因在于 `spinForTimeoutThreshold` 已经非常小了，非常短的时间等待无法做到十分精确，如果这时再次进行超时等待，相反会让`nanosTimeout` 的超时从整体上面表现得不是那么精确，所以在超时非常短的场景中，AQS会进行无条件的快速自旋。

流程图如下：

![image](http://bloghello.oursnail.cn/thread7-8.jpg)

## 十、共享式-共享式同步状态获取

共享式与独占式的最主要区别在于同一时刻独占式只能有一个线程获取同步状态，而共享式在同一时刻可以有多个线程获取同步状态。例如读操作可以有多个线程同时进行，而写操作同一时刻只能有一个线程进行写操作，其他操作都会被阻塞。

`AQS`提供`acquireShared(int arg)`方法共享式获取同步状态：

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```
从上面程序可以看出，方法首先是调用`tryAcquireShared(int arg)`方法尝试获取同步状态，如果获取失败则调用`doAcquireShared(int arg)`自旋方式获取同步状态，共享式获取同步状态的标志是返回 >= 0 的值表示获取成功。自旋方式获取同步状态如下：

```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
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
逻辑几乎和独占式锁的获取一模一样，这里的自旋过程中能够退出的条件是当前节点的前驱节点是头结点并且`tryAcquireShared(arg)`返回值大于等于0即能成功获得同步状态。

`acquireShared(int arg)`方法不响应中断，与独占式相似，AQS也提供了响应中断、超时的方法，分别是：`acquireSharedInterruptibly(int arg)`、`tryAcquireSharedNanos(int arg,long nanos)`，这里就不做解释了。

## 十一、共享式同步状态释放

获取同步状态后，需要调用`release(int arg)`方法释放同步状态，方法如下：


```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

因为可能会存在多个线程同时进行释放同步状态资源，所以需要确保同步状态安全地成功释放，一般都是通过CAS和循环来完成的。


## 十二、再回过头来看看ReentrantLock的原理

在对AQS原理进行大概了梳理之后，再来理解`ReentrantLock`就比较容易了，因为大部分的事情都由AQS做完了，剩下的只要重写几个个性化的方法即可。

还是要看看最核心的方法：`lock()`方法


```java
public void lock() {
    sync.lock();
}
```

下面来看看这个`lock()`，一点点进了抽象静态内部类`Sync`中去了：


```java
abstract void lock();
```

上面说过，`ReentrantLock`里面大部分的功能都是委托给`Sync`来实现的，同时`Sync`内部定义了`lock()`抽象方法由其子类去实现的，所以这个`lock`方法的具体实现是在子类中完成的。`Sync`的子类有`NonfairSync`和`FairSync`这两个，一看就知道了，一个是非公平一个是公平。

## 十三、非公平锁

先来看看比较简单的非公平锁：


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

我们看到，这个`lock()`方法里面首先用`CAS`尝试获取锁，获取不到则执行`acquire()`方法，这个方法就恰好是完全由`AQS`实现，那么就回到了上面介绍过的内容了。这里为了方便再贴一下源码：


```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

首先就是调用`tryAcquire()`这个方法，即尝试获取锁，这个方法上面也提过，是留给具体的类自己去实现的，所以我们还要回到`ReentrantLock`中来看看，果然，在上面贴的`NonfairSync`这个类中对这个方法进行了重写。即：


```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
```
调用的方法就是实现尝试获取锁的核心代码：


```java
final boolean nonfairTryAcquire(int acquires) {
    //当前线程
    final Thread current = Thread.currentThread();
    //获取同步状态
    int c = getState();
    //state == 0,表示该锁未被任何线程占有，该锁能被当前线程获取
    if (c == 0) {
        //获取锁成功，设置为当前线程所有
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //线程重入
    //判断锁持有的线程是否为当前线程
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
就很简单了，值得注意的是，为了支持重入性，在第二步增加了处理逻辑，如果该锁已经被线程所占有了，会继续检查占有线程是否为当前线程，如果是的话，同步状态加1返回true，表示可以再次获取成功。每次重新获取都会对同步状态进行加一的操作。

另外需要注意的是，这是非公平锁，就是说，一个线程进来，可能是比先进来的线程先获取锁，就像在开车的时候，总是会由一些车插到你的前面一样。但是如果它没有获取锁，则入队。

那么尝试获取锁的逻辑我们知道了，那么释放锁呢？


```java
protected final boolean tryRelease(int releases) {
	//1. 同步状态减1
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
		//2. 只有当同步状态为0时，锁成功被释放，返回true
        free = true;
        setExclusiveOwnerThread(null);
    }
	// 3. 锁未被完全释放，返回false
    setState(c);
    return free;
}
```

需要注意的是，重入锁的释放必须得等到同步状态为0时锁才算成功释放，否则锁仍未释放。如果锁被获取n次，释放了n-1次，该锁未完全释放返回false，只有被释放n次才算成功释放，返回true。


## 十四、公平锁

何谓公平性，是针对获取锁而言的，如果一个锁是公平的，那么锁的获取顺序就应该符合请求上的绝对时间顺序，满足`FIFO`。`ReentrantLock`的构造方法无参时是构造非公平锁。

提供了有参构造函数，可传入一个`boolean`值，`true`时为公平锁，`false`时为非公平锁，源码为：


```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

在上面非公平锁获取时（`nonfairTryAcquire`方法）只是简单的获取了一下当前状态做了一些逻辑处理，并没有考虑到当前同步队列中线程等待的情况。我们来看看公平锁的处理逻辑是怎样的，核心方法为：


```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
  }
}
```

这段代码的逻辑与`nonfairTryAcquire`基本上一致，唯一的不同在于增加了`hasQueuedPredecessors`的逻辑判断，方法名就可知道该方法用来判断当前节点在同步队列中是否有前驱节点的判断，如果有前驱节点说明有线程比当前线程更早的请求资源，根据公平性，当前线程请求资源失败。如果当前节点没有前驱节点的话，再才有做后面的逻辑判断的必要性。公平锁每次都是从同步队列中的第一个节点获取到锁，而非公平性锁则不一定，有可能刚释放锁的线程能再次获取到锁。


## 十五、公平锁 VS 非公平锁

- 公平锁每次获取到锁为同步队列中的第一个节点，保证请求资源时间上的绝对顺序，而非公平锁有可能刚释放锁的线程下次继续获取该锁，则有可能导致其他线程永远无法获取到锁，造成“饥饿”现象。
- 公平锁为了保证时间上的绝对顺序，需要频繁的上下文切换，而非公平锁会降低一定的上下文切换，降低性能开销。因此，`ReentrantLock`默认选择的是非公平锁，则是为了减少一部分上下文切换，保证了系统更大的吞吐量。


