### 一、Eureka



### 二、Ribbon



### 三、Nacos

#### 1. Nacos注册中心

1. **Nacos服务分级存储模型**

   一级是服务，例如userservice

   二级是集群，例如无锡、上海

   三级是实例，例如某机房部署了userservice的服务器

2. **如何设置实例集群属性**

   修改application.yml文件，添加spring.cloud.nacos.discovery.cluster-name属性

``` yaml
spring:
    cloud:
        nacos:
          discovery:
            server-addr: localhost:8848
            cluster-name: WX
```

3. **NacosRule负载均衡**

   优先选择同集群服务列表

   本地集群找不到提供者，才会去其他集群找

   确定了可用实例列表后，采用随机负载均衡挑选实例

``` yaml
userservice:
  ribbon:
    NFLoadBalancerRuleClassName: com.alibaba.cloud.nacos.ribbon.NacosRule # 负载均衡
```

4. **实例的权重控制**

   Nacos控制台设置实例的权重，0~1之间

   权重设置为0完全不会被访问

5. **环境隔离**

   namespace用于环境隔离

   每个namespace都有唯一id

   不同namespace下的服务不可见

   ``` yaml
     cloud:
       nacos:
         discovery:
           server-addr: localhost:8848
           cluster-name: WX
           ephemeral: false # 是否是临时实例
           namespace: bae45e1a-15ab-4d23-a24f-2c77ebdf2aa8
   ```

6. **Nacos和eureka**

   **共同点：**

   都支持服务注册和服务拉取

   都支持服务提供者心跳方式做健康检测

   **区别：**

   Nacos支持服务端主动检测提供者状态（临时实例采用心跳模式，非临时实例采用主动检测模式）

   临时实例心跳不正常会被剔除，非临时实例则不会被剔除

   Nacos支持服务列表变更的消息推送模式，服务列表更新更及时

   Nacos集群默认采用AP模式，当集群中存在非临时实例时，采用CP模式；eureka采用AP模式

   

#### 2. Nacos配置管理

1. **微服务配置拉取**

   在Nacos中添加配置文件

   在微服务中引入nacos的config依赖

   在微服务中添加bootstrap.yml，配置Nacos地址、当前环境、服务名称、文件后缀名。（这决定了程序启动时去nacos读取哪个文件）

   **读取配置顺序：bootstrap.yml > nacos中的配置 > 本地配置application.yml**

2. **配置实现热更新的两种方式**

   一、通过@Value注解注入，结合@RefreshScope来刷新

   二、通过@ConfigurationProperties注入，自动刷新（推荐使用）

``` java
// 1 通过@Value注解注入，结合@RefreshScope来刷新
@RefreshScope
public class UserController {
    @Value("${pattern.dateformat}")
    private String dateformat;
}

// 2 通过@ConfigurationProperties注入，自动刷新
// 创建Properties类
@Data
@ConfigurationProperties(prefix = "pattern")
@Component
public class PatternProperties {
    private String dateformat;
}
// 注入PatternProperties
@Slf4j
@RestController
@RequestMapping("/user")
//@RefreshScope
public class UserController {
    @Autowired
    private PatternProperties properties;

}
```

3. **多服务共享配置**

   微服务会从nacos读取的配置文件：

   服务名-profile.yaml和服务名.yaml

   多种配置优先级：

   **服务名-profile.yaml(userservice-dev.yaml) > 服务名.yaml(userservice.yaml) > 本地配置(application.yaml)**

#### 3. Nacos集群搭建

搭建集群：

1. 搭建数据库，初始化数据库表结构（Nacos数据库，以集群方式启动需要nacos数据库及相关表）

2. 配置nacos集群

   配置cluster.conf，集群配置（ip+端口号）

   ``` properties
   127.0.0.1:8845
   127.0.0.1.8846
   127.0.0.1.8847
   ```

   复制nacos文件为三份分别修改application.properties

   ``` properties
   spring.datasource.platform=mysql
   
   db.num=1
   
   db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
   db.user.0=root
   db.password.0=123
   ```

3. 配置nginx.conf

   ``` nginx
   upstream nacos-cluster {
       server 127.0.0.1:8845;
   	server 127.0.0.1:8846;
   	server 127.0.0.1:8847;
   }
   
   server {
       listen       80;
       server_name  localhost;
   
       location /nacos {
           proxy_pass http://nacos-cluster;
       }
   }
   ```

4. java程序修改bootstrap.yml, 把原先Nacos端口号改为nginx端口号80（nginx反向代理）

   ``` yaml
   cloud:
       nacos:
         server-addr: localhost:8848 # Nacos地址
         
   cloud:
       nacos:
         server-addr: localhost:80 # Nacos集群地址
   ```

5. 在nacos中添加的配置会存储到之前创建的nacos数据库中，在config.info表里

**在2、3配置完成后分别启动多个nacos节点**

#### 4. Feign远程调用

1. **编写Feign客户端**，在cn.itcast.order包里创建clients包，并创建UserClient接口

   ```java
   @FeignClient("userservice")
   public interface UserClient {
       @GetMapping("/user/{id}")
       User findById(@PathVariable("id") Long id);
   }
   // 服务名称：userservice
   // 请求方式：GET
   // 请求路径：/user/{id}
   // 请求参数：Long id
   // 返回值类型User
   ```

   在orderservice里注入UserClient,调用其方法即可使用（并且在启动类上加上开启Feign注解@EnableFeignClients）

   ```java
   @Autowired
   private UserClient userClient;
   User user = userClient.queryById(order.getUserId());
   ```

   引入依赖

   添加@EnableFeignClients注解

   编写FeignClient接口

   调用接口

2. **自定义Feign配置**

   方式一、配置文件方式

   ```yaml
   feign:
     client:
       config:
         default: # default为全局配置，写服务名称则只针对单个微服务 eg:userservice
           loggerLevel: FULL
   ```

   方式二、Java代码方式，需要先声明一个bean

   ```java
   public class DefaultFeignConfiguration {
       @Bean
       public Logger.Level logLevel(){
           return Logger.Level.BASIC;
       }
   }
   
   ```

   1. 如果是全局配置，则把他放到启动类上的@EnableFeignClients注解中：

      @EnableFeignClients(defaultConfiguration = FeignConfiguration.class)

   2. 如果是局部配置，则放到@FeignClient注解中：

      @FeignClient(value = "userservice"，defaultConfiguration = FeignConfiguration.class)

3. **Feign的性能优化**

   1. 使用连接池代替默认的URLConnection

      如果要使用HttpClint替换，则需引入HttpClient的依赖，然后在配置文件里配置连接池

      ``` yaml
      feign:
        httpclient:
          enable: true # 开启Feign对HttpClient的支持
          max-connection: 200 # 最大的连接数
          max-connection-per-route: 50 # 每个路径最大连接数
      ```

      

   2. 修改日志级别为Basic或none

   

4. **Feign的最佳实践（继承和抽取）**

   方式一、继承，给消费者的FeignClient和提供者的controller定义统一的父接口作为标准

   缺点：服务紧耦合，父接口参数列表不会被继承

   方式二、抽取，将FeignClient抽取为独立模块，并且把接口有关的POJO、默认的Feign配置都放到这个模块中，提供给消费者使用

#### 5. 统一网关Gateway

1. **网关功能：**

   身份认证和权限校验

   服务路由（请求路由到微服务）和负载均衡

   请求限流

2. **网关的技术实现（gateway和zuul）**

   Zuul是基于Servlet实现，属于阻塞式编程。而SpringCloundGateway是基于Spring5提供的WebFlux，属于响应式编程的实现，具备更好的性能。

3. **搭建网关服务**

   1. 创建新的module，引入SpringCloudGateway依赖和nacos的服务发现依赖

   2. 编写路由配置及Nacos地址

      ``` yaml
      server：
        port: 10010
      spring:
        application:
          name: gateway
        cloud: 
          nacos:
            server-addr: localhost:8848 # nacos地址
          gateway:
            routes: # 网关路由配置
              - id: user-service # 路由id，自定义，唯一即可
                uri: lb://userservice # 路由的目标地址 lb就是负载均衡，后面跟服务名称
                predicates: # 路由断言，判断请求是否符合路由规则的条件
                  - Path=/user/** # 按路径匹配，只要以/user/开头就符合要求
      ```

      注：还可配置filters: 路由过滤器，处理请求或响应，uri: 路由目的地支持http和lb两种

   3. 路由断言工厂Route Predicate Factory（Spring提供了11中基本的断言工厂）

      After: 是某个时间点后的请求

      Before: 是某个时间点前的请求

      Between：是两个时间点之间的请求

      Cookie: 请求必须包含某些cookie

      Header: 请求必须包含某些header

      Host: 请求必须是访问某个host（域名）

      Method：请求必须是指定方式 (- Method=GET,POST)

      Path： 请求路径必须符合指定规则

      Query：请求必须包含指定参数

      RemoteAddr：请求的ip必须是指定范围 (- RemoteAddr=19.168.1.1/24)

      Weight：权重处理

4. **路由过滤器** **GatewayFilter**

   GatewayFilter是网关中提供的一种过滤器，可以对进入网关的请求和微服务的响应做处理

   eg：给所有进入userszervice的请求添加一个请求头：Truth=excelsior0657 is awesome!

   ``` yaml
   spring:
     cloud:
       gateway:
         routes: # 网关路由配置
           - id: user-service # 路由id，自定义，唯一即可
             uri: lb://userservice # 路由的目标地址 lb就是负载均衡，后面跟服务名称
             predicates: # 路由断言，判断请求是否符合路由规则的条件
               - Path=/user/** # 按路径匹配，只要以/user/开头就符合要求
             filter: # 过滤器
             - AddRequestHeader=Truth, excelsior0657 is awesome! # 添加请求头
         default-filters: # 默认过滤器，会对所有的请求路径都生效
           - AddRequestHeader=Truth, excelsior0657 is awesome!
   ```

5.  **全局过滤器** **GlobalFilter**

   全局过滤器的作用和Gateway一样，也是处理一切进去网关的请求和微服务的响应

   区别在于GatewayFilter通过配置定义，处理逻辑是固定的。而GlobalFilter的逻辑需要自己写代码实现。定义的方式是实现GlobalFilter接口。

   ```java
   public interface GlobalFilter{
       // 处理当前请求，有必要时，可通过{@link GatewayFilterChain}将请求交给下一个过滤器处理
       // @param exchange 请求上下文，里面可以获取Request、Respose等信息
       // @param chain 用来把请求委托给下一个过滤器
       // @return {@code Mono<Void>} 返回标示当前过滤器业务结束
       Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
   }
   ```

   ```java
   @Order(-1) // 设置过滤器优先级，值越小，优先级越高，越先执行
   @Component
   public class AutoorzeFilter implements GlobalFilter{
       @Override
       public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterCgai chain){
           // 1.获取请求参数
       ServerHttpRequest request = exchange.getRequest();
       MultiValueMap<string, String> params = request.getQueryParams();
       // 2.获取参数中的 authorization 参数
       String auth = params.getFirst("authorization");
       // 3.判断参数是否等于 admin
       if("admin".equals(auth)){
           // 是，则放行
           return chain.filter(exchange);
       }
       // 不是，则拦截
       // 设置状态码
       exchange.getResponse().setStatusCode(HttpStatus.UNAUTHRIZEO);
       // 拦截请求
       return exchange.getResponse().setComplete();
   }
   }
   ```

6. **过滤器执行顺序**

   请求进入网关会碰到三类过滤器：当前的路由过滤器、DefaultFilter、GlobalFilter

   请求路由后，会将当前的路由过滤器、DefaultFilter、GlobalFilter，合并到一个过滤器链（集合）中，排序后依次执行

   **执行顺序：**

   1. 每个过滤器都必须指定一个int类型的order值，order值越小，优先级越高，排序越靠前

   2. GlobalFilter通过实现Ordered接口，或者添加@Order注解来指定order值，有我们自己指定

   3. 路由过滤器和defaultFilter的order有Spring指定，默认是按声明顺序从1递增

      ``` yaml
      spring:
        cloud:
          gateway:
            routes: 
              - id: user-service 
                uri: lb://userservice 
                predicates: 
                  - Path=/user/** 
                filter: 
                - AddRequestHeader=Truth, excelsior0657 is awesome! # order=1
                - AddRequestHeader=Truth, excelsior0657 is awesome! # order=2
                - AddRequestHeader=Truth, excelsior0657 is awesome! # order=3
            default-filters: 
              - AddRequestHeader=Truth, excelsior0657 is awesome! # order=1
              - AddRequestHeader=Truth, excelsior0657 is awesome! # order=2
              - AddRequestHeader=Truth, excelsior0657 is awesome! # order=3
      ```

      当过滤器的值一样时，按照 defaultFilter  > 路由过滤器 > GlobalFilter 的顺序执行

7. **跨域问题处理**

   跨域：域名不一致就是跨域，主要包括：

   域名不同：www.taobao.com 和 www.taobao.org 和 www.jd.com 和 miaosha.jd.com

   域名相同，端口不同：localhost:8080 和 localhost:8081

   跨域问题：浏览器禁止请求发起者与服务端发生跨域ajax请求，请求被浏览器拦截的问题

   **网关跨域处理采用CORS方案，只需简单配置即可实现**

   ``` yaml
   spring:
     cloud:
       gateway:
         globalcors: # 全局的跨域处理
           add-to-simple-url-handler-mapping: true # 解决options被拦截的问题
           corsConfiguration:
           '[/**]':
             allowedOrigins: # 允许哪些网站的跨域请求
             - "http://localhost:8090"
             - "http://www.leyou.com"
             allowedMethods: # 允许的跨域ajax的请求方式
               - "GET"
               - "POST"
             allowedHeaders: "*" # 允许在请求头上携带信息
             allowedCredentials: true # 是否允许携带cookie
             maxAge: 360000 # 允许跨域检测的有效期
   ```

   
