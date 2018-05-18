---
layout: post
comments: true
title: "Java学习笔记——LinkedHashMap"
description: "Java学习笔记——LinkedHashMap"
category: Java
tags: [Java]
---

**1、LinkedHashMap数据结构**    
**2、LRU实现**    
**3、参考文档**

<!--more-->

### 1、LinkedHashMap数据结构

![](/image/2018-04-10-learning-notes-linkedhashmap/linkedhashmap.png)

上面结构图非常直观且形象的体现了LinkedHashMap的数据结构：同时兼具HashMap和双向链表的特性。    
（1）首先，它是一个HashMap，内部包含一个数组，数组的每个元素要么是一个单链表要么是一个红黑树。单链表和红黑树上的结点根据next属性相连接。    
（2）其次，它将HashMap中的所有结点按照访问顺序或插入顺序连接成一个双向链表，双向链表的结点间通过before，after属性连接。    

我们不难推断出LinkedHashMap的相关特性：    
（1）双向链表的头结点和尾结点：head，tail   
（2）每个结点包含next，before，after属性    
（3）结点访问(put/get/remove)后，存在一个双向链表结构的调整过程。如果按照插入顺序排列，则新结点插入双向链表的尾部；如果按照访问顺序排列，则最新访问的结点移动到双向链表的尾部    

**注意：**    
以访问顺序(accessOrder=true)为例，双向链表的顺序是`非最近访问 -> 最近访问`。    
每次get(key)后，之后将key对应的元素调整到最后；每次put(key,value)，也是将key对应的元素插入调整到最后。    
所以第一个元素是最老的。

结合源码来看：

    public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>
    {
        /**
         * The head (eldest) of the doubly linked list.
         */
        transient LinkedHashMap.Entry<K,V> head;

        /**
         * The tail (youngest) of the doubly linked list.
         */
        transient LinkedHashMap.Entry<K,V> tail;

        static class Entry<K,V> extends HashMap.Node<K,V> {
            Entry<K,V> before, after;
            Entry(int hash, K key, V value, Node<K,V> next) {
                super(hash, key, value, next);
            }
        }
    ｝
    
然后再来看双向链表中结构调整的相关代码：

#### （1）新增结点：

创建结点后，首先将新结点加到双向链表结尾，其次调整head和tail的指向。

    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }

    // link at the end of list
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
    }

#### （2）访问某个结点后：

访问（put／set）某个结点后，会调用afterNodeAccess，若accessOrder为true，会将该结点移动到结尾，同时调整head和tail的指向。

    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
    
#### （3）删除结点后：

会先从双向链表删除该结点，然后调整head和tail。

    void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a == null)
            tail = b;
        else
            a.before = b;
    }

#### （4）插入结点后：

根据条件，可能会删除双向链表最老的结点，以保证HashMap的size不变大。

    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }

和HashMap的比较：    
（1）结点结构上LinkedHashMap.Entry多了before、after属性    
（2）下面几个方法LinkedHashMap增加了对双向链表的结点操作，和头结点尾结点指针的调整    

    newNode(int hash, K key, V value, Node<K,V> e)   
    afterNodeAccess(Node<K,V> e)    
    afterNodeRemoval(Node<K,V> e)    
    afterNodeInsertion(boolean evict)    


### 2、LRU实现

由于LinkedHashMap支持双向链表的访问顺序排序和插入顺序排序方式，因此可以实现LruCache和FIFOCache。    
如果要实现一个LruCache，accessorOrder=true即可；如果要实现一个FIFOCache，accessorOrder=false即可。        

关于元素删除的条件一般有2种情况：    
（1）按`元素个数`，当size > maxSize时    
（2）按`内存大小`，当size > maxSize时，比如在Bitmap场景，肯定不是按照Bitmap个数，而是按照Bitmap总内存。    

都是在调用put(key, value)之后，进行删除操作。

如果按元素个数，可以直接按照示例1代码，复写removeEldestEntry()方法即可。    
如果按内存大小，需要在put之后，再调用trimToSize(maxSize)方式。

使用trimToSize(maxSize)这种方式，同时适用于按照元素个数和内存大小删除的情况。

示例1：

    public class LruCache<K,V> {
        private Map<K, V> cache;

        public LruCache(final int capacity) {
            cache = new LinkedHashMap<K, V>(capacity, 0.75f, true){
                @Override
                protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                    return size() > capacity;
                }
            };
        }

        public V get(K key) {
            if (cache.containsKey(key)) {
                return cache.get(key);
            }
            return null;
        }

        public void set(K key, V val) {
            cache.put(key, val);
        }
    }

示例2:    

    public class LruMemoryCache implements MemoryCache {

	    private final LinkedHashMap<String, Bitmap> map;

    	private final int maxSize;
	    /** Size of this cache in bytes */
    	private int size;

	    public LruMemoryCache(int maxSize) {
		    if (maxSize <= 0) {
			    throw new IllegalArgumentException("maxSize <= 0");
		    }
		    this.maxSize = maxSize;
		    this.map = new LinkedHashMap<String, Bitmap>(0, 0.75f, true);
	    }

	    @Override
	    public final Bitmap get(String key) {
		    if (key == null) {
			    throw new NullPointerException("key == null");
		    }

		    synchronized (this) {
			    return map.get(key);
		    }
	    }

    	@Override
	    public final boolean put(String key, Bitmap value) {
		    if (key == null || value == null) {
			    throw new NullPointerException("key == null || value == null");
		    }

		    synchronized (this) {
			    size += sizeOf(key, value);
			    Bitmap previous = map.put(key, value);
			    if (previous != null) {
				    size -= sizeOf(key, previous);
			    }
		    }

		    trimToSize(maxSize);
		    return true;
	    }

    	private void trimToSize(int maxSize) {
	    	while (true) {
		    	String key;
			    Bitmap value;
			    synchronized (this) {
				    if (size < 0 || (map.isEmpty() && size != 0)) {
					    throw new IllegalStateException(getClass().getName() + ".sizeOf() is reporting inconsistent results!");
				    }

				    if (size <= maxSize || map.isEmpty()) {
					    break;
				    }

				    Map.Entry<String, Bitmap> toEvict = map.entrySet().iterator().next();
				    if (toEvict == null) {
					    break;
				    }
				    key = toEvict.getKey();
				    value = toEvict.getValue();
				    map.remove(key);
				    size -= sizeOf(key, value);
			    }
		    }
	    }

    	@Override
	    public final Bitmap remove(String key) {
		    if (key == null) {
			    throw new NullPointerException("key == null");
		    }

		    synchronized (this) {
			    Bitmap previous = map.remove(key);
			    if (previous != null) {
				    size -= sizeOf(key, previous);
			    }
			    return previous;
		    }
	    }

    	@Override
	    public Collection<String> keys() {
		    synchronized (this) {
			    return new HashSet<String>(map.keySet());
		    }
	    }

    	@Override
	    public void clear() {
		    trimToSize(-1); // -1 will evict 0-sized elements
	    }

    	private int sizeOf(String key, Bitmap value) {
	    	return value.getRowBytes() * value.getHeight();
    	}
    }

### 3、参考文档

（1）[Java 容器源码分析之 LinkedHashMap](http://blog.jrwang.me/2016/java-collections-linkedhashmap/)    
（2）[Java类集框架--LinkedHashMap源码分析](https://juejin.im/post/59c4ded56fb9a00a402df44d)    
