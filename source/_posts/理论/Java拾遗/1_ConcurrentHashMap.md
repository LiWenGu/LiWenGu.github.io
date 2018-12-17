---
title: ConcurrentHashMap
date: 2018-04-12 23:00:00
updated: 2018-04-13 01:00:00
comments: true
categories: 
- Java 拾遗
permalink: A2B_Java/1_ConcurrentHashMap.html    
---

# java7 的 ConcurrentHashMap

在构造函数就初始化了 Segment  
使用 Segment 可重入分段锁 + 链表结构的 HashEntry

## Segment

 ```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {

        private static final long serialVersionUID = 2249069246763182397L;
        // 如果是多处理器则是 64，否则是 1 次
        static final int MAX_SCAN_RETRIES =
            Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;

        final V put(K key, int hash, V value, boolean onlyIfAbsent) {
            // put 的时候，通过 scanAndLockForPut 自旋锁，for 循环 64/1 次尝试获取锁，如果一直没获取到，则 lock() 将自己挂起，然后等待 put 完之后的 unlock() 将自己唤醒
            HashEntry<K,V> node = tryLock() ? null :
                scanAndLockForPut(key, hash, value);
            V oldValue;
            try {
                HashEntry<K,V>[] tab = table;
                int index = (tab.length - 1) & hash;
                HashEntry<K,V> first = entryAt(tab, index);
                for (HashEntry<K,V> e = first;;) {
                    if (e != null) {
                        K k;
                        if ((k = e.key) == key ||
                            (e.hash == hash && key.equals(k))) {
                            oldValue = e.value;
                            if (!onlyIfAbsent) {
                                e.value = value;
                                ++modCount;
                            }
                            break;
                        }
                        e = e.next;
                    }
                    else {
                        if (node != null)
                            node.setNext(first);
                        else
                            node = new HashEntry<K,V>(hash, key, value, first);
                        int c = count + 1;
                        if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                            rehash(node);
                        else
                            setEntryAt(tab, index, node);
                        ++modCount;
                        count = c;
                        oldValue = null;
                        break;
                    }
                }
            } finally {
                unlock();
            }
            return oldValue;
        }
}
public int size() {
        final Segment<K,V>[] segments = this.segments;
        int size;
        boolean overflow; 
        long sum;         
        long last = 0L;   
        int retries = -1;
        try {
            // 获取 size 时，先进行三次无锁 CAS 的合并计算，如果前后两次相同则返回结果，如果前后两次都不一样则再对每个 Segment 加锁后获取 size 进行合并
            for (;;) {
                if (retries++ == RETRIES_BEFORE_LOCK) {
                    for (int j = 0; j < segments.length; ++j)
                        ensureSegment(j).lock();
                }
                sum = 0L;
                size = 0;
                overflow = false;
                for (int j = 0; j < segments.length; ++j) {
                    Segment<K,V> seg = segmentAt(segments, j);
                    if (seg != null) {
                        sum += seg.modCount;
                        int c = seg.count;
                        if (c < 0 || (size += c) < 0)
                            overflow = true;
                    }
                }
                if (sum == last)
                    break;
                last = sum;
            }
        } finally {
            if (retries > RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    segmentAt(segments, j).unlock();
            }
        }
        return overflow ? Integer.MAX_VALUE : size;
    }
 ```

 # java8 的 ConcurrentHashMap

在 put 的时候构造 Node  
使用 Node + CAS + Synchronized

## Node

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
    /**
     * Virtualized support for map.get(); overridden in subclasses.
     */
    Node<K,V> find(int h, Object k) {
        Node<K,V> e = this;
        if (k != null) {
            do {
                K ek;
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
            } while ((e = e.next) != null);
        }
        return null;
    }
}

public V put(K key, V value) {
    return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 初始化 Node
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 如果该 Node 节点未初始化则通过 CAS 插入数据
            if (casTabAt(tab, i, null,
                            new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            // 如果该 Node 下有数据，进行链表插入则使用 synchronized
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
                    // 如果该节点是红黑树结构
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
                // 判断该链表长度是否是 8 ，进行红黑树处理
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 通过 binCount 判断是否是插入新数据还是更新数据，来更新数据 size baseCount
    addCount(1L, binCount);
    return null;
}
```

获取 size 通过 baseCount 和 CounterCell 数组。其中每次 put 都会进行更新 baseCount 和 CounterCell，但是对其中不懂。

# 总

分段锁虽然好，但是获取 size 复杂  
在 jdk7 中：
1. put 先采用 64 次自旋锁，获取不到再进行挂起自己等待 put 完唤醒  
2. size() 则是采用三次尝试无锁操作，如果 size 不一致才对每个 Segment 加锁获取 size

在 jdk8 中：
1. put  如果是新数据则使用 CAS  插入，如果是链表已有数据则使用 synchronized，如果一个 Node 链表超过 8 个就会变成红黑树，避免了 HashDos 攻击，最后统计 size
2. size() 直接获取 size

参考：https://www.jianshu.com/p/e694f1e868ec
