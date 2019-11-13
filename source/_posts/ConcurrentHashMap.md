---
title: ConcurrentHashMap
date: 2019-11-13 21:19:02
author: Chopin
tags: #标签
    - 并发编程
---

# 前言
在JUC包体现了许多Doug Lee的并发设计理念，有许多在我们看来十分难懂且在我们编码过程中很少使用的并发设计，十分精妙。虽然在我们看来没有必要如此极限的压榨CPU的性能，但是学习其中的一些并发思路也是有一定必要的。这篇文章对JDK7和8中Doug Lee对ConcurrentHashMap并发安全的设计进行分析。
<!-- more -->

# HashMap

众所周知，HashMap是线程不安全的，并不单单只是JDK8之前扩容时可能出现环链的问题，而是他数据结构的设计没有考虑并发环境，没有采用任何措施保证put、remove操作的线程安全。

## 环链问题

JDK7版本中，HashMap仍存在环链，数据结构采用数组+链表的方式

![](/png/CHM/164c47f32e1066e8.png)

在添加元素时，当数组大小超过扩容阈值且插入的位置已经有元素存在时，会触发数组扩容

```java
	void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        // 遍历旧的数组
        for (Entry<K,V> e : table) {
            // 遍历旧的链表
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                // 计算元素在新的集合里的位置
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }
```

单线程情况下，正常的rehash过程：

1.遍历旧数组

2.遍历旧链表

3.头插法迁移数据

![](/png/CHM/36a01be9fbb625c0bed24bc5aff5ab01.png)

处理key为3的元素，计算元素在新数组中的下标，然后将元素插入到新下标链表的头部

![](/png/CHM/92b2a9045aba0224585a552d211d61b8.png)

一次正常的rehash完后，HashMap存储情况应该是



![](/png/CHM/4.png)

但是在并发情况下，如果两个线程同时进入transfer方法

假如对于线程1，执行到`Entry<K,V> next = e.next;`线程挂起

![](/png/CHM/5.png)

线程2rehash完毕后，线程1再继续执行

![](/png/CHM/6.png)

此时对于线程1，e是key为3的元素，next是key为7的元素，在扩容时就会出现

![](/png/CHM/7.png)

在两个元素之间陷入死循环。

**多线程put时出现resize，由于使用头部插入法，可能会导致链表闭环，从而CPU占用率达到100%。** 

在JDK8中HashMap对扩容的方法做了修改，将原来链表上的节点插入新链表的尾部，不改变链表中元素的顺序，避免了扩容时出现环链问题。

```java
for (int binCount = 0; ; ++binCount) {
    if ((e = p.next) == null) {
		p.next = newNode(hash, key, value, null);
		if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
			treeifyBin(tab, hash);
		break;
	}
    ...
}
```

# ConcurrentHashMap

与HashMap相比，虽然ConcurrentHashMap仅仅只增加了并发特性，但是其复杂度却极大的上升了。因为考虑到并发性能，它没有像Hashtable一样简单的给每个公有方法加上synchronize，而是利用了JUC包中提供的多种并发特性，在尽量保持性能的前提下实现了多线程安全。 

## JDK1.7

HashTable 容器在竞争激烈的并发环境下表现出效率低下的原因是所有访问 HashTable 的线程都必须竞争同一把锁，那假如容器里有多把锁，每一把锁用于锁容器其中一部分数据，那么当多线程访问容器里不同数据段的数据时，线程间就不会存在锁竞争，从而可以有效的提高并发访问效率，这就是JDK1.7中 ConcurrentHashMap 所使用的锁分段技术。

### Segment

ConcurrentHashMap维护了一个Segment数组，Segment这个类继承了重入锁ReentrantLock，并且该类里面维护了一个 HashEntry<K,V>[] table数组，在写操作put，remove，扩容的时候，会对Segment加锁，所以仅仅影响这个Segment，不同的Segment还是可以并发的，所以解决了线程的安全问题，也提升了并发的效率，在最理想的情况下，ConcurrentHashMap可以最高同时支持Segment数量大小的写操作。 

ConcurrentHashMap底层采用HashEntry存储数据，与JDK1.7的HashMap一样采用数组+链表的数据结构

![](/png/CHM/ConcurrentHashMap数据结构.jpg)

和 HashMap 非常类似，唯一的区别就是其中的核心数据如 value ，以及链表都是 volatile 修饰的，保证了获取时的可见性。 

```java
static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;

    HashEntry(int hash, K key, V value, HashEntry<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
}
```

Segment 是 ConcurrentHashMap 的一个内部类，主要的组成如下： 

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {

	private static final long serialVersionUID = 2249069246763182397L;
        
	// 和 HashMap 中的 HashEntry 作用一样
	transient volatile HashEntry<K,V>[] table;
	
    // Segment对象包含的HashEntry的个数
	transient int count;

    // table被更新的次数
    transient int modCount;

    // 扩容阈值
    transient int threshold;

    // 负载因子
    final float loadFactor;
}
```

ConcurrentHashMap初始化过程通过initialCapacity、loadFactor、concurrencyLevel参数来初始化segmentShift、segmentMask和segment数组。 

```java
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    // segment数组长度ssize是由concurrencyLevel计算得出，当ssize < concurrencyLevel时，ssize *= 			2，至于为什么一定要保证ssize是2的N次方是为了可以通过按位与来定位segment；
    int sshift = 0;
    int ssize = 1;
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    // segmentShift和segmentMask在定位segment使用，segmentShift = 32 - ssize向右移位的次数，			segmentMask = ssize - 1。ssize的最大长度是65536，对应的 segmentShift最小值为16，				segmentMask最大值是65535，对应的二进制16位全1；
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c)
        cap <<= 1;
    // create segments and segments[0]
    // HashEntry长度cap同样也是2的N次方，默认情况，ssize = 16，initialCapacity = 16，loadFactor = 		0.75f，那么cap = 1，threshold = (int) cap * loadFactor = 0。
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```

#### 定位

并发级别的默认值为DEFAULT_CONCURRENCY_LEVEL=16。segmentShift是用来计算segments数组索引的位移量，而segmentMask则是用来计算索引的掩码值。例如并发度为16时（即segments数组长度为16），segmentShift为32-4=28（因为2的4次幂为16），而segmentMask则为1111（二进制）。

ConcurrentHashMap将hashCode进行位运算来定位具体的segment： 

```java
	private Segment<K,V> segmentForHash(int h) {
        // 使用索引值(x<<SSHIFT) + SBASE计算出segments中相应Segment的地址，最后使用UNSAFE.getObjectVolatile(segments, u)取出相应的Segment
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        return (Segment<K,V>) UNSAFE.getObjectVolatile(segments, u);
    }
```

**volatile的数组只针对数组的引用具有volatile的语义，而不是它的元素**。 通过Unsafe.getObjectVolatile可以直接获取指定内存的数据，保证了每次拿到数据都是最新的。 

#### 再散列

ConcurrentHashMap使用Segment来保护数据，在插入和读取数据之前，需要先通过hash算法来定位Segment。ConcurrentHashMap使用了变种hash算法对元素的hashCode再散列。 

```java
	private int hash(Object k) {
        int h = hashSeed;

        if ((0 != h) && (k instanceof String)) {
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();

        // Spread bits to regularize both segment and index locations,
        // using variant of single-word Wang/Jenkins hash.
        h += (h <<  15) ^ 0xffffcd7d;
        h ^= (h >>> 10);
        h += (h <<   3);
        h ^= (h >>>  6);
        h += (h <<   2) + (h << 14);
        return h ^ (h >>> 16);
    }
```

再散列的目的是为了扰乱hash值的高位和低位，减少冲突，让元素可以近似均匀的分布在不同的Segment上，从而提升存储效率。如果hash算法不好，最差的情况是所有的元素都在一个Segment中，这时候hash表将退化成链表，查询插入的时间复杂度都会从理想的O(1)退化成O(n^2)，同时，分段锁也会失去存在的意义。

### get

由于 HashEntry 中的 value 属性是用 volatile 关键词修饰的，保证了内存可见性，所以每次获取时都是最新值，且get的时候无需加锁。

```java
	public V get(Object key) {
        Segment<K,V> s; // manually integrate access methods to reduce overhead
        HashEntry<K,V>[] tab;
        // 计算键对应的散列值，找到对应的segment
        int h = hash(key);
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
            // 根据hashTable的索引(tab.length-1)&hash,找到对应table的位置
            for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                     (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    return e.value;
            }
        }
        return null;
    }
```

### put

```java
	public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        int j = (hash >>> segmentShift) & segmentMask;
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            s = ensureSegment(j);
        return s.put(key, hash, value, false);
    }
```

首先是通过 key 定位到 Segment，之后在对应的 Segment 中进行具体的 put操作。 

```java
		final V put(K key, int hash, V value, boolean onlyIfAbsent) {
            // 尝试获取锁，如果获取失败，则利用scanAndLockForPut保证获取到锁
            HashEntry<K,V> node = tryLock() ? null :
                scanAndLockForPut(key, hash, value);    
            V oldValue;
            try {
                ...
                }
            } finally {
                // 释放当前segment持有的锁
                unlock();
            }
            return oldValue;
        }
```

```java
		private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
            HashEntry<K,V> first = entryForHash(this, hash);
            HashEntry<K,V> e = first;
            HashEntry<K,V> node = null;
            int retries = -1; 
            while (!tryLock()) {   // 尝试获取锁
                HashEntry<K,V> f; 
                    if (e == null) {
                        if (node == null) 
                            node = new HashEntry<K,V>(hash, key, value, null);
                        retries = 0;
                    }
                    else if (key.equals(e.key))
                        retries = 0;
                    else
                        e = e.next;
                }
            	// 如果重试次数达到MAX_SCAN_RETRIES，改为阻塞获取锁
                else if (++retries > MAX_SCAN_RETRIES) {
                    lock();
                    break;
                }
                else if ((retries & 1) == 0 &&
                         (f = entryForHash(this, hash)) != first) {
                    e = first = f; // re-traverse if entry changed
                    retries = -1;
                }
            }
            return node;
        }
```

首先调用tryLock，如果加锁失败，则进入scanAndLockForPut(key, hash, value)方法，该方法实际上是先自旋等待其他线程解锁，直至指定的次数MAX_SCAN_RETRIES。若自旋过程中，其他线程释放了锁，导致本线程直接获得了锁，就避免了本线程进入等待锁的场景，提高了效率。若自旋一定次数后，仍未获取锁，则调用lock方法进入等待锁的场景。 

采用这种**自旋锁和独占锁结合**的方法，能够提高Segment并发操作数据的效率。 

## JDK1.8

JDK1.8版本舍弃了segment，并且大量使用了synchronized，以及CAS无锁操作以保证ConcurrentHashMap操作的线程安全性。不用ReentrantLock而是Synchronzied是因为synchronzied做了很多的优化，包括偏向锁，轻量级锁，重量级锁，可以依次向上升级锁状态。另外，底层数据结构改变为采用数组+链表+红黑树的数据形式。

![](D:\Desktop\分享\HashMap&ConcurrentHashMap\png\JDK8ConcurrentHashMap数据结构.jpg)

和HashMap类似，ConcurrentHashMap使用了一个table来存储Node，ConcurrentHashMap同样使用记录的key的hashCode来寻找记录的存储index，而处理哈希冲突的方式与HashMap也是类似的，冲突的记录将被存储在同一个位置上，形成一条链表，当链表的长度等于8的时候会将链表转化为一棵红黑树，从而将查找的复杂度从O(N)降到了O(lgN)。

### initTable

```java
	private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

在初始化Node数组的时候，有一个重要的属性sizeCtl

```java
	/**
     * -1 :代表table正在初始化,其他线程应该交出CPU时间片
     * -N: 表示正有N-1个线程执行扩容操作（高 16 位是 length 生成的标识符，低 16 位是扩容的线程数）
     */
    private transient volatile int sizeCtl;
```

sizeCtl是一个用于同步多个线程的共享变量，如果它的当前值为负数，则说明table正在被某个线程初始化或者扩容，所以，如果某个线程想要初始化table，需要去竞争sizeCtl这个共享变量，获得变量的线程才有许可去进行接下来的操作，没能获得的线程将会一直自旋来尝试获得这个共享变量，所以获得sizeCtl这个变量的线程在完成工作之后需要设置回来，使得其他的线程可以走出自旋进行接下来的操作。

```java
		while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            ...
        }	
```

而在initTable方法中我们可以看到，当线程发现sizeCtl小于0的时候，他就会让出CPU时间，而稍后再进行尝试，当发现sizeCtl不再小于0的时候，就会通过调用方法compareAndSwapInt来将sizeCtl共享变量变为-1,`U.compareAndSwapInt(this, SIZECTL, sc, -1)`，以告诉其他试图获得sizeCtl变量的线程，目前正在由本线程在享用该变量，在完成初始化table的任务之后，线程需要将sizeCtl设置成可以使得其他线程获得变量的状态。

```java
	finally {
		sizeCtl = sc;
	}
```

这其中还有一个地方需要注意，就是在某个线程通过U.compareAndSwapInt方法设置了sizeCtl之前和之后进行了两次check，来检测table是否被初始化过了，这种检测是必须的，因为在并发环境下，可能前一个线程正在初始化table但是还没有成功初始化，也就是table依然还为null，而有一个线程发现table为null他就会进行竞争sizeCtl以进行table初始化，但是当前线程在完成初始化之后，那个试图初始化table的线程获得了sizeCtl，但是此时table已经被初始化了，所以，如果没有再次判断的话，可能会将之后进行put操作的线程的更新覆盖掉，这是极为不安全的行为。

### put

ConcurrentHashMap采用CAS+synchronized实现并发插入或更新操作。

```java
	/** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            ...
        }
        addCount(1L, binCount);
        return null;
    }
```

#### hash定位

```java
	static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }

	index = (n - 1) & hash // 通过位运算获取索引
```

hash算法相比之前版本简化会带来弊端，哈希冲突加剧，因此在链表节点数量大于等于8时，会将链表转化为红黑树进行存储。 

#### Unsafe+CAS

ConcurrentHashMap定义了三个数组元素的原子操作，用于对指定内存位置的节点进行操作，保证了并发操作的可见性以及禁止指令重排序。

```java
	@SuppressWarnings("unchecked")
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }

    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```

 如果索引位置为null，说明table中这个位置第一次插入元素，利用Unsafe.compareAndSwapObject方法插入Node节点，如果当前线程的值与内存中的值不相等，不进行修改，防止覆盖其他线程的修改结果。

```java
		else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
             if (casTabAt(tab, i, null,
                     new Node<K,V>(hash, key, value, null)))
                 break;                   // no lock when adding to empty bin
             }
```

如果CAS失败，说明有其它线程提前插入了节点，自旋重新尝试在这个位置插入节点。

#### helpTransfer()

```java
else if ((fh = f.hash) == MOVED)
	tab = helpTransfer(tab, f);

static final int MOVED     = -1; // hash for forwarding nodes
```

ForwardingNode是一个用于连接旧表和扩容的表的节点类。它包含一个nextTable指针，用于指向新表。而且这个节点的key、value、next指针全部为null，它的hash值为-1 

```java
	static final class ForwardingNode<K,V> extends Node<K,V> {
        final Node<K,V>[] nextTable;
        ForwardingNode(Node<K,V>[] tab) {
            super(MOVED, null, null, null);
            this.nextTable = tab;
        }
    }
```

线程进行扩容时，会把自己处理区间内的链表/树的首节点设为ForwardingNode，插入时判断链表头节点是ForwardingNode时，说明要插入的链表有其他线程正在进行扩容，当前线程于是协助进行扩容。

```java
	/**
     * Helps transfer if a resize is in progress.
     */
    final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            // 表长度标识
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                // 扩容结束的判断条件（这里有bug）
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                // 如果以上都不是, 将 sizeCtl + 1（表示增加了一个线程帮助其扩容）
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
```

#### 插入节点

插入节点的代码简化如下

```java
synchronized (f) {
	// 节点插入之前，再次利用tabAt(tab, i) == f判断，防止被其它线程修改。
	if (tabAt(tab, i) == f) {
		// 如果f.hash >= 0，说明f是链表结构的头节点
		if (fh >= 0) {
			binCount = 1;
			// 遍历链表，如果找到对应的node节点，则修改value，否则在链表尾部加入节点。
		}
        // 如果f是TreeBin类型节点，说明f是红黑树根节点，在树结构上遍历元素，更新或增加节点。
		else if (f instanceof TreeBin) {
			binCount = 2;
			...
		}
	}
}
if (binCount != 0) {
    // 如果链表中节点数binCount >= TREEIFY_THRESHOLD(默认是8)，则把链表转化为红黑树结构。
	if (binCount >= TREEIFY_THRESHOLD)
		treeifyBin(tab, i);
	break;
}
```

### transfer()

因为ConcurrentHashMap的扩容是高度并发的，十分复杂，精简后可分为4个步骤：

1. 计算每个线程可以处理的桶区间，最小 16。
2. 初始化临时变量 nextTable，扩容 2 倍。
3. 死循环，计算下标。完成总体判断。
4. 如果桶内有数据，同步转移数据。

```java
	/**
     * Moves and/or copies the nodes in each bin to new table. See
     * above for explanation.
     */
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        // 计算线程可以处理的桶大小
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; 
        // 初始化nextTab，设置扩容表为原表2倍大小
        if (nextTab == null) {            
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            // nextTable是扩容时用到的volatile变量，保证线程间可见
            nextTable = nextTab;
            // transferIndex初始化为旧表长度
            transferIndex = n;
        }
        int nextn = nextTab.length;
        // 创建ForwardingNode，指针指向扩容表
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                // 这里是关键，通过CAS确定线程的处理区间，bound是区间的下界
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                // 完成扩容，将nextTable置空，更新table，table也是volatile变量
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                // 尝试将sc减1，说明当前线程协助扩容结束
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    // 如果sc-2等于标识符左移16位，说明扩容结束了
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            // 如果首节点是Null，通过CAS设为Fwd
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            // 如果首节点是Fwd，说明当前节点已经有其他线程完成扩容处理
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            // 需要当前线程进行扩容处理
            else {
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            ...
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                            ...
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                }
            }
        }
    }
```

ConcurrentHashMap实现方式实际上还是，将整个表拆分成小区间，让线程并发进行处理

![](/png/CHM/并发扩容.png)

#### sizeCtl

判断是否所有线程扩容完成，通过sizeCtl来判断

![](/png/CHM/sizeCtl.png)

低16位保存协助扩容的线程数，默认第一个线程设置 sc等于rs 左移16位加2，rs是表长度的标识，第一个线程进行扩容时，会将sc+2

```java
	else if (U.compareAndSwapInt(this, SIZECTL, sc,
                 (rs << RESIZE_STAMP_SHIFT) + 2))
		transfer(tab, null);
```

当有一个线程协助扩容，就将sc+1

```java
if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
	transfer(tab, nextTab);
    ...
}
```

 当一个线程结束扩容时，会将sizeCtl-1

```java
if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
	if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
		return;
	finishing = advance = true;
	i = n; // recheck before commit
}
```

 如果sc-2等于标识符左移16位，说明所有线程结束扩容，扩容完成

### get

JDK1.8中ConcurrentHashMap的get操作相对1.7版本要复杂一些，因为链表长度大于等于8时，会将链表转化为红黑树，因此在对红黑树进行遍历时，需要考虑如果存在并发写的情况，不能按红黑树的方式进行查找

```java
		static final int WRITER = 1; // set while holding write lock
        static final int WAITER = 2; // set when waiting for write lock
        static final int READER = 4; // increment value for setting read lock				

		final Node<K,V> find(int h, Object k) {
            if (k != null) {
                for (Node<K,V> e = first; e != null; ) {
                    int s; K ek;
                    // 如果存在并发写，按遍历链表的方式查找
                    if (((s = lockState) & (WAITER|WRITER)) != 0) {
                        if (e.hash == h &&
                            ((ek = e.key) == k || (ek != null && k.equals(ek))))
                            return e;
                        e = e.next;
                    }
                    // 如果只是加了读锁，按红黑树方式进行查找
                    else if (U.compareAndSwapInt(this, LOCKSTATE, s,
                                                 s + READER)) {
                        TreeNode<K,V> r, p;
                        try {
                            p = ((r = root) == null ? null :
                                 r.findTreeNode(h, k, null));
                        } finally {
                            Thread w;
                            if (U.getAndAddInt(this, LOCKSTATE, -READER) ==
                                (READER|WAITER) && (w = waiter) != null)
                                LockSupport.unpark(w);
                        }
                        return p;
                    }
                }
            }
            return null;
        }
```

Node本身可以链成一个链表，而TreeBin和TreeNode也继承自Node节点，也自然继承了next属性，同样拥有链表的性质，其实真正在存储时，红黑树仍然是以链表形式存储的，只是逻辑上TreeBin和TreeNode多了支持红黑树的root，first, parent，left，right，red属性，在附加的属性上进行逻辑上的引用和关联，也就构造成了一颗树。