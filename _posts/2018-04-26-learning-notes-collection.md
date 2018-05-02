---
layout: post
comments: true
title: "Java学习笔记——集合类总结篇"
description: "Java学习笔记——集合类总结篇"
category: Java
tags: [Java]
---


**1、集合类整体框架**    
**2、List系列**    
**（1）ArrayList**    
**（2）LinkedList**    
**（3）Vector、Stack**    
**3、Set系列**    
**（1）HashSet、LinkedHashSet**    
**（2）TreeSet**    
**4、Map系列**    
**（1）HashMap、LinkedHashMap**    
**（2）HashTable**    
**（3）TreeMap**  
**（4）WeakHashMap**  
**5、Queue系列**    
**（1）Deque、ArrayDeque**    
**（2）PriorityQueue**    
**（3）BlockingQueue、LinkedBlockingQueue、PriorityBlockingQueue**    
**6、Sparse系列**    
**（1）SparseArray/SparseBooleanArray/SparseIntArray/SparseLongArray**    
**7、集合类总结**    
**8、参考文档**    

<!--more-->

### 1、集合类整体框架    

![](/image/2018-04-26-learning-notes-collection/collection.png)

### 2、List系列    

#### （1）ArrayList    

底层结构：数组、非线程安全     
transient Object[] elementData；    
扩容机制    

protected transient int modCount = 0;//fail-fast机制
ListIterator遍历时，先检查modCount，防止创建ListIterator后，数组结构发生变化    

#### （2）LinkedList

implements List, Deque    
底层结构：双端链表、    
transient Node<E> first;    
transient Node<E> last;    
transient int size = 0;    

根据index查找规则：比较index和size>>1的大小决定是从first还是last开始。

#### （3）Vector、Stack

Vector：和ArrayList实现类似，数组所有操作都synchronized，`线程安全`    
Stack：继承Vector，栈结构，synchronized，`线程安全`    
    
### 3、Set系列    

#### （1）HashSet、LinkedHashSet
    
HashSet：    
底层结构：通过HashMap实现    
private transient HashMap<E,Object> map;    
private static final Object PRESENT = new Object();//map中的value占位对象    

    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }

由于是基于HashMap实现，其特性由HashMap特性决定：    
允许传空值null（因为HashMap中key可以为空，此时hash(key)=0）    
没有重复的对象，即不会同时出现2个null、2个equal相等的非空对象（因为HashMap中的key不会重复）        
    
    
LinkedHashSet：    
extends HashSet implements Set    
底层结构：通过LinkedHashMap实现，accessOrder = false，除此之外所有方法的实现和HashSet一样    
按插入顺序访问元素    
    
#### （2）TreeSet
   
基于TreeMap实现   

#### （3）CopyOnWriteArrayList
   
### 4、Map系列

解决冲突方式：拉链法（ThreadLocal用的线性探测法）    

#### （1）HashMap、LinkedHashMap    

HashMap：    
底层结构：元素类型的单链表的数组    
transient Node<K,V>[] table;

LinkedHashMap：    
extends HashMap
底层结构：元素类型的单链表的数组＋每个元素按照插入序或访问序构成双向链表    
transient Node<K,V>[] table;    
transient LinkedHashMap.Entry<K,V> head;
transient LinkedHashMap.Entry<K,V> tail;

#### （2）HashTable    

实现和HashMap差不多，差别是：    
synchronized，线程安全    
插入数据时，新结点是插入链表首部而不是尾部    

官方建议用ConcurrentHashMap取代Hashtable

#### （3）TreeMap  

底层结构：基于红黑树实现，并按结点的key进行排序，排序规则是Comparator或Comparable    
private final Comparator<? super K> comparator;
private transient Entry<K,V> root;
private transient int size = 0;

结点结构（key-value对）：    
static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left;
        Entry<K,V> right;
        Entry<K,V> parent;
        boolean color = BLACK;
｝

遍历下一个结点的过程就是找后继结点的过程。

    static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
        if (t == null)
            return null;
        else if (t.right != null) {
            Entry<K,V> p = t.right;
            while (p.left != null)
                p = p.left;
            return p;
        } else {
            Entry<K,V> p = t.parent;
            Entry<K,V> ch = t;
            while (p != null && ch == p.right) {
                ch = p;
                p = p.parent;
            }
            return p;
        }
    }

线程安全用法：    
SortedMap m = Collections.synchronizedSortedMap(new TreeMap(...));

#### （4）WeakHashMap  

和HashMap实现类似，差别在于，结点的结构中key是一个弱引用，期间会对key==null的结点进行清理    
ThreadLocalMap就是这种实现    

    private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
        V value;
        final int hash;
        Entry<K,V> next;

        /**
         * Creates new entry.
         */
        Entry(Object key, V value,
              ReferenceQueue<Object> queue,
              int hash, Entry<K,V> next) {
            super(key, queue);
            this.value = value;
            this.hash  = hash;
            this.next  = next;
        }
    ｝

### 5、Queue系列    

#### （1）Deque、ArrayDeque    

Deque：双端队列接口    
ArrayDeque：双端环形队列，内部通过一个可变数组实现    
底层结构：    
    
    transient Object[] elements;
    transient int head;//队列头部索引
    transient int tail;//队列尾部索引

可以操作队头和队尾，head和tail的值会跟着变化，且是环形变化，即index＝length-1的下一个索引是index=0    

![](/image/2018-04-26-learning-notes-collection/arraydeque.png)

扩容条件：    
在add操作后，若head == tail会执行doubleCapacity()双倍扩容

addFirst()：    
head = (head - 1) & (elements.length - 1)

removeFirst()     
head = (head + 1) & (elements.length - 1);    

addLast()    
tail = (tail + 1) & (elements.length - 1)

removeLast()    
tail = (tail - 1) & (elements.length - 1);

#### （2）PriorityQueue    

基于数组实现的完全二叉树，小顶堆
[深入理解Java PriorityQueue](http://www.cnblogs.com/CarpenterLee/p/5488070.html)

#### （3）BlockingQueue、PriorityBlockingQueue、LinkedBlockingQueue    

BlockingQueue：阻塞队列接口    
PriorityBlockingQueue：在PriorityQueue基础上，增加了线程安全（Lock）和队列阻塞（Condition、await/signal）    
LinkedBlockingQueue：基于链表实现的FIFO线程安全性阻塞队列（Lock，await/signal，head/tail）    

await/signal/Lock配合一起使用，wait/notify/synchronized配合一起使用，两者功能基本类似，但是前者功能更丰富，具体区别到时候再总结。    

基本概念参考 [1](https://zh.wikipedia.org/wiki/%E6%A0%91_(%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84\))：    
完全二叉树    
平衡二叉树    
满二叉树    
小顶堆／大顶堆    
层数、度    

### 6、Sparse系列    

#### （1）SparseArray/SparseBooleanArray/SparseIntArray/SparseLongArray    

Android平台提供的类，JDK没有。    

以SparseArray为例。

数据结构：key为int[]递增数组，value为Object[]数组    

    private int[] mKeys;
    private Object[] mValues;
    private int mSize;

put(int key, E value)：    
1、先二分法找到key所在的索引，如果找到则返回i，如果没找到则返回要插入位置的取非~i（取反操作，值为负）    
2、如果返回值大于等于0，说明已经找到key所在的索引，直接更新mKeys[]，mValues[]数组对应的值    
3、如果返回值小于0，说明没找到key的索引，然后将~i取反得到i，得到将插入的位置。
如果mValues[i]=DELETED，说明原来的值是要删除的，直接更新mValues[i] = value即可。
如果垃圾回收标志位mGarbage为true且mSize达到数组容量，则需要进行gc()操作（将所有非DELETED数据往前移位），然后再二分法重新找要插入的位置i。如果mSize达到数组容量，需要进行2倍扩容操作并将value插入合适位置。
mSize++    

get(int key)：    
先二分查找找到索引i，然后返回mValues[i]

SparseArray和Java8 HashMap比较：    
（1）数据结构和内存占用：SparseArray是int[]数组＋Object[]数组，HashMap是Node<Integer,V>[]链表数组（链表每个结点结构包含K、V对象和hash值），HashMap处理基本数据类型需要进行装箱操作（int -> Integer对象），所以整体来说SparseArray更省内存。    
（2）时间复杂度：SparseArray的索引是二分查找，时间复杂度O(lgn)，但是可能需要进行数组插入操作，所以会有元素的移位操作；HashMap通过hash求索引，且对于每个桶中长度>=7的单链表会转化为红黑树处理，所以时间复杂度O(lgn)。    

所以整体来说，SparseArray相比于HashMap，除了内存占用要少一点，在时间复杂度上并不占优势，相反元素比较多时还需要大量的移位操作。    

其他几个除了value类型不同且没有DELETED标志位外，其他逻辑一样：    
SparseBooleanArray：int[]+boolean[]    
SparseIntArray：int[]+int[]    
SparseLongArray：int[]+long[]    


TODO:    
transient    
Map的initialCapacity,loadFactor        
writeObject/readObject    
hashCode()/equal()    

### 7、集合类总结    

![](/image/2018-04-26-learning-notes-collection/collection-compare.png)

### 8、参考文档    

(1) [关于Java集合的小抄](http://calvin1978.blogcn.com/articles/collection.html)        
(2) [Java集合框架总结](https://blog.csdn.net/justdb/article/details/7481493)    
(3) [Java 容器源码分析](http://blog.jrwang.me/archives/)    

