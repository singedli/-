

## ArrayBlockingQueue介绍

ArrayBlockingQueue是采用数组实现的有界阻塞线程安全队列。如果向已满的队列继续塞入元素，将导致当前的线程阻塞。如果向空队列获取元素，那么将导致当前线程阻塞。

## ArrayBlockingQueue类的几个主要成员属性：

```java
final Object[] items; //用于存放队列元素的数组
int takeIndex;	//消费者取的元素的数组下标
int putIndex; //生产者存元素的数组下标
int count; //阻塞队列的元素个数
final ReentrantLock lock; //数据访问的重入锁
private final Condition notEmpty;//消费者取元素的条件变量
private final Condition notFull; //生产者存元素的条件变量
```



## ArrayBlockingQueue的构造函数

```java
public ArrayBlockingQueue(int capacity) {
    this(capacity, false);
}

public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];//初始化数组
    lock = new ReentrantLock(fair);//创建可重入锁
    notEmpty = lock.newCondition();//创建消费者取元素的条件变量
    notFull =  lock.newCondition();//创建生产者存元素的条件变量
}
/**
*  获取锁资源，以同步的方式将集合c的元素批量放入阻塞队列中
*  
*  如果集合的元素个数大于初始化容量数，则会抛IllegalArgumentException异常
*/
public ArrayBlockingQueue(int capacity, boolean fair,
                          Collection<? extends E> c) {
    this(capacity, fair);//调用重载的构造方法

    final ReentrantLock lock = this.lock;
    lock.lock(); 
    try {
        int i = 0;
        try {
            for (E e : c) {
                checkNotNull(e);
                items[i++] = e;
            }
        } catch (ArrayIndexOutOfBoundsException ex) {
            throw new IllegalArgumentException();
        }
        count = i;
        //如果几个c的大小等于队列容量，则生产者存元素的数组下标设置为0，否则为i
        putIndex = (i == capacity) ? 0 : i;
    } finally {
        lock.unlock();
    }
}
```







## 生产者线程存放元素的方法

### 1、put(E e)

```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);//非空校验
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();//获取可以响应中断的锁
    try {
        while (count == items.length)//如果阻塞队列满，则将当前生产者线程阻塞在条件变量notFull上
            notFull.await();
        //如果当前阻塞队列不满，则将元素入队
        enqueue(e);
    } finally {
        lock.unlock();
    }
}
```



#### enqueue(E x )

```java
//将元素入队
private void enqueue(E x) {
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length)//如果putIndex指针已经指向了数组的最后一个元素，
        //则将putIndex指针设置为0
        putIndex = 0;
    count++;
    notEmpty.signal();//唤醒阻塞在notEmpty上的消费者线程(如果有的话)
}
```



### 2、add(E e)

```java
//模板模式，调用父类AbstractQueue的add(E e)方法
public boolean add(E e) {
    return super.add(e);
}
```

#### AbstractQueue#add(E e)

```java
//调用offer()添加元素，如果失败则直接抛出IllegalStateException异常
public boolean add(E e) {
    if (offer(e)) 
        return true;
    else
        throw new IllegalStateException("Queue full");
}
```



#### 3、offer(E e)

```java
public boolean offer(E e) {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        if (count == items.length)//队列元素已满
            return false;
        else {
            enqueue(e);//将元素入队
            return true;
        }
    } finally {
        lock.unlock();
    }
}
```



## 消费者线程从阻塞队列中获取元素的方法

### 1、take()

```java
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
```



#### dequeue()

```java
//元素出队的方法
private E dequeue() {
    final Object[] items = this.items;
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length) //逻辑和入队方法中putIndex指针的逻辑相同
        takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal();
    return x;
}
```



### 2、poll

```java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}
```



### 3、peak



```java
public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return itemAt(takeIndex); // 直接使用数组指针获取
    } finally {
        lock.unlock();
    }
}
```



```java
final E itemAt(int i) {
    return (E) items[i];
}
```





## ArrayBlockingQueue类常用方法对比

存元素的方法

| 方法  | 返回值  |       当队列已满时        |        方法来源         |
| :---: | :-----: | :-----------------------: | :---------------------: |
|  put  |  void   |       线程会被阻塞        | 实现自BlockingQueue接口 |
|  add  | boolean | 抛出IllegalStateException |     实现自Queue接口     |
| offer | boolean |     立即返回是否成功      |     实现自Queue接口     |



取元素的方法：

| 方法 | 返回值 |        当队列为空时        |        方法来源         |
| :--: | :----: | :------------------------: | :---------------------: |
| take |   E    |   线程会被阻塞，直到非空   | 实现自BlockingQueue接口 |
| poll |   E    |  直接返回(调用出队的方法)  |     实现自Queue接口     |
| peek |   E    | 直接返回(使用数组下标查找) |     实现自Queue接口     |



