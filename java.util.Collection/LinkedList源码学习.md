LinkedList简介
============

LinkedList是一个实现了List接口和Deque接口的双端链表。 
LinkedList除了可以当做链表来操作外，它还可以当做栈、队列和双端队列来使用。
LinkedList同样是非线程安全的，只在单线程下适合使用。
LinkedList实现了Serializable接口，因此它支持序列化，能够通过序列化传输，实现了Cloneable接口，能被克隆。


LinkedList的内部结构
===============
linkedList是由一个个的节点Node组成，每个节点都包含上一个节点的引用、下一个节点的引用以及该节点引用的具体对象。节点对象Node的代码如下：

```
private static class Node<E> {
        //该节点引用的具体对象
        E item;
        //上一个节点的引用
        LinkedList.Node<E> next;
        //下一个节点的引用
        LinkedList.Node<E> prev;

        Node(LinkedList.Node<E> prev, E element, LinkedList.Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```
节点的示意图如下：
![node示意图](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWctYmxvZy5jc2RuLm5ldC8yMDE4MDUwNTEwMzUxMjMyOD93YXRlcm1hcmsvMi90ZXh0L0x5OWliRzluTG1OelpHNHVibVYwTDNGeFh6TTFPRE0xTmpJMC9mb250LzVhNkw1TDJUL2ZvbnRzaXplLzQwMC9maWxsL0kwSkJRa0ZDTUE9PS9kaXNzb2x2ZS83MC9ncmF2aXR5L1NvdXRoRWFzdA?x-oss-process=image/format,png)

LinkedList的继承和实现结构
================
LinkedList的类图如下：
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWctYmxvZy5jc2RuLm5ldC8yMDE4MDUwNTExMDY0Njk3OT93YXRlcm1hcmsvMi90ZXh0L0x5OWliRzluTG1OelpHNHVibVYwTDNGeFh6TTFPRE0xTmpJMC9mb250LzVhNkw1TDJUL2ZvbnRzaXplLzQwMC9maWxsL0kwSkJRa0ZDTUE9PS9kaXNzb2x2ZS83MC9ncmF2aXR5L1NvdXRoRWFzdA?x-oss-process=image/format,png)
由上图可知，LinkedList继承了抽象类AbstractSequentialList并实现了List，Deque，Cloneable，Serializable接口。

LinkedList的成员变量
===============

```
	//链表的长度
    transient int size = 0;
    //链表的第一个节点
    transient LinkedList.Node<E> first;
    //链表的最后一个节点
    transient LinkedList.Node<E> last;
```

LinkedList常见API的源码学习
====================

1、boolean add(E e)
------------------

大致逻辑：添加新节点newNode到链表的末尾，将原先链表的最后一个节点last的next指向newNode，并将newNode的prev指向last。
```
public boolean add(E e) {
    linkLast(e);
    //该方法永远都返回True
    return true;
}
//向链表的最后一位添加节点
void linkLast(E e) {
    final LinkedList.Node<E> l = last;
    //调用Node的构造器来创建一个新的节点
    // 第一个参数为该新节点的上一个节点，第二个参数为当前节点引用的对象
    //第三个参数为该新节点的下一个节点
    final LinkedList.Node<E> newNode = new LinkedList.Node<>(l, e, null);
    last = newNode;
    if (l == null)
        //l == null即链表的长度为0，该新增的节点是第一个节点
        first = newNode;
    else
        l.next = newNode;
    //链表长度+1
    size++;
    modCount++;
}
```

2、void add(int index, E element)
--------------------------------

```
public void add(int index, E element) {
        //检查索引index的大小是否超出LinkedList的size。
        checkPositionIndex(index);
        //判断是否是临界值
        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }
    //先将查找的范围缩小一半，然后再通过遍历去获取指定索引处的节点
    //当指定的索引小于size/2时，则进行正向的遍历查找
    //当指定的索引大于size/2时，则进行逆向的遍历查找
    LinkedList.Node<E> node(int index) {
        if (index < (size >> 1)) {
            LinkedList.Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            LinkedList.Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }

    // e为要插入的节点的对象引用，succ为指定索引处的节点
    void linkBefore(E e, LinkedList.Node<E> succ) {
        //获取index处的节点succ的上一个节点pred
        final LinkedList.Node<E> pred = succ.prev;
        //创建一个新的节点newNode，newNode的上一个节点为pred，下一个节点为succ
        final LinkedList.Node<E> newNode = new LinkedList.Node<>(pred, e, succ);
        //将succ的上一个节点指向newNode
        succ.prev = newNode;
        //pred的下一个节点指向newNode
        if (pred == null)
            //处理临界情况
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }
```
大致过程如下图所示：
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWctYmxvZy5jc2RuLm5ldC8yMDE4MDUwNTE2MTQyMTI1MD93YXRlcm1hcmsvMi90ZXh0L0x5OWliRzluTG1OelpHNHVibVYwTDNGeFh6TTFPRE0xTmpJMC9mb250LzVhNkw1TDJUL2ZvbnRzaXplLzQwMC9maWxsL0kwSkJRa0ZDTUE9PS9kaXNzb2x2ZS83MC9ncmF2aXR5L1NvdXRoRWFzdA?x-oss-process=image/format,png)

3、void addLast(E e）和 void addFirst(E e)
---------------------------------------
见名知意，addLast(E e)调用的是linkedLast(E e)方法,同boolean add(E e)方法调用的同一个方法。

两者的区别:
1、两个方法的返回值不同，一个是无返回值，另一个永远都返回true。
2、无返回值的是重写抽象类AbstractSequentialList中的add方法，而有返回值的是实现的Deque接口的方法。

void addFirst(E e)方法方用的是void linkFirst(E e)方法，代码如下：

```
private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;
        if (f == null)
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
    }
```
可以看到该方法和void linkLast(E e)方法的实现思路相同，只是一个是向链表的头处添加节点，一个是向链表的末尾添加节点。

4、boolean addAll(Collection c)和boolean addAll(int index, Collection c)
-------------------------------------------
boolean addAll(int index,Collection c)方法的实现思路同void add(int index, E element)方法相似，只是要添加的是一个集合，所以多了一步遍历集合插入的步骤。
```
//向链表的末尾添加一个集合
    public boolean addAll(Collection<? extends E> c) {
        return addAll(size, c);
    }

    //参数：index为要向链表插入的索引；c：为将要添加的集合
    public boolean addAll(int index, Collection<? extends E> c) {
        //检查索引是否超出链表的长度
        checkPositionIndex(index);

        Object[] a = c.toArray();
        //numNew为将要添加的节点的个数
        int numNew = a.length;
        if (numNew == 0)
            return false;

        //pred为上一个节点，succ为当前节点
        LinkedList.Node<E> pred, succ;

        if (index == size) {
            succ = null;
            pred = last;
        } else {
            succ = node(index);
            pred = succ.prev;
        }

        //遍历集合c，向链表插入新的节点
        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            LinkedList.Node<E> newNode = new LinkedList.Node<>(pred, e, null);
            if (pred == null)
                first = newNode;
            else
                pred.next = newNode;
            pred = newNode;
        }

        if (succ == null) {
            last = pred;
        } else {
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }
```

5、E remove() && E removeFirst()
------------

E remove()方法和E removeFirst()方法的作用都是删除链表的第一个节点，返回被删除的节点。代码如下：

```
public E remove() {
        return removeFirst();
}

public E removeFirst() {
    final LinkedList.Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}

//参数f为当前链表的第一个节点
//大致逻辑：获取第一个节点，并将第一个节点的item和next置为null。再将第二个节点的prev置为null。
private E unlinkFirst(LinkedList.Node<E> f) {
    //获取第一个节点f中的对象引用
    final E element = f.item;
    final LinkedList.Node<E> next = f.next;
    f.item = null;
    f.next = null;
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
```

6、boolean remove(Object o) && boolean removeFirstOccurrence(Object o)
--------------------------
该方法为删除链表中指定的对象，返回值为boolean类型。
删除指定对象的大致过程如下：
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWctYmxvZy5jc2RuLm5ldC8yMDE4MDUwNTIxMDU0NTc1Mz93YXRlcm1hcmsvMi90ZXh0L0x5OWliRzluTG1OelpHNHVibVYwTDNGeFh6TTFPRE0xTmpJMC9mb250LzVhNkw1TDJUL2ZvbnRzaXplLzQwMC9maWxsL0kwSkJRa0ZDTUE9PS9kaXNzb2x2ZS83MC9ncmF2aXR5L1NvdXRoRWFzdA?x-oss-process=image/format,png)

相关代码如下：

```
public boolean removeFirstOccurrence(Object o) {
	     return remove(o);
}
```

```
public boolean remove(Object o) {
        //先在链表中找到需要删除的元素，然后再调用unlink方法进行删除
        if (o == null) {
            for (LinkedList.Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                 }
            }
        } else {
            for (LinkedList.Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
}
```

```
E unlink(LinkedList.Node<E> x) {

     final E element = x.item;
     final LinkedList.Node<E> next = x.next;
     final LinkedList.Node<E> prev = x.prev;
     
     //临界值，该节点为链表的第一个节点
     if (prev == null) {
         first = next;
     } else {
         prev.next = next;
         x.prev = null;
     }
     
     //临界值，该节点为链表的最后一个节点
     if (next == null) {
         last = prev;
     } else {
         next.prev = prev;
         x.next = null;
     }

     x.item = null;
     size--;
     modCount++;
     return element;
}
```

7、E remove(int index)
---------------------
删除指定索引处的节点

```
 public E remove(int index) {
        checkElementIndex(index);
        //先获取到执行索引处的节点元素，然后再调用unlink(E e)方法进行删除
        return unlink(node(index));
 }
```

8、E removeLast()
--------------

移除链表的最后一个节点，逻辑同上。
```
public E removeLast() {
        final LinkedList.Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return unlinkLast(l);
    }
        
    private E unlinkLast(LinkedList.Node<E> l) {
        final E element = l.item;
        final LinkedList.Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; 
        last = prev;
        if (prev == null)
            first = null;
        else
            prev.next = null;
        size--;
        modCount++;
        return element;
    }
```

9、boolean contains(Object o)
----------------------------
判断链表中是否包含某个对象

```
public boolean contains(Object o) {
        return indexOf(o) != -1;
}
```

10、E element() && E getFirst()
------------------------------

```
public E element() {
	//获取第一个节点中的对象
    return getFirst();
}
```

```
public E getFirst() {
   final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return f.item;
}
```

11、E get(int index)
-------------------
获取指定索引处的节点的对象
```
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
```

12、E getLast()
--------------

逻辑与E element() && E getFirst()相似
```
public E getLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return l.item;
}
```

13、int indexOf(Object o) && lastIndexOf(Object o）
------------------------
int indexOf(Object o)：正向查找，获取链表中第一个指定对象的索引，如果没有则返回-1。
lastIndexOf(Object o):反向查找，其余逻辑同int indexOf(Object o)相同。
```
//从first节点开始遍历，直到找到指定对象
    public int indexOf(Object o) {
        int index = 0;
        if (o == null) {
            for (LinkedList.Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            for (LinkedList.Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }
```

14、boolean offer(E e) && boolean offerFirst(E e) && boolean offerLast(E e)
------------------------------------------------------------------------

boolean offer(E e)：向链表的末尾添加元素，调用add(E e)方法。
boolean offerFirst(E e)：向链表的首部添加元素，调用addFitst(E e)方法。
boolean offerLast(E e)：向链表的末尾添加元素，调用addLast(E e)方法。

```
public boolean offer(E e) {
	return add(e);
}
```

```
public boolean offerFirst(E e) {
	addFirst(e);
	return true;
}
```

```
public boolean offerLast(E e) {
    addLast(e);
    return true;
}
```

15、E peek() && E peekFirst() && E peekLast()
--------------------------------------------

```
//当链表不为空时，返回链表的第一个元素
public E peek() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
}
```

```
//当链表不为空时，返回链表的第一个元素
public E peekFirst() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
}
```

```
//当链表不为空时，返回链表的最后一个元素
public E peekLast() {
        final Node<E> l = last;
        return (l == null) ? null : l.item;
}
```

16、E poll() && E pollFirst() && E pollLast()
--------------------------------------------

```
//当链表不为空时，返回链表的第一个元素,并删除第一个元素
public E poll() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
}
```

```
//当链表不为空时，返回链表的第一个元素,并删除第一个元素
public E pollFirst() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
}
```

```
//当链表不为空时，返回链表的第一个元素,并删除最后一个元素
public E pollLast() {
        final Node<E> l = last;
        return (l == null) ? null : unlinkLast(l);
}
```

17、E pop()

```
//删除链表的第一个元素，并返回被删除的元素
public E pop() {
        return removeFirst();
}
```