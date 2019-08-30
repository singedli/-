本文的源码来自于jdk1.8版本，然而并不会涉及jdk8新特性。

ArrayList简介
===========

ArrayList是基于数组实现的，是一个动态数组，其容量能自动增长，类似于C语言中的动态申请内存，动态增长内存。

ArrayList不是线程安全的，只能用在单线程环境下，多线程环境下可以考虑用Collections.synchronizedList(List l)函数返回一个线程安全的ArrayList类，也可以使用concurrent并发包下的CopyOnWriteArrayList类。

ArrayList实现了Serializable接口，因此它支持序列化，能够通过序列化传输，实现了RandomAccess接口，支持快速随机访问，实际上就是通过下标序号进行快速访问，实现了Cloneable接口，能被克隆。

ArrayList源码解析
=============

在进行源码解析之前需要有以下的准备
-----------------

1、transient关键字的作用
答：我们都知道一个对象只要实现了Serilizable接口，这个对象就可以被序列化，java的这种序列化模式为开发者提供了很多便利，我们可以不必关系具体序列化的过程，只要这个类实现了Serilizable接口，这个类的所有属性和方法都会自动序列化。
然而在实际开发过程中，我们常常会遇到这样的问题，这个类的有些属性需要序列化，而其他属性不需要被序列化，打个比方，如果一个用户有一些敏感信息（如密码，银行卡号等），为了安全起见，不希望在网络操作（主要涉及到序列化操作，本地序列化缓存也适用）中被传输，这些信息对应的变量就可以加上transient关键字。换句话说，这个字段的生命周期仅存于调用者的内存中而不会写到磁盘里持久化。
总之，java 的transient关键字为我们提供了便利，你只需要实现Serilizable接口，将不需要序列化的属性前添加关键字transient，序列化对象的时候，这个属性就不会序列化到指定的目的地中。
2、源码中多次使用的Arrays类中的copyOf方法

```
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        @SuppressWarnings("unchecked")
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }
```
该方法的作用是创建一个数组，并将original数组中newLength个元素复制到新创建的数组中。

ArrayList的成员变量
--------------
ArrayList的成员变量如下：

```
	/*定义集合底层数组的默认容量为10*/
    private static final int DEFAULT_CAPACITY = 10;
    
    
    /*当用户指定该 ArrayList 容量为 0 时，返回该空数组 */
    private static final Object[] EMPTY_ELEMENTDATA = {};
    

    /*当用户没有指定 ArrayList 的容量时(即调用无参构造函数)，返回的是该数组*/
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};


    /*ArrayList基于数组实现，用该数组保存数据, ArrayList 的容量就是该数组的长度*/
    transient Object[] elementData;

    /*ArrayList中所包含元素的个数*/
    private int size;
```
DEFAULTCAPACITY_EMPTY_ELEMENTDATA这个属性貌似是jdk8才加入的，它和EMPTY_ELEMENTDATA的区别不大，只是两者的使用情况不同。


ArrayList的构造函数
--------------
看代码可以发现ArrayList共有三个构造函数，其代码很易懂。

1、空参构造函数

```
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```
2、创建集合时用户指定ArrayList的初始容量

```
public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
```

3、创建集合时用户传入一个指定的集合

```
public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            if (elementData.getClass() != Object[].class)
                //将传入的集合复制一份，并将其赋给elementData
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            //若传入的集合为空，则使用空数组替代
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

常用API源码
-------

1、add(E e)方法
ArrayList的add方法几乎包含了ArrayList最主要的流程，其中包含了是否需要对数组进行动态扩容的判断，也包含了数组动态扩容的具体方式。

```
public boolean add(E e) {
	   ensureCapacityInternal(size + 1); 
	   elementData[size++] = e;
	   return true;
}
```
ensureCapacityInternal(size + 1)方法的作用是在添加元素之前确定ArrayList的容量大小，其中size变量的值为当前ArrayList中的数组的长度。这样做的目的是用于判定本次添加元素的操作是否需要对底层的数组进行扩容。

```
	private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }

    //如果当前是首次向数组中添加元素，则返回当前所需要的最小容量和默认数组容量的最大值
    //该值最终为本次添加元素操作所需要的最小数组长度
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }

    //判定是否需要对数组elementData进行扩容
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;
        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```

数组的动态扩容

```
	//动态扩容的方法
    private void grow(int minCapacity) {
        //旧的数组的长度
        int oldCapacity = elementData.length;
        //新的数组的长度为旧数组长度的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        //如果旧数组的长度依旧无法满足要求，则使用minCapacity为新数组的长度
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        //如果新数组的长度超过了规定的数组的最大长度（MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8）
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            //判定是否抛出异常
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        //minCapacity < 0，即minCapacity的值已经超出int类型的最大值
        if (minCapacity < 0) 
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
                Integer.MAX_VALUE :
                MAX_ARRAY_SIZE;
    }
```
根据以上的学习可以总结出
1、ArrayList底层数组的最大长度为int类型的长度。
2、ArrayList底层数组在进行动态扩容时，数组的长度每次增加1.5倍。
3、数组添加单个元素的主要流程为，先计算出本次添加元素所需要的最小数组长度x，再比较x与elementData.length的大小，当x的大小大于elementData.length的长度时，再对ArrayList底层数组elementData进行扩容。


2、将元素插入到指定的所引处，**public void add(int index,E element);**

```
public void add(int index, E element) {
        //对指定的索引进行校验
        rangeCheckForAdd(index);
        //确定数组的长度
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //将原数组index索引之前的元素保留，index所以之后的元素全体像后移一位
        System.arraycopy(elementData, index, elementData, index + 1,
                size - index);
        //将element元素插入到数组的index索引处
        elementData[index] = element;
        size++;
    }
```

```
	private void rangeCheckForAdd(int index) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
```

3、一次向ArrayList中添加一个集合**public boolean addAll(Collection<? extends E> c)**

```
public boolean addAll(Collection<? extends E> c) {
        //将集合转成对象数组
        Object[] a = c.toArray();
        int numNew = a.length;
        //确定数组是否需要扩容
        ensureCapacityInternal(size + numNew);
        //数组复制
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }
```

4、在指定索引处添加集合**public boolean addAll(int index, Collection<? extends E> c)**

```
public boolean addAll(int index, Collection<? extends E> c) {
        //检查索引index是否合法
        rangeCheckForAdd(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        //判断是否需要进行数组扩容
        ensureCapacityInternal(size + numNew);  // Increments modCount
        //需要被移动的元素的数量
        int numMoved = size - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData, index + numNew,
                    numMoved);

        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }
```
5、删除指定索引处的元素 public E remove(int index) 

```
public E remove(int index) {
        //检查索引是否合法
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);
        //需要移动的元素的个数
        int numMoved = size - index - 1;
        if (numMoved > 0)
            //将index+1索引之后的所有元素向前移动一位
            System.arraycopy(elementData, index+1, elementData, index,
                    numMoved);
        //将最后一位的引用置空，等待GC回收，并将数组的长度-1
        elementData[--size] = null; // clear to let GC do its work
        //将被删除的值返回
        return oldValue;
    }

    private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
```
6、删除指定的元素 public boolean remove(Object o)，与上面的逻辑相同。

```
public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }


    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            //将index+1索引之后的所有元素向前移动一位
            System.arraycopy(elementData, index+1, elementData, index,
                    numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```
7、删除某个集合removeAll和retainAll两个方法是相通的，区别在于removeAll(Collection<?> c)是删除ArrayList中c集合的元素，而retainAll(Collection<?> c)是保留c集合中的元素，将其他的元素删除。

```
public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, false);
    }

    public boolean retainAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, true);
    }
```
观察removeAll和retainAll的代码可发现，两个方法的区别只是在调用删除方法batchRemove时所传的标志位不同。batchRemove方法是根据方法第二个参数的值来判断当前操作是进行删除还是保留。

```
	/**
     * 该方法的主要逻辑为：
     * 遍历数组elementData，根据用户传入的判定条件，将符合条件的元素插入到elementData数组的前面,
     * 最后再将其余不符合规则的元素删除。
     * @param c
     * @param complement
     * @return
     */
    private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        //r变量用于记录遍历elementData数组的长度
        //w变量用于记录符合条件(complement)的元素的索引
        int r = 0, w = 0;
        boolean modified = false;
        try {
            for (; r < size; r++)
                //将符合条件的元素插入到数组的前面
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            //当c.contains(elementData[r])抛出异常时执行
            if (r != size) {
                System.arraycopy(elementData, r,
                        elementData, w,
                        size - r);
                w += size - r;
            }
            //将符合条件的元素留下，将不符合条件的元素删除
            if (w != size) {
                // clear to let GC do its work
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }
```

8、根据索引查询 get(int index)

```
public E get(int index) {
       rangeCheck(index);

       return elementData(index);
}
```
9、据元素查询其在集合中的索引  indexOf(Object o)和lastIndexOf(Object o)

```
//根据元素查询其在集合中的索引（正向查找），返回的是符合条件的第一个值的索引
   public int indexOf(Object o) {
       if (o == null) {
           for (int i = 0; i < size; i++)
               if (elementData[i]==null)
                   return i;
       } else {
           for (int i = 0; i < size; i++)
               if (o.equals(elementData[i]))
                   return i;
       }
       //没找到就返回-1
       return -1;
   }
```

```
//据元素查询其在集合中的索引（反向查找）
   public int lastIndexOf(Object o) {
       if (o == null) {
           for (int i = size-1; i >= 0; i--)
               if (elementData[i]==null)
                   return i;
       } else {
           for (int i = size-1; i >= 0; i--)
               if (o.equals(elementData[i]))
                   return i;
       }
       return -1;
   }
```

10、修改指定索引处的元素 set(int index, E element)

```
public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        //返回的是被替换的元素的值
        return oldValue;
}
```
11、其他的常用方法

```
public void clear() {
        modCount++;

        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            //全部置空
            elementData[i] = null;

        size = 0;
}
```

```
public boolean contains(Object o) {
        return indexOf(o) >= 0;
}
```

```
public boolean isEmpty() {
        return size == 0;
}
```

```
 public int size() {
        return size;
 }
```