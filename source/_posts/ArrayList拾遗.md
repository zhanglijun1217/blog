---
title: ArrayList拾遗
copyright: true
date: 2020-03-29 01:00:44
tags:
	- ArrayList
categories:
	- Java基础
---

# 1.ArrayList简介
- ArrayList底层是object数组，容量能在添加元素的过程中动态扩容。并且在可预知添加大量元素时，调用ensureCapactiy方法提前扩容，减少递增式的扩容次数。

- 实现了RandomAccess接口，表示可以快速随机访问。根据下标访问。

- 实现了Cloneable接口，覆盖了函数克隆，不过也是潜拷贝

- 实现了Serializable接口，支持序列化进行传输

  <!-- more -->

## 和Vector容器的区别
- 两者都是List接口的实现类，但是ArrayList线程不安全，Vector线程安全，方法都加了synchronize锁。

## 和LinkedList区别
- 两者都不保证线程安全
- 底层数据结构：ArrayList底层是object数组，而LinledList底层是双向链表。（jdk1.6是双向循环链表，而1.7之后取消了循环。）
- LinkedList不支持高效的快速随机访问，而ArrayList支持快速随机根据下标访问。
- 插入和删除受元素位置的影响
    - ArrayList在末尾add方法追加元素时，时间复杂度是O(1)，而如果是在指定位置插入和删除元素时，时间复杂度就是O(N-i)，因为这时需要把第i个和后面的n-i个元素向后/前移动一位。
    - LinkedList采用链表存储，对于add(E e)方法的插入、删除时间复杂度不受元素位置影响。对于指定位置i的插入和删除元素，因为也需要遍历到要插入位置，时间复杂度近似为O(N)
- 内存占用：ArrayList的内存浪费主要体现在元素个数比数组长度小时，会预留长度的空间浪费。而LinkedList没有扩容问题，但其每个节点除了存储元素之外，还要维护前驱结点和后继节点，所以单节点空间消耗大。


# ArrayList关键源码走读
## 静态和实例字段

```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    private static final long serialVersionUID = 8683452581122892189L;

    /**
     * 默认初始容量大小
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * 空数组（用于空实例）。
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

     //用于默认大小空实例的共享空数组实例。
      //我们把它从EMPTY_ELEMENTDATA数组中区分出来，以知道在添加第一个元素时容量需要增加多少。
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * 保存ArrayList数据的数组
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * ArrayList 所包含的元素个数
     */
    private int size;
```

## 构造函数
### 无参数
```
  /**
     *默认无参构造函数
     *DEFAULTCAPACITY_EMPTY_ELEMENTDATA 为0.初始化为10，也就是说初始其实是空数组 当添加第一个元素的时候数组容量才变成10
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```

### 带有初始容量参数的构造函数
```
 public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            //如果传入的参数大于0，创建initialCapacity大小的数组
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            //如果传入的参数等于0，创建空数组
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            //其他情况，抛出异常
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
```

### 包含一个指定集合，按照集合的迭代器返回的顺序
```
public ArrayList(Collection<? extends E> c) {
        //将指定集合转换为数组
        elementData = c.toArray();
        //如果elementData数组的长度不为0
        if ((size = elementData.length) != 0) {
            // 如果elementData不是Object类型数据（c.toArray可能返回的不是Object类型的数组所以加上下面的语句用于判断）
            if (elementData.getClass() != Object[].class)
                //将原来不是Object类型的elementData数组的内容，赋值给新的Object类型的elementData数组
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // 长度为0，用空数组代替
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

### contains和containsAll方法
```
    /**
     * 如果此列表包含指定的元素，则返回true 。
     */
    public boolean contains(Object o) {
        //indexOf()方法：返回此列表中指定元素的首次出现的索引，如果此列表不包含此元素，则为-1
        return indexOf(o) >= 0;
    }
    
     /**
     *返回此列表中指定元素的首次出现的索引，如果此列表不包含此元素，则为-1
     */
    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                //equals()方法比较
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
    
    // lastIndexOf方法
     /**
     * 返回此列表中指定元素的最后一次出现的索引，如果此列表不包含元素，则返回-1。.
     */
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

### get方法
```
  public E get(int index) {
        // 检查在size内
        rangeCheck(index);
        // 直接从数组中根据下标读
        return elementData(index);
    }
    
      private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
    
    @SuppressWarnings("unchecked")
    E elementData(int index) {
        return (E) elementData[index];
    }

```

### add 和 set方法
add方法会在ensureCapacityInternal中对modCount+1。
```
/**
     * 用指定的元素替换此列表中指定位置的元素。
     */
    public E set(int index, E element) {
        //对index进行界限检查
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        //返回原来在这个位置的元素
        return oldValue;
    }

    /**
     * 将指定的元素追加到此列表的末尾。
     */
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //这里看到ArrayList添加元素的实质就相当于为数组赋值
        elementData[size++] = e;
        return true;
    }

    /**
     * 在此列表中的指定位置插入指定的元素。
     *先调用 rangeCheckForAdd 对index进行界限检查；然后调用 ensureCapacityInternal 方法保证capacity足够大；
     *再将从index开始之后的所有成员后移一个位置；将element插入index位置；最后size加1。
     */
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //arraycopy()这个实现数组之间复制的方法一定要看一下，下面就用到了arraycopy()方法实现数组自己复制自己
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
    
    /**
     * add和addAll使用的rangeCheck的一个版本
     */
    private void rangeCheckForAdd(int index) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }


```

### remove方法
remove也会对modCount+1
```
    /**
     * 删除该列表中指定位置的元素。 将任何后续元素移动到左侧（从其索引中减去一个元素）。
     */
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
      //从列表中删除的元素
        return oldValue;
    }

    /**
     * 从列表中删除指定元素的第一个出现（如果存在）。 如果列表不包含该元素，则它不会更改。
     *返回true，如果此列表包含指定的元素
     */
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

    /*
     * Private remove method that skips bounds checking and does not
     * return the value removed.
     */
     // 不校验index的快速删除方法
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```

### 扩容相关
从上边的构造函数中可以知道，调用无参数构造函数时，elementData数组初始化为DEFAULTCAPACITY_EMPTY_ELEMENTDATA空数组，只有当真正对数组添加元素操作时，才真正分配容量，向数据添加第一个元素时，数组容量扩容到默认的capacity=10.


在add方法中调用了ensureCapacityInternal方法来修改了modCount和进行扩容的。
```
   private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        // 如果现在object数组时初始化时的默认空数组 则重新计算minCapacity值
        // 计算规则 default_capacity和minCapacity中较大的值。
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        // 增加modCount
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
        // 计算出的minCapacity如果比当前数组长度大  则调用grow方法进行扩容。
            grow(minCapacity);
    }
```

- 当调用了无参数构造函数之后，add第一个元素时，minCapacity = size+1 =1，此时和默认容量相比比较大的是默认容量，此时minCapacity传入grow进行扩容
- 当add第2个元素时，minCapacity = size + 1 = 2，此时因为mincapacity - length = 2-10 <0 ,所以不会调用grow方法进行扩容，同理第3、4..10个元素被add时都不会扩容。
- 当第11个元素add进去时，会调用grow方法进行扩容

grow方法是真正对ArrayList中的数组进行扩容。
```
   /**
     * 要分配的最大数组大小
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * ArrayList扩容的核心方法。
     */
    private void grow(int minCapacity) {
        // oldCapacity为旧容量，newCapacity为新容量
        int oldCapacity = elementData.length;
        
        //将oldCapacity 右移一位，其效果相当于oldCapacity /2，
        //我们知道位运算的速度远远快于整除运算，整句运算式的结果就是将新容量更新为旧容量的1.5倍，
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        
        //然后检查新容量是否大于最小需要容量，若还是小于最小需要容量，那么就把最小需要容量当作数组的新容量，
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
            
       // 如果新容量大于 MAX_ARRAY_SIZE,进入(执行) `hugeCapacity()` 方法来比较 minCapacity 和 MAX_ARRAY_SIZE，
       //如果minCapacity大于最大容量，则新容量则为`Integer.MAX_VALUE`，否则，新容量大小则为 MAX_ARRAY_SIZE 即为 // `Integer.MAX_VALUE - 8`。
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
- 当add第一个元素时，会grow方法扩容到10。当add第二个元素时，此时不会调用grow方法，容量不变
- 当add第11个元素时，会调用grow方法，扩容容量为1.5倍数组长度，即为15