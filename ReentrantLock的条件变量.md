



```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
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
//将当前线程封装成一个Node节点，并将其添加到等待队列的末尾
private Node addConditionWaiter() {
    Node t = lastWaiter;
    //链表不为空且尾节点的状态不等于Node.CONDITION
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
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
//释放锁资源，将当前线程对象的节点从同步队列中移除
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        if (release(savedState)) {
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







```java
//判断该节点是否在同步队列
final boolean isOnSyncQueue(Node node) {
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    if (node.next != null) // If has successor, it must be on queue
        return true;
   
    return findNodeFromTail(node);
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

![image-20190820144551823](/Users/lijiaxing/Library/Application Support/typora-user-images/image-20190820144551823.png)





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



