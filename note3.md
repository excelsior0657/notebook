### 一、String、StringBuffer和StringBuilder的区别

1. String是一个不可变类，每次修改String的值的时候都**会产生一个新的类**。而StringBuffer和StringBuilder是可变类
2. String和StringBuffer是线程安全的（StringBuffer的每个方法都用了一个synchronized同步关键字），**StringBuilder是线程不安全的**
3. 性能方面：StringBuilder > StringBuffer > String
4. 储存方面：**String存储在常量池**里，StringBuilder和StringBuffer存储在堆内存空间里



### 二、Redis和MySQL如何保证数据的一致性

一般情况下，Redis是应用层和数据库之间的一个读操作的**缓存层**，主要目的是**减少数据库的IO，还可以提升数据的IO性能**

当应用层需要读取数据时，首先会尝试去Redis里加载，如果有数据则直接返回，否则，直接从数据库里查询，查询到后再缓存到Redis里

1. 先更新数据库，在更新缓存。如果更新缓存失败，则会导致数据不一致
2. 先删除缓存，在更新数据库。一般情况下，数据是一致的，但由于删除缓存和更新数据库并不是原子的，在极端情况下（如其他线程来访问），也可能出现数据不一致。
3. 采用最终一致性的方案 （1）基于RocketMQ的可靠性消息通信。（2）通过Canal组件，监控MySQL的binlog日志，把更新后的数据同步到Redis中。由于是基于最终一致性实现的，数据在短期是不一致的。



### 三、@Resource和@Autowired的区别

@Autowire是**根据类型**来匹配， @Resource**可以根据name和type来匹配**

@Resource可以支持ByName好ByType两种注入方式，如果没有声明用哪种方式注入，会先根据定义的属性名字去匹配，如果没有匹配到，则会根据类型去匹配。如果都没匹配到则会报错。

``` java
@Resource(name = "userService1")
private UserService1 userService1;

@Resource(type = UserService2.class)
private UserService2 userService2;
```



### 四、Spring IOC和DI

Spring IOC即控制反转--Inversion of Control, 对象的创建和查找依赖、对象的控制全部交给IOC容器来管理。（这样的设计理念使得对象与对象之间，**降低耦合性，提升了程序的灵活性和复用性**）

IOC它是一个bean的容器，可以通过xml或者注解的方式把bean装载到IOC容器里，取bean的时候直接去IOC容器获取即可

DI表示依赖注入，对于IOC容器管理的bean,如果bean之间存在依赖关系，那么IOC容器需要自动去实现依赖对象的实力注入。

三种注入方式（关系？）：

1. 接口注入
2. setter注入
3. 构造器注入



### 五、SpringBoot中自动装配机制的原理

自动装配，就是**自动把第三方组件的bean装载到IOC容器里**，不需要开发人员再去写bean相关的一个配置。在SpringBoot里只需要在启动类上加上@SpringBootApplication注解就可以实现自动装配。@SpringBootApplication注解是一个复合注解，真正实现自动装配的注解是@EnableAutoConfiguration

例如：如果依赖了Starter启动依赖，它会自动把启动依赖里相关的一些bean装载到容器里，这样就可以直接通过一个DI的方式去完成依赖注入的动作

