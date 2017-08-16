---
layout:     post
title:      "JDK 1.8 ArrayList源码解析"
subtitle:   "预防间竭性失忆的ArrayList记事"
author:     "Chris"
header-img: "img/post-bg-7.jpg"
tags:
    - Java
    - 源码研读
---

## 简介

ArrayList是List接口的可扩容数组实现，在日常开发中为我们免去了思考数组扩容的逻辑，本文基于`JDK 1.8.0_111`的源码进行研究，由于ArrayList过于基础与常见，故免去冗长的描述。

## 结构

在深入研读源码之前，我们首先需要对ArrayList的Outline有个大概的了解：

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    /**
     * 初始化容量
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * Shared empty array instance used for empty instances.
     * <p>
     * 用于空数组时所返回的实例
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * Shared empty array instance used for default sized empty instances. We
     * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
     * first element is added.
     * <p>
     * 使用默认容量时的数组实例
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * 真正存放元素的内部数组。在该引用等于DEFAULTCAPACITY_EMPTY_ELEMENTDATA时，在第一个元素被添加时就会扩容至DEFAULT_CAPACITY
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * 实际存放的元素总数
     */
    private int size;

}
```

上面列出了ArrayList中的大部分域，其他特别注意`elementData`代表ArrayList实际存储的元素的内部结构，`EMPTY_ELEMENTDATA`表示空的`elementData`时所返回的实例，而`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`表示使用默认容量（10）时的实例，对于两者的区别可以看javadoc上的注释，但是可能接着往下看会找到更直观的答案。

### 构造器

```java
    /**
     * Constructs an empty list with the specified initial capacity.
     * <p>
     * 使用指定的初始化容量去构建一个空的List
     *
     * @param initialCapacity the initial capacity of the list
     * @throws IllegalArgumentException if the specified initial capacity
     *                                  is negative
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            // 初始化容量大于0则按照指定容量初始化List
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            // 初始化容量为0的时候，使用EMPTY_ELEMENTDATA（注意与DEFAULTCAPACITY_EMPTY_ELEMENTDATA区分）
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            // 当初始化容量小于0的时候抛出IllegalArgumentException
            throw new IllegalArgumentException("Illegal Capacity: " +
                    initialCapacity);
        }
    }

    /**
     * Constructs an empty list with an initial capacity of ten.
     * <p>
     * 若使用默认构造器则将elementData指定为DEFAULTCAPACITY_EMPTY_ELEMENTDATA（注意区分EMPTY_ELEMENTDATA）
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     * Constructs a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param c the collection whose elements are to be placed into this list
     * @throws NullPointerException if the specified collection is null
     */
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            // toArray()返回的是实际类型，不一定是Object[]。这样做能以防抛出ArrayStoreException
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            // 空的List全部用EMPTY_ELEMENTDATA替代
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

ArrayList提供了3个构造器，在指定容量的构造器`ArrayList(int)`中可以看到，指定容量大小为0时，`elementData`被赋值为`EMPTY_ELEMENTDATA`；而在无参构造器中，`elementData`被直接赋值为`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`，这样就简单明了地看清楚了这两个很相似但是却又不尽相同的变量到底代表了什么。

### 扩容

```java
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            // 如果是默认容量的空List的话则取10和minCapacity之间的最大值
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        // 提防负增长
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    /**
     * The maximum size of array to allocate.
     * Some VMs reserve some header words in an array.
     * Attempts to allocate larger arrays may result in
     * OutOfMemoryError: Requested array size exceeds VM limit
     * <p>
     * 划重点，某些VM会保留一些header words在数组里，所以设定数组大小的时候不能"赶尽杀绝"
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     */
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        // 每次扩容50%
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0) {	// (1)
            // 保证minCapacity一定大于newCapacity
            // 基本除了首次扩容至10之后都不会执行到此分支
            newCapacity = minCapacity;
        }
        if (newCapacity - MAX_ARRAY_SIZE > 0) {
            newCapacity = hugeCapacity(minCapacity);
        }
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
                Integer.MAX_VALUE :
                MAX_ARRAY_SIZE;
    }
```

在ArrayList内部扩容的调用链为`ensureCapacityInternal(int)` —> `ensureExplicitCapacity(int)` —> `grow(int)`。

每当ArrayList的`add`或`addAll`方法及其衍生方法被调用，都会先调用`ensureCapacityInternal(int)`以进行容量的确认，往上翻一下源码可以看到该方法中有如下语句

```java
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
```

回忆一下`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`的作用，该if语句告知我们要么取`DEFAULT_CAPACITY`（也就是10）要么取外部传入的变量，选其最大值作为`ensureExplicitCapacity(int)`的实参。在`ensureExplicitCapacity(int)`内自增modCount（该变量从AbstractList继承），检测正增长的之后传入`grow(int)`，同时需要注意，**本方法也是唯一的`grow(int)`入口**。

扩容的真正主角是函数`grow(int)`，在此之前我们中途插一下ArrayList中添加元素的方法，比较有代表性的是`add(E)`和`addAll(Collection<? extends E c>)`。前者在真正将元素插入至elementData之前会调用`ensureCapacityInternal(size + 1)`以确认数组容量充足，而后者将会调用`ensureCapacityInternal(size + c.toArray().lenth)`，一个是当前容量加一，另一个是当前容量加新增集合大小。

**<u>以下皆是在ArrayList不使用Collection进行初始化时的状态说明。</u>**

回到主角`grow(int)`上，将灯光打在代码块中的标注`(1)`上，在首次调用grow(int)时，即当elementData为`EMPTY_ELEMENTDATA`或`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`时`oldCapacity`都为0，所以计算出的`newCapacity`同样为0。而`minCapacity`为外部传入，同样是在首次调用grow(int)时，当elementData为`EMPTY_ELEMENTDATA`时，`minCapacity`可能为1(add)或者是新添加的集合大小(addAll)；当elementData为`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`时，`minCapacity`会被赋值为10，这是在`ensureCapacityInternal(int)`中所决定的。

综上所述，标注`(1)`的if语句在首次grow时为true，此时ArrayList有可能为1(调用add时)，有可能为新增的集合大小(调用addAll时)，也有可能为10(使用无参构造器时)。在以后的扩容中每次增加50%的容量，最后使用native方法`System.arraycopy`复制到新的数组中，这样就完成了数组动态的大小调整。

### 迭代器

在JDK 5之后，Java引入了新的语法糖`Foreach`，在实现接口`Iterable<T>`后都可直接使用该语法糖。

但是我们主要关心的是产生`ConcurrentModificationException`的相关变量`modCount`，相关迭代器的知识可以阅读《Thinking In Java》 the 4th edition。我们看看迭代器相关的实现类`ArrayList.Itr`中的方法，为了阅读方便我们先除去Java 8中lambda相关的函数

```java
    /**
     * An optimized version of AbstractList.Itr
     */
    private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        public boolean hasNext() {
            return cursor != size;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```

当外部对象调用ArrayList的实例方法`iterator()`时返回实例`ArrayList.Itr`，在初始化时int `expectedModCount`会被赋值为`modCount`，在`next()`、`remove()`执行具体逻辑之前都会检测modCount是否有被更改，如果有则会抛出`ConcurrentModificationException`。

事实上并不是任何操作都会导致`ConcurrentModificationException`，在`java.util.ArrayList.Itr#remove`和`java.util.ArrayList.ListItr#add`两个方法里面会重新将`modCount`赋值给`expectedModCount`。所以在使用迭代器的时候，不要使用`ArrayList#remove`及其衍生方法，而是使用单向迭代器中的`ArrayList.Itr#remove()`就能有效预防`ConcurrentModificationException`。

## 总结

ArrayList默认容量为10，但不是马上就为其申请内存空间，而是在首次`add()`或`addAll()`时才进行空间分配，除首次扩容以外，每次扩容提升50%。所以我们在初始化ArrayList的时候就**必须**指定期望容量，避免多次扩容造成不必要的性能开销。但是具有疑惑的一点ArrayList所说的最大数组容量为`MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8`，这8个Slot是为某些虚拟机所预留的，在Javadoc上明确说明了`Some VMs reserve some header words in an array`。但是在`hugeCapacity(int)`中是允许将数组的最大值变成`Integer.MAX_VALUE`，也就是说把原本预留给某些虚拟机的8个slot全部“赶尽杀绝”。在此我们需要思考的是，ArrayList的设计者应该不会为了多给调用者8个Slot，铤而走险地冒着OOM的风险把这些内存给省出来。或者是说这预留给某些虚拟机的8个Slot是可有可无的，如果是这样的话就完全没有必要将其上界首先限制为`Integer.MAX_VALUE - 8`。这个问题记录在此，希望将来的自己回顾这篇文章的时候能有一个鲜明的答案。