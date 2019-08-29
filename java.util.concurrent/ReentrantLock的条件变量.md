

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









1、创建锁对象，因为条件变量是有ReentrantLock创建的

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



## await()方法

当某个线程调用await()方法后：

1、该线程首先会被封装成一个Node节点，并将该节点添加到ConditionObject对象的等待队列中。

2、接着当前线程会释放所持有的锁资源

3、调用LockSupport.park()方法将当前线程阻塞(线程状态为waiting)



```java
/**
*
*/
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    
    Node node = addConditionWaiter();//将当前线程封装成一个节点并将该节点添加到条件变量的等待队列中
    int savedState = fullyRelease(node);//完全释放锁资源
    int interruptMode = 0;
    //这里使用while是为了防止过早唤醒
    while (!isOnSyncQueue(node)) {//判断当前线程的节点是否在AQS的同步队列中
        LockSupport.park(this);//阻塞当前线程，当前线程被其他线程唤醒后，就从这里开始继续向下执行
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```





### addConditionWaiter()

将节点添加ConditionObject的等待队列的过程是不需要使用cas操作完成的，因为当前线程是持有锁状态。

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

因为ReentrantLock是可重入锁，所以必须要完全释放锁资源，即将AQS的状态变量减为0

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

```java
//判断该节点是否在同步队列
final boolean isOnSyncQueue(Node node) {
    //如果节点状态是Node.CONDITION表示该节点不在同步队列中
    //如果该节点的prev属性为空表示该节点不在同步队列中，因为同步队列的头结点是new Node(),
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
//从等待队列的尾节点开始向上查找
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
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);//唤醒等待队列中的第一个节点所封装的线程
}
```





### doSignal(Node first)

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
final boolean transferForSignal(Node node) {
   
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))//从cas更新节点的状态
        return false;

    Node p = enq(node);//将当前节点添加到AQS的同步队列中,返回值p为当前节点在阻塞队列的前驱节点
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```







测试代码如下：

```java
public class ConditionTest {
    
    static ReentrantLock lock = new ReentrantLock();
    static Condition condition = lock.newCondition();
    
    public static void main(String[] args) throws InterruptedException {
        
            Thread thread = new Thread(new SignalThread());
            thread.setName("子线程");
            thread.start();
        
            lock.lock();
            try {
                System.out.println("主线程等待通知");
                condition.await();
            } finally {
                lock.unlock();
            }
        
            System.out.println("主线程恢复运行");
    }
    
    static class SignalThread implements Runnable {
        @Override
        public void run() {
            lock.lock();
            try {
                condition.signal();
                System.out.println("子线程通知");
            } finally {
                lock.unlock();
            }
        }
    }
}

```



运行结果如下：





```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();//将当前线程封装成一个Node节点，添加到等待队列中
    int savedState = fullyRelease(node);//释放锁资源
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {//判断当前线程是否在AQS的同步队列中
        LockSupport.park(this);//阻塞当前线程
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```





```java
//将当前线程封装为一个Node节点，并将其添加到ConditionObject对象维护的等待队列中
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();//从头开始遍历队列，将状态不为Node.CONDITION的节点从队列中剔除
        t = lastWaiter;
    }
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```



```java
final boolean isOnSyncQueue(Node node) {
    //如果节点的状态是Node.Condition 则表示当前节点在ConditionObject维护的等待队列中
    //如果当前节点在AQS的等待队列中，那么node.prev不可能为空，因为队列的头结点是new Node()
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    //如果当前节点有后继节点，那么该节点一定在AQS的同步队列中
    if (node.next != null) 
        return true;
 	//如果上述条件都不满足，则遍历整个队列进行查找
    return findNodeFromTail(node);
}
```





```java
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



唤醒线程的方法

```java
public final void signal() {
    if (!isHeldExclusively()) //判断当前线程是否是持有独占锁的线程
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;//等待队列中的头节点
    if (first != null)
        doSignal(first);
}
```



```java
private void doSignal(Node first) {
    do {
        if ((firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;//将节点first出队
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```





```java
final boolean transferForSignal(Node node) {
   	//使用cas修改当前节点的状态，如果修改失败则表示该节点所封装的线程已经被取消(Cancelled)
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))//如果cas替换失败，则该节点一定是取消状态
        return false;

    Node p = enq(node);//调用AQS的方法将等待队列的头节点添加到AQS的阻塞队列的队尾
    int ws = p.waitStatus;
    //ws >0 表示前驱节点的状态为取消
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL)) 
        LockSupport.unpark(node.thread);
    return true;
}
```







```java
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}
```



