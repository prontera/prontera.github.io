---
layout:     post
title:      "JDK 1.8 HashMap源码解析"
subtitle:   "领略JDK大师设计，了解HashMap的基本工作原理"
author:     "Chris"
header-img: "img/post-bg-1.jpg"
tags:
    - Java
    - 源码研读
---

## 简介

JDK 1.8的HashMap相较于JDK 1.7有了更多的优化，单纯在体量上HashMap的代码就足足增加了一倍。这些体量的增加更多的是底层的一些优化，如红黑树的引入和扩容时的rehash优化等。

![](https://4.bp.blogspot.com/-qWRZSkyBDGc/V6M41S237DI/AAAAAAAAGxQ/xKS9RMu57tAsLHPFzw7JfSTq6tLoBvIjACLcB/s1600/How%2Bdoes%2BHashMap%2Bworks%2Bin%2BJava%2B8.jpg)

## 内部结构

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {

    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
    static final int MAXIMUM_CAPACITY = 1 << 30;
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    static final int TREEIFY_THRESHOLD = 8;
    static final int MIN_TREEIFY_CAPACITY = 64;

    transient Node<K,V>[] table;
    transient int size;
    int threshold;
    final float loadFactor;

  	// 链表节点
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
    }
  
  	// 红黑树节点
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
    }
}
```

- DEFAULT_INITIAL_CAPACITY

  初始化HashMap时的默认容量，无论何时table的容量必然为2^n，如果在初始化时所传入的值不满足该条件，HashMap将会往上调节大小以达到2^n的要求。

- MAXIMUM_CAPACITY

  HashMap的最大容量为1 << 30，达到此数值后将不再扩容。

- DEFAULT_LOAD_FACTOR

  通常译为“负载因子”，是时间与空间上的一种平衡。

- TREEIFY_THRESHOLD

  将链表树化的阈值，这个值为8，表示如果一个哈希桶上的链表长度大于等于8后将**有机会尝试**将其转换成红黑树，但实际的转换条件还要根据下面的MIN_TREEIFY_CAPACITY来进行综合判断。

- MIN_TREEIFY_CAPACITY

  转换成红黑树时的最小容量，最小的值也应该为TREEIFY_THRESHOLD * 4，这个值默认为64，即如果table.length如果小于64的话则不选择将哈希桶转换成红黑树，而是选择扩容。

- table

  哈希表，在JDK1.8中是`Node<K,V>[]`，在JDK1.7是`Entry<K,V>[]`

- size

  实际上HashMap的实际元素的个数

- threshold

  进行扩容的容量阈值 (capacity * load factor)

- loadFactor

  哈希表的负载因子，一般默认为DEFAULT_LOAD_FACTOR，即0.75f

## 取模的位运算

取模同样也可以被位运算所替代，当余数为2^n时，且被余数不小于0的时候，以下等式成立

`x % 2^n == x & (2^n - 1)`

我们列举几个例子（x不为负数）

```java
x % 2 == x & 1
x % 4 == x & 3
x % 8 == x & 7
```

更多关于位运算取模的信息可以翻阅一下维基百科的[相关词条](https://en.wikipedia.org/wiki/Modulo_operation#Performance_issues)。

上述的操作是HashMap的优化点之一，同样也是HashMap的capacity为什么取2^n的原因之一。

## 负载因子

起初我学习的时候会有以下的疑问：为什么会有负载因子这个东西？不应该是物尽其用吗？其实当负载因子比较大的时候的确能够起到物尽其用的作用，但相对地哈希桶的长度也会增加，查找的效率就会降低，这是时间换空间；如果负载因子较小，则会引起更多的扩容，对于原本的哈希桶在扩容之后会有相应地减少长度，从而让哈希表有更高的检索效率，这是典型的空间换时间。HashMap中的负载因子默认为0.75，对于该数值我也是持有“默认即有理”的态度，一般也不会更改。

## 内部实现

以下将会介绍HashMap中的哈希计算、添加元素和扩容这几项操作。

### hash(Object) - 计算哈希

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

计算哈希的方法我们可知两点

1. 当key为null的时候必定存放在第0位的哈希桶上
2. 结合了hashCode()，并将其高16bit进行异或操作，使得高位都能参与计算

### putVal(int, K, V, boolean, boolean) - 添加Node至哈希表

```java
/**
 * Implements Map.put and related methods
 *
 * @param hash hash for key
 * @param key the key
 * @param value the value to put
 * @param onlyIfAbsent if true, don't change existing value
 * @param evict if false, the table is in creation mode.
 * @return previous value, or null if none
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab;
  	// p为计算所得的哈希桶的首节点
  	Node<K,V> p;
  	// i为桶的下标，n为哈希表长度(注意是哈希表，而不是哈希桶)
  	int n, i;
  	// 如果table为null或table.length为0就进行初始化(扩容)
    if ((tab = table) == null || (n = tab.length) == 0) {
      	// n为当前的table.length
      	// a1
        n = (tab = resize()).length;
    }
  	// hash由外部传入，经hashCode及与高16位异或所得
  	// (n - 1) & hash与hash % n等同
  	// 如果该桶为null就直接占用
  	// a2
    if ((p = tab[i = (n - 1) & hash]) == null) {
        tab[i] = newNode(hash, key, value, null);
    }
  	// 以下详细解析被占用的情况
    else {
      	// e存储已经存在的节点
        Node<K,V> e;
      	// p的key，即哈希桶首节点的key
      	K k;
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k)))) {
          	// 如果桶的首节点的key与传入的key一致，会在下面if(e != null)块中覆盖旧val
          	// a3
            e = p;
        }
      	// 如果p为红黑树节点，则在树中插入键值对
        else if (p instanceof TreeNode) {
          	// a4
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        }
        else {
          	// 如果与链表的头结点e不匹配，也不是树节点，说明还是链表状态，用拉链法
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                  	// 还是链表的时候会将节点放到末尾
                    p.next = newNode(hash, key, value, null);
                  	// TREEIFY_THRESHOLD默认为8，即如果长度大于等于8就有可能变形成红黑树
                  	// binCount是从0开始的，所以TREEIFY_THRESHOLD-1
                    if (binCount >= TREEIFY_THRESHOLD - 1) {// -1 for 1st
                      	// a5
                        treeifyBin(tab, hash);
                    }
                  	// 此时e为null
                    break;
                }
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
                  	// 在哈希桶中找到equals key的Node，内存地址一样也行
                  	// a6
                    break;
                }
              	// 遍历下一个节点
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
          	// onlyIfAbsent这变量是方法putIfAbsent与put的唯一区别
          	// 特别注意如果putIfAbsent的val为null的话，下次putIfAbsent会覆盖这个null值
            if (!onlyIfAbsent || oldValue == null) {
              	// a7
                e.value = value;
            }
          	// 这个回调方法目前仅供LinkedHashMap使用
            afterNodeAccess(e);
          	// 返回旧val
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold) {
      	// 扩容
      	// a8
        resize();
    }
  	// 这个回调方法与afterNodeAccess一样目前仅供LinkedHashMap使用
    afterNodeInsertion(evict);
    return null;
}
```

真正计算哈希桶下标的操作在a2，hash由外部传入，其实质就是调用了上述介绍的hash(Object)函数

```java
public V put(K key, V value) {
  return putVal(hash(key), key, value, false, true);
}
```

回到a2处可以看到有一条赋值语句`tab[i = (n - 1) & hash]`， 该语句与`hash % n`相当，但hash必须不为负数，这点hashCode()已经有保证，而且n必须为2的幂，这点由HashMap的特质所决定。可以比较一下位运算取模和直接通过运算符取模的性能差异，本机测试大概差距一倍有多，这可能是HashMap不采用素数作为capacity的原因之一。

整个putVal的逻辑大致如下，根据外部传入的hash计算出哈希桶index。先判断哈希桶的**头结点**是否null，是就将其占用，不是的话就判断该哈希桶的**头结点**的key是否相等，如果是则覆盖其值，如果头结点是树节点的话则交给红黑树相关的函数处理，再不符合条件就要开始遍历该哈希桶。在遍历的过程中如果发现hash相等且key相等的Node则覆盖其val，但如果整个桶都遍历了依然没有找到的话就将其添加到该桶的末尾，并且判断该桶的大小是否超过8，是就发起转换红黑树的请求，如果该table的capacity不小于64的话，就将该桶正式转换为红黑树。

### resize() - 扩容

```java
/**
 * Initializes or doubles table size.  If null, allocates in
 * accord with initial capacity target held in field threshold.
 * Otherwise, because we are using power-of-two expansion, the
 * elements from each bin must either stay at same index, or move
 * with a power of two offset in the new table.
 *
 * @return the table
 */
final Node<K,V>[] resize() {
  	// 旧table的引用
    Node<K,V>[] oldTab = table;
  	// 旧table的容量
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
  	// 旧table的阈值
    int oldThr = threshold;
  	// 新table的容量与阈值
    int newCap, newThr = 0;
  	// 通过任一构造函数初始化后，首次put的时候因为table为null，所以oldCap为0 ->>
  	// 所以本if块只为table已经初始化过后扩容所用
    if (oldCap > 0) {
      	// 如果旧table已经达到最大上限，就将threshold设为Integer.MAX_VALUE
      	// MAXIMUM_CAPACITY < Integer.MAX_VALUE
      	// 专门对付 if (++size > threshold)
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
          	// 因为已经达到上限，所以不再提供任何扩容功能
            return oldTab;
        }
      	// 如果小于MAXIMUM_CAPACITY而且HashMap(int)初始化的容量大于16
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
          	// 如果上面判断为false就调到下面if (newThr == 0)
          	// b1 扩容一倍
            newThr = oldThr << 1; // double threshold
    }
  	// 如果是通过HashMap()初始化，threshold默认为0 ->>
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
              	// 桶只有一个节点的情况
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
              	// 红黑树的情况则不展开讨论
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                      	// 复制e的下一个节点的引用
                        next = e.next;
                      	// b2
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                      	// b3
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                  	// loTail执行到这里指向桶的最后的一个节点
                    if (loTail != null) {
                        loTail.next = null;
                      	// b4
                        newTab[j] = loHead;
                    }
                  	// hiTail也是指向桶的最后一个节点
                    if (hiTail != null) {
                        hiTail.next = null;
                      	// b5 等同左移1位
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

在javadoc中有以下一句话

> Otherwise, because we are using power-of-two expansion, the elements from each bin must either stay at same index, or move with a power of two offset in the new table.

因为capacity使用的是2^n的增长方式，在b1处我们可以看到扩容是每次将容量扩大一倍，所以扩容后的哈希桶index要么不变，要么移动index乘2，美团点评的有一篇文章就将其解释得很清楚，先看看美团大师的讲解。

经过观测可以发现，我们使用的是2次幂的扩展(指长度扩为原来2倍)，所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。看下图可以明白这句话的意思，n为table的长度，图（a）表示**扩容前**的key1和key2两种key确定索引位置的示例，图（b）表示**扩容后**key1和key2两种key确定索引位置的示例，其中hash1是key1对应的哈希与高位运算结果。

![hashMap 1.8 哈希算法例图1](http://tech.meituan.com/img/java-hashmap/hashMap%201.8%20%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E4%BE%8B%E5%9B%BE1.png)

元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：

![hashMap 1.8 哈希算法例图2](http://tech.meituan.com/img/java-hashmap/hashMap%201.8%20%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E4%BE%8B%E5%9B%BE2.png)

因此，我们在扩充HashMap的时候，不需要像JDK1.7的实现那样重新计算hash，**只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”**，可以看看下图为16扩充为32的resize示意图：

![jdk1.8 hashMap扩容例图](http://tech.meituan.com/img/java-hashmap/jdk1.8%20hashMap%E6%89%A9%E5%AE%B9%E4%BE%8B%E5%9B%BE.png)

这个设计确实非常的巧妙，既省去了重新计算hash值的时间，而且同时，由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的bucket了。这一块就是JDK1.8新增的优化点。有一点注意区别，JDK1.7中rehash的时候，旧链表迁移新链表的时候，如果在新表的数组索引位置相同，则链表元素会倒置，但是从上图可以看出，JDK1.8不会倒置。

以上就是美团的工程师们对其的详尽解释，我们回顾一下源码b2处，如果`(e.hash & oldCap) == 0`就放到低位，这点可以对照上述图a处，将n-1还原为n，再对hash1或hash2进行相与操作，这一位进位刚好就是扩容之后(<<1)对应的bit，我们可以根据其计算值决定该Node是stay at same index，还是move with a power of two offset in the new table。

## 后话

以上就是JDK 1.8 HashMap的源码阅读笔记，如有错误可在下方留言或私信给我，万分感谢！

