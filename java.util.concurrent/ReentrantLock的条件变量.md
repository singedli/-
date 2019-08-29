

测试代码如下：

```java
public class ConditionTest {
	
    static ReentrantLock lock = new ReentrantLock();				//创建锁对象
    static Condition condition = lock.newCondition();				//创建条件变量
    public static void main(String[] args) throws InterruptedException {

        Thread.currentThread().setName("main");
        Thread thread = new Thread(new SignalThread());
        thread.setName("子线程");
        thread.start();

        lock.lock();
        System.out.println("主线程获取锁资源");
        try {
            System.out.println("主线程等待通知");
            condition.await();										//主线程释放锁资源，并被阻塞
        } finally {
            lock.unlock();
            System.out.println("主线程释放锁资源");
        }
    }
	
	
    /**
    * 子线程的任务类
    */
    static class SignalThread implements Runnable {
        @Override
        public void run() {
            lock.lock();
            System.out.println("子线程获取锁资源");
            try {
                condition.signal();
                System.out.println("子线程通知");
            } finally {
                lock.unlock();
                System.out.println("子线程释放锁资源");
            }
        }
    }


}
```



执行结果如下图所示：









1、创建锁对象，因为条件变量是由ReentrantLock创建的

```java
    static ReentrantLock lock = new ReentrantLock();
```

创建条件变量

```
    static Condition condition = lock.newCondition();
```



Lock.newCondition()方法的实现如下：

```java
    public Condition newCondition() {
        return sync.newCondition();
    }
```

sync.newCondition()方法的实现如下：

```java
    final ConditionObject newCondition() {
        return new ConditionObject();
    }
```

可见条件变量Condition的具体实现是AQS中的一个内部类，即ConditionObject类，该类实现了Condition接口。

ConditionObject类中维护了一个FIFO的等待队列，且该队列为单向队列，等待队列中的节点复用了AQS同步队列的节点。节点的nextWaiter属性指向等待队列的下一个节点。而ConditionObject类的firstWaiter和lastWaiter属性分别表示等待队列的头结点和尾节点。

AQS阻塞队列示意图：



ConditionObject等待队列示意图：







## await()方法

await()方法的大致流程：

1、该线程首先会被封装成一个Node节点，并将该节点添加到ConditionObject对象的等待队列中。

2、接着当前线程会释放所持有的锁资源

3、调用LockSupport.park()方法将当前线程阻塞(线程状态为waiting)

4、线程被唤醒，判断标志位interruptMode的值，决定后续逻辑



```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    
    Node node = addConditionWaiter();//将当前线程封装成一个节点并将该节点添加到条件变量的等待队列中
    int savedState = fullyRelease(node);//完全释放锁资源
    int interruptMode = 0;
    //这里使用while是为了防止过早唤醒
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) 
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

await()方法详解：

1、调用addConditionWaiter()方法将当前线程封装成一个Node节点，并将该节点添加到ConditionObject的等待队列中

2、调用fullyRelease(node)方法将当前线程所持有的锁资源完全释放

3、 while (!isOnSyncQueue(node)){...}，判断当前线程的节点是否在AQS的同步队列中，因为当前线程的节点刚刚被加入到ConditionObject的等待队列中，所以isOnSyncQueue(node)方法返回false，取反则会进入到while循环体中

4、调用LockSupport.park(this)方法，将当前线程阻塞，此时线程的状态为WAITING。此时线程等待被唤醒，唤醒后的线程会从LockSupport.park(this)方法开始继续向下执行。

5、当线程被唤醒后，执行语句if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)判断当前线程的唤醒方式，以及对应的后续操作。

有两种方法可以将一个状态为WAITING的线程唤醒

- 调用LockSupport.unpark(Thread thread)方法。
- 调用调用thread.interrupt方法。没错，调用中断方法会将一个线程唤醒，thread.interrupt方法的cpp代码如下图所示，该方法确实会唤醒一个线程。



thread.interrupt方法的cpp代码



所以当一个线程被唤醒后就需要判断唤醒自己的方式是什么，从大的范围来说，此时共有两种可能性，即：

- 其他的线程调用Condition#signal()方法

  ​	如果是其他线程调用了Condition#signal()方法，则interruptMode标志位为0，跳出while循环。

  ​	跳出循环后面三个if条件都不会满足，所以会接着执行condition.await();后的代码

- 其他线程执行了中断方法thread.interrupt

  ​	其他线程执行了当前线程的中断方法分为两种情况。

  1. 执行thread.interrupt在Condition#signal()之前执行，此时将会抛出异常
  2. 执行thread.interrupt在Condition#signal()之后执行，此时会再次将线程的中断标志位置为true	



### addConditionWaiter()

将节点添加ConditionObject的等待队列的过程是不需要使用cas操作完成的，因为ConditionObject是基于lock锁的，而当前线程是持有锁的状态。

```java
//将当前线程封装成一个Node节点，并将其添加到等待队列的末尾
private Node addConditionWaiter() {
    Node t = lastWaiter;
    //链表不为空且尾节点的状态不等于Node.CONDITION
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();//将状态不为Node.CONDITION的节点从等待队列中剔除
        t = lastWaiter;
    }
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)//如果链表为空则直接添加到头结点
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```



### fullyRelease(Node node)

因为ReentrantLock是可重入锁，所以必须要完全释放锁资源，即将AQS的锁状态变量减为0

```java
//完全释放锁资源，将当前线程对象的节点从同步队列中移除
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        if (release(savedState)) {//释放锁资源，并唤醒AQS同步队列中的头结点
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```



### isOnSyncQueue(Node node)

判断当前节点是否在AQS的阻塞队列中

```java
//判断该节点是否在同步队列
final boolean isOnSyncQueue(Node node) {
    //如果节点状态是Node.CONDITION表示该节点不在同步队列中，Node.CONDITION状态表示节点在Condition的
    //等待队列中
    //如果该节点的prev属性为空表示该节点不在同步队列中，因为阻塞队列的头结点是new Node(),
    //所以node.prev属性不可能为空
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    //node.next != null表示当前节点有后继节点，表示当前节点一定在同步队列
    if (node.next != null) 
        return true;
   //如果当前节点是AQS同步队列的尾节点，则会进入下面findNodeFromTail(node)方法
    return findNodeFromTail(node);
}
```



### findNodeFromTail(Node node)

```java
//从阻塞队列的尾节点开始向前查找
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```



## signal()

```java
//唤醒等待队列中的线程
public final void signal() {
    if (!isHeldExclusively())//判断当前持有锁资源的线程是否是当前线程
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);//唤醒等待队列中的第一个节点所封装的线程
}
```



### doSignal(Node first)

将Condition的等待队列的头结点出队，并将符合条件的节点添加到AQS的等待队列中

```java
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```



### transferForSignal（Node node）

```JAVA
/**
*	true表示节点被成功入队
*	false表示节点在被通知之前被中断
*/
final boolean transferForSignal(Node node) {
   	//如果node节点的等待状态不为Node.CONDITION，则表示该节点的线程已经被取消或者在调用signal()
    //方法之前已经被其他线程中断，所以直接将该节点从condition的等待队列中剔除
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    Node p = enq(node);//将当前节点添加到AQS的同步队列中,返回值p为当前节点在阻塞队列的前驱节点
    int ws = p.waitStatus;
    //1、如果前驱节点的等待状态>0（状态为取消）,则直接唤醒当前线程，根据acquireQueued方法的逻辑，
    // 唤醒该线程后发现该线程的前驱节点不是AQS同步队列的头结点，则会将前驱节点从AQS同步队列中剔除，
    // 并中断当前线程的节点
    //2、!compareAndSetWaitStatus(p, ws, Node.SIGNAL)
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```



















