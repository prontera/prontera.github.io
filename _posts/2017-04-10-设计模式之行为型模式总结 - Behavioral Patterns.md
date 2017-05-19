---
layout:     post
title:      "设计模式之行为型模式总结"
subtitle:   "Behavioral Patterns"
author:     "Chris"
header-img: "img/post-bg-1.jpg"
tags:
    - 设计模式
---

## 前言

本篇主要使用实例代码与UML进行总结, 代码已上传至[GitOSC](http://git.oschina.net/witless/design-patterns/)

## 责任链模式/职责链模式 - [Chain of responsibility](https://en.wikipedia.org/wiki/Chain_of_responsibility_pattern)

标准的职责链: 只要有Handler处理了请求, 那么这个请求就不再被传递

功能链: 每个职责对象都会这个请求进行一定的功能处理

请求不一定会被处理, 可以在最后添加一个缺省职责对象, 表示不支持此功能的处理

在客户端组合链, 再由客户端发起请求, 此时成为外部链

在职责对象里面组合链成为内部链

#### UML

外部链的一个例子

![](/img/in-post/behavioral-patterns/chain-of-responsibility.png)

#### Usage

```java
/**
 * @author Zhao Junjian
 */
public class Client {
    public static void main(String[] args) {
        final CtoHandler cto = new CtoHandler();
        final CeoHandler ceo = new CeoHandler();
        final NobodyHandler nobody = new NobodyHandler();
        cto.setSuccessor(ceo);
        ceo.setSuccessor(nobody);
        System.out.println(cto.handleRequest("ceo"));
    }
}
```

## 命令模式 - [Command](https://en.wikipedia.org/wiki/Command_pattern)

命令模式实质上就是回调, 将请求封装为对象, 从而允许对具有不同请求的客户端进行参数化

#### UML

![](/img/in-post/behavioral-patterns/command.png)

###### ChineseCooker

真正执行命令的接收者

```java
/**
 * @author Zhao Junjian
 */
public class ChineseCooker implements Cooker {
    @Override
    public void cook(String name) {
        System.out.println(this.getClass().getSimpleName() + " is cooking " + name);
    }
}
```

###### MakeSeafoodCommand

将请求封装成了Command, 如`MakeSoupCommand`, `MakeSeafoodCommand`

```java
/**
 * @author Zhao Junjian
 */
public class MakeSeafoodCommand implements MakeFoodCommand {
    private Cooker cooker;

    public MakeSeafoodCommand(Cooker cooker) {
        this.cooker = cooker;
    }

    @Override
    public void execute() {
        cooker.cook(" seafood ");
    }
}
```

###### BatchMakeCommand

实质上是宏命令

```java
/**
 * @author Zhao Junjian
 */
public class BatchMakeCommand implements MakeFoodCommand {
    private Set<MakeFoodCommand> commands = new LinkedHashSet<>();

    public void addCommand(MakeFoodCommand command) {
        commands.add(command);
    }

    @Override
    public void execute() {
        for (MakeFoodCommand command : commands) {
            command.execute();
        }
    }
}
```

#### Usage

实际应用中可以单独分离出Invoker, 在本例中Invoker退化了与Client融为一体

```java
/**
 * @author Zhao Junjian
 */
public class Client {
    public static void main(String[] args) {
        final BatchMakeCommand batch = new BatchMakeCommand();
        final MakeSoupCommand makeSoupCommand = new MakeSoupCommand(new IndianCooker());
        batch.addCommand(makeSoupCommand);
        final MakeSeafoodCommand makeSeafoodCommand = new MakeSeafoodCommand(new ChineseCooker());
        batch.addCommand(makeSeafoodCommand);
        batch.execute();
    }
}
```

## 迭代器模式 - [Iterator](https://en.wikipedia.org/wiki/Iterator_pattern)

Java中的迭代器模式

```java
/**
 * @author Zhao Junjian
 */
public class Laptop implements Iterator<Integer> {
    private int[] cores;
    private AtomicInteger index = new AtomicInteger(0);

    public Laptop(int[] cores) {
        this.cores = Arrays.copyOf(cores, cores.length);
    }

    @Override
    public boolean hasNext() {
        return cores != null && index.get() < cores.length;
    }

    @Override
    public Integer next() {
        final int nowIndex = index.getAndAdd(1);
        return cores[nowIndex];
    }

    @Override
    public void remove() {
        throw new UnsupportedOperationException("remove");
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
        final Laptop laptop = new Laptop(new int[]{1, 2, 3, 4, 5, 6, 7, 8, 9, 0});
        while (laptop.hasNext()) {
            System.out.println(laptop.next());
        }
    }
}
```

## 中介者模式 - [Mediator](https://en.wikipedia.org/wiki/Mediator_pattern)

所有的交互都封装到中介者对象里面, 各个对象就不再需要维护这些关系了. 扩展关系的时候也只需要扩展或修改中介者对象就可以了

但是当对象较多的时候, 中介者就会变得十分臃肿, 难于管理和维护

下面是产品经理充当中介者, 研发充当使用对象的例子

#### UML

![](/img/in-post/behavioral-patterns/mediator.png)

###### ProjectManager

中介者类, 可见日后当对象较多时该类将难以维护

```java
/**
 * @author Zhao Junjian
 */
public class ProductManager implements Mediator {
    private QualityAssuranceEngineer qa;

    private BackendEngineer java;

    private FrontendEngineer angular;

    public void setQa(QualityAssuranceEngineer qa) {
        this.qa = qa;
    }

    public void setJava(BackendEngineer java) {
        this.java = java;
    }

    public void setAngular(FrontendEngineer angular) {
        this.angular = angular;
    }

    @Override
    public void communicate(Engineer engineer) {
        final Class<? extends Engineer> sourceClass = engineer.getClass();
        if (QualityAssuranceEngineer.class.isAssignableFrom(sourceClass)) {
            // 到qa部门就说明已经执行完
            System.out.println("test done!");
        } else if (BackendEngineer.class.isAssignableFrom(sourceClass)) {
            System.out.println("java done, web now");
            angular.coding();
        } else if (FrontendEngineer.class.isAssignableFrom(sourceClass)) {
            System.out.println("web done, qa now");
            qa.test();
        } else {
            throw new IllegalStateException("none");
        }
    }
}
```

###### QualityAssuranceEngineer

```java
/**
 * @author Zhao Junjian
 */
public class QualityAssuranceEngineer extends Engineer {
    public QualityAssuranceEngineer(Mediator mediator) {
        super(mediator);
    }

    public void test() {
        System.out.println(this.getClass().getSimpleName());
        getMediator().communicate(this);
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
        // 产品经理
        final ProductManager mediator = new ProductManager();
        // 后端的同事
        final BackendEngineer backendEngineer = new BackendEngineer(mediator);
        // 前端的同事
        final FrontendEngineer frontendEngineer = new FrontendEngineer(mediator);
        // QA的同事
        final QualityAssuranceEngineer qa = new QualityAssuranceEngineer(mediator);
        mediator.setAngular(frontendEngineer);
        mediator.setJava(backendEngineer);
        mediator.setQa(qa);
        // 后端的同事开始工作
        backendEngineer.coding();
    }
}
```

## 观察者模式 - [Observer](https://en.wikipedia.org/wiki/Observer_pattern) or [Publish/subscribe](https://en.wikipedia.org/wiki/Publish/subscribe)

当被观察的对象的状态更新时, 就会触发相应的动作, 主动通知订阅者.

下面是一个Github仓库更新时, 就会主动通知关注该仓库的用户的例子

#### UML

![](/img/in-post/behavioral-patterns/observer.png)

###### Subject/Observable

```java
/**
 * @author Zhao Junjian
 */
public class Github extends Observable {
    public void newRepo(String repoName) {
        setChanged();
        notifyObservers(repoName);
    }
}
```

###### Observer

```java
/**
 * @author Zhao Junjian
 */
public class User implements Observer {
    @Override
    public void update(Observable o, Object arg) {
        System.out.println("observable class: " + o.getClass().getSimpleName());
        System.out.println("content: " + arg);
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
        final User user = new User();
        final Github github = new Github();
        github.addObserver(user);
        github.newRepo("https://github.com/prontera/spring-cloud-rest-tcc");
    }
}
```

## 策略模式 - [Strategy](https://en.wikipedia.org/wiki/Strategy_pattern)

策略模式是让客户端去选择需要使用的策略算法, 这一点与状态模式大相径庭. 状态模式一般由Context负责维护状态数据, 客户端不进行状态的指定.

策略模式中的策略是可以相互替代的, 而状态模式因为状态不一样所以对应的功能类之间是**绝不可**被替代的

#### UML

![](/img/in-post/behavioral-patterns/strategy.png)

#### Usage

```java
/**
 * @author Zhao Junjian
 */
public class Client {
    public static void main(String[] args) {
        final SqlLoggerStrategy logger = new SqlLoggerStrategy();
        final Context context = new Context(logger);
        context.someBusiness();
    }
}
```

## 状态模式 - [State](https://en.wikipedia.org/wiki/State_pattern)

策略模式与状态模式的结构几乎一样, 两者的区别已经在策略模式中说明, 不再赘述

#### UML

![](/img/in-post/behavioral-patterns/state.png)

#### Usage

```java
/**
 * @author Zhao Junjian
 */
public class Client {
    public static void main(String[] args) {
        final Context context = new Context();
        context.login();
        context.login();
        context.login();
        context.login();
        context.login();
    }
}
```

## 模板方法模式 - [Template method](https://en.wikipedia.org/wiki/Template_method_pattern)

通过定义公用模板达到复用的作用

模板方式模式与工厂方法模式的区别在于, 模板方法用于定义公用的函数与流程, 工厂方法只用于生成对象

#### UML

![](/img/in-post/behavioral-patterns/template.png)

#### Usage

```java
/**
 * @author Zhao Junjian
 */
public class Client {
    public static void main(String[] args) {
        final SwaggerConfiguration configuration = new SwaggerConfiguration();
        configuration.afterProperties();
    }
}
```

## 访问者模式 - [Visitor](https://en.wikipedia.org/wiki/Visitor_pattern)

在不修改指定对象结构的情况下, 给指定对象添加新的功能, 其核心思想是多路分发

下面是为MacBook和Xps13添加Touch Bar

#### UML

![](/img/in-post/behavioral-patterns/visitor.png)

###### TouchBarVisitor

```java
/**
 * @author Zhao Junjian
 */
public class TouchBarVisitor implements Visitor {
    @Override
    public void visit(Laptop laptop) {
        // 可能会调用一下Laptop的方法
        System.out.println(laptop.getClass().getSimpleName() + " got touch bar now!");
    }
}
```

###### MacBook

预留了回调方法accept()

```java
/**
 * @author Zhao Junjian
 */
public class MacBook extends Laptop {
    @Override
    public void accept(Visitor visitor) {
        visitor.visit(this);
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
        final TouchBarVisitor visitor = new TouchBarVisitor();
        final Xps13 xps13 = new Xps13();
        final MacBook macBook = new MacBook();
        xps13.accept(visitor);
        macBook.accept(visitor);
    }
}
```