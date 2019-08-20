

## ReentrantLock类

> A reentrant mutual exclusion Lock with the same basic behavior and semantics as the implicit monitor lock accessed using synchronized methods and statements, but with extended capabilities.
>
> ReentrantLock是一个可重入的互斥锁，它具有与使用synchronized的方法和语句访问的隐式监视器锁相同的基本行为和语义，但它具有可扩展的能力。------JDK API



ReentrantLock类有三个静态内部类：

- Sync类
- NonfairSync类
- FairSync类。

其中Sync类继承自AbstractQueuedSynchronizer(抽象队列同步器)，而FairSync和NonfairSync为Sync类的两个实现，分别应用于公平锁和非公平锁的场景。

ReentrantLock类的大部分逻辑，都是由Sync类实现的。

ReentrantLock提供了公平锁机制，构造方法接收一个可选的公平参数。当设置为true时，它是公平锁，这时锁会将访问权授予等待时间最长的线程。否则该锁将无法保证线程获取锁的访问顺序。

公平锁与非公平锁相比，使用非公平锁的程序会有较高的吞吐量，但使用公平锁能有效减少线程饥饿的发生。

## AbstractQueuedSynchronizer类

AbstractQueuedSynchronizer即抽象队列同步器，AbstractQueuedSynchronizer类内部维护了一个FIFO的队列，是构建锁或者其他相关同步装置的基础框架。使用队列的好处，可以有效避免惊群效应，防止线程的过度唤醒。

AQS(AbstractQueuedSynchronizer)同步器拥有三个成员变量：队列的头结点**head**、队列的尾节点**tail**和状态变量**state**。

```java
private transient volatile Node head;
private transient volatile Node tail;
private volatile int state;
```

简单来说，当一个线程访问锁资源时，大致会经历如下步骤：

1. 当前线程首先判断同步器的state变量的值，如果state为0，则表示锁资源没有被任何线程获取，经过一系列的操作，当前线程获取锁资源。
2. 如果state>0,则表示当前锁资源已经被某个线程获取，此时当前线程会继续获取同步器exclusiveOwnerThread属性的值，该属性表示当前锁资源被哪个线程占有，如果exclusiveOwnerThread属性的值是当前线程对象，则表示该锁被重入，当前线程成功获取锁资源。
3. 如果exclusiveOwnerThread属性的值不是当前线程对象，则表示锁资源被其他线程占有，此时线程会将自己封装成一个Node节点，将该节点添加到AQS同步器所维护的队列的队尾，并阻塞当前线程。



示意图如下：

![image-20190812175747591](/Users/lijiaxing/Desktop/2019/blog/image-20190812175747591.png)



整体类图关系如下图所示：

![image-20190812140142468](/Users/lijiaxing/Desktop/2019/blog/image-20190812140142468.png)



## ReentrantLock获取非公平锁的源码学习

##### 例子：

```java
public static void main(String[] args) throws InterruptedException {
    Lock lock = new ReentrantLock();//创建非公平锁
    try {
        lock.lock();//获取锁资源
        TimeUnit.SECONDS.sleep(2);//模拟业务代码执行
    }catch (Exception e){

    }finally {
        lock.unlock();//释放锁资源
    }
}
```

##### ReentrantLock类的构造方法

ReentrantLock类有两个构造方法，不传参数则默认使用非公平模式

```java
public ReentrantLock() {
    sync = new NonfairSync();//默认使用非公平模式
}
```

```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

##### lock.lock()方法

获取锁资源的流程从lock.lock()方法开始

```java
public void lock() {
    sync.lock();//调用内部类Sync中的lock方法
}
```

前面说过ReentrantLock类的大部分逻辑，都是在Sync类及其子类中实现的。所以sync.lock()才是获取锁资源真正的入口。

##### sync.lock()方法

```java
/**
*	获取锁资源的入口
*/
final void lock() {
    if (compareAndSetState(0, 1))//使用cas算法更新AQS中的同步状态变量(state)
        setExclusiveOwnerThread(Thread.currentThread());//将当前线程设置为持有锁的线程
    else
        acquire(1);
}
```

lock()方法的大致逻辑为，先使用cas算法尝试将AQS的state属性更新为1，只有在state的值为0时才能更新成功。因为state表示的是当前锁资源被重入的次数，所以state等于0表示当前锁资源未被任何线程持有，如果cas替换成功，则表示当前线程获取到了锁资源，那么只需要将当前线程标记为持有锁资源的线程（将当前线程对象赋值给AQS的exclusiveOwnerThread属性），整个获取锁的流程就结束了。如果cas替换失败，则执行acquire(1);

##### acquire(int arg)方法

```java
public final void acquire(int arg) {
	if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
		selfInterrupt();
}
```

acquire(int arg)方法中涉及三个重要的方法

```java
boolean tryAcquire(int arg)
```

```java
boolean acquireQueued(final Node node, int arg)
```

```java
Node addWaiter(Node mode)
```

先分析tryAcquire(int arg)方法

##### AQS的tryAcquire(int arg)方法

tryAcquire(int acquires)在AQS中的定义如下，该方法直接抛出了一个异常

```java
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();//模板模式，具体的实现交给子类完成
}
```

##### NonFairSync的tryAcquire(int arg)

因为现在分析的是非公平锁的获取，所以非公平模式的实现如下：

```java
protected final boolean tryAcquire(int acquires) {
	return nonfairTryAcquire(acquires);
}
```

tryAcquire调用nonfairTryAcquire() 

```java
final boolean nonfairTryAcquire(int acquires) {
	final Thread current = Thread.currentThread();
	int c = getState();//获取AQS中的同步状态变量的值
	if (c == 0) {//state为0，表示锁没有被任何线程持有
		if (compareAndSetState(0, acquires)) {//使用cas算法将state的值设置为1，即获取锁
			setExclusiveOwnerThread(current);//设置持有锁的线程为当前线程
			return true;
		}
	}else if (current == getExclusiveOwnerThread()) {//如果持有锁的线程是当前线程
		int nextc = c + acquires;
		if (nextc < 0) // 一个线程支持的最大重入次数为 2147483647，超出则会抛出Error
			throw new Error("Maximum lock count exceeded");
		setState(nextc);//设置state值
		return true;
	}
	return false;
}

```

上面代码是非公平模式下获取锁资源的流程，分为三种情况：

1. 如果锁资源没有被任何线程所持有，则直接获取锁资源
2. 如果锁已经被当前线程获取，则对AQS的状态变量加1，表示锁的重入次数加1
3. 否则将返回false，表示获取锁资源失败



返回到上面的的acquire方法：

```java
public final void acquire(int arg) {
	if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
		selfInterrupt();
}
```

由上面代码可以看出只有当tryAcquire(arg)返回false（没有获取到锁资源）才会执行后面的代码

```
acquireQueued(addWaiter(Node.EXCLUSIVE), arg)
```

##### addWaiter(Node.EXCLUSIVE)

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);//将当前线程封装到一个Node节点中
    Node pred = tail;
    //---start---
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //---end---
    enq(node);//enq方法中也有上面注释包围的逻辑
    return node;
}
```

上面代码的注释括起来的部分可以不看，因为在下面的enq(node)方法中也有同样的逻辑。

##### enq(final Node node)方法

```java
//参数为封装了当前线程的node对象
private Node enq(final Node node) {
    for (;;) { 
        Node t = tail;
        if (t == null) { //链表为空，此时表示链表需要初始化
            if (compareAndSetHead(new Node()))//创建一个空节点，并将其设置为链表的head
                tail = head;//Q2
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {//使用cas算法将当前节点设置为链表的尾节点
                t.next = node;//Q2
                return t;
            }
        }
    }
}
```

enq(final Node node)方法的大体逻辑为如果链表为空则初始化链表，头结点为一个空节点，第二次循环时将当前线程的节点添加在队列的末尾。

##### 问题1：既然enq(final Node node)方法中已经包含了上面addWaiter(Node mode)方法中的部分逻辑，为什么相同的逻辑要写两遍？

```
在通用的逻辑处理之前提前处理某些特殊的逻辑，优点是可以提升性能，缺点是牺牲代码的简洁性。
```

##### 问题2：enq(final Node node)方法中Q2注释处的代码为什么不用cas算法，这样做是线程安全的吗？

```java
for (;;) { 
    Node t = tail;
    if (t == null) { 
        if (compareAndSetHead(new Node()))//1
            tail = head;
    } else {
        node.prev = t;
        if (compareAndSetTail(t, node)) {
            t.next = node;
            return t;
        }
    }
}
```

```java
private final boolean compareAndSetHead(Node update) {
    //调用Unsafe类的compareAndSwapObject方法设置头结点
    //compareAndSwapObject是一个native方法，参数列表如下
    //第一个参数：要修改的对象
    //第二个参数：要修改的对象的属性在内存中地址的偏移量
    //第三个参数：预期值
    //第四个参数：要设置的值
    return unsafe.compareAndSwapObject(this, headOffset, null, update);
}
```

相关代码如上面所示，假设有两个线程同时执行到注释1处的if语句，因为compareAndSetHead方法使用cas算法实现，所以两个线程中只会有一个线程成功设置头结点，而另外一个线程设置头结点时因为实际值和预期值不相等，则会设置失败。所以后面的 tail = head;语句不会有线程安全问题。

##### acquireQueued(final Node node, int arg)

返回到上面的代码：

```java
public final void acquire(int arg) {
	if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
		selfInterrupt();
}
```

由上面的代码可知，调用addWaiter方法将当前线程封装成一个Node节点，并将该节点添加到队列的末尾后将会执行acquireQueued(final Node node, int arg)方法，该方法的逻辑如下所示：

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;//线程中断标志
        for (;;) {
            final Node p = node.predecessor();//获取node节点的前驱节点
            if (p == head && tryAcquire(arg)) {//判断node节点的前驱节点是否是头结点且再次尝试获取锁
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //shouldParkAfterFailedAcquire方法：判断当前线程是否应该被中断
            //parkAndCheckInterrupt方法：中断当前线程
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

上面代码的主要逻辑为：

1. 先获取当前节点前驱节点，判断前驱节点是否是头结点。
2. 如果前驱节点是头结点，那么会再次尝试获取锁资源。
3. 如果前驱节点是头结点且再次获取锁资源成功，则将当前节点设置为头结点，返回false。
4. 如果if p == head && tryAcquire(arg)的结果为false，即前驱节点不是头结点或前驱节点是头结点但是尝试获取锁资源失败，则会执行shouldParkAfterFailedAcquire(Node pred, Node node)方法判断当前线程是否应该被中断。
5. 如果shouldParkAfterFailedAcquire(Node pred, Node node)方法返回true则执行parkAndCheckInterrupt()方法中断当前线程，并返回true，否则将继续自旋直到满足上述某一条件。



```java
/*
* 	Node.SHARED = new Node(); 状态位，表示当前节点使用共享模式等待
*	Node.EXCLUSIVE = null; 状态位，表示当前节点使用独占模式等待
*	Node.CANCELLED = 1; 状态位，表示当前线程已被取消
*	Node.SIGNAL = -1; 状态位，表示后继线程需要取消阻塞
*	Node.CONDITION = -2; 状态位，表示线程由于阻塞而处于等待状态
*	Node.PROPAGATE = -3; 状态位，表示处于共享模式下，下一次的acquire需要无条件的传播
* 	None of the above    0 
*
*	可以看到代码中会将前驱节点的状态设置为Node.SIGNAL，因为在后面线程释放锁并唤醒其他线程的逻辑中，只有	
*	waitStatus为Node.SIGNAL的节点才能唤醒后继节点封装的线程
*/
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;//获取前驱节点的等待状态
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {//将状态为“取消”的线程从队列中剔除
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);//使用cas将waitStatus的值设置为Node.SIGNAL
    }
    return false;
}
```



```java
//中断当前线程
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```



时序图如下：

![image-20190813152543032](/Users/lijiaxing/Desktop/2019/blog/image-20190813152543032.png)



![image-20190812171348453](/Users/lijiaxing/Desktop/2019/blog/image-20190812171348453.png)





## ReentrantLock释放非公平锁的源码学习

##### 调用ReentrantLock类的unlock()方法

```java
public void unlock() {
    sync.release(1);//调用内部类Sync中的release方法
}
```

##### 调用Sync的release(int arg)方法

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {//尝试释放锁资源
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);//唤醒后继节点所封装的线程
        return true;
    }
    return false;
}
```



```java
//模板方法，具体逻辑由子类实现
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
```



```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;//将状态变量state的值减1
    //如果当前线程不是持有锁资源的线程，则抛出非法锁监视器状态异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;//当状态变量state为0时意味着锁被完全释放
        setExclusiveOwnerThread(null);//设置持有锁的线程为空
    }
    setState(c);
    return free;
}
```



```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)//如果节点的waitStatus<0，则将其设置为0(初始状态)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;//获取头节点的后继节点
    //如果后继节点为null或后继节点的状态为canceled，则从尾节点开始遍历，
    if (s == null || s.waitStatus > 0) {
        s = null;
        //找到离头结点最近的非cancelled状态的节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        //唤醒该线程
        LockSupport.unpark(s.thread);
}
```