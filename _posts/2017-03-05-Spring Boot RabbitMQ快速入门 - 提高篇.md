---
layout:     post
title:      "Spring Boot RabbitMQ快速入门 - 提高篇"
subtitle:   "prefetch, ack, retry and dead letter queue"
author:     "Chris"
header-img: "img/post-bg-8.jpg"
tags:
    - Spring Boot
    - RabbitMQ
---

## Prefetch设置

当我们进入RabbitMQ的GUI管理界面, 点入某个队列查看消费者的属性时, 有记录如下

| Channel                                  | Consumer tag                    | Ack required | Exclusive | Prefetch count | Arguments |
| ---------------------------------------- | ------------------------------- | ------------ | --------- | -------------- | --------- |
| [172.22.0.1:57382 (1)](http://localhost:15672/#/channels/172.22.0.1%3A57382%20-%3E%20172.22.0.3%3A5672%20(1)) | amq.ctag-Gsix2DEjaFI9zVlsJJZp3Q | ●            | ○         | 1              |           |
| [172.22.0.1:57378 (1)](http://localhost:15672/#/channels/172.22.0.1%3A57378%20-%3E%20172.22.0.3%3A5672%20(1)) | amq.ctag-_FIcIOpflMXXaBQN7xLYcA | ●            | ○         | 1              |           |

上面的表格说明消息的消费需要手工ack, 且是公平分发的. 设置prefetch的方式有两种

- 全局式设定

  在application.yml文件中设定`spring.rabbitmq.listener.prefetch`即可, 这会影响到本Spring Boot应用中所有使用默认`SimpleRabbitListenerContainerFactory`的消费者

  ```yaml
  spring:
    rabbitmq:
      host: localhost
      username: chris
      password: 123123
      virtual-host: prontera
      listener:
        prefetch: 100
  ```

- 特定消费者设置

  在消费者的配置中自定义一个`SimpleRabbitListenerContainerFactory`

  ```java
  @Bean
  public SimpleRabbitListenerContainerFactory myContainerFactory(
    SimpleRabbitListenerContainerFactoryConfigurer configurer, 
    ConnectionFactory connectionFactory) {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setPrefetchCount(100);
    configurer.configure(factory, connectionFactory);
    return factory;
  }

  ```

  然后在消费者上声明使用该ContainerFactory即可达到对特定消费者配置prefetch的作用

  ```java
  @RabbitListener(queues = "#{rabbitConfiguration.TOPIC_QUEUE}", containerFactory = "myContainerFactory")
  public void processBootTask2(WorkUnit content) {
    System.out.println(content);
  }

  ```

## Ack机制

Spring Boot Rabbit使用手工应答机制, 当@RabbitListener修饰的方法被调用且没有抛出异常时, Spring Boot会为我们自动应答. 否则会根据设定的重试机制而作出nack或reject等行为.

## 重试机制

重试分两种, template的重试与listener的重试, 分别代表生产者与消费者

### 生产者端的重试

```Yaml
spring:
  rabbitmq:
    template:
      retry:
        enabled: true
```

通过以上配置可以启动`AmqpTemplate`的重试机制, 例如与RabbitMQ连接丢失的时候将会自动重试事件的发布, 这个特性默认是关闭的

### 消费者端的重试

消费者一端, 即`@RabbitListener`也有像`AmqpTemplate`一样的重试机制, 当重试次数(默认是3)耗尽的时候, 该特性同样也是默认关闭的, 可以通过以下配置打开

```yaml
spring:
  rabbitmq:
    host: localhost
    username: chris
    password: 123123
    virtual-host: prontera
    listener:
      retry:
        enabled: true
```

如果消费一端的重试机制没有被启动, 而且Listener抛出异常的话, 那么该消息就会**无限地**被重试(刚开始我也晕, retry都关了居然会被无限地重试, 这个不是bug, 官方文档就是这么写的, 实测结果也是一样). 通常我们的做法是抛出`AmqpRejectAndDontRequeueException`以reject该消息, 同时如果有dead-letter queue被设置的话该消息就会被置入, 否则被丢弃.

如果启动消费端的重试机制, 我们可以设置其最大的尝试次数(默认为3次)

```yaml
spring:
  rabbitmq:
    listener:
      retry:
        enabled: true
        max-attempts: 5
```

### 死信队列

```java
@Bean
public DirectExchange directExchange() {
  return new DirectExchange(DEFAULT_DIRECT_EXCHANGE, true, true);
}

@Bean
public Queue tradeQueue() {
  final ImmutableMap<String, Object> args = 
    ImmutableMap.of("x-dead-letter-exchange", DEFAULT_DIRECT_EXCHANGE,
                    "x-dead-letter-routing-key", TRADE_DEAD_ROUTE_KEY);
  return new Queue(TRADE_QUEUE, true, false, true, args);
}

@Bean
public Binding tradeBinding() {
  return BindingBuilder.bind(tradeQueue()).to(directExchange()).with(TRADE_ROUTE_KEY);
}

@Bean
public Queue deadLetterQueue() {
  return new Queue(TRADE_DEAD_QUEUE, true, false, true);
}

@Bean
public Binding deadLetterBinding() {
  return BindingBuilder.bind(deadLetterQueue()).to(directExchange()).with(TRADE_DEAD_ROUTE_KEY);
}
```

## 队列定义不一致

对于已经存在的Queue配置将不会被后来的覆盖, 且会在Spring Boot控制台抛出一条WARN日志

```shell
Caused by: com.rabbitmq.client.ShutdownSignalException: channel error; protocol method: #method<channel.close>(reply-code=406, reply-text=PRECONDITION_FAILED - inequivalent arg 'durable' for queue 'boot_task' in vhost 'prontera': received 'false' but current is 'true', class-id=50, method-id=10)
```

## Spring Boot RabbitMQ Properties

```properties
# RABBIT (RabbitProperties)
spring.rabbitmq.addresses= # Comma-separated list of addresses to which the client should connect.
spring.rabbitmq.cache.channel.checkout-timeout= # Number of milliseconds to wait to obtain a channel if the cache size has been reached.
spring.rabbitmq.cache.channel.size= # Number of channels to retain in the cache.
spring.rabbitmq.cache.connection.mode=CHANNEL # Connection factory cache mode.
spring.rabbitmq.cache.connection.size= # Number of connections to cache.
spring.rabbitmq.connection-timeout= # Connection timeout, in milliseconds; zero for infinite.
spring.rabbitmq.dynamic=true # Create an AmqpAdmin bean.
spring.rabbitmq.host=localhost # RabbitMQ host.
spring.rabbitmq.listener.acknowledge-mode= # Acknowledge mode of container.
spring.rabbitmq.listener.auto-startup=true # Start the container automatically on startup.
spring.rabbitmq.listener.concurrency= # Minimum number of consumers.
spring.rabbitmq.listener.default-requeue-rejected= # Whether or not to requeue delivery failures; default `true`.
spring.rabbitmq.listener.max-concurrency= # Maximum number of consumers.
spring.rabbitmq.listener.prefetch= # Number of messages to be handled in a single request. It should be greater than or equal to the transaction size (if used).
spring.rabbitmq.listener.retry.enabled=false # Whether or not publishing retries are enabled.
spring.rabbitmq.listener.retry.initial-interval=1000 # Interval between the first and second attempt to deliver a message.
spring.rabbitmq.listener.retry.max-attempts=3 # Maximum number of attempts to deliver a message.
spring.rabbitmq.listener.retry.max-interval=10000 # Maximum interval between attempts.
spring.rabbitmq.listener.retry.multiplier=1.0 # A multiplier to apply to the previous delivery retry interval.
spring.rabbitmq.listener.retry.stateless=true # Whether or not retry is stateless or stateful.
spring.rabbitmq.listener.transaction-size= # Number of messages to be processed in a transaction. For best results it should be less than or equal to the prefetch count.
spring.rabbitmq.password= # Login to authenticate against the broker.
spring.rabbitmq.port=5672 # RabbitMQ port.
spring.rabbitmq.publisher-confirms=false # Enable publisher confirms.
spring.rabbitmq.publisher-returns=false # Enable publisher returns.
spring.rabbitmq.requested-heartbeat= # Requested heartbeat timeout, in seconds; zero for none.
spring.rabbitmq.ssl.enabled=false # Enable SSL support.
spring.rabbitmq.ssl.key-store= # Path to the key store that holds the SSL certificate.
spring.rabbitmq.ssl.key-store-password= # Password used to access the key store.
spring.rabbitmq.ssl.trust-store= # Trust store that holds SSL certificates.
spring.rabbitmq.ssl.trust-store-password= # Password used to access the trust store.
spring.rabbitmq.ssl.algorithm= # SSL algorithm to use. By default configure by the rabbit client library.
spring.rabbitmq.template.mandatory=false # Enable mandatory messages.
spring.rabbitmq.template.receive-timeout=0 # Timeout for `receive()` methods.
spring.rabbitmq.template.reply-timeout=5000 # Timeout for `sendAndReceive()` methods.
spring.rabbitmq.template.retry.enabled=false # Set to true to enable retries in the `RabbitTemplate`.
spring.rabbitmq.template.retry.initial-interval=1000 # Interval between the first and second attempt to publish a message.
spring.rabbitmq.template.retry.max-attempts=3 # Maximum number of attempts to publish a message.
spring.rabbitmq.template.retry.max-interval=10000 # Maximum number of attempts to publish a message.
spring.rabbitmq.template.retry.multiplier=1.0 # A multiplier to apply to the previous publishing retry interval.
spring.rabbitmq.username= # Login user to authenticate to the broker.
spring.rabbitmq.virtual-host= # Virtual host to use when connecting to the broker.
```
