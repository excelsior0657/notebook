### 1. 同步通讯和异步通讯

**同步通讯缺点：**

1. 耦合度高：每次加入新的需求都需要修改原来的代码
2. 性能下降：调用者需要等待服务提供者响应调用时间链长=每次调用时间之和
3. 资源浪费：调用链中的每个服务在等待响应过程中，不能释放请求占用的资源
4. 级联失败：如果服务提供者出现问题，所有调用方都会跟着出现问题

**异步通信优点：**

1. 耦合度低
2. 吞吐量提升
3. 故障隔离
4. 流量削峰

**异步通信缺点：**

1. 依赖于broker的可靠性、安全性、吞吐能力
2. 架构更复杂、业务没有明显的流程线，不好追踪处理

### 2. 常见MQ技术

1. **RabbitMQ**

   公司：Rabbit

   开发语言：Erlang

   协议支持：AMQP、XMPP、SMTP、STOMP

   可用性：高

   单机吞吐量：一般

   消息延迟：**微秒级**

   消息可靠性：**高**

2. **ActiveMQ**

   公司：apache

   开发语言：java

   协议支持：AMQP、XMPP、SSTOMP、OpenWire、REST

   可用性：一般

   单机吞吐量：差

   消息延迟：毫秒级

   消息可靠性：一般

3. **RocketMQ**

   公司：阿里

   开发语言：java

   协议支持：自定义协议

   可用性：**高**

   单机吞吐量：**高**

   消息延迟：毫秒级

   消息可靠性：**高**

4. **Kafka**

   公司：apache

   开发语言：java&Scala

   协议支持：自定义协议

   可用性：**高**

   单机吞吐量：**非常高**

   消息延迟：**毫秒以内**

   消息可靠性：一般

### 3. 基本队列模型

1. 只包括publisher、queue、consumer三个角色

2. publisher发送消息代码

   建立connection

   创建channel

   利用channel声明队列

   利用channel向队列发送消息

   ```java
   public class PublisherTest {
       @Test
       public void testSendMessage() throws IOException, TimeoutException {
           // 1.建立连接
           ConnectionFactory factory = new ConnectionFactory();
           // 1.1.设置连接参数，分别是：主机名、端口号、vhost、用户名、密码
           factory.setHost("192.168.40.130");
           factory.setPort(5672);
           factory.setVirtualHost("/");
           factory.setUsername("root");
           factory.setPassword("root");
           // 1.2.建立连接
           Connection connection = factory.newConnection();
   
           // 2.创建通道Channel
           Channel channel = connection.createChannel();
   
           // 3.创建队列
           String queueName = "simple.queue";
           channel.queueDeclare(queueName, false, false, false, null);
   
           // 4.发送消息
           String message = "hello, rabbitmq!";
           channel.basicPublish("", queueName, null, message.getBytes());
           System.out.println("发送消息成功：【" + message + "】");
   
           // 5.关闭通道和连接
           channel.close();
           connection.close();
           
       }
   }
   
   ```

3. consumer消费消息代码(订阅消息的处理消息过程为**异步调用**，主程序会继续执行，不等待消息处理完成)

   建立connection

   创建channel

   利用channel声明队列

   定义consumer的消费行为handleDelivery()

   利用channel将消费者与队列绑定 

   ```java
   public class ConsumerTest {
   
       public static void main(String[] args) throws IOException, TimeoutException {
           // 1.建立连接
           ConnectionFactory factory = new ConnectionFactory();
           // 1.1.设置连接参数，分别是：主机名、端口号、vhost、用户名、密码
           factory.setHost("192.168.40.130");
           factory.setPort(5672);
           factory.setVirtualHost("/");
           factory.setUsername("root");
           factory.setPassword("root");
           // 1.2.建立连接
           Connection connection = factory.newConnection();
   
           // 2.创建通道Channel
           Channel channel = connection.createChannel();
   
           // 3.创建队列
           String queueName = "simple.queue";
           channel.queueDeclare(queueName, false, false, false, null);
   
           // 4.订阅消息
           channel.basicConsume(queueName, true, new DefaultConsumer(channel){
               @Override
               public void handleDelivery(String consumerTag, Envelope envelope,
                                          AMQP.BasicProperties properties, byte[] body) throws IOException {
                   // 5.处理消息
                   String message = new String(body);
                   System.out.println("接收到消息：【" + message + "】");
               }
           });
           // 5.关闭连接
           channel.close();
           connection.close();
           System.out.println("等待接收消息。。。。");
       }
   }
   ```

### 4. SpringAMQP

利用SpringAMQP完成基本消息队列模型

1. 引入AMQP依赖

   ```java
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-amqp</artifactId>
   </dependency>
   ```

   

2. **在publisher中编写测试方法，向simple.queue发送消息**

   2.1：在publisher服务中编写application.yaml, 添加mq连接信息

   ``` yaml
   spring:
     rabbitmq:
       host: 192.168.40.130 # 主机名
       port: 5672 # 端口
       virtual-host: / # 虚拟主机
       username: root # 用户名
       password: root # 密码
   ```

   2.2：在publisher服务中编写测试方法

   注意：要关闭消费者连接，否则queue里的消息很快就会被消费

   ```java
   @RunWith(SpringRunner.class)
   @SpringBootTest
   public class SpringAmqpTest {
       @Autowired
       private RabbitTemplate rabbitTemplate;
   
       @Test
       public void testSend(){
           String queueName = "simple.queue";
           String msg = "excelsior0657!!";
           rabbitTemplate.convertAndSend(queueName, msg);
       }
   }
   ```

3. **在consumer编写消费逻辑**

   3.1：在consumer中编写application.yml, 添加mq连接信息(和publisher一致)

   ```yaml
   spring:
     rabbitmq:
       host: 192.168.40.130 # 主机名
       port: 5672 # 端口
       virtual-host: / # 虚拟主机
       username: root # 用户名
       password: root # 密码
   ```

   

   3.2：在consumer服务中新建一个类，编写消费逻辑（注意需要添加@Component注解）

   ```java
   @Component
   public class SpringRabbitListener {
   
       @RabbitListener(queues = "simple.queue")
       public void listenSimpleQueue(String msg){
           System.out.println("消费者接收到simple.queue的消息：" + msg);
       }
   }
   ```

   

### 5. Work Queue 工作队列（一个队列绑定多个消费者）

1. publisher发送消息

   ```java
   @Test
       public void testSendWorkQueue() throws InterruptedException {
           String queueName = "simple.queue";
           String msg = "message__";
           for(int i=1;i<=100;i++){
               rabbitTemplate.convertAndSend(queueName, msg + i);
               Thread.sleep(20);
           }
       }
   ```

2. 构造两个消费者消费消息

   ```java
       @RabbitListener(queues = "simple.queue")
       public void listenSimpleWorkQueue1(String msg) throws InterruptedException {
           System.out.println("消费者1接收到的消息：" + msg + "*********" + LocalTime.now());
           Thread.sleep(20);
       }
   
       @RabbitListener(queues = "simple.queue")
       public void listenSimpleWorkQueue2(String msg) throws InterruptedException {
           System.err.println("消费者2..........接收到的消息：" + msg + "*********" + LocalTime.now());
           Thread.sleep(200);
       }
   ```

   

**消息预取机制**：当消息发送到queue时，channel会提前把消息（批量）投递给消费者

故：虽然，**两个消费者消费能力不一样。最终消费的消息量却是一样的（消息量小时）**

**改进，消费预取限制：**修改application.yml，设置preFetch的值，控制预取消息的上限

``` yaml
spring:
  rabbitmq:
    host: 192.168.40.130 # 主机名
    port: 5672 # 端口
    virtual-host: / # 虚拟主机
    username: root # 用户名
    password: root # 密码
    listener:
      simple:
        prefetch: 1 # 每次只能获取一条消息，处理完才能获取下一条消息

```

通过上述设置后，消费者即可按照能力进行消息消费**（消息量大时可以不设置）**

### 6. 发布订阅模型（FanoutExchange,DirectExchange,TopicExchange）

发布订阅模式就是允许同一消息发给多个消费者。（实现方式就是加入了exchange）

exchange类型包括：fanout(广播)，direct(路由)，topic(话题)

1. #### Fanout Exchange：将接受到的消息路由到每个跟其绑定的queue

   实现思路：创建config包，在config包里定义FanoutConfig类，实现fanout模型

   **注：exchange和queue均在FanoutConfig类里创建，一定先启动consumer运行类，否则没有创建exchange和queue，发送消息会失败**

   ```java
   // 1.在consumer服务中，利用代码声明队列、交换机、并将二者绑定
   // 2.在consumer服务中，编写两个消费方法，分别监听fanout.queue1和fanout.queue2
   // 3.在publisher编写测试方法，向itcast.fanout发送消息
   // 注意Queue的类型
   import org.springframework.amqp.core.Queue;
   @Configuration
   public class FanoutConfig {
       // itcast.fanout
       @Bean
       public FanoutExchange fanoutExchange(){
           return new FanoutExchange("itcast.fanout");
       }
       // fanout.queue1
       @Bean
       public Queue fanoutQueue1(){
           return new Queue("fanout.queue1");
       }
       // 绑定队列1到交换机
       @Bean
       public Binding fanoutBinding1(Queue fanoutQueue1, FanoutExchange fanoutExchange){
           return BindingBuilder.bind(fanoutQueue1).to(fanoutExchange);
       }
       // fanout.queue2
       @Bean
       public Queue fanoutQueue2(){
           return new Queue("fanout.queue2");
       }
       // 绑定队列2到交换机
       @Bean
       public Binding fanoutBinding2(Queue fanoutQueue2, FanoutExchange fanoutExchange){
           return BindingBuilder.bind(fanoutQueue2).to(fanoutExchange);
       }
   }
   ```

   publisher发送消息，写一个测试类发送消息

   **测试类中，只需指定交换机名，即可，无需知道队列名。消息是发到交换机，而不是队列了**

   ```java
   @Test
   public void testSendExchange() throws InterruptedException {
       // 交换机名称
       String exchangeName = "itcast.fanout";
       // 消息
       String msg = "message__exchange";
       // 发送消息
       rabbitTemplate.convertAndSend(exchangeName,"",msg);
   }
   ```

2. #### DirectExchange：将接受到的消息根据规则路由发送到指定的queue（路由模式）

   1. 每一个Queue都与Exchange设置了一个BindingKey
   2. 发布者发送消息时，指定消息的RoutingKey
   3. Exchange将消息路由到BindingKey与RouttngKey一致的队列

   **实现思路：利用@RabbitListener注解声明Exchange,  Queue, routingKey**

   （用@RabbitListener注解方式，**比上述声明bean的方式更简洁、方便**）

   **consumer实现**：在SpringRabbitListener类中定义两个消费者及其queue和exchange的绑定关系，并指定exchange类型

   在@RabbitListener注解中，**声明所需的队列和exchange以及BindingKey**，会自动创建queue和exchange和绑定关系，无需在Config类定义bean

   ```java
   @RabbitListener(bindings = @QueueBinding(
               value = @Queue(name = "direct.queue1"),
               exchange = @Exchange(name = "itcast.direct", type = ExchangeTypes.DIRECT),
               key = {"red", "blue"}
       ))
       public void listenDirectQueue1(String msg){
           System.out.println("direct.queue1："+msg);
       }
   
       @RabbitListener(bindings = @QueueBinding(
               value = @Queue(name = "direct.queue2"),
               exchange = @Exchange(name = "itcast.direct", type = ExchangeTypes.DIRECT),
               key = {"red", "yellow"}
       ))
       public void listenDirectQueue2(String msg){
           System.out.println("direct.queue2："+msg);
       }
   ```

   **publisher实现：**在测试类中指定交换机名称，并指定routingKey，与上述方式，没太大变化

   ```java
   @Test
   public void testSendDirect() throws InterruptedException {
       // 交换机名称
       String exchangeName = "itcast.direct";
       // 消息
       String msg = "message__blue";
       // 发送消息
       rabbitTemplate.convertAndSend(exchangeName,"blue",msg);
   }
   ```

3. #### TopicExchange：与DirectExchange类似，区别在于routingKey必须是多个单词的列表，并且以 '.' 分割

   eg：china.news, china.weather, japan.news, japan.weather

   queue与exchange指定bindingKey时可以使用通配符（#：代指0或多个单词，*：代指一个单词）

   **实现思路：**（与DirectExchange基本一致）

   1. 利用@RabbitListener声明Exchange,Queue,RoutingKey
   2. 在consumer服务中，编写两个消费者方法，分别监听topic.queue1和topic.queue2
   3. 在publisher中编写测试方法，向itcast.topic发送消息

   consumer实现：

   ```java
   @RabbitListener(bindings = @QueueBinding(
               value = @Queue(name = "topic.queue1"),
               exchange = @Exchange(name = "itcast.topic", type = ExchangeTypes.TOPIC),
               key = "china.#"
       ))
       public void listenTopicQueue1(String msg){
           System.out.println("topic.queue1："+msg);
       }
   
       @RabbitListener(bindings = @QueueBinding(
               value = @Queue(name = "topic.queue2"),
               exchange = @Exchange(name = "itcast.topic", type = ExchangeTypes.TOPIC),
               key = "#.news"
       ))
       public void listenTopicQueue2(String msg){
           System.out.println("topic.queue2："+msg);
       }
   ```

   publisher实现：

   ```java
   @Test
       public void testSendTopic() throws InterruptedException {
           // 交换机名称
           String exchangeName = "itcast.topic";
           // 消息
           String msg = "message__topic";
           // 发送消息
           rabbitTemplate.convertAndSend(exchangeName,"china.weather",msg);
       }
   ```

**声明队列和交换机的两种方式，1. 在xxx-config类中声明bean ，2. 在xxx-listener类中用@RabbitListener注解**

### 7. SpringAMQP-消息转换器

在SpringAMQP发送方法中，接受消息的类型是Object，也就是说我们可以发送任意对象类型的消息。SpringAMQP会帮我们序列化为字节后发送

**消息转换器：**

Spring的消息对象处理是由MessageConverter来处理的，而**默认实现**是SimpleMessageConverter，**基于JDK**的ObjectOutputStream**完成序列化**。（不推荐）

要修改只需要定义一个MessageConverter类型的Bean即可。推荐使用JSON方式序列化（传输速度更快，更简洁）。步骤如下

1. 引入两个依赖（建议在父工程中引入）

   ```java
   <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
   </dependency>
   <dependency>
         <groupId>com.fasterxml.jackson.dataformat</groupId>
         <artifactId>jackson-dataformat-xml</artifactId>
   </dependency>
   ```

2. 在publisher配置类中声明MessageConverter（启动类也是配置类，在启动类声明也可）

```java
@Bean
public MessageConverter messageConverter(){
    return new Jackson2JsonMessageConverter();
}
```

