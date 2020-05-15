---
title: 对比源码分析 ConcurrentHashMap 是如何成为一个线程安全的 HashMap
date: 2020-05-13 13:39:36
cover: https://gitee.com/dongzl/article-images/raw/master/cover/java_study.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文通过对比源码的方式，分析 ConcurrentHashMap 是如何保证线程安全，成为一个线程安全的 HashMap 容器。

categories: 
  - java开发

tags: 
  - Map
---

## 背景描述

`HashMap` 是我们在工作中使用非常广泛的容器类，前面已经有一篇文章，结合自己工作中的一个使用场景（[一个工作的真实场景聊聊 HashMap 的实现](https://dongzl.github.io/2019/09/29/02-JDK8-HashMap-Code/)），简单分析过 `HashMap` 的实现原理；但是我们也知道，`HashMap` 并不是线程安全的，如果要在多线程场景中保证线程安全，一般推荐使用 `ConcurrentHashMap`，当然这也不是唯一选择，还有 `HashTable`、`Collections.synchronizedMap()` 等方式，不过这些肯定都是不推荐的了。`ConcurrentHashMap` 和 `HashMap` 的在代码层面的实现非常类似，基本上可以说 `ConcurrentHashMap` 就是在可能会引发线程安全问题的关键点上做了处理，这篇文章我们就通过分析一下这些关键点，来理解 `ConcurrentHashMap` 是如何保证线程安全的。

> 关于 HashMap 可能引发的线程安全问题，可以参考这篇文章 [深入解读HashMap线程安全性问题](https://juejin.im/post/5c8910286fb9a049ad77e9a3)

## ConcurrentHashMap 构造方法实现

```java
/**
 * Creates a new, empty map with the default initial table size (16).
 * 创建空的 map，默认初始容量 16
 */
public ConcurrentHashMap() {

}

/**
 * Creates a new, empty map with an initial table size
 * accommodating the specified number of elements without the need
 * to dynamically resize.
 *
 * @param initialCapacity The implementation performs internal
 * sizing to accommodate this many elements.
 * @throws IllegalArgumentException if the initial capacity of
 * elements is negative
 */
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
            MAXIMUM_CAPACITY :
            tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}
```
`ConcurrentHashMap` 类的构造方法还是有些特殊的：

- 在构造方法内部并没有进行 `table` 的初始化操作，初始化的过程放到了 `put` 方法中来完成，后面我们还会讲到；
- 在带有 `initialCapacity` 参数的构造方法中提供初始容量值，计算了  `sizeCtl` 大小，`sizeCtl = [( 1.5 * initialCapacity + 1)`，然后向上取最近的 2 的 n 次方]。如 `initialCapacity` 为 10，那么得到 `sizeCtl` 为 16，如果 `initialCapacity` 为 11，得到 `sizeCtl` 为 32。`sizeCtl` 这个属性有很多使用场景，后面我们来分析。

## ConcurrentHashMap 类 spread 方法理解

```java
/**
 * Spreads (XORs) higher bits of hash to lower and also forces top
 * bit to 0. Because the table uses power-of-two masking, sets of
 * hashes that vary only in bits above the current mask will
 * always collide. (Among known examples are sets of Float keys
 * holding consecutive whole numbers in small tables.)  So we
 * apply a transform that spreads the impact of higher bits
 * downward. There is a tradeoff between speed, utility, and
 * quality of bit-spreading. Because many common sets of hashes
 * are already reasonably distributed (so don't benefit from
 * spreading), and because we use trees to handle large sets of
 * collisions in bins, we just XOR some shifted bits in the
 * cheapest possible way to reduce systematic lossage, as well as
 * to incorporate impact of the highest bits that would otherwise
 * never be used in index calculations because of table bounds.
 */
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

`ConcurrentHashMap` 中 `spread` 方法的作用类似于 HashMap 中 hash(Object key) 方法，但是在实现上也是略有差别。

`ConcurrentHashMap` 类中的 `spread(int h)` 方法类似于 `HashMap` 中的 `hash(Object key)` 方法，两个方法的作用应该说是相同的，都是通过将 `Map` 中的 `key` 的 `hashCode` 值进行无符号右移 16 位的方式，让 `hashCode` 的高低 16 位都参与到运算中，尽量降低 `hash` 碰撞的概率，但是两者的实现又略有不同。

```java
static final int hash(Object key) {
    int h = key.hashCode();
    return (key == null) ? 0 : h ^ (h >>> 16);
}
```

`HashMap` 中只是做了无符号右移操作，然后和原值进行异或操作；而在`ConcurrentHashMap` 中无符号右移，和原值异或之后，还和 `HASH_BITS` 常量进行了 & 操作，`HASH_BITS` 常量在类中的定义为：

```java
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
```

这个值其实是 Integer.MAX_VALUE，如果转换成二进制是：

```java
0111 1111 1111 1111 1111 1111 1111 1111
```

对于任意一个二进制的数据，和上面的数据做 & 操作，其实结果都是：

- 如果这个数 > 0，那么 `& HASH_BITS` 结果是不变的，还是原值；
- 如果这个数 < 0，那么 `& HASH_BITS` 后低 31 位不变，最高位符号位 `1 & 0` 后结果为 `0`，最终结果 > 0。

这个操作的含义应该就是注释中描述的作用 `forces top bit to 0`：

> Spreads (XORs) higher bits of hash to lower and also forces top bit to 0.

<font color="red">对于这里的操作不是太理解作用是什么，为了防止 hashCode 出现负数的情况吗，即使是负数，最终计算 table 数组中位置的时候是会进行 h & (n-1) 计算，也该不会有什么问题；而且在 HashMap 的源码的确是没有这个操作的，没理解透彻，这个问题暂时遗留吧，看看是不是有朋友能回答这个问题。</font>

## ConcurrentHashMap 中的 put 操作

```java
/**
 * Maps the specified key to the specified value in this table.
 * Neither the key nor the value can be null.
 *
 * <p>The value can be retrieved by calling the {@code get} method
 * with a key that is equal to the original key.
 *
 * @param key key with which the specified value is to be associated
 * @param value value to be associated with the specified key
 * @return the previous value associated with {@code key}, or
 *         {@code null} if there was no mapping for {@code key}
 * @throws NullPointerException if the specified key or value is null
 */
public V put(K key, V value) {
    return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) {
        throw new NullPointerException();
    }
    // 计算hash值
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table; ;) {
        Node<K,V> f;
        int n;
        int i;
        int fh;
        if (tab == null || (n = tab.length) == 0) {
            // 初始化tab
            tab = initTable();
            // 找到tab中的第一个元素
        } else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 如果第一个元素为空，执行cas操作，添加元素
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null))) {
                // 当向一个空bin添加元素时使用无锁的CAS
                break; // no lock when adding to empty bin
            }
        } else if ((fh = f.hash) == MOVED) {
            // 处理map正在扩容中的场景
            tab = helpTransfer(tab, f);
        } else {
            V oldVal = null;
            // 加锁执行添加元素操作
            // 锁f是通过tabAt方法获取的
            // 也就是说，当发生hash碰撞时，会以链表的头结点作为锁
            synchronized (f) {
                // 这个检查的原因在于：
                // tab引用的是成员变量table，table在发生了rehash之后，原来index上的Node可能会变
                // 这里就是为了确保在put的过程中，没有收到rehash的影响，指定index上的Node仍然是f
                // 如果不是f，那这个锁就没有意义了
                if (tabAt(tab, i) == f) {
                    // 确保put没有发生在扩容的过程中，fh=-1时表示正在扩容
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f; ; ++binCount) {
                            K ek;
                            if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent) {
                                    e.val = value;
                                }
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    }
                    // 如果是红黑树，执行红黑树插入操作
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent) {
                                p.val = value;
                            }
                        }
                    }
                }
            }
            // 如果超过设定阈值，转换成红黑树
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD) {
                    treeifyBin(tab, i);
                }
                if (oldVal != null) {
                    return oldVal;
                }
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/25-JDK8-ConcurrentHashMap/JDK8-ConcurrentHashMap-01.png" width="600px">

通过对代码分析，我们可以了解到，在执行 `put` 操作时会进行如下操作：

- 如果 table 为空，执行初始化操作，这一点是 ConcurrentHashMap 和 HashMap 比较大的区别，ConcurrentHashMap 在构造方法中不会对 table 进行初始化，而 HashMap 会在构造方法中对 table 进行初始化；
- 如果 table 某个位置元素为空，也就是这个位置还没有元素，put 的内容是这个位置的第一个元素，ConcurrentHashMap 会通过 CAS 操作，将该元素放入该位置；
- 在 put 过程中，有可能 ConcurrentHashMap 正在执行扩容操作，需要等到扩容操作，继续进行 put 操作；
- 如果 table 某个位置元素不为空，需要使用 synchronized 对该位置第一个元素 f 加锁，完成 put 操作；
- 至于后面执行的操作，根据判断是链表 or 红黑树来执行不同操作，与 HashMap 中的操作基本是相同的。

## ConcurrentHashMap 中的 get 操作

```java
/**
 * Returns the value to which the specified key is mapped,
 * or {@code null} if this map contains no mapping for the key.
 *
 * <p>More formally, if this map contains a mapping from a key
 * {@code k} to a value {@code v} such that {@code key.equals(k)},
 * then this method returns {@code v}; otherwise it returns
 * {@code null}.  (There can be at most one such mapping.)
 *
 * @throws NullPointerException if the specified key is null
 */
public V get(Object key) {
    Node<K, V>[] tab; Node<K, V> e; Node<K, V> p;
    int n; int eh; K ek;
    int h = spread(key.hashCode());
    tab = table; n = tab.length; e = tabAt(tab, (n - 1) & h);
    if (tab != null && n > 0 && e != null) {
        eh = e.hash;
        if (eh == h) {
            ek = e.key;
            if (ek == key || (ek != null && key.equals(ek))) {
                return e.val;
            }
        } else if (eh < 0) {
            p = e.find(h, key);
            return p != null ? p.val : null;
        }
        e = e.next;
        while (e != null) {
            ek = e.key;
            if (e.hash == h && (ek == key || (ek != null && key.equals(ek)))) {
                return e.val;
            }
            e = e.next;
        }
    }
    return null;
}
```
通过上面的源码，我们可以看到，在 `ConcurrentHashMap` 类的 `get` 方法并没有任何加锁操作，那是如何来保证线程安全的。

```java
/**
 * Key-value entry.  This class is never exported out as a
 * user-mutable Map.Entry (i.e., one supporting setValue; see
 * MapEntry below), but can be used for read-only traversals used
 * in bulk tasks.  Subclasses of Node with a negative hash field
 * are special, and contain null keys and values (but are never
 * exported).  Otherwise, keys and vals are never null.
 */
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;

    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }

    public final K getKey()       { return key; }
    public final V getValue()     { return val; }
    public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
    public final String toString(){ return key + "=" + val; }
    public final V setValue(V value) {
        throw new UnsupportedOperationException();
    }

    ... ...
}
```

`get` 操作可以无锁是由于 `Node` 的元素 `val` 和指针 `next` 是用 `volatile` 修饰的，在多线程环境下一个线程修改结点的 `val` 或者新增节点的时候保证对另外一个线程可见的。

## ConcurrentHashMap 中的扩容操作

[解读Java8中ConcurrentHashMap是如何保证线程安全的--5 线程安全的扩容](https://juejin.im/post/5ca89afa5188257e1d4576ff#heading-10)

## 参考资料

- [解读Java8中ConcurrentHashMap是如何保证线程安全的](https://juejin.im/post/5ca89afa5188257e1d4576ff)

