ReentrantLock公平锁的测试代码如下：

```java
    public static void main(String[] args) {
        ReentrantLock lock = new ReentrantLock(true);//创建一个公平锁
        lock.lock();
        try{
            TimeUnit.SECONDS.sleep(1);
        }catch (Exception e){

        }finally{
            lock.unlock();
        }
    }

```



ReentrantLock的构造函数：

```java
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

当参数fair为true时则创建FairSync类，当fair为false时创建NonfairSync类



接下来进入lock.lock()方法查看具体获取锁的逻辑：

```java
    public void lock() {
        sync.lock();
    }
```



查看sync的lock()方法可以发现，sync类中的lock()方法为一个钩子方法，因为上面创建的是公平锁，所以此时具体的获取锁资源的实现是在FairSync类中。

![]([https://github.com/singedli/juc/blob/master/pic/sycn%E7%9A%84lock()%E6%96%B9%E6%B3%95.png](https://github.com/singedli/juc/blob/master/pic/sycn的lock()方法.png))



FairSync类的lock()方法

```java
        final void lock() {
            acquire(1);
        }
```



AQS的acquire(int arg)方法是模板方法，公平模式和非公平模式都是用同样的流程

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

尝试获取锁资源该方法的逻辑大致如下：

1. 判断当前锁资源是否被某个线程占用
2.  如果锁资源已经被占用，则判断占用锁资源的线程是否是当前线程，如果是当前线程，则直接对AQS的状态变量加acquires，表示重入
3. 如果当前锁资源没有被占用，判断AQS队列中优先级最高的节点节点所对应的线程是否是当前线程
4. 如果是当前线程，则获取锁成功
5. 如果不是当前线程，意味着AQS队列中有更高优先级的线程，于是将当前线程封装成一个Node节点，并将这个节点添加到AQS同步队列的队尾
6. 阻塞当前线程，等待后续被唤醒

```java
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();//获取当前线程
            int c = getState();//获取AQS中状态变量的值
            if (c == 0) {//c == 0表示当前没有以其他线程占用锁资源
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {//如果占用锁资源的线程是当前线程
                int nextc = c + acquires;//AQS状态变量值+1
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```

​	

```java
	//判断当前线程节点在AQS同步队列中是否还有前驱节点    
	public final boolean hasQueuedPredecessors() {
        Node t = tail;
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```

1.  h != t  表示AQS的同步队列中至少有两个节点

2. (s = h.next) == null  处理的是一种中间状态

   ![[https://github.com/singedli/juc/blob/master/pic/%E7%BA%BF%E7%A8%8B%E8%8A%82%E7%82%B9%E5%85%A5%E9%98%9F%E7%9A%84%E6%96%B9%E6%B3%95.png](https://github.com/singedli/juc/blob/master/pic/线程节点入队的方法.png)]()

   线程节点入队的方法如上图所示：

   ​	1、先将尾节点设置为当前节点的prev

   ​	2、cas设置队列的尾节点

   ​	3、将旧为节点的next指向当前节点

   (s = h.next) == null ，就是用来处理当前有一个其他线程刚好执行到上面的步骤2，但还没来得及执行步骤3的情况，此时将会返回true，因为当前线程不是队列中优先级最高的线程。

3. s.thread != Thread.currentThread()  判断队列中第二个节点所维护的线程是否是当前线程



其余封装线程为一个节点并添加到AQS同步队列的逻辑通非公平模式相同

```java
//将当前线程封装成一个Node节点，并添加到AQS的同步队列中    
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
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



```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();//获取当前节点的前驱节点
                if (p == head && tryAcquire(arg)) {//如果前驱节点是head节点，则尝试获取锁资源
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

