---
layout: post
title: 'ConcurrentHashMap：源码剖析与应用场景分析'
date: 2024-03-16
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: 系统



---

> ConcurrentHashMap 是 Java 并发包（java.util.concurrent）中提供的线程安全的哈希表实现。它允许完全并发的读取，并且支持高并发环境下的更新操作。本文将深入剖析 JDK 1.7 和 JDK 1.8 两个版本的 ConcurrentHashMap，从数据结构、实现原理、初始化、put 方法、get 方法、扩容机制以及并发控制等方面进行详细分析。此外，我们还将结合实际应用场景，展示如何在高并发场景下使用 ConcurrentHashMap，并分析其优势。

# 1. 概述

ConcurrentHashMap 是 Java 并发包（java.util.concurrent）中提供的线程安全的哈希表实现。它允许完全并发的读取，并且支持高并发环境下的更新操作。本文将深入剖析 JDK 1.7 和 JDK 1.8 两个版本的 ConcurrentHashMap，从数据结构、实现原理、初始化、put 方法、get 方法、扩容机制以及并发控制等方面进行详细分析。此外，我们还将结合实际应用场景，展示如何在高并发场景下使用 ConcurrentHashMap，并分析其优势。

# 2. 数据结构

## 2.1 JDK 1.7

![]({{ '/assets/img/ConcurrentHashMap/1.png' | prepend: '' }})
![](.\img\ConcurrentHashMap\1.png)

- **Segment数组：**ConcurrentHashMap在JDK 1.7中使用了分段锁的概念，内部维护了一个Segment数组，每个Segment相当于一个ReentrantLock。
- **HashEntry链表：**每个Segment下挂载了若干个HashEntry链表，用于存储键值对。

**分析：**

Segment 数组通过将整个哈希表划分为多个独立的段（segments），每个段可以独立加锁，从而降低了锁的竞争。每个 Segment 内部使用链表来处理哈希冲突，但当链表过长时，性能会受到影响。

## 2.2 JDK 1.8

![]({{ '/assets/img/ConcurrentHashMap/2.png' | prepend: '' }})
![](.\img\ConcurrentHashMap\2.png)

- **Node数组 + 链表/红黑树：**JDK 1.8摒弃了Segment概念，直接采用Node数组，当链表长度超过一定阈值时会转换为红黑树以提高查找效率。

**分析：**

JDK 1.8 的设计更加简洁高效，取消了 Segment，改为使用 CAS 操作和同步块相结合的方式，减少了锁的竞争。同时，当链表长度超过一定阈值时会自动转换为红黑树，提高了查找效率。

# 3. 实现原理

**JDK 1.7：**

1. **当进行数据操作时，先根据哈希值定位到对应的 Segment，然后获取该 Segment 的锁，再对其内部的 HashEntry 数组进行操作。这样不同 Segment 之间的操作可以并发执行，大大提高了并发性能。**
2. **哈希冲突处理采用链表法，每个 HashEntry 节点存储键值对，通过 next 指针链接下一个冲突节点。**

**JDK 1.8：**

1. **基于 CAS 无锁算法实现部分操作，例如在插入新节点时，先尝试使用 CAS 操作将新节点设置到对应位置，如果失败则说明有其他线程同时操作，进入自旋重试或其他处理逻辑。**
2. **利用红黑树的自平衡特性保证查询、插入、删除等操作在最坏情况下的时间复杂度为 O (log n)，与链表结构结合，兼顾了空间和时间性能。**

## 3.1 初始化

### 3.1.1 JDK 1.7

初始化时，创建一个包含指定数量 Segment 的数组，默认并发级别（concurrencyLevel）为 16。每个 Segment 内的 HashEntry 数组初始大小根据初始容量和并发级别计算得出，同时设置负载因子等参数。

```java
public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    
    // 计算 Segment 数量
    int sshift = 0;
    int ssize = 1;
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;
    this.segments = Segment.newArray(ssize);
    
    // 初始化 Segments
    if (initialCapacity > 0) {
        int c = initialCapacity;
        int cap = 1;
        while (cap < c)
            cap <<= 1;
        for (int i = 0; i < this.segments.length; ++i)
            this.segments[i] = new Segment(cap, loadFactor);
    }
}

```

**分析：**

- **initialCapacity：初始容量，表示ConcurrentHashMap的初始大小。**
- **loadFactor：负载因子，表示当哈希表中的元素数量达到总容量的多少倍时，需要进行扩容。**
- **concurrencyLevel：并发级别，表示Segment的数量，即可以同时并发操作的线程数。**
- **segmentShift 和 segmentMask：用于计算某个key属于哪个Segment。**
- **segments：Segment数组，每个Segment是一个独立的哈希表。**

初始化过程中，ConcurrentHashMap 根据 concurrencyLevel 参数计算出合适的 Segment 数量，并初始化每个 Segment。concurrencyLevel 表示预计的最大并发线程数，Segment 数量会根据这个参数进行调整，以确保在高并发环境下能够有效减少锁竞争。

### 3.1.2 JDK 1.8

初始化主要是创建一个初始容量的 Node 数组，根据传入参数计算合适的容量、负载因子等。与 JDK 1.7 相比，简化了一些复杂的分段计算逻辑。

```java
public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAXIMUM_CONCURRENCY_LEVEL)
        concurrencyLevel = MAXIMUM_CONCURRENCY_LEVEL;
    
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int s = tableSizeFor(size);
    this.sizeCtl = (s < (1 << 16)) ? ((1 << 16) - 1) : (int)(size >>> 1);
}

```

**分析：**

- **initialCapacity 和 loadFactor 的含义与JDK 1.7相同。**
- **concurrencyLevel 不再直接影响Segment的数量，而是作为参数传递给构造函数。**
- **sizeCtl：控制表初始化和扩容的状态变量。**

JDK 1.8 的初始化过程相对简单，主要计算初始容量和加载因子，并设置 sizeCtl 变量用于控制扩容。由于取消了 Segment，初始化过程不需要创建多个锁对象，简化了代码逻辑。

## 3.2 put方法

### 3.2.1 JDK 1.7

1. 先根据键的哈希值定位到对应的 Segment，调用 tryLock () 尝试获取锁，如果获取成功则进入后续操作，否则执行自旋等待锁或其他阻塞策略。
2. 找到对应 Segment 内的 HashEntry 数组位置，遍历链表查找是否已存在相同键，如果存在则更新值，不存在则创建新的 HashEntry 节点插入链表头部。关键源码片段：

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int j = (hash & 0x7FFFFFFF) % s.count;
    HashEntry<K,V>[] tab = s.table;
    int i = indexFor(j, tab.length);

    // 查找是否存在相同键
    for (HashEntry<K,V> e = tab[i]; e != null; e = e.next) {
        K k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k)))) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            return oldValue;
        }
    }

    // 加锁并插入新元素
    s.lock();
    try {
        HashEntry<K,V>[] tab = s.table;
        int i = indexFor(j, tab.length);
        for (HashEntry<K,V> e = tab[i]; e != null; e = e.next) {
            K k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k)))) {
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                return oldValue;
            }
        }

        modCount++;
        HashEntry<K,V> e = tab[i];
        s.count++;
        tab[i] = new HashEntry<K,V>(hash, key, value, e);
        return null;

    } finally {
        s.unlock();
    }
}

```

**分析：**

- **hash：通过hashCode计算出的哈希值。**
- **j：确定key属于哪个Segment。**
- **tab：Segment中的HashEntry数组。**
- **i：确定key在Segment中的位置。**
- **lock：获取Segment的锁，确保写操作的原子性。**
- **modCount：记录修改次数，用于迭代器检测是否被修改。**
- **count：记录Segment中的元素数量。**

put 方法首先尝试在当前 Segment 中查找是否存在相同的键。如果找到，则根据 onlyIfAbsent 参数决定是否更新值。如果没有找到，则加锁后重新检查并插入新元素。这种设计保证了线程安全性，但每次插入都需要获取锁，可能会导致性能瓶颈。

### 3.2.2 JDK 1.8

1. 根据键的哈希值找到对应的 Node 数组位置，先通过自旋结合 CAS 操作尝试插入新节点，如果成功则直接返回旧值（如果是更新操作）或 null（新插入）。
2. 若 CAS 失败，说明有冲突，判断当前位置节点类型，如果是链表，遍历链表插入节点，过程中可能触发链表转红黑树操作；如果是红黑树，直接在红黑树上插入节点。关键源码片段：

```java
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
                break;                   // 无锁插入到空桶
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}

```

**分析：**

- **spread：扩展哈希值，减少哈希冲突。**
- **tab：Node数组。**
- **casTabAt：CAS操作，确保在无竞争的情况下插入新节点。**
- **MOVED：表示正在迁移。**
- **helpTransfer：帮助其他线程完成迁移。**
- **synchronized (f)：锁定当前节点，确保写操作的原子性。**
- **treeifyBin：当链表长度超过阈值时，转换为红黑树。**

JDK 1.8 的 put 方法使用了 CAS 操作和同步块相结合的方式，减少了锁的竞争。对于空桶的情况，可以直接使用 CAS 操作插入新元素，无需加锁。只有在链表或红黑树中插入元素时才会使用同步块，大大提高了并发性能。

## 3.3 get方法

### 3.3.1 JDK 1.7

1. 根据键的哈希值定位到对应的 Segment，由于读操作不需要修改数据结构，所以不需要获取锁。
2. 在 Segment 内的 HashEntry 数组中遍历链表查找对应键的值，如果找到则返回，未找到返回 null。关键源码：

```java
V get(Object key, int hash) {
    if (count != 0) { // 读取 volatile
        HashEntry<K,V> e = getFirst(hash);
        while (e != null) {
            if (e.hash == hash && key.equals(e.key))
                return e.value;
            e = e.next;
        }
    }
    return null;
}

```

**分析：**

- **count：检查Segment中的元素数量。**
- **getFirst：获取第一个HashEntry。**
- **hash：比较哈希值和键值。**

get 方法通过遍历链表查找键值对，不涉及任何锁操作，因此读取操作是完全并发的。这使得 ConcurrentHashMap 在高并发读取场景下具有很高的性能。

### 3.3.2 JDK 1.8

1. 根据键的哈希值定位到 Node 数组位置，遍历链表或红黑树查找对应键的值，由于读操作无锁且基于 volatile 修饰的节点引用保证可见性，所以多个线程可以同时读。关键源码：

```java
final V get(Node<K,V>[] tab, int h, Object k) {
    Node<K,V> e; K u; V v;
    int n = tab.length;
    if ((e = tabAt(tab, (n - 1) & h)) != null) {
        if ((k == (u = e.key) || (u != null && k.equals(u))) &&
            (v = e.val) != null)
            return v;
        synchronized (e) {
            if (tabAt(tab, (n - 1) & h) == e) {
                if (e.hash >= 0) {
                    for (Node<K,V> d = e.next; d != null; d = d.next) {
                        if (d.key == k || (d.key != null && k.equals(d.key))) {
                            v = d.val;
                            break;
                        }
                    }
                }
                else if (e instanceof TreeBin) {
                    Node<K,V> p;
                    if ((p = ((TreeBin<K,V>)e).getTreeNode(h, k)) != null)
                        v = p.val;
                }
            }
        }
    }
    return v;
}

```

**分析：**

- **tab：Node数组。**
- **h：哈希值。**
- **k：键。**
- **synchronized (e)：锁定当前节点，确保读操作的一致性。**
- **TreeBin：处理红黑树的情况。**

JDK 1.8 的 get 方法同样不涉及锁操作，直接通过 CAS 操作读取数据。对于链表或红黑树中的节点，使用同步块进行访问，确保线程安全。这种方式在高并发读取场景下表现优异。

## 3.4 扩容

### 3.4.1 JDK 1.7

当某个 Segment 内的元素数量超过阈值（负载因子 * 桶数组长度）时，会触发该 Segment 的扩容操作。扩容过程是创建一个容量翻倍的新 HashEntry 数组，然后将原数组中的元素重新哈希到新数组中，这个过程中会持有 Segment 的锁，保证数据一致性。关键源码涉及 rehash 方法，较为复杂，这里不完整展示，其核心是遍历原链表，根据新的哈希值将节点插入新数组合适位置。

扩容通过增加 Segment 的数量来实现，每次扩容后容量变为原来的两倍。扩容过程中需要对所有 Segment 进行重新分配，可能导致较大的停顿时间。

### 3.4.2 JDK 1.8

采用多线程协助扩容机制，当某个线程发现需要扩容时，会先尝试帮助其他正在扩容的线程完成部分工作，然后再处理自己负责的桶。扩容过程中通过标记位（如 MOVED 状态）等协调多线程操作，保证数据在扩容期间的一致性。关键源码涉及 transfer 方法，同样复杂，核心是按步长将原数组元素迁移到新数组。

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // initializing
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<?,?>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    ThresholdControl tc = null;
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
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit)
                                lastRun = p;
                        }
                        if (lastRun.hash & n == 0) {
                            setTabAt(nextTab, i, lastRun);
                            Node<K,V> ln = lastRun.next;
                            setTabAt(nextTab, i + n, f);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else {
                            setTabAt(nextTab, i + n, lastRun);
                            setTabAt(nextTab, i, f);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        if (lo != null && (lc <= UNTREEIFY_THRESHOLD || (lc >= MIN_TREEIFY_CAPACITY && tab == null))) {
                            setTabAt(nextTab, i, untreeify(lo));
                        }
                        else {
                            setTabAt(nextTab, i, (lc == 0) ? null : (lc == 1) ? lo : t);
                            if (hi != null)
                                ((TreeBin<K,V>) setTabAt(nextTab, i + n, (hc == 0) ? null : (hc == 1) ? hi : t)).treeify(tab);
                        }
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}

```

**分析：**

- **transfer：负责扩容操作。**
- **stride：步长，用于划分迁移任务。**
- **nextTab：新的Node数组。**
- **ForwardingNode：用于标记正在迁移的桶。**
- **synchronized (f)：锁定当前节点，确保迁移操作的原子性。**
- **TreeBin：处理红黑树的情况。**

JDK 1.8 的扩容机制采用了渐进式扩容策略，通过多线程协作完成扩容任务，减少了停顿时间。扩容过程中，旧表中的元素会被逐步迁移到新表中，确保在扩容期间仍能正常进行读写操作。

# 4. 并发控制机制

## 4.1 JDK 1.7

**分段锁：**使用 Segment 数组中的 ReentrantLock 进行分段锁定，降低了锁粒度，提高了并发性能。

通过分段锁，将整个哈希表划分为多个独立的 Segment，不同 Segment 的读写操作互不干扰，只要多个线程操作不同的 Segment 就能并行执行，大大减少了锁竞争，提高并发效率。但对于同一个 Segment 内的操作，仍需获取锁来保证线程安全。

## 4.2 JDK 1.8

**CAS操作与synchronized结合：**取消了 Segment，改为 CAS 操作与 synchronized 结合的方式，进一步减少了锁的竞争，提升了并发性能。

1. 读操作基本无锁，依靠 volatile 关键字保证可见性，多个线程可以并发读。
2. 写操作利用 CAS 实现乐观锁，减少不必要的锁开销，在冲突发生时配合自旋和锁（如链表操作的头节点锁）保证数据一致性，同时引入红黑树优化结构，减少锁粒度，进一步提升并发性能。

# 5. 应用场景分析

## 5.1缓存系统

在分布式系统或单机应用中，缓存是提高性能的重要手段。缓存系统需要频繁地进行读写操作，并且必须保证线程安全。

```java
import java.util.concurrent.ConcurrentHashMap;

public class CacheService {
    private final ConcurrentHashMap<String, String> cache = new ConcurrentHashMap<>();

    public String get(String key) {
        return cache.get(key);
    }

    public void put(String key, String value) {
        cache.put(key, value);
    }

    public void remove(String key) {
        cache.remove(key);
    }
}

```

**使用原因分析：**

- 高效并发读写：ConcurrentHashMap 支持多线程并发读写操作，极大提升了在高并发环境下的性能。
- 低锁开销：通过分段锁（JDK 1.7）或 CAS 操作与同步块结合（JDK 1.8），减少了锁的竞争，降低了锁开销。
- 线程安全：所有操作都是线程安全的，无需额外加锁，简化了开发和维护。

## 5.2 频率统计

在日志分析、用户行为跟踪等场景中，需要对某些事件的发生频率进行统计。例如，统计每个用户的访问次数或某个页面的点击次数。

```java
import java.util.concurrent.ConcurrentHashMap;

public class FrequencyCounter {
    private final ConcurrentHashMap<String, Integer> frequencyMap = new ConcurrentHashMap<>();

    public void recordEvent(String event) {
        frequencyMap.compute(event, (key, oldValue) -> oldValue == null ? 1 : oldValue + 1);
    }

    public int getFrequency(String event) {
        return frequencyMap.getOrDefault(event, 0);
    }
}


```

**使用原因分析：**

- 原子更新：compute 方法提供了原子更新功能，确保在高并发环境下不会出现数据不一致的问题。
- 高效并发读写：ConcurrentHashMap 的设计使得多个线程可以同时读取和更新不同键值对，提高了统计效率。
- 线程安全：所有操作都是线程安全的，无需额外加锁，简化了开发和维护。

## 5.3 会话管理

在 Web 应用中，通常需要管理用户的会话信息。每个用户的会话信息存储在一个独立的键值对中，要求能够快速查找和更新。

```java
import java.util.concurrent.ConcurrentHashMap;

public class SessionManager {
    private final ConcurrentHashMap<String, UserSession> sessionMap = new ConcurrentHashMap<>();

    public void createSession(String sessionId, UserSession session) {
        sessionMap.put(sessionId, session);
    }

    public UserSession getSession(String sessionId) {
        return sessionMap.get(sessionId);
    }

    public void invalidateSession(String sessionId) {
        sessionMap.remove(sessionId);
    }
}


```

**使用原因分析：**

- 高效并发读写：ConcurrentHashMap 支持多线程并发读写操作，适合处理大量用户的会话信息。
- 线程安全：所有操作都是线程安全的，确保在高并发环境下不会出现数据不一致的问题。
- 快速查找：基于哈希表的实现提供了 O(1) 的查找时间复杂度，能够快速获取会话信息。

## 5.4 动态配置管理

在微服务架构中，动态配置管理是一个常见的需求。配置项需要在运行时根据不同的条件进行更新，并且要保证线程安全。

```java
import java.util.concurrent.ConcurrentHashMap;

public class ConfigManager {
    private final ConcurrentHashMap<String, String> configMap = new ConcurrentHashMap<>();

    public void updateConfig(String key, String value) {
        configMap.put(key, value);
    }

    public String getConfig(String key) {
        return configMap.get(key);
    }
}


```

**使用原因分析：**

- 高效并发读写：ConcurrentHashMap 支持多线程并发读写操作，适合处理动态配置的频繁更新。
- 线程安全：所有操作都是线程安全的，确保在高并发环境下不会出现数据不一致的问题。
- 快速查找：基于哈希表的实现提供了 O(1) 的查找时间复杂度，能够快速获取配置项。

## 5.5 数据共享与协作

在多线程协作任务中，多个线程可能需要共享一些中间结果或状态信息。ConcurrentHashMap 可以作为共享数据结构，确保数据的一致性和安全性。

```java
import java.util.concurrent.ConcurrentHashMap;

public class TaskCoordinator {
    private final ConcurrentHashMap<String, Object> sharedData = new ConcurrentHashMap<>();

    public void shareData(String key, Object data) {
        sharedData.put(key, data);
    }

    public Object getData(String key) {
        return sharedData.get(key);
    }
}


```

**使用原因分析：**

- 高效并发读写：ConcurrentHashMap 支持多线程并发读写操作，适合处理多个线程之间的数据共享。
- 线程安全：所有操作都是线程安全的，确保在多线程环境下不会出现数据不一致的问题。
- 灵活扩展：可以根据实际需求灵活添加或移除共享数据项，适应不同的协作场景。

# 6. 总结

Java7 中 `ConcurrentHashMap` 使用的分段锁，也就是每一个 Segment 上同时只有一个线程可以操作，每一个 `Segment` 都是一个类似 `HashMap` 数组的结构，它可以扩容，它的冲突会转化为链表。但是 `Segment` 的个数一但初始化就不能改变。

Java8 中的 `ConcurrentHashMap` 使用的 `Synchronized` 锁加 CAS 的机制。结构也由 Java7 中的 `**Segment**` 数组 + `**HashEntry**` 数组 + 链表 进化成了 Node 数组 + 链表 / 红黑树，Node 是类似于一个 HashEntry 的结构。它的冲突再达到一定大小时会转化成红黑树，在冲突小于一定数量时又退回链表。





<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>