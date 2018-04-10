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
（4）数据是怎么遍历的？

#### （1）存数据put(K key, V value)

#### （2）取数据get(Object key)    

#### （3）删数据remove(Object key)    

#### （4）遍历HashIterator    

### 3、参考文档

