**MQ的一些常见问题**

* 消息可靠性问题（消息可靠性）：如何确保发送的消息只被消费一次
* 消息堆积问题（惰性队列）：如何解决数百万消息堆积，无法及时消费的问题
* 延迟消息问题（死信交换机）：如何实现消息的延迟投递
* 高可用问题（MQ集群）：如何避免单点的MQ故障而导致的不可用问题



消息丢失的情况：

* 发送时丢失：
  * 生产者发送的消息未到达exchange
  * 消息到达exchange后未到达queue
* MQ宕机，queue将消息丢失
* consumer接收到消息后未消费就宕机



### 一、消息可靠性

#### 1. 生产者消息确认

RabbitMQ提供了publisher confirm机制来避免消息发送到MQ过程中丢失。消息发送到MQ以后，会返回一个结果给发送者，表示消息是否处理成功。结果有两种请求：

* publisher-confirm，发送者确认
  * 消息成功投递到交换机，返回ack
  * 消息未投递到交换机，返回nack
* publisher-return，发送者回执
  * 消息投递到交换机了，但是没有路由到队列。返回ACK，及路由失败原因

**注：确认机制发送消息时，需要给每个消息设置一个全局唯一的id，以区分不同消息，避免ack冲突**

#### 2. 消息持久化

MQ默认是内存存储消息，开启持久化功能可以确保缓存在MQ中的消息不丢失

**在SpringAMQP封装了消息持久化功能，因此交换机，队列，消息默认都是持久的**

1. 交换机持久化：

   ```java
   @Bean
   public DirectExchange simpleExchange(){
       // 三个参数：交换机名称，是否持久化，当没有queue与其绑定时是否自动删除
       return new DirectExchange("simpleExchange", true, false);
   }
   ```

2. 队列持久化：

   ```java
   @Bean
   public Queue simpleQueue(){
       // 使用QueueBuilder构建队列，durable就是持久化的
       return QueueBuilder.durable("simple.queue").build();
   }
   ```

3. 消息持久化：SpringAMQP中的消息默认是持久的，可以通过MessageProperties中的DeliveryMode来指定

   ```java
   Message msg = Message.getBytes(StandardCharsets.UTF_8)//消息体
                        .setDeliveryMode(MessageDeliveryMode.PERSISTENT)//持久化                  
                        .build();
   ```

   

#### 3. 消费者消息确认

RabbitMQ支持消费者确认机制，即：消费者处理消息后可以向MQ发送ack回执，MQ收到ack回执后才会删除消息。SpringAMQP允许配置三种确认模式：

* manual：手动ack，需要业务代码结束后，调用api发送ack
* auto：自动ack，由spring监测listener代码是否出现异常，没有异常则返回ack，抛出异常则返回nack
* none：关闭ack，MQ假定消费者获取消息后会成功处理，因此消息投递后立即被删除

配置方式：

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch:
          acknowledge-mode: none # none, munual, auto
```



#### 4. 消费失败重试机制

消费者失败重试：当消费者出现异常后，消息会不断requeue（重新入队），再重新发送给消费者，然后再次异常，再次requeue，无限循环，导致mq的消息处理飙升，带来不必要的压力

**解决办法：**利用Spring的Retry机制，在消费者出现异常时利用**本地重试**，而不是无限制的requeue到mq队列

配置：

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch:
        retry:
          enabled: true # 开启消费者失败重试
          initial-interval: 1000 # 初始的失败等待时长为1s
          multiplier: 1 # 下次失败等待时长倍数
          max-attempts: 3 # 最大重试次数
          stateless: true # true无状态，false有状态，如果业务中包含事务，这里改为false
```

**消费者失败消息处理策略**

在开启重试模式后，重试次数耗尽，如果消息依然失败，则需要有MessageRecoverer接口来处理，它包含三种不同的实现：

* RejectAndDontRequeueRecoverer：重试次数耗尽后，直接reject，丢弃消息，也是默认方式
* ImmediateRequeueMessageRecoverer：重试次数耗尽后，返回nack，消息重新入队
* RepublishMessageRecoverer：重试耗尽后，将失败的消息投递到指定的交换机



