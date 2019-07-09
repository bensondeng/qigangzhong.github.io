---
layout: post
title:  "集合"
categories: java-basics
tags:  java-basics
author: 网络
---

* content
{:toc}









## 概览

![java_collections_all.gif](/images/collection-map/java_collections_all.gif)

### Iterator

> 方法列表：
>
> * hasNext()
> * next() //比较expectedModCount与modCount，不相等则抛出ConcurrentModificationException异常
> * remove() //比较expectedModCount与modCount，不相等则抛出ConcurrentModificationException异常
> * forEachRemaining() //jdk1.8新增对lambda表达式支持
>
> Iterator是一种fail-fast的实现，对元素遍历的时候不允许并发修改，如果发生并发修改操作则抛出ConcurrentModificationException异常
>
> for-each是通过iterator来实现的

### ListIterator

> 可以通过List.listIterator()方法产生ListIterator，它继承自Iterator，比Iterator多了以下方法：
>
> * hasPrevious()、previous() //不光能向后遍历，还能向前遍历
> * nextIndex()、previousIndex() //返回索引位置
> * add() //可以在游标前添加元素
> * set() //可以修改游标指向的元素

## Collection

### List

#### ArrayList

非线程安全，通过数组实现，初始大小10，默认扩容%50

适合随机访问get(index)，在中间add/delete性能差

#### CopyOnWriteArrayList

线程安全

#### LinkedList

非线程安全，通过双向链表实现，实现Deque接口(链表双向队列)，不会扩容，元素是有序的，允许Null，所有指定位置操作都是从头开始遍历

#### Vector、Stack（不推荐使用）

* `Vector`：线程安全，通过数组实现，初始大小10，默认扩容1倍(可以构造函数设置capacityIncrement)，类似于ArrayList，但是由于所有操作都加锁，性能差，一般实际场景ArrayList足够，如果需要线程安全，推荐使用Collections.synchronizedList替代
* `Stack`：线程安全，继承自Vector，是实现了FILO（先进后出）的堆栈，提供了pop/posh/peek方法，性能差，推荐使用Dequeue/ArrayDeque（Double Ended Queue）替代，实现更完善

### Set

#### TreeSet

默认初始容量16，超过0.75倍则扩容100%，支持自然顺序，add/delete/contains效率低

通过TreeMap实现，仅使用TreeMap的Key

#### HashSet

不保证有序，性能与容量有关

通过HashMap实现，仅使用HashMap的Key

#### LinkedHashSet

继承自HashSet，保证元素的插入顺序

通过LinkedHashMap实现

### Queue

#### PriorityQueue

默认按自然顺序排列，也就是数字默认是小的在队列头，字符串则按字母顺序排列，可以指定Comparator自定义元素排序规则

#### Deque

继承自Queue接口，有两个实现类：ArrayDeque、LinkedList

可以FIFO当队列用，也可以FILO当栈用（替代Stack）

### Collections工具类

Collections.synchronizedList

## Map

### HashMap

非线程安全，默认初始容量16，负载因子0.75，超过0.75倍则扩容100%，允许一个null键，自然排序（比如按字母排序，按汉字拼音首字母排序）

JDK1.8中的HashMap使用数组+链表形式，链表长度>8，则链表转换为红黑树

### HashTable

### LinkedHashMap

非线程安全，继承自HashMap，维护双向链表，保持元素插入顺序

### TreeMap

非线程安全，间接实现SortedMap接口，按照key排序，可以指定comparator自定义排序规则

### ConcurrentHashMap

## 常见问题

## 参考

[Java集合框架](https://www.runoob.com/java/java-collections.html)
