---
layout: post
comments: true
title: "Java学习笔记——HashMap"
description: "Java学习笔记——HashMap"
category: Java
tags: [Java]
---

**1、HashMap数据结构**    
**2、数据的增删改查**    
**（1）存数据put(K key, V value)**    
**（2）取数据get(Object key)**    
**（3）删数据remove(Object key)**    
**（4）遍历HashIterator**    
**3、参考文档**


<!--more-->


### 1、HashMap数据结构

先结合源码来看。（备注：仅分析JDK8源码    

    Node<K,V>[] table;
    int size;
    int threshold;
    float loadFactor;


    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }

![](/image/2018-04-10-learning-notes-hashmap/hashmap.png)

HashMap的结构图如上所示：    
（1）内部包含一个数组，数组中每个元素可以认为是一个桶Bin，每个桶里对应一个单链表或红黑树。    
（2）当某个桶里单链表结点数>=7，需要转换成红黑树结构    
（3）当某个桶里红黑树结点数<=6，需要退化成单链表结构    

从时间复杂度来分析，这样设计的优势：    
假设单链表的长度为n，则查找某个结点的时间复杂度是O(n)，而红黑树的时间复杂度是O(logn)

### 2、数据的增删改查

知道了整个结构图，我接下来有几个问题会比较好奇：    
（1）怎么存数据？    
（2）怎么取数据？    
（3）怎么删数据？    
（4）数据怎么遍历？

具体每一行代码注释，下面参考文档 **[Java 容器源码分析之 HashMap]** 的大佬讲的很好，直接看那个就行了。我自己只总结流程。

#### （1）存数据put(K key, V value)

基本逻辑：    
1.计算key的hash值    
2.根据i = (n - 1) & hash计算索引，得到在数组（桶）第几个位置    
3.针对table[i]，进行put操作，考虑如下情况：    

- （1）若table[i]头结点p为空，则新建一个结点Node，作为table[i]头结点    
- （2）若table[i]头结点满足 `p.hash == hash && (p.key == key || key.equals(p.key))`，则更新头结点的value    
- （3）若table[i]是红黑树，则走红黑树的put逻辑    
- （4）若table[i]是单链表，则遍历单链表。若找到符合条件`p.hash == hash && (p.key == key || key.equals(p.key))`的结点，则更新结点值；若没找到则新建一个结点添加到单链表结尾。        

4.数据put后，相关扫尾操作：    

- （1）若单链表长度>=7，调用`treeifyBin`将单链表转化成红黑树
- （2）如果是更新结点，则针对被更新的结点，调用`afterNodeAccess(e)`进行访问后处理
- （3）如果是添加结点，则size+1，且若size > threshold，调用`resize()`进行扩容处理，之后调用`afterNodeInsertion(evict)`进行插入后处理    

计算hash(key)的逻辑：

    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

扩容逻辑：



#### （2）取数据get(Object key)    

#### （3）删数据remove(Object key)    

#### （4）遍历HashIterator    

#### （5）扩容


### 3、参考文档

（1）[Java 容器源码分析之 HashMap](http://blog.jrwang.me/2016/java-collections-hashmap/)    
（2）