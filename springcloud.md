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

