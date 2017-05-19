---
layout:     post
title:      "设计模式之结构型模式总结"
subtitle:   "Structural Patterns"
author:     "Chris"
header-img: "img/post-bg-2.jpg"
tags:
    - 设计模式
---

## 前言

本篇依然使用Java作为核心语言来总结<u>结构型模式</u>, 重点使用UML类图以减少来回切换代码而引起的逻辑混乱性.

代码已上传至[GitOSC](http://git.oschina.net/witless/design-patterns/)

## 适配器模式 - [Adapter](https://en.wikipedia.org/wiki/Adapter_pattern)

适配器模式是将其他接口的功能进行**转换**, 本质是转调已有功能. 相对于装饰者模式, 适配器模式转换后的接口会变, 装饰者则保持同一个接口, 所以装饰者模式适合递归.

下面是将Android手机适配为假iPhone的例子

#### UML

![](/img/in-post/structural-patterns/adapter.png)

#### Usage

```java
/**
 * @author Zhao Junjian
 */
public class Client {
    public static void main(String[] args) {
        // 我们有一台普通的安卓手机
        final AndroidPhone androidPhone = new AndroidPhone();
        // 被奸商拿去改成iPhone骗小白
        final IPhoneAdaptor iPhone = new IPhoneAdaptor(androidPhone);
        // 视频通话时玛德黑屏
        iPhone.videoCall();
    }
}
```

## 桥接模式 - [Bridge](https://en.wikipedia.org/wiki/Bridge_pattern)

维基上的对该模式的描述如下

> Decouple an abstraction from its implementation allowing the two to vary independently.

读起来不太像人话, "从实现部分解耦出抽象部门, 使他们可以独立变化"

其实我们平常写的Service, ServiceImpl就是桥接模式. 桥接模式适合控制多种变量, 例如"中国人发短信", "中国发邮件", "美国人发短信", "美国人发邮件". 这里有两个变量"人"和"发布信息的方式", 这样就抽象成两个Service了.

#### UML

![](/img/in-post/structural-patterns/bridge.png)

#### Usage

```java
/**
 * @author Zhao Junjian
 */
public class Client {
    public static void main(String[] args) {
        final MessageService messageService = new EmailMessageService();
        final UserService userService = new ChineseService(messageService);
        userService.giveBirth();
    }
}
```

## 装饰者模式 - [Decorator](https://en.wikipedia.org/wiki/Decorator_pattern)

装饰者模式主要功能是以动态的透明的方式给对象添加职责, 如**增强**功能, 控制访问

下面是对MacBook添加Touch Bar并添加颜色Space Grey的例子

#### UML

![](/img/in-post/structural-patterns/decorator.png)

#### Usage

```java
/**
 * @author Zhao Junjian
 */
public class Client {
    public static void main(String[] args) {
        final MacBook macBook = new MacBook();
        final TouchBarDecorator greyMacBook = new TouchBarDecorator(macBook);
        final SpaceGreyDecorator touchBar = new SpaceGreyDecorator(greyMacBook);
        System.out.println(touchBar.feature());
    }
}
```

## 外观模式 - [Facade](https://en.wikipedia.org/wiki/Facade_pattern)

为多个组件提供一个同一的接口, 使系统更加易用.

外观模式正是体现迪米特法则, 客户端只需要外观类进行交互, 屏蔽了底层模块.

下面是Apple被夷为平地和Dell破产的例子

#### UML

![](/img/in-post/structural-patterns/facade.png)

#### Usage

```java
/**
 * @author Zhao Junjian
 */
public class Client {
    public static void main(String[] args) {
        new TroubleFacade().boom();
    }
}
```

## 代理模式

此处仅介绍Java中的动态代理, 这里就直接看代码

```java
/**
 * @author Zhao Junjian
 */
public class MacBookProxy implements InvocationHandler {
    private Laptop target;

    public Laptop getProxy(MacBook macBook) {
        this.target = macBook;
        return (Laptop) Proxy.newProxyInstance(
                macBook.getClass().getClassLoader(),
                macBook.getClass().getInterfaces(),
                this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("dynamic proxy");
        return method.invoke(target, args);
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
        final MacBook macBook = new MacBook();
        final Laptop proxy = new MacBookProxy().getProxy(macBook);
      	// shutdown()是Laptop中的方法
        proxy.shutdown();
    }
}
```

