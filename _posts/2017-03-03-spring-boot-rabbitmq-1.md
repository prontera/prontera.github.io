---
layout:     post
title:      "Spring Boot RabbitMQ快速入门 - 基础篇"
subtitle:   "direct, topic and fanout"
author:     "Chris"
header-img: "img/post-bg-1.jpg"
tags:
    - Spring Boot
    - RabbitMQ
---

## Preface

Spring Boot集成RabbitMQ, 其属性可直接通过application.yml中的`spring.rabbitmq.*`前缀配置. 

Sprint Boot RabbitMQ的消费者默认是Fair dispatch, 即prefetch=1

*为了方便调试, 我将所有Exchange与Queue设置为auto delete.*

以下所有示例基于`spring-boot-starter-amqp 1.4.4.RELEASE`, 若在未来的版本中有更加优雅的使用方法可以给我留言哈.

### Prerequisite

`application.yml`中先定义账户密码等连接信息, 确保连接成功后再进行下一步操作

```yaml
spring:
  rabbitmq:
    host: localhost
    username: chris
    password: 123123
    virtual-host: prontera
```

## Main Concepts

org.springframework.amqp.core.Queue: 队列

org.springframework.amqp.core.Binding: 建立交换器与队列的绑定关系

org.springframework.amqp.core.DirectExchange: Direct交换器

org.springframework.amqp.core.TopicExchange: Topic交换器

org.springframework.amqp.core.FanoutExchange: Fanout交换器

org.springframework.amqp.support.converter.MessageConverter: 消息转换器, 如将Java类转换JSON类型发送至Broker, 从Broker处获取JSON消息转换为Java类型

org.springframework.amqp.core.AmqpTemplate 多用于生产者端发布消息

org.springframework.amqp.core.AmqpAdmin 用于Exchange, Queue等的**动态**管理

## Configuration

### Work queues

![img](http://www.rabbitmq.com/img/tutorials/python-two.png)

生产者与消费者的配置一样, 因为监听的是同一个队列, 所以队列名要先约定好

```java
/**
 * @author Zhao Junjian
 */
@Configuration
public class RabbitConfiguration {

    public static final String DEFAULT_DIRECT_EXCHANGE = "prontera.direct";
    public static final String TRADE_QUUE = "funds";
    public static final String TRADE_ROUTE_KEY = "trading";

    @Bean
    public DirectExchange pronteraExchange() {
        return new DirectExchange(DEFAULT_DIRECT_EXCHANGE, true, true);
    }

    @Bean
    public Queue tradeQueue() {
        return new Queue(TRADE_QUEUE, true, false, true);
    }

    @Bean
    public Binding tradeBinding() {
        return BindingBuilder.bind(tradeQueue()).to(pronteraExchange()).with(TRADE_ROUTE_KEY);
    }

    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }

}
```

生产消息, 这里我是借助Controller发起的请求, 无论怎样只要能注入AmqpTemplate就行

```java
@Autowired
private AmqpTemplate amqpTemplate;

@RequestMapping(value = "/echo", method = RequestMethod.GET)
public Map<String, ?> hello() {
  final WorkUnit unit = new WorkUnit();
  unit.setId("1");
  unit.setMessage("hello world");
  amqpTemplate.convertAndSend(RabbitConfiguration.DEFAULT_DIRECT_EXCHANGE, RabbitConfiguration.TRADE_ROUTE_KEY, unit);
  return ImmutableMap.of("code", 20000);
}
```

消费消息

```java
@RabbitListener(queues = {RabbitConfiguration.TRADE_QUEUE})
public void processBootTask(WorkUnit content) {
  System.out.println(content);
}
```

> NOTE: 
>
> 如果先启动生产者, 那么要先与RabbitMQ进行过通讯之后, 才会在RabbitMQ Management处看到Exchange与Queue. 如果是先启动消费者, 那么@RabbitListener会自动监听队列, 所以可以直接在RabbitMQ Management处看到我们所定义的组件

### Publish/Subscribe

![img](http://www.rabbitmq.com/img/tutorials/python-three-overall.png)

要使用Fanout的话那么两端的配置就不一样了, 根据AMQP规范, 生产者其实仅关注Exchange与Route Key, 消费者仅关注Queue, 根据这条规则我们来写一下生产端的配置

```java
/**
 * @author Zhao Junjian
 */
@Configuration
public class RabbitConfiguration {

    public static final String DEFAULT_FANOUT_EXCHANGE = "prontera.fanout";

    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    public FanoutExchange fanoutExchange() {
        return new FanoutExchange(DEFAULT_FANOUT_EXCHANGE, true, true);
    }

}
```

消费端我们就需要将Exchange与临时Queue进行绑定, 这里我们使用UUID为Queue命名

```java
/**
 * @author Zhao Junjian
 */
@Configuration
public class RabbitConfiguration {

    public static final String DEFAULT_FANOUT_EXCHANGE = "prontera.fanout";
    public static final String FANOUT_QUEUE = "p-" + UUID.randomUUID();

    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    public FanoutExchange fanoutExchange() {
        return new FanoutExchange(DEFAULT_FANOUT_EXCHANGE, true, true);
    }

    @Bean
    public Queue randomQueue() {
        return new Queue(FANOUT_QUEUE, true, false, true);
    }

    @Bean
    public Binding fanoutBinding() {
        return BindingBuilder.bind(randomQueue()).to(fanoutExchange());
    }

}
```

其实也可以使用UniquelyNamedQueue, 只不过我个人觉得没那么顺手而已

以下是生产端发送消息的示例, Route Key "chris"在Fanout Exchange中是会被忽略的, 这里只是提醒一下自己

```java
@Autowired
private AmqpTemplate amqpTemplate;

@RequestMapping(value = "/echo", method = RequestMethod.GET)
public Map<String, ?> hello() {
  // fanout
  amqpTemplate.convertAndSend(RabbitConfiguration.DEFAULT_FANOUT_EXCHANGE, "chris", unit);
  return ImmutableMap.of("code", 20000);
}

```

消费端获取消息

```java
@RabbitListener(queues = "#{rabbitConfiguration.FANOUT_QUEUE}")
public void processBootTask(WorkUnit content) {
  System.out.println(content);
}
```

需要注意的是, 因为注解中的值必须是常量, 如果我们直接写`RabbitConfiguration.FANOUT_QUEUE`是会抛出编译期的异常, 而Spring则为该注解进行SpEL扩展使其支持动态变量. 因为Topic与Fanout都必须使用临时队列(随机且唯一的对列名才有意义), 所以这里我们使用UUID为其命名.

## Routing

![img](http://www.rabbitmq.com/img/tutorials/python-four.png)

与Work queues一样使用Direct Exchange, 就是消费者监听的队列不是同一个, 这里就不作演示了

### Topics

![img](http://www.rabbitmq.com/img/tutorials/python-five.png)

生产端配置

```java
/**
 * @author Zhao Junjian
 */
@Configuration
public class RabbitConfiguration {

    public static final String DEFAULT_TOPIC_EXCHANGE = "prontera.topic";
    public static final String TOPIC_ROUTE_KEY = "NYSE.TECH.MSFT";

    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    public TopicExchange topicExchange() {
        return new TopicExchange(DEFAULT_TOPIC_EXCHANGE, true, true);
    }
}
```

消费端同样跟Fanout一样也是配置临时队列

```java
/**
 * @author Zhao Junjian
 */
@Configuration
public class RabbitConfiguration {

    public static final String DEFAULT_TOPIC_EXCHANGE = "prontera.topic";
    public static final String TOPIC_QUEUE = "p-" + UUID.randomUUID();
    public static final String TOPIC_ROUTE_KEY = "#.#";	// 下表有例子

    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    public TopicExchange topicExchange() {
        return new TopicExchange(DEFAULT_TOPIC_EXCHANGE, true, true);
    }

    @Bean
    public Queue randomQueue() {
        return new Queue(TOPIC_QUEUE, true, false, true);
    }

    @Bean
    public Binding topicBinding() {
        return BindingBuilder.bind(randomQueue()).to(topicExchange()).with(TOPIC_ROUTE_KEY);
    }
}
```

生产端发送请求, 其实就是指定Exchange与Route Key, 对于Queue来说生产者不关心

```java
@Autowired
private AmqpTemplate amqpTemplate;

@RequestMapping(value = "/echo", method = RequestMethod.GET)
public Map<String, ?> hello() {
  final WorkUnit unit = new WorkUnit();
  unit.setId("1");
  unit.setMessage("hello world");
  amqpTemplate.convertAndSend(RabbitConfiguration.DEFAULT_TOPIC_EXCHANGE, RabbitConfiguration.TOPIC_ROUTE_KEY, unit);
  return ImmutableMap.of("code", 20000);
}
```

消费端跟Fanout中的例子是一样的

```java
@RabbitListener(queues = "#{rabbitConfiguration.TOPIC_QUEUE}")
public void processBootTask(WorkUnit content) {
  System.out.println(content);
}
```

生产端指定Route Key为NYSE.TECH.MSFT, 下面是消费端绑定Route Key的不同情况

| BINDING KEY ON CONSUMER SIDE | MATCH? |
| ---------------------------- | ------ |
| NYSE.TECH.MSFT               | Yes    |
| #                            | Yes    |
| NYSE.#                       | Yes    |
| *.*                          | No     |
| NYSE.*                       | No     |
| NYSE.TECH.*                  | Yes    |
| NYSE.*.MSFT                  | Yes    |

## 小结

本篇介绍了Spring Boot RabbitMQ的常用模型, 在下篇将会介绍prefetch, dead-letter与重试机制.