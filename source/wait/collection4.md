---
title: Java集合详解4：HashMap和HashTable
date: 2018-05-10 23:34:08
tags:
    - Java集合框架
categories:
	- 后端
	- Java集合类
---
今天我们来探索一下HashMap和HashTable机制与比较器的源码。

具体代码在我的GitHub中可以找到

https://github.com/h2pl/MyTech

喜欢的话麻烦star一下哈

文章首发于我的个人博客：

https://h2pl.github.io/2018/05/10/collection4

更多关于Java后端学习的内容请到我的CSDN博客上查看：https://blog.csdn.net/a724888

我的个人博客主要发原创文章，也欢迎浏览
https://h2pl.github.io/

<!-- more -->

# HashMap

 HashMap也是我们使用非常多的Collection，它是基于哈希表的 Map 接口的实现，以key-value的形式存在。在HashMap中，key-value总是会当做一个整体来处理，系统会根据hash算法来来计算key-value的存储位置，我们总是可以通过key快速地存、取value。下面就来分析HashMap的存取。



## 定义

  HashMap实现了Map接口，继承AbstractMap。其中Map接口定义了键映射到值的规则，而AbstractMap类提供 Map 接口的骨干实现，以最大限度地减少实现此接口所需的工作，其实AbstractMap类已经实现了Map，这里标注Map LZ觉得应该是更加清晰吧！

    public class HashMap<K,V>
        extends AbstractMap<K,V>
        implements Map<K,V>, Cloneable, Serializable
    
## 构造函数

      HashMap提供了三个构造函数：

      HashMap()：构造一个具有默认初始容量 (16) 和默认加载因子 (0.75) 的空 HashMap。

      HashMap(int initialCapacity)：构造一个带指定初始容量和默认加载因子 (0.75) 的空 HashMap。

      HashMap(int initialCapacity, float loadFactor)：构造一个带指定初始容量和加载因子的空 HashMap。

  在这里提到了两个参数：初始容量，加载因子。
  
>   这两个参数是影响HashMap性能的重要参数，其中容量表示哈希表中桶的数量，初始容量是创建哈希表时的容量，加载因子是哈希表在其容量自动增加之前可以达到多满的一种尺度，它衡量的是一个散列表的空间的使用程度，负载因子越大表示散列表的装填程度越高，反之愈小。
>   
>   对于使用链表法的散列表来说，查找一个元素的平均时间是O(1+a)，因此如果负载因子越大，对空间的利用更充分，然而后果是查找效率的降低；如果负载因子太小，那么散列表的数据将过于稀疏，对空间造成严重浪费。系统默认负载因子为0.75，一般情况下我们是无需修改的。

  HashMap是一种支持快速存取的数据结构，要了解它的性能必须要了解它的数据结构。

## 数据结构

>   我们知道在Java中最常用的两种结构是数组和模拟指针(引用)，几乎所有的数据结构都可以利用这两种来组合实现，HashMap也是如此。实际上HashMap是一个“链表散列”，如下是它的数据结构：

HashMap数据结构图

下图的table数组的每个格子都是一个桶。负载因子就是map中的元素占用的容量百分比。比如负载因子是0.75，初始容量（桶数量）为16时，那么允许装填的元素最大个数就是16*0.75 = 12，这个最大个数也被成为阈值，就是map中定义的threshold。超过这个阈值时，map就会自动扩容。

![image](http://cmsblogs.com/wp-content/uploads/2014/01/HashMap.png)

>   从上图我们可以看出HashMap底层实现还是数组，只是数组的每一项都是一条链。其中参数initialCapacity就代表了该数组的长度。下面为HashMap构造函数的源码：

    public HashMap(int initialCapacity, float loadFactor) {
            //初始容量不能<0
            if (initialCapacity < 0)
                throw new IllegalArgumentException("Illegal initial capacity: "
                        + initialCapacity);
            //初始容量不能 > 最大容量值，HashMap的最大容量值为2^30
            if (initialCapacity > MAXIMUM_CAPACITY)
                initialCapacity = MAXIMUM_CAPACITY;
            //负载因子不能 < 0
            if (loadFactor <= 0 || Float.isNaN(loadFactor))
                throw new IllegalArgumentException("Illegal load factor: "
                        + loadFactor);
    
            // 计算出大于 initialCapacity 的最小的 2 的 n 次方值。
            int capacity = 1;
            while (capacity < initialCapacity)
                capacity <<= 1;
            
            this.loadFactor = loadFactor;
            //设置HashMap的容量极限，当HashMap的容量达到该极限时就会进行扩容操作
            threshold = (int) (capacity * loadFactor);
            //初始化table数组，也就是桶数组。
            table = new Entry[capacity];
            init();
        }
          从源码中可以看出，每次新建一个HashMap时，都会初始化一个table数组。table数组的元素为Entry节点。

    static class Entry<K,V> implements Map.Entry<K,V> {
            final K key;
            V value;
            Entry<K,V> next;
            final int hash;
    
            /**
             * Creates new entry.
             */
            Entry(int h, K k, V v, Entry<K,V> n) {
                value = v;
                next = n;
                key = k;
                hash = h;
            }
            .......
        }
          其中Entry为HashMap的内部类，它包含了键key、值value、下一个节点next，以及hash值，这是非常重要的，正是由于Entry才构成了table数组的项为链表。
    
          上面简单分析了HashMap的数据结构，下面将探讨HashMap是如何实现快速存取的。

## 存储实现：put(key,vlaue)

      首先我们先看源码

    public V put(K key, V value) {
            //当key为null，调用putForNullKey方法，保存null与table第一个位置中，这是HashMap允许为null的原因
            if (key == null)
                return putForNullKey(value);
            //计算key的hash值，此处对原来元素的hashcode进行了再次hash
            int hash = hash(key.hashCode());                  ------(1)
            //计算key hash 值在 table 数组中的位置
            int i = indexFor(hash, table.length);             ------(2)
            //从i出开始迭代 e,找到 key 保存的位置
            for (Entry<K, V> e = table[i]; e != null; e = e.next) {
                Object k;
                //判断该条链上是否有hash值相同的(key相同)
                //若存在相同，则直接覆盖value，返回旧value
                if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                    V oldValue = e.value;    //旧值 = 新值
                    e.value = value;
                    e.recordAccess(this);
                    return oldValue;     //返回旧值
                }
            }
            //修改次数增加1
            modCount++;
            //将key、value添加至i位置处
            addEntry(hash, key, value, i);
            return null;
        }
        
  通过源码我们可以清晰看到HashMap保存数据的过程为：首先判断key是否为null，若为null，则直接调用putForNullKey方法。
  
  若不为空则先计算key的hash值，然后根据hash值搜索在table数组中的索引位置，如果table数组在该位置处有元素，则通过比较是否存在相同的key，若存在则覆盖原来key的value，==否则将该元素保存在链头（最先保存的元素放在链尾）==。
  
  若table在该处没有元素，则直接保存。这个过程看似比较简单，其实深有内幕。有如下几点：

>   1、 先看迭代处。此处迭代原因就是为了防止存在相同的key值，若发现两个hash值（key）相同时，HashMap的处理方式是用新value替换旧value，这里并没有处理key，这就解释了HashMap中没有两个相同的key。
> 
>   2、 在看（1）、（2）处。这里是HashMap的精华所在。首先是hash方法，该方法为一个纯粹的数学计算，就是计算h的hash值。

    static int hash(int h) {
            h ^= (h >>> 20) ^ (h >>> 12);
            return h ^ (h >>> 7) ^ (h >>> 4);
        }
        
  我们知道对于HashMap的table而言，数据分布需要均匀（最好每项都只有一个元素，这样就可以直接找到），不能太紧也不能太松，太紧会导致查询速度慢，太松则浪费空间。计算hash值后，怎么才能保证table元素分布均与呢？我们会想到取模，但是由于取模的消耗较大，HashMap是这样处理的：调用indexFor方法。

    static int indexFor(int h, int length) {
            return h & (length-1);
        }
        
  HashMap的底层数组长度总是2的n次方，在构造函数中存在：capacity <<= 1;这样做总是能够保证HashMap的底层数组长度为2的n次方。当length为2的n次方时，h&(length - 1)就相当于对length取模，而且速度比直接取模快得多，这是HashMap在速度上的一个优化。至于为什么是2的n次方下面解释。
  
 ==对length取模来得到hash是常用的hash索引方法，这里采用位运算的话效率更高。==

  我们回到indexFor方法，该方法仅有一条语句：h&(length - 1)，这句话除了上面的取模运算外还有一个非常重要的责任：均匀分布table数据和充分利用空间。

  这里我们假设length为16(2^n)和15，h为5、6、7。

table1

![image](http://cmsblogs.com/wp-content/uploads/2014/01/table1_thumb.jpg)
  
  当n=15时，6和7的结果一样，这样表示他们在table存储的位置是相同的，也就是产生了碰撞，6、7就会在一个位置形成链表，这样就会导致查询速度降低。诚然这里只分析三个数字不是很多，那么我们就看0-15。

    
table2

![image](http://cmsblogs.com/wp-content/uploads/2014/01/table2_thumb.jpg)

  从上面的图表中我们看到总共发生了8次碰撞，同时发现浪费的空间非常大，有1、3、5、7、9、11、13、15处没有记录，也就是没有存放数据。
  
  这是因为他们在与14进行&运算时，得到的结果最后一位永远都是0，即0001、0011、0101、0111、1001、1011、1101、1111位置处是不可能存储数据的，空间减少，进一步增加碰撞几率，这样就会导致查询速度慢。
  
>   而当length = 16时，length – 1 = 15 即1111，那么进行低位&运算时，值总是与原来hash值相同，而进行高位运算时，其值等于其低位值。所以说当length = 2^n时，不同的hash值发生碰撞的概率比较小，这样就会使得数据在table数组中分布较均匀，查询速度也较快。
> 
>   这里我们再来复习put的流程：当我们想一个HashMap中添加一对key-value时，系统首先会计算key的hash值，然后根据hash值确认在table中存储的位置。若该位置没有元素，则直接插入。否则迭代该处元素链表并依此比较其key的hash值。
>   
>   如果两个hash值相等且key值相等(e.hash == hash && ((k = e.key) == key || key.equals(k))),则用新的Entry的value覆盖原来节点的value。如果两个hash值相等但key值不等 ，则将该节点插入该链表的链头。具体的实现过程见addEntry方法，如下：


    void addEntry(int hash, K key, V value, int bucketIndex) {
            //获取bucketIndex处的Entry
            Entry<K, V> e = table[bucketIndex];
            //将新创建的 Entry 放入 bucketIndex 索引处，并让新的 Entry 指向原来的 Entry 
            table[bucketIndex] = new Entry<K, V>(hash, key, value, e);
            //若HashMap中元素的个数超过极限了，则容量扩大两倍
            if (size++ >= threshold)
                resize(2 * table.length);
        }
        
  这个方法中有两点需要注意：

    后面添加的entry反而会接到前面。

  一、是链的产生。
  
  这是一个非常优雅的设计。系统总是将新的Entry对象添加到bucketIndex处。如果bucketIndex处已经有了对象，那么新添加的Entry对象将指向原有的Entry对象，形成一条Entry链，但是若bucketIndex处没有Entry对象，也就是e==null,那么新添加的Entry对象指向null，也就不会产生Entry链了。

  二、扩容问题。

  随着HashMap中元素的数量越来越多，发生碰撞的概率就越来越大，所产生的链表长度就会越来越长，这样势必会影响HashMap的速度，为了保证HashMap的效率，系统必须要在某个临界点进行扩容处理。
  
  该临界点在当HashMap中元素的数量等于table数组长度*加载因子。但是扩容是一个非常耗时的过程，因为它需要重新计算这些数据在新table数组中的位置并进行复制处理。所以如果我们已经预知HashMap中元素的个数，那么预设元素的个数能够有效的提高HashMap的性能。
 
## JDK1.8的hashmap：put方法
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                       boolean evict) {
            Node<K,V>[] tab; Node<K,V> p; int n, i;
            if ((tab = table) == null || (n = tab.length) == 0)
                n = (tab = resize()).length;
            if ((p = tab[i = (n - 1) & hash]) == null)
                tab[i] = newNode(hash, key, value, null);
            else {
                Node<K,V> e; K k;
                if (p.hash == hash &&
                    ((k = p.key) == key || (key != null && key.equals(k))))
                    e = p;
                    //如果p是红黑树节点，则用另外的处理方法
                else if (p instanceof TreeNode)
                    e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
                else {
                    for (int binCount = 0; ; ++binCount) {
                        if ((e = p.next) == null) {
                            p.next = newNode(hash, key, value, null);
                            if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            //当链表节点数超过8个，则直接进行红黑树化。
                                treeifyBin(tab, hash);
                            break;
                        }
                        if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                            break;
                        p = e;
                    }
                }
                if (e != null) { // existing mapping for key
                    V oldValue = e.value;
                    if (!onlyIfAbsent || oldValue == null)
                        e.value = value;
                    afterNodeAccess(e);
                    return oldValue;
                }
            }
            ++modCount;
            if (++size > threshold)
                resize();
            afterNodeInsertion(evict);
            return null;
        }
        
JDK1.8在链表长度超过8时会转换为红黑树。
转换方法如下：

    final void treeifyBin(Node<K,V>[] tab, int hash) {
            int n, index; Node<K,V> e;
            if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            //如果节点数变小小于红黑树的节点数阈值时，调整空间
                resize();
            else if ((e = tab[index = (n - 1) & hash]) != null) {
                TreeNode<K,V> hd = null, tl = null;
                do {
                //该方法直接返回一个红黑树结点。
                    TreeNode<K,V> p = replacementTreeNode(e, null);
                    if (tl == null)
                        hd = p;
                    else {
                    //从链表头开始依次插入红黑树
                        p.prev = tl;
                        tl.next = p;
                    }
                    tl = p;
                } while ((e = e.next) != null);
                if ((tab[index] = hd) != null)
                    hd.treeify(tab);
            }
        }
        
            // For treeifyBin
    TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
        return new TreeNode<>(p.hash, p.key, p.value, next);
    }
  
## 扩容

    final Node<K,V>[] resize() {
            Node<K,V>[] oldTab = table;
            int oldCap = (oldTab == null) ? 0 : oldTab.length;
            int oldThr = threshold;
            int newCap, newThr = 0;
            if (oldCap > 0) {
                //如果原容量大于最大空间，则让阈值为最大值。因为不能再扩容了，最大容量就是整数最大值。
                if (oldCap >= MAXIMUM_CAPACITY) {
                    threshold = Integer.MAX_VALUE;
                    return oldTab;
                }
                //两倍扩容，阈值也跟着变为两倍
                else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                         oldCap >= DEFAULT_INITIAL_CAPACITY)
                    newThr = oldThr << 1; // double threshold
            }
            else if (oldThr > 0) // initial capacity was placed in threshold
                newCap = oldThr;
            else {               // zero initial threshold signifies using defaults
                newCap = DEFAULT_INITIAL_CAPACITY;
                newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
            }
            if (newThr == 0) {
                float ft = (float)newCap * loadFactor;
                newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                          (int)ft : Integer.MAX_VALUE);
            }
            threshold = newThr;
            @SuppressWarnings({"rawtypes","unchecked"})
                Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
            table = newTab;
            if (oldTab != null) {
                for (int j = 0; j < oldCap; ++j) {
                    Node<K,V> e;
                    if ((e = oldTab[j]) != null) {
                        oldTab[j] = null;
                        if (e.next == null)
                            //当后面没有节点时，直接插入即可 //每个元素重新计算索引位置，此处的hash值并没有变，只是改变索引值
                            newTab[e.hash & (newCap - 1)] = e;
                        else if (e instanceof TreeNode)
                            ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                        else { // preserve order
                        //否则，就从头到尾依次将节点进行索引然后插入新数组，这样插入后的链表顺序会和原来的顺序相反。
                            Node<K,V> loHead = null, loTail = null;
                            Node<K,V> hiHead = null, hiTail = null;
                            Node<K,V> next;
                            do {
                                next = e.next;
                                if ((e.hash & oldCap) == 0) {
                                    if (loTail == null)
                                        loHead = e;
                                    else
                                        loTail.next = e;
                                    loTail = e;
                                }
                                else {
                                    if (hiTail == null)
                                        hiHead = e;
                                    else
                                        hiTail.next = e;
                                    hiTail = e;
                                }
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
  
## 读取实现：get(key)

            相对于HashMap的存而言，取就显得比较简单了。通过key的hash值找到在table数组中的索引处的Entry，然后返回该key对应的value即可。

    public V get(Object key) {
            // 若为null，调用getForNullKey方法返回相对应的value
            if (key == null)
                return getForNullKey();
            // 根据该 key 的 hashCode 值计算它的 hash 码  
            int hash = hash(key.hashCode());
            // 取出 table 数组中指定索引处的值
            for (Entry<K, V> e = table[indexFor(hash, table.length)]; e != null; e = e.next) {
                Object k;
                //若搜索的key与查找的key相同，则返回相对应的value
                if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
                    return e.value;
            }
            return null;
        }
        
> 在这里能够根据key快速的取到value除了和HashMap的数据结构密不可分外，还和Entry有莫大的关系，在前面就提到过，HashMap在存储过程中并没有将key，value分开来存储，而是当做一个整体key-value来处理的，这个整体就是Entry对象。
> 
> 同时value也只相当于key的附属而已。在存储的过程中，系统根据key的hashcode来决定Entry在table数组中的存储位置，在取的过程中同样根据key的hashcode取出相对应的Entry对象。
> 
> 在java中与有两个类都提供了一个多种用途的hashTable机制，他们都可以将可以key和value结合起来构成键值对通过put(key,value)方法保存起来，然后通过get(key)方法获取相对应的value值。

# HashTable

一个是前面提到的HashMap，还有一个就是马上要讲解的HashTable。对于HashTable而言，它在很大程度上和HashMap的实现差不多，如果我们对HashMap比较了解的话，对HashTable的认知会提高很大的帮助。他们两者之间只存在几点的不同，这个后面会阐述。

## 定义

      HashTable在Java中的定义如下：

    public class Hashtable<K,V>
        extends Dictionary<K,V>
        implements Map<K,V>, Cloneable, java.io.Serializable
          从中可以看出HashTable继承Dictionary类，实现Map接口。其中Dictionary类是任何可将键映射到相应值的类（如 Hashtable）的抽象父类。每个键和每个值都是一个对象。在任何一个 Dictionary 对象中，每个键至多与一个值相关联。Map是"key-value键值对"接口。



      HashTable采用"拉链法"实现哈希表，它定义了几个重要的参数：table、count、threshold、loadFactor、modCount。



      table：为一个Entry[]数组类型，Entry代表了“拉链”的节点，每一个Entry代表了一个键值对，哈希表的"key-value键值对"都是存储在Entry数组中的。



      count：HashTable的大小，注意这个大小并不是HashTable的容器大小，而是他所包含Entry键值对的数量。



      threshold：Hashtable的阈值，用于判断是否需要调整Hashtable的容量。threshold的值="容量*加载因子"。



      loadFactor：加载因子。



      modCount：用来实现“fail-fast”机制的（也就是快速失败）。所谓快速失败就是在并发集合中，其进行迭代操作时，若有其他线程对其进行结构性的修改，这时迭代器会立马感知到，并且立即抛出ConcurrentModificationException异常，而不是等到迭代完成之后才告诉你（你已经出错了）。

## 构造方法

      在HashTabel中存在5个构造函数。通过这5个构造函数我们构建出一个我想要的HashTable。

    public Hashtable() {
            this(11, 0.75f);
        }
          默认构造函数，容量为11，加载因子为0.75。
    
    public Hashtable(int initialCapacity) {
            this(initialCapacity, 0.75f);
        }
          用指定初始容量和默认的加载因子 (0.75) 构造一个新的空哈希表。
    
    public Hashtable(int initialCapacity, float loadFactor) {
            //验证初始容量
            if (initialCapacity < 0)
                throw new IllegalArgumentException("Illegal Capacity: "+
                                                   initialCapacity);
            //验证加载因子
            if (loadFactor <= 0 || Float.isNaN(loadFactor))
                throw new IllegalArgumentException("Illegal Load: "+loadFactor);
    
            if (initialCapacity==0)
                initialCapacity = 1;
    
            this.loadFactor = loadFactor;
    
            //初始化table，获得大小为initialCapacity的table数组
            table = new Entry[initialCapacity];
            //计算阀值
            threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
            //初始化HashSeed值
            initHashSeedAsNeeded(initialCapacity);
        }
        
        
>  用指定初始容量和指定加载因子构造一个新的空哈希表。其中initHashSeedAsNeeded方法用于初始化hashSeed参数，其中hashSeed用于计算key的hash值，它与key的hashCode进行按位异或运算。这个hashSeed是一个与实例相关的随机值，主要用于解决hash冲突。

    private int hash(Object k) {
            return hashSeed ^ k.hashCode();
        }
        
  构造一个与给定的 Map 具有相同映射关系的新哈希表。

    public Hashtable(Map<? extends K, ? extends V> t) {
            //设置table容器大小，其值==t.size * 2 + 1
            this(Math.max(2*t.size(), 11), 0.75f);
            putAll(t);
        }
        
## 主要方法

HashTable的API对外提供了许多方法，这些方法能够很好帮助我们操作HashTable，但是这里我只介绍两个最根本的方法：put、get。

      首先我们先看put方法：将指定 key 映射到此哈希表中的指定 value。注意这里键key和值value都不可为空。

    public synchronized V put(K key, V value) {
            // 确保value不为null
            if (value == null) {
                throw new NullPointerException();
            }
    
            /*
             * 确保key在table[]是不重复的
             * 处理过程：
             * 1、计算key的hash值，确认在table[]中的索引位置
             * 2、迭代index索引位置，如果该位置处的链表中存在一个一样的key，则替换其value，返回旧值
             */
            Entry tab[] = table;
            int hash = hash(key);    //计算key的hash值
            int index = (hash & 0x7FFFFFFF) % tab.length;     //确认该key的索引位置
            //迭代，寻找该key，替换
            for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {
                if ((e.hash == hash) && e.key.equals(key)) {
                    V old = e.value;
                    e.value = value;
                    return old;
                }
            }
    
            modCount++;
            if (count >= threshold) {  //如果容器中的元素数量已经达到阀值，则进行扩容操作
                rehash();
                tab = table;
                hash = hash(key);
                index = (hash & 0x7FFFFFFF) % tab.length;
            }
    
            // 在索引位置处插入一个新的节点
            Entry<K,V> e = tab[index];
            tab[index] = new Entry<>(hash, key, value, e);
            //容器中元素+1
            count++;
            return null;
        }
        
>   put方法的整个处理流程是：计算key的hash值，根据hash值获得key在table数组中的索引位置，然后迭代该key处的Entry链表（我们暂且理解为链表），若该链表中存在一个这个的key对象，那么就直接替换其value值即可，否则在将改key-value节点插入该index索引位置处。如下：

首先我们假设一个容量为5的table，存在8、10、13、16、17、21。他们在table中位置如下：

![](http://cmsblogs.com/wp-content/uploads/2014/04/1_thumb.png)

然后我们插入一个数：put(16,22)，key=16在table的索引位置为1，同时在1索引位置有两个数，程序对该“链表”进行迭代，发现存在一个key=16,这时要做的工作就是用newValue=22替换oldValue16，并将oldValue=16返回。

![image](http://cmsblogs.com/wp-content/uploads/2014/04/2014040302_thumb.png)


 在put(33,33)，key=33所在的索引位置为3，并且在该链表中也没有存在某个key=33的节点，所以就将该节点插入该链表的第一个位置。

![image](http://cmsblogs.com/wp-content/uploads/2014/04/2014040303vsd_thumb.png)

 在HashTabled的put方法中有两个地方需要注意：

1、HashTable的扩容操作，在put方法中，如果需要向table[]中添加Entry元素，会首先进行容量校验，如果容量已经达到了阀值，HashTable就会进行扩容处理rehash()，如下:

    protected void rehash() {
            int oldCapacity = table.length;
            //元素
            Entry<K,V>[] oldMap = table;
    
            //新容量=旧容量 * 2 + 1
            int newCapacity = (oldCapacity << 1) + 1;
            if (newCapacity - MAX_ARRAY_SIZE > 0) {
                if (oldCapacity == MAX_ARRAY_SIZE)
                    return;
                newCapacity = MAX_ARRAY_SIZE;
            }
    
            //新建一个size = newCapacity 的HashTable
            Entry<K,V>[] newMap = new Entry[];
    
            modCount++;
            //重新计算阀值
            threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
            //重新计算hashSeed
            boolean rehash = initHashSeedAsNeeded(newCapacity);
    
            table = newMap;
            //将原来的元素拷贝到新的HashTable中
            for (int i = oldCapacity ; i-- > 0 ;) {
                for (Entry<K,V> old = oldMap[i] ; old != null ; ) {
                    Entry<K,V> e = old;
                    old = old.next;
    
                    if (rehash) {
                        e.hash = hash(e.key);
                    }
                    int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                    e.next = newMap[index];
                    newMap[index] = e;
                }
            }
        }
        
 在这个rehash()方法中我们可以看到容量扩大两倍+1，同时需要将原来HashTable中的元素一一复制到新的HashTable中，这个过程是比较消耗时间的，同时还需要重新计算hashSeed的，毕竟容量已经变了。
 
 这里对阀值啰嗦一下：比如初始值11、加载因子默认0.75，那么这个时候阀值threshold=8，当容器中的元素达到8时，HashTable进行一次扩容操作，容量 = 8 * 2 + 1 =17，而阀值threshold=17*0.75 = 13，当容器元素再一次达到阀值时，HashTable还会进行扩容操作，依次类推。

下面是计算key的hash值，这里hashSeed发挥了作用。

    private int hash(Object k) {
            return hashSeed ^ k.hashCode();
        }
        
 相对于put方法，get方法就会比较简单，处理过程就是计算key的hash值，判断在table数组中的索引位置，然后迭代链表，匹配直到找到相对应key的value,若没有找到返回null。
    
    public synchronized V get(Object key) {
            Entry tab[] = table;
            int hash = hash(key);
            int index = (hash & 0x7FFFFFFF) % tab.length;
            for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {
                if ((e.hash == hash) && e.key.equals(key)) {
                    return e.value;
                }
            }
            return null;
        }
        
四、HashTable与HashMap的区别

  HashTable和HashMap存在很多的相同点，但是他们还是有几个比较重要的不同点。


> 
>   第一：我们从他们的定义就可以看出他们的不同，HashTable基于Dictionary类，而HashMap是基于AbstractMap。Dictionary是什么？它是任何可将键映射到相应值的类的抽象父类，而AbstractMap是基于Map接口的骨干实现，它以最大限度地减少实现此接口所需的工作。
> 
> 
> 
>   第二：HashMap可以允许存在一个为null的key和任意个为null的value，但是HashTable中的key和value都不允许为null。如下：
> 
> 
> 
>   当HashMap遇到为null的key时，它会调用putForNullKey方法来进行处理。对于value没有进行任何处理，只要是对象都可以。
> 
    if (key == null)
                return putForNullKey(value);
          而当HashTable遇到null时，他会直接抛出NullPointerException异常信息。
    
    if (value == null) {
        throw new NullPointerException();
    }
            
>   第三：Hashtable的方法是同步的，而HashMap的方法不是。所以有人一般都建议如果是涉及到多线程同步时采用HashTable，没有涉及就采用HashMap，但是在Collections类中存在一个静态方法：synchronizedMap()，该方法创建了一个线程安全的Map对象，并把它作为一个封装的对象来返回，所以通过Collections类的synchronizedMap方法是可以我们你同步访问潜在的HashMap。这样君该如何选择呢？？？