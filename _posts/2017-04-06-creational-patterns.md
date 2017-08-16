---
layout:     post
title:      "设计模式之创建型模式总结"
subtitle:   "Creational Patterns"
author:     "Chris"
header-img: "img/post-bg-3.jpg"
tags:
    - 设计模式
---

## 前言

以前学习设计模式的时候, 通常都是咖啡馆, 披萨店, 各种各样的红绿头鸭. 明显地, 这些概念需要更新一下, 这次我们用MacBook和XPS13两台电脑作为例子来快速复习创建型设计模式.

创建型模型简单直接, 所以大部分模式直接通过UML进行展示, 个别需要特别注意的将会以代码的形式展示.

代码已上传至[GitOSC](http://git.oschina.net/witless/design-patterns/)

## 简单工厂

简单工厂可以为客户端选择具体的实现

下面是LaptopFactory生产MacBook和Xps13的例子

#### UML

![](/img/in-post/creational-patterns/simple-factory.png)

#### Usage

```java
/**
 * @author Zhao Junjian
 */
public class Client {
    public static void main(String[] args) {
        // 生成rmbp
        final MacBook implA = LaptopFactory.createMacBook();
        implA.doSomething();
        // 生产xps
        final Xps13 implB = LaptopFactory.createXps13();
        implB.doSomething();
    }
}
```

## 工厂方法模式 - [Factory method](https://en.wikipedia.org/wiki/Factory_method_pattern)

与简单工厂的区别在于**工厂方法模式允许将类的实例化推迟到子类中**, 如果把工厂方法中的具体实现放到父类中, 那么就等同于简单工厂

下面是LaptopFactory将生产Laptop的功能分别推迟至`AppleFactory`和`DellFactory`的例子

#### UML

![](/img/in-post/creational-patterns/factory-method.png)

#### Usage

```java
/**
 * @author Zhao Junjian
 */
public class Client {
    public static void main(String[] args) {
        final LaptopFactory laptopFactory = new AppleFactory();
        laptopFactory.someWork();
    }
}
```

#### 小结

工厂父类是一个抽象类, 里面有个protected abstract方法就是工厂方法

一般工厂方法返回的是被创建对象的接口对象, 当然也可以是抽象类或者一个具体的类的实例

工厂方法可以看成一个生成对象的抽象方法

## 抽象工厂模式 - [Abstract factory](https://en.wikipedia.org/wiki/Abstract_factory_pattern)

创建一系列的产品对象, 而且这一系列对象是构建新的对象所需要的组成部分, 简而言之就是这一系列被创建的对象相互之间是有约束的.

如今不仅需要生产电脑, 还要生产电源适配器, 这两个组成了产品簇. 显而易见的是XPS不能使用MacBook的电源.

#### UML

![](/img/in-post/creational-patterns/abstract-factory.png)

#### Client Usage

```java
/**
 * @author Zhao Junjian
 */
public class Client {
    public static void main(String[] args) {
        final LaptopFactory factory = new AppleFactory();
        factory.createLaptop();
        factory.createPowerAdaptor();
    }
}
```

## 原型模式 - [Prototype](https://en.wikipedia.org/wiki/Prototype_pattern)

<u>可忽略</u>, 在实际工作中更多地会使用第三方类库完成深拷贝

下面是克隆MacBook的例子

#### UML

![](/img/in-post/creational-patterns/prototype.png)

#### Usage

```java
/**
 * @author Zhao Junjian
 */
public class PrototypeTester {
    @Test
    public void testClone() throws Exception {
        // 你就只管我们生成一个List对象就行
        final List<MacBook.Cpu> cpuList = new ArrayList<>();
        for (int i = 1; i <= 4; i++) {
            final MacBook.Cpu cpu = new MacBook.Cpu(i);
            cpuList.add(cpu);
        }
        final MacBook macBook = new MacBook();
        macBook.setCpuArray(cpuList.toArray(new MacBook.Cpu[cpuList.size()]));
        macBook.setCpuList(cpuList);
        macBook.setKeyboard(87);
        macBook.setMemory("16G");
        // 全部准备完成, 我们就开始克隆一个新的实体
        final MacBook newMacBook = macBook.clone();
        Assert.assertEquals(macBook, newMacBook);
        // 修改引用类型
        final MacBook.Cpu newCpu = new MacBook.Cpu(5);
        cpuList.add(newCpu);
        newMacBook.setCpuList(cpuList);
        // 如果这里抛出异常, 就证明是浅拷贝. 实际情况就是浅拷贝.
        Assert.assertNotEquals(macBook, newMacBook);
    }
}
```

#### 小结

`MacBook`需要实现标识接口Cloneable, 并在`clone()`方法里面显式地调用父类的`clone()`

```java
@Override
protected MacBook clone() throws CloneNotSupportedException {
  return (MacBook) super.clone();
}
```

## 单例模式 - [Singleton](https://en.wikipedia.org/wiki/Singleton_pattern)

这里着重于讲解懒汉式, 因为有需要注意的细节, 所以直接贴代码.

实现单例有3种模式, 嵌套类, 双重检查和枚举

##### 嵌套类 holder

```java
/**
 * @author Zhao Junjian
 */
public class DellFactory {

    private DellFactory() {
        System.out.println("dell is the best");
    }

    private static final class DellFactoryHolder {
        private static final DellFactory FACTORY = new DellFactory();
    }

    public static DellFactory getInstance() {
        return DellFactoryHolder.FACTORY;
    }

}
```

##### 双重检查 double-checked

```java
/**
 * @author Zhao Junjian
 */
public class AppleFactory {
    private static volatile AppleFactory FACTORY;

    public static AppleFactory getInstance() {
        if (FACTORY == null) {
            //Client.LATCH.await();
            synchronized (AppleFactory.class) {
                if (FACTORY == null) {
                    FACTORY = new AppleFactory();
                }
            }
        }
        return FACTORY;
    }

    private AppleFactory() {
        System.out.println(Thread.currentThread().getName() + " wanna get a hurt");
    }

}
```

##### 枚举 enum

```java
/**
 * @author Zhao Junjian
 */
public enum RazerFactory {
    STEALTH;

    public void doSomething() {
        // ...
    }
}
```

## 建造者模式/生成器模式 - [Builder](https://en.wikipedia.org/wiki/Builder_pattern)

实际中有两种方法可以快速构建Fluent API, 第一是安装插件, 第二是使用Lombok

被构建者的私有构造器, 公开静态的builder()方法

下面是通过建造者生成MacBook的例子

```java
/**
 * @author Zhao Junjian
 */
@Data
public class MacBook {
    private String processor;

    private int memory;

    private long capacity;

    private String keyboard;

    private MacBook(Builder builder) {
        processor = builder.processor;
        memory = builder.memory;
        capacity = builder.capacity;
        keyboard = builder.keyboard;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static final class Builder {
        private String processor;
        private int memory;
        private long capacity;
        private String keyboard;

        public Builder processor(String val) {
            processor = val;
            return this;
        }

        public Builder memory(int val) {
            memory = val;
            return this;
        }

        public Builder capacity(long val) {
            capacity = val;
            return this;
        }

        public Builder keyboard(String val) {
            keyboard = val;
            return this;
        }

        public MacBook build() {
            return new MacBook(this);
        }
    }
}
```

#### Usage

```java
/**
 * @author Zhao Junjian
 */
public class Client {
    public static void main(String[] args) {
        final MacBook macBook = MacBook.builder().processor("core").memory(9481).capacity(102931L).build();
    }
}
```

