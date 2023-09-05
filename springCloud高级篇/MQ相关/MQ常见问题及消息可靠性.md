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



### 二、死信交换机

#### 1. 初始死信交换机

当一个队列中的消息满足下列情况之一时，可以成为死信

* 消费者使用basic.reject或basic.nack声明消费失败，并且消息的requeue参数设置为false
* 消息是一个过期消息，超时无人消费
* 要投递的队列消息堆积满了，最早的消息可能成为死信

如果该队列配置了dead-letter-exchange属性，指定了一个交换机，那么队列中的死信就会投递到这个交换机中，而这个交换机称为死信交换机

如何绑定死信交换机：

* 给队列设置dead-letter-exchange属性，指定一个交换机
* 给队列设置dead-letter-routing-key属性，设置死信交换机与死信队列的RoutingKey



#### 2. TTL

TTL，即Time-To-Live。如果一个队列中的消息TTL结束仍未消费，则会变为死信，TTL超时分为两种情况：

* 消息所在队列设置了存活时间
* 消息本身设置了存活时间





**延迟队列**

利用TTL结合死信交换机，我们实现了消息发出后，消费者延迟收到消息的效果。

延迟队列使用场景：

* 延迟发送短信
* 用户下单，如果用户在15钟内未支付，则自动取消（取消订单这个动作是消费者）
* 预约工作会议，20min后自动通知所有参会人员



### 三、消息堆积

当消费者发送消息速度大于消费者处理消息速度，就会导致队列中的消息堆积，直到队列存储消息达到上限。最早接收到的消息，可能成为死信，会被丢弃，这就是消息堆积问题

解决消息堆积的三种思路：

* 增加更多消费者，提高消费速度
* 在消费者内开启线程池加快消息处理速度
* 扩大队列容积，提高堆积上限

**惰性队列：**

* 接收到消息后直接存入磁盘而非内存
* 消费者要消费消息时才会从磁盘中读取并加载到内存
* 支持数百万条的消息存储

**注：**要设置一个队列为惰性队列，只需要在声明队列时，指定x-queue-mode属性为lazy即可。可以通过命令行将一个运行中的队列修改为惰性队列



### 四、集群分类

RabbitMQ的集群有两种模式

* **普通集群：**一种分布式集群，将队列分散到集群各个节点，从而提高集群的并发能力
* **镜像集群：**一种主从集群，普通集群的基础上，添加了主从备份功能，提高集群的数据可用性

镜像集群虽然支持主从，但主从同步并**不是强一致**的，某些情况下可能有数据丢失的风险。因此在RabbitMQ3.8版本以后，推出了新功能：**仲裁队列**来代替镜像集群，底层采用**Raft协议**确保主从数据一致性。

**普通集群：**

* 会在集群各个节点间共享部分数据，包括：交换机、队列元信息。不包含队列中的消息
* 当访问集群某节点时，如果队列不在该节点，会从数据所在节点传递到当前节点并返回（数据传递）
* 队列所在节点宕机，队列中的消息丢失

**镜像集群：**

* 交换机、队列、队列中的消息会在各个mq的镜像之间同步备份
* 创建队列的节点被称为该队列的**主节点**，备份到的其他节点称为**镜像节点**
* 一个节点的主节点可能是另一个队列的镜像节点
* 所有操作都是主节点完成，然后同步给镜像节点
* 主机宕机，镜像节点对替代成为新的主节点

**仲裁模式：**

* 与镜像队列一样，都是主从模式，支持主从数据同步
* 使用非常简单，没有复杂配置
* 主从同步基于raft协议，强一致

