---
layout:     post
title:      "剑指JVM - 内存区域"
subtitle:   "经得起推敲的JVM内存区域考究"
author:     "Chris"
header-img: "img/post-bg-8.jpg"
tags:
    - JVM
---

![](/img/in-post/jvm-memory-area/overview.png)

## 运行时数据区域 - Runtime Data Areas

### Java堆 - [Heap](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.5.3)

![](/img/in-post/jvm-memory-area/heap.png)

对于大多数应用来说，Java堆是Java虚拟机所管理的内存中最大的一块。Java堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，**几乎所有**的对象实例都在这里分配内存，而**例外**的情况就关乎JVM的优化技术之一，逃逸分析（Escape Analysis）。

逃逸分析的基本行为就是分析对象动态作用域：当一个对象在方法中被定义后，它可能被外部方法所引用，称为方法逃逸。甚至还有可能被外部线程访问到，称为线程逃逸。若一个**对象**被证明为非逃逸对象，那么该对象则可能被JIT编译进行优化。对于不会有逃逸行为的对象，有可能会使用**栈上分配**优化手段将该对象的内存空间直接分配至栈上，随栈帧出栈而销毁。或是**标量替换**将其拆分成若干个成员变量，使其可以在栈上分配和读写。对于逃逸分析的启用或关闭可以使用VM参数`DoEscapeAnalysis`确定，在鄙人的机器上java `1.7.0_80`和`1.8.0_111`都已经默认启用。

需要额外注意的是，对于HotSpot而言，还有Class对象这个特殊的例子，在JDK7及JDK8中Class对象存放于**Java Heap**之内，而不是方法区，关于这点在知乎中已经有前辈列出[论据](https://www.zhihu.com/question/59174759)。

从内存回收的角度来观察Java堆，可以细分为：新生代和老年代；再细致一点的有Eden空间、From Survivor空间、To Survivor空间等。从内存分配的角度来看，Java堆中可能划分出多个线程私有的分配缓冲区（Thread Local Allocation Buffer，TLAB）。不过无论如何划分，都与存放内容无关，无论哪个区域，存储的都仍然是对象实例，进一步划分的目的是为了更好地回收内存，或者更快地分配内存。

根据Java虚拟机规范的规定，Java堆可以处于物理上不连续的内存空间，只要逻辑上是连续的即可。在实现时，既可以实现成固定大小的，也可以是可扩展的，不过当前主流的虚拟机都是按照可扩展来实现的（通过-Xmx和-Xms控制）。如果在堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出OutOfMemoryError异常。

### 方法区 - [Method Area](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.5.4)

![](/img/in-post/jvm-memory-area/method-area.png)

方法区属于Java堆的一个逻辑部分，是各个线程共享的内存区域，用于存储已被虚拟机加载的类信息、常量、JIT编译后的代码等数据。

在过去（自定义类加载器还不是很常见的时候），类大多是”static”的，很少被卸载或收集，因此被称为“永久的(Permanent)”。同时，由于类Class是JVM实现的一部分，并不是由程序创建的，所以又被认为是“非堆(non-heap)”内存。

在JDK8之前的HotSpot JVM，存放这些”永久的”的区域叫做“永久代(permanent generation)”。永久代是一片连续的堆空间，在JVM启动之前通过在命令行设置参数-XX:MaxPermSize来设定永久代最大可分配的内存空间，默认大小是64M（64位JVM由于指针膨胀，默认是85M）。永久代的垃圾收集是和老年代(old generation)捆绑在一起的，因此无论谁满了，都会触发永久代和老年代的垃圾收集。不过一个明显的问题是，如果对永久代的内存大小设置不当，当JVM加载的类信息容量超过了参数-XX:MaxPermSize设定的值时，应用将会报OOM的错误。而JDK8里的Metaspace，也可以通过参数-XX:MetaspaceSize 和-XX:MaxMetaspaceSize设定大小，但如果不指定MaxMetaspaceSize的话，Metaspace的大小仅受限于native memory的剩余大小。

在JDK7以后的版本，对于HopSpot JVM，**字符串常量池（[准确地说是StringTable所引用的java.lang.String实例](https://www.zhihu.com/question/49044988)）**已经从永久代中移出至Java堆。并且根据[JDK Enhancement Proposals-122](http://openjdk.java.net/jeps/122)中的描述，永久代已经在JDK8中彻底废弃，取而代之的是Metaspace。该计划提出的实现将在native memory中分配类的元信息，并将字符串常量池和类变量迁移至Java堆。

> The proposed implementation will allocate class meta-data in native memory and move interned Strings and class statics to the Java heap. 

虽说永久代已经从JDK8中移除，但不代表方法区被移除，本质上两者并不等价，仅仅是因为HotSpot虚拟机的设计团队在之前使用永久代来实现方法区而已。在The Java® Virtual Machine Specification for Java SE 8 Edition中我们依然可以看到[Method Area](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5.4)的身影。

**永久代的垃圾回收和老年代的垃圾回收是绑定的，一旦其中一个区域被占满，这两个区都要进行垃圾[回收](http://www.infoq.com/cn/articles/Java-PERMGEN-Removed/)，永久代中的元数据可能会随着每一次Full GC发生而进行移动**。这个区域的内存回收目标主要是针对类型的卸载，类型卸载条件相当苛刻，类需要同时满足以下3个条件才能算是可回收的类：

1. 这个类没有任何活的实例
2. 这个类的java.lang.Class对象不能被任何活引用所引用
3. 这个类的defining class loader不能被任何活引用所引用，而且它加载的所有类都满足前两点。

换句话说，一个ClassLoader与其加载的所有Class必须共存亡。

我们可以使用-verbose:class以及-XX:+TraceClassLoading查看类加载信息

对于每个Class文件来说，其中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是**Class文件常量池**（Constant Pool Table），用于存放**编译期**生成的各种字面量和符号引用，这部分数据将在**类加载**后进入方法区的**运行时常量池**（[Run-Time Constant Pool](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.5.5)）。一般来说，除了保存Class文件中描述的符号引用外，还会把翻译出来的直接引用也存储在运行时常量池中。

既然运行时常量池是方法区的一部分，自然会受到方法区内存的限制，当运行时常量池无法再申请到内存时则抛出OutOfMemoryError异常。

### 程序计数器 - [PC Register](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.5.1)

由于Java虚拟机的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现的，在任何一个确定的时刻，一个CPU处理器都只会执行一条线程中的指令。因此，为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，并且是线程私有的。

如果线程正在执行的是一个Java方法，这个计数器记录的是**正在执行**的虚拟机字节码指令的地址；如果正在执行的是native method，这个计数器的值则为Undefined。

由于程序计数器中存储的数据所占空间的大小不会随程序的执行而发生改变，因此，对于程序计数器并不会产生OutOfMemoryError。

### JVM栈 - [Java Virtual Machine Stacks](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.5.2)

![](/img/in-post/jvm-memory-area/vm-stack.png)

与程序计数器一样，Java虚拟机栈也是线程私有的，他的生命周期与线程相同。栈帧（Stack Frame）是用于支持虚拟机进行方法调用和方法执行的数据结构，他是虚拟机运行时数据区中的虚拟机栈的栈元素。栈帧存储了方法的局部变量表、操作数栈、动态连接、方法返回地址和一些额外的附加信息。每一个方法从调用开始至执行完成的过程，都对应着一个栈帧在虚拟机栈里面从入栈到出栈的过程。

局部变量表存放了编译期可知的各种基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference类型，他不等用于对象本身，可能是一个指向对象起始地址的引用指针、也可能是指向一个代表对象的句柄或与此对象相关的位置）和returnAddress类型（指向了一条字节码指令的地址）。

其中64bit长度的long和double类型的数据会占用2个局部变量空间（Slot），其余的数据类型只占用1个。局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在栈帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小。

在Java虚拟机规范中，对这个区域规定了两种异常状况：如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError异常；如果虚拟机栈可以动态扩展（当前大部分的Java虚拟机都可以动态扩展，只不过Java虚拟机规范中也允许固定长度的虚拟机栈），如果扩展时无法申请到足够的内存，就会抛出OutOfMemoryError异常。

### 本地方法栈 - [Native Method Stacks](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.5.6)

本地方法栈与虚拟机栈所发回的作用是非常相似的，他们之间的区别不过是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则为虚拟机使用到的Native方法服务。在虚拟机规范中对本地方法栈中方法使用的语言、使用方式与数据结构并没有强制规定，因此具体的虚拟机可以自有实现他。**甚至有的虚拟机（如HotSpot虚拟机）直接就把本地方法栈和虚拟机栈合二为一**。与虚拟机栈一样，本地方法栈预期也会抛出StackOverflowError和OutOfMemoryError异常。