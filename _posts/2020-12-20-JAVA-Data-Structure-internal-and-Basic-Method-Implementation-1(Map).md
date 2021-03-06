---
layout: post
title:  "JAVA数据结构内部及基础方法实现一（Map）"
author: "陈宇瀚"
date:   2020-12-20 19:50:02 +0800
header-img: "img/img-head/img-head-java.jpg"
categories: article
tags:
  - JAVA 
  - 数据结构
---

JAVA有几种常用的数据结构，主要是继承**Collection**和**Map**这两个主要接口的数据实现类

在jdk1.7和jdk1.8中，实现会有些许不同，之后会在注解中添加两版本区别
下面分别介绍几个常用的数据结构(按照继承的接口分为两类)，以下代码截取自**基于JAVA8的android SDK 28**
# Map
**Map**接口提供**key**到**value**的映射。一个**Map**中不能包含相同的**key**，每个**key**只能映射一个**value**。**Map**接口提供3种集合的视图，**Map**的内容可以被当作一组**key**集合，一组**value**集合，或者一组**key-value**映射。

## HashMap(以下源码基于JAVA8，与JAVA7有较大差别)
**HashMap**继承**Map**接口，实现一个key-value映射的哈希表。是非同步的，同时允许n**ull value**和**null key**。
**HashMap**的父类接口，以及内部有几个主要的变量，如下：
```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable 

//默认的初始容量-必须是2的幂。
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
//最大容量，如果两个带参数的构造函数中的任何一个隐式指定了更高的值，则使用该值。一定是2的幂<= 1<<30。
static final int MAXIMUM_CAPACITY = 1 << 30;
//未设置负载系统时默认的负载系数/加载因子。
static final float DEFAULT_LOAD_FACTOR = 0.75f;
//容器计数阈值,当table对应hash位置的链表元素数量超过这个阈值，该位置的链表会转换为红黑树
static final int TREEIFY_THRESHOLD = 8;
//容器计数阈值,当table对应hash位置的红黑树元素数量小雨这个阈值，该位置的红黑树会转换为单链表
static final int UNTREEIFY_THRESHOLD = 6;

static final int MIN_TREEIFY_CAPACITY = 64;
//存放数据的表，就是HashMap数据基类的数组，在第一次使用时进行初始化，并根据需要调整大小。分配时，长度总是2的幂。
transient Node<K,V>[] table;
//此Map映射中包含的键-值对的数量。
transient int size;
//(threshold = 容量*负载系数/加载因子)扩容阈值，可以说是一个是否需要扩容的判断条件，当HashMap的size>threshold时会进行resize操作。
int threshold;
//负载系数/加载因子，可以说是当前HashMap满的程度，
//假设它等于0.75，那么在HashMap的键值对数量超过容量*0.75时，则会进行扩容。
//保证有足够的空间进行数据存放，同时不会经常进行扩容。
final float loadFactor;
```
**Node**:**HashMap**的数据基类
```java
static class Node<K,V> implements Map.Entry<K,V> {
        //hash值
        final int hash;
        final K key;
        V value;
        //下一个对象
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
    
    //JAVA8引入红黑树TreeNode，之后会分析
    static final class TreeNode<K,V> extends LinkedHashMap.LinkedHashMapEntry<K,V>
```
**Node**是一个单链表结构的类，由于**HashMap**存储健值对时，会先将**key**转成**hash**与当前容量-1进行一个与操作( (n - 1) & hash)转换成一个**下标值**，在数组中对应一个**Node**，所以会存在转换出来的**下标值**相同(**哈希冲突**)，但**key**不同的情况，所以同一个**下标值**可能对应不同的健值对。所以table数组采用**Node**这个单链表的作为存储基类，用于存放这些**下标值**相同，但键值不同的数据；

在**JAVA8**中考虑到如果哈希冲突多的情况，单链表**Node**的长度会越来越长，此时通过单链表来寻找对应**Key**对应的**Value**的时候就会使得时间复杂度达到**O(n)**，因此在**JAVA8**中引入了**TreeNode(红黑树)**，当链表长度超过**TREEIFY_THRESHOLD(8)**的时候，会将单链表**Node**转换成红黑树**TreeNode**。

*红黑树是一种易于增删改查的二叉树，他对与数据的查询的时间复杂度是O(logn)，所以利用红黑树的特点就可以更高效的对 HashMap 中的元素进行操作。*
#### 构造方法
```java
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
    //构造一个具有指定的初始容量和默认的负载系数(0.75)的空的HashMap，。
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
    
    //构造一个具有指定的初始容量和负载系数的空的HashMap，
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
    //构造一个与指定的Map相同键值对映射的新的HashMap。该HashMap是用默认的负载系数(0.75)创建的，初始容量足以容纳指定的Map中的映射。
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
    
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        //得到推入的map大小
        int s = m.size();
        if (s > 0) {
            if (table == null) { // pre-size
                //根据map大小和负载因子计算出设置的容量大小
                float ft = ((float)s / loadFactor) + 1.0F;
                //判断容量是否超出最大容量
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            else if (s > threshold)
                resize();
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
    //tableSizeFor的功能（不考虑大于最大容量的情况）是返回大于输入参数且最近的2的整数次幂的数。比如10，则返回16。
    static final int tableSizeFor(int cap) {
	//>>：带符号右移。正数右移高位补0，负数右移高位补1 >>> 无符号右移。无论是正数还是负数，高位通通补0
        
	//假设cap=10，则有n=9
	int n = cap - 1;
        n |= n >>> 1; // 1001 |=0101 -> n=1101
        n |= n >>> 2; // 1101 |=0011 -> n=1111
        n |= n >>> 4; // 1111 |=0000 -> n=1111
        n |= n >>> 8;//  1111 |=0000 -> n=1111
        n |= n >>> 16;// 1111 |=0000 -> n=1111
	//最后n+1=1111+1=16
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
    /**********************以下两个方法对于整个HashMap极其重要**********************/
    
    //HashMap添加值的方法
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //判断当前HashMap的table大小
        if ((tab = table) == null || (n = tab.length) == 0)
            //table未创建，调用resize()进行初始化(扩容)
            n = (tab = resize()).length;
            
        //去table中最后一位与hash进行按位与操作，得到的值赋值给i，
        //判断位置的值是否为null，为null则代表还没有与添加数据键值相应的hash相同的数据推入过;
        if ((p = tab[i = (n - 1) & hash]) == null)
            //为空则直接将键值对添加的table的i位置
            tab[i] = newNode(hash, key, value, null);
        else {
            //若不为空，则对应键值hash位置已有数据
            //e代表最后存储的键值对
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                //键值hash相同，键值内容也相同，代表将值替换
                e = p;
            else if (p instanceof TreeNode)
                //此时p对应位置的数据量超过TREEIFY_THRESHOLD(8)，Node已转换为TreeNode,所以采用TreeNode的新增节点方式
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //此时p对应位置的数据量未超过TREEIFY_THRESHOLD(8)，所以还没转化成红黑树。仍是一个Node链表
                
                //binCount用于计算当前链表的节点数，binCount从0开始，代表binCount = 节点数+1；
                for (int binCount = 0; ; ++binCount) {
                    //判断是否到了链表的末尾节点
                    if ((e = p.next) == null) {
                        //在链表末尾添加新生成的Node
                        p.next = newNode(hash, key, value, null);
                        //判断当前节数是否超过TREEIFY_THRESHOLD(8)，超过则将链表Node转换成红黑树
                        if (binCount >= TREEIFY_THRESHOLD， - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //判断是否存在hash相同，key值相同的键值，存在则跳出循环。
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //e!=null代表map种存在key值相同的键值，将对应的valuet替换成新的，同时返回旧value
            if (e != null) { 
                V oldValue = e.value;
                //onlyIfAbsent调用方法时传入，如果为true，则不会覆盖值
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);//一个回调方法
                return oldValue;
            }
        }
        ++modCount;
        //判断是否需要扩容map
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);//一个回调方法
        return null;
    }
    
    //HashMap的扩容方法
    final Node<K,V>[] resize() {
        //获得当前table
        Node<K,V>[] oldTab = table;
        //获取当前map容量，扩容阈值
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                //当前容量超过了可设的最大值
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                //容量翻倍后不超过可设最大容量，旧容量超过课设最小容量
                newThr = oldThr << 1; // 扩容阈值也翻倍
        }
        else if (oldThr > 0) 
            newCap = oldThr;// 此时oldCap《=0,代表hashmap未初始化，但设置了扩容阈值，初始容量设置为阈值
        else {               // 初始扩容阈值为0时
            newCap = DEFAULT_INITIAL_CAPACITY;//初始容量为默认容量(16)
            //初始扩容阈值 = 初始容量(16) * 负载系数/加载因子(0.75)
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            //上面第二种情况下未计算新的扩容阈值，这里计算并赋值
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        //根据新的容量创建一个新的table
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        //将之前的数据转移到新的table
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        //该Node只有一个数据，根据hash和容量重新计算下标放入新table
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        //该Node是一个红黑树Node的情况
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { 
                        //该Node是一个有多个数据的链表(1<数据数量<TREEIFY_THRESHOLD(8))
                        //loHead是用来保存新链表上的低位区的头元素的，loTail是用来保存低位区的尾元素的，
                        Node<K,V> loHead = null, loTail = null;
                        //hiHead是用来保存新链表上的高位区的头元素的，hiTail是用来保存高位区的尾元素的，
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        //进行链表遍历至末尾节点，这部分可能会将会将当前链表分列成两个链表，取决于hash的位数
                        do {
                            next = e.next;
                            //等于0时，则将该头节点放到新数组时的索引位置等于其在旧数组时的索引位置,记为低位区链表lo开头-low;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {//不等于0时,则将该头节点放到新数组时的索引位置等于其在旧数组时的索引位置再加上旧数组长度，记为高位区链表hi开头high.
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                            /**
                             * 这部分举个例子比较好理解，举个栗子来看，假设oldCap = 16 = 10000，两个数据的hash分别为hash1 = 7 = 0111, hash2 = 23 = 10111
                             * 未扩容的时候两个hash对应的下标分别是:
                             * (hash1 & oldCap -1) = 0111 & 01111 = 111
                             * (hash2 & oldCap -1) = 10111 & 01111 = 111 是同样的位置；
                             * 此时我们将容量扩大一倍 newCap = 32 = 100000，在此计算下标
                             * (hash1 & newCap -1) = 0111 & 11111 = 111;位置不变
                             * hash2 & oldCap -1) = 10111 & 11111 = 10111;位置变了，比原来的位置多了10000，即oldCap
                             * 再来算代码中的e.hash & oldCap == 0这部分
                             * 可以知道hash1 & oldCap = 0111 & 10000 = 0；hash2 & oldCap = 10111 & 10000 = 1；
                             * 所以可以得出，如果e.hash & oldCap == 0，那么对应hash的Node在表扩容后也在数组同样的位置
                             * 而e.hash & oldCap != 0的Node，则会移动，移动的间隔就是oldCap的长度
                             */
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```
可以看出其实**HashMap**初始化时是不会初始化我们的数据表**table**，只会初始化一些容量大小和负载因子的值，在第一次使用时才会创建，例如在第四个构造函数，根据传入的**map**构造一个**HashMap**，调用**putVal**方法时内部的**resize**函数会初始化**table**，当单链表数据项超过**TREEIFY_THRESHOLD(8)**，会将单链表**Node**转换成红黑树**TreeNode**，**TreeNode**比较复杂，留到最后讲。

接下来来看一些常用的方法
#### put(K,V)
**put(K,V)**:推入指定的键值对的映射。如果该映射先前包含了该键的映射，则旧值将被替换。
```java
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    
```
#### remove(Object)
**remove(Object)**:从该映射中移除指定键的映射(如果存在)。
```java
public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }
    
final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        // 判断table和对应的key值Node是否为空
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            //node存放键值相同的Node
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                //找到了对应的Node，此处结束node = p;
                node = p;
            else if ((e = p.next) != null) {//当前链表头不是对应的键值对映射
                if (p instanceof TreeNode)
                    // 如果是红黑树TreeNode，则在红黑树内寻找
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    //否则在单链表内循环查找，此处结束node = p.next;
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            //如果matchValue为true,则只移除value相等的键值对映射
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    //调用红黑树的方式移除
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    //node是单链表头
                    tab[index] = node.next; 
                else
                    //此时node就会从单链表中分离
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```
#### get(Object)
**get(Object)**:返回指定键映射到的值，如果该映射不包含该键的映射，则返回null。
```java
 public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    
 final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //判断table是否为空，是否存在数据，是否存在对应键值的Node
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                //单链表头就是对应的Node，直接返回
                return first;
            //在链表内搜索hash和key值相同的Node并返回
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```
#### putAll(Map<? extends K, ? extends V>)
**putAll(Map<? extends K, ? extends V>)**:将指定Map的所有键值对映射复制到此映射Map。这些映射将替换该映射对指定映射中对应存在的键值对映射。
```java
 public void putAll(Map<? extends K, ? extends V> m) {
        putMapEntries(m, true);
    }
    
```
#### clear()
**clear()**:清空**HashMap**中的所有键值对映射；
```java
 public void clear() {
        Node<K,V>[] tab;
        modCount++;
        //将table数组每一位都设置为null;
        if ((tab = table) != null && size > 0) {
            size = 0;
            for (int i = 0; i < tab.length; ++i)
                tab[i] = null;
        }
    }
```
## HashTable
**Hashtable**继承**Map**接口，实现一个key-value映射的哈希表。任何非空（**non-null**）的对象都可作为key或者value。
**HashTable**的父类接口，以及内部有几个主要的变量，如下：
```java
public class Hashtable<K,V>
    extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable
//hash数据表
private transient HashtableEntry<?,?>[] table;
//HashTable中的数据数量
private transient int count;
//(threshold = 容量*负载系数/加载因子)扩容阈值，可以说是一个是否需要扩容的判断条件，当HashMap的size>threshold时会进行resize操作。
int threshold;
private int threshold;
//负载系数/加载因子，可以说是当前HashMap满的程度，
//假设它等于0.75，那么在HashMap的键值对数量超过容量*0.75时，则会进行扩容。
//保证有足够的空间进行数据存放，同时不会经常进行扩容。
final float loadFactor;
private float loadFactor;

//可分配的内部数组的最大大小
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```
**HashtableEntry**:HashTable的数据基类
```java
 private static class HashtableEntry<K,V> implements Map.Entry<K,V> {
    // END Android-changed: Renamed Entry -> HashtableEntry.
        final int hash;
        final K key;
        V value;
        HashtableEntry<K,V> next;

        protected HashtableEntry(int hash, K key, V value, HashtableEntry<K,V> next) {
            this.hash = hash;
            this.key =  key;
            this.value = value;
            this.next = next;
        }

        @SuppressWarnings("unchecked")
        protected Object clone() {
            return new HashtableEntry<>(hash, key, value,
                                  (next==null ? null : (HashtableEntry<K,V>) next.clone()));
        }

        // Map.Entry Ops

        public K getKey() {
            return key;
        }

        public V getValue() {
            return value;
        }

        public V setValue(V value) {
            if (value == null)
                throw new NullPointerException();

            V oldValue = this.value;
            this.value = value;
            return oldValue;
        }

        public boolean equals(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;

            return (key==null ? e.getKey()==null : key.equals(e.getKey())) &&
               (value==null ? e.getValue()==null : value.equals(e.getValue()));
        }

        public int hashCode() {
            return hash ^ Objects.hashCode(value);
        }

        public String toString() {
            return key.toString()+"="+value.toString();
        }
    }
```
**HashTable**与**HashMap**非常相似，有很多作用一样的变量。
#### 构造方法
```java
 //构造一个新的Hashtable，具有默认的初始容量(11)和加载系数(0.75)。
 public Hashtable() {
        this(11, 0.75f);
    }
    
 //构造一个指定初始容量的新的Hashtable，具有默认的加载系数(0.75)。
 public Hashtable(int initialCapacity) {
        this(initialCapacity, 0.75f);
    }
    
 //构造一个指定初始容量和指定加载系数的新的Hashtable。
 public Hashtable(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal Load: "+loadFactor);

        if (initialCapacity==0)
            initialCapacity = 1;
        this.loadFactor = loadFactor;
        table = new HashtableEntry<?,?>[initialCapacity];
        threshold = (int)Math.min(initialCapacity, MAX_ARRAY_SIZE + 1);
    }

 //根据传入的Map一个新的HashTable。HashTable的初始容量足以容纳给定Map中的映射数据，并具有默认的加载因子(0.75)。
 public Hashtable(Map<? extends K, ? extends V> t) {
        this(Math.max(2*t.size(), 11), 0.75f);
        putAll(t);
    }
     
 public synchronized void putAll(Map<? extends K, ? extends V> t) {
        for (Map.Entry<? extends K, ? extends V> e : t.entrySet())
            put(e.getKey(), e.getValue());
    }
    
 //添加键值对，是一个同步的方法
 public synchronized V put(K key, V value) {
        //禁止添加空的value
        if (value == null) {
            throw new NullPointerException();
        }

        HashtableEntry<?,?> tab[] = table;
        int hash = key.hashCode();
        //根据key的hash值和表容量计算下标
        //0x7FFFFFFF是一个用16进制表示的整型,是整型里面的最大值
        //转成2进制由31个1组成，而整形的最高是32位
        //hash与其按位与得到一个正数
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        //获得对应下标的单链表
        HashtableEntry<K,V> entry = (HashtableEntry<K,V>)tab[index];
        //如果存在key值相同的键值对，则替换value
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }
        
        addEntry(hash, key, value, index);
        return null;
    }
    
private void addEntry(int hash, K key, V value, int index) {
        modCount++;

        HashtableEntry<?,?> tab[] = table;
        //判断当前容量是否超出或等于扩容阈值
        if (count >= threshold) {
            // 如果超过阈值，则扩容
            rehash();
            
            //扩容后重新计算下标index
            tab = table;
            hash = key.hashCode();
            index = (hash & 0x7FFFFFFF) % tab.length;
        }

        //创建新的HashtableEntry，并赋值
        @SuppressWarnings("unchecked")
        HashtableEntry<K,V> e = (HashtableEntry<K,V>) tab[index];
        tab[index] = new HashtableEntry<>(hash, key, value, e);
        //数据量+1
        count++;
    }
    
  //扩容，并对内部的键值对重新排列
  protected void rehash() {
        int oldCapacity = table.length;
        HashtableEntry<?,?>[] oldMap = table;

        //新容量为两倍旧容量+1
        int newCapacity = (oldCapacity << 1) + 1;
        //判断新容量是否超出最大容量
        if (newCapacity - MAX_ARRAY_SIZE > 0) {
            if (oldCapacity == MAX_ARRAY_SIZE)
                // Keep running with MAX_ARRAY_SIZE buckets
                return;
            newCapacity = MAX_ARRAY_SIZE;
        }
        //使用新容量创建一个新的数组table
        HashtableEntry<?,?>[] newMap = new HashtableEntry<?,?>[newCapacity];

        modCount++;
        //计算新的扩容阈值
        threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
        table = newMap;
        
        //将旧map数组的数据导入新map数组,重新计算下标
        for (int i = oldCapacity ; i-- > 0 ;) {
            for (HashtableEntry<K,V> old = (HashtableEntry<K,V>)oldMap[i] ; old != null ; ) {
                HashtableEntry<K,V> e = old;
                old = old.next;

                int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                e.next = (HashtableEntry<K,V>)newMap[index];
                newMap[index] = e;
            }
        }
    }
    
```
可以看出其实**HashTable**与**HashMap**的初始化类似，在初始化时是不会初始化我们的数据表**table**，只会初始化一些容量大小和负载因子的值，在第一次使用时才会创建。例如**putAll**。
#### put(K,V)
**put(K,V)**:推入指定的键值对的映射。如果该映射先前包含了该键的映射，则旧值将被替换。**key**和**value**都不能为null。
```java
 public synchronized V put(K key, V value) {
        //确保valu不为null
        if (value == null) {
            throw new NullPointerException();
        }

        HashtableEntry<?,?> tab[] = table;
        //计算下标
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        //获得对应下标的单链表
        HashtableEntry<K,V> entry = (HashtableEntry<K,V>)tab[index];
        //如果存在在key值已存在，则替换value
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }
        //不存在添加键值对
        addEntry(hash, key, value, index);
        return null;
    }
```
#### remove(Object)
**remove(Object)**:从这个**HashTable**移除**key**(及其对应的**value**)。如果**key**不在**HashTable**中，此方法不执行任何操作。
```
 public synchronized V remove(Object key) {
        //计算下标
        HashtableEntry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        
        @SuppressWarnings("unchecked")
        HashtableEntry<K,V> e = (HashtableEntry<K,V>)tab[index];
        //判断是否存在对应的key值
        for(HashtableEntry<K,V> prev = null ; e != null ; prev = e, e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                modCount++;
                if (prev != null) {
                    prev.next = e.next;
                } else {
                    tab[index] = e.next;
                }
                count--;
                V oldValue = e.value;
                e.value = null;
                return oldValue;
            }
        }
        return null;
    }

```
#### get(Object)
**get(Object)**:返回指定**key**对应的**value**,若**key**不在**HashTable**中，则返回**null**
```java
 public synchronized V get(Object key) {
        //计算下标
        HashtableEntry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        //找key对应的value
        for (HashtableEntry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return (V)e.value;
            }
        }
        return null;
    }
```
#### clear()
**clear()**:清空此**HashTable**
```java
  public synchronized void clear() {
        HashtableEntry<?,?> tab[] = table;
        modCount++;
        for (int index = tab.length; --index >= 0; )
            tab[index] = null;
        count = 0;
    }
```
**HashTable**的方法基本都是**synchronized**的，相对于**HashMap**是同步的。
## HashMap与HashTable的不同
### 一 线程安全性
- **HashTable**方法添加了**synchronized**关键字，是同步的，是线程安全的
- - **HashMap**是非同步的，线程不安全

### 二 内部方法
- **Hashtable**则保留了**contains**，**containsValue**和**containsKey**三个方法，其中**contains**和**containsValue**功能相同。
- **HashMap**没有**contains**方法去掉了，改成**containsValue**和**containsKey**，因为**contains**方法容易让人引起误解。

方法内容对比：
**HashTable**
```java
  public synchronized boolean contains(Object value) {
        //value不能为null
        if (value == null) {
            throw new NullPointerException();
        }
        //包含返回true,不包含返回false
        HashtableEntry<?,?> tab[] = table;
        for (int i = tab.length ; i-- > 0 ;) {
            for (HashtableEntry<?,?> e = tab[i] ; e != null ; e = e.next) {
                if (e.value.equals(value)) {
                    return true;
                }
            }
        }
        return false;
    }
  //查看是否包含value
  public boolean containsValue(Object value) {
        return contains(value);
    }
    
  //查看是否包含key
  public synchronized boolean containsKey(Object key) {
        HashtableEntry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (HashtableEntry<?,?> e = tab[index] ; e != null ; e = e.next) {
             //包含返回true,不包含返回false
            if ((e.hash == hash) && e.key.equals(key)) {
                return true;
            }
        }
        return false;
    }
```
**HashMap**
```java
 public boolean containsValue(Object value) {
        Node<K,V>[] tab; V v;
        if ((tab = table) != null && size > 0) {
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                    if ((v = e.value) == value ||
                        (value != null && value.equals(v)))
                        return true;
                }
            }
        }
        return false;
    }

  public boolean containsKey(Object key) {
        return getNode(hash(key), key) != null;
    }
```
### key和value的限制
根据上面的**containsValue**和**containsKey**方法得出：
**Hashtable**:**key**和**value**都不允许出现**null**值；
**HashMap**:**null**可以作为**key**，这样的**key**只有一个；但是可以有一个或多个**key**所对应的值为**null**。当**get()**方法返回n**ull**值时，可能是 **HashMap**中没有该键，也可能使该键所对应的值为**null**。因此，在**HashMap**中不能由**get()**方法来判断**HashMap**中是否存在某个**key**， 而应该用**containsKey()**方法来判断。
### 两个遍历方式的内部实现上不同
**Hashtable**:使用了 **Iterator**
**HashMap**:使用了 **Iterator**。由于历史原因，还使用了**Enumeration**的方式 。
### hash值不同
**Hashtable**:直接使用对象的**hashCode**,**hashCode**是jdk根据对象的地址或者字符串或者数字算出来的**int**类型的数值。**Hashtable**在求**hash**值对应的位置索引时，用取模运算;
**HashMap**:重新计算**hash**值,在求位置索引时，则用与运算;

### 内部实现使用的数组初始化和扩容方式不同
**Hashtable**:在不指定容量的情况下的默认容量为11,**Hashtable**不要求底层数组的容量一定要为2的整数次幂;扩容时，**Hashtable**将容量变为原来的2倍加1
**HashMap**:在不指定容量的情况下的默认容量为16,**HashMap**要求底层数组的容量一定为2的整数次幂,扩容时，**HashMap**将容量变为原来的2倍。

## TreeMap
**TreeMap**是一个非同步有序的**key-value**集合，它通过**红黑树**实现，实现了**NavigableMap**接口，意味着它支持一系列的导航方法。比如返回**有序的key集合**。该映射根据其键的自然顺序进行排序，或者根据创建映射时提供的 **Comparator** 进行排序，具体取决于使用的构造方法。
**TreeMap**的基本操作 **containsKey**、**get**、**put** 和 **remove** 的时间复杂度是 **log(n)** 。使用不多，就暂不介绍了。
