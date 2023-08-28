**redis基本数据类型：**

* 字符串：String（存储普通字符串）
* 哈希：hash（适合存储对象）
* 列表：list（按照插入顺序排序，可以有重复元素）
* 集合：set（无序集合，没有重复元素）
* 有序集合：sorted set（有序集合，没有重复元素）

**Redis通用命令：**

* KEYS pattern：查找所有符合给定模式pattern的key
* EXISTS key：检查给定key是否存在
* TYPE key：返回key存储值的类型
* TTL key：返回给定key的剩余生存时间(TTL, time to live)，以秒为单位
* DEL key：该命令用于在key存在时删除key

**String操作命令：**

* SET key value（设定指定key的值）
* GET key（获取指定key的值）

* SETEX key seconds value（设定指定key的值，并将key的过期时间设置为seconds秒）
* SETNX key value（只有在key不存在时设置key的值）

**hash操作命令：**

* HSET key field value：将哈希表key中字段field设置为value
* HGET key field：获取存储在哈希表中指定字段的值
* HDEL key field：删除存储在哈希表中的指定字段
* HKEYS key：获取哈希表中的所有字段
* HVALS key：获取哈希表中所有值
* HGETALL key：获取哈希表中的指定key的所有字段和值

**list操作命令：**

* LPUSH key value1 [value2]：将一个或多个值插入到列表头部
* LRANGE key start stop：获取列表指定范围内的元素
* RPOP key：移除并获取列表最后一个元素
* LLEN key：获取列表长度
* BRPOP key1 [key2] timeout：移除并获取最后一个元素，如果列表没有元素会阻塞列表直至超时或发现可弹出元素为止

**set操作命令：**

* SADD key member1 [member2]：向集合添加一个或多个成员
* SMEMBERS key：返回集合中的所有成员
* SCARD key：获取集合的成员数
* SUNION key1 [key2]：返回所有给定集合的并集
* SDIFF key1 [key2]：返回所有给定集合的差集
* SREM key member1 [member2]：移除集合中一个或多个成员

**sorted set：Redis sorted set有序集合是string类型元素的集合，且不允许重复的成员。每个元素都会关联一个double类型的分数（score）。Redis通过分数来为集合中的成员从小到大排序。成员唯一，但是分数可以重复**

* ZADD  key score1 member1 [score2 member2]：向有序集合添加一个或多个成员，或者更新一存在成员的分数
* ZRANGE key start stop [WITHSCORES]：通过索引区间返回有序集合中指定区间内的成员
* ZINCRBY key increment member：有序集合中对指定成员的分数加上增量increment
* ZREM key member [member ...]：移除有序集合中一个或多个成员



#### 在Java中操作Redis

Redis的Java客户端有很多，官方推荐的有三种：

* Jedis
* Lettuce
* Redission

Spring对Redis客户端进行了整合，提供了Spring Data Redis，在Spring Boot项目中还提供了对应的starter，即spring-boot-starter-data-redis



**通过Jedis操作Redis**

首先引入在redis.clients的jedis依赖

```java
public void testRedis(){
    // 获取连接
    Jedis jedis = new Jedis("localhost", 6379);
    
    // 执行操作
    jedis.set("username", "xiaoming");
    
    //关闭连接
    jedis.close();
}
```



**通过Spring Data Redis操作Redis（使用广泛）**

Spring Data Redis中提供了一个高度分装的类：RedisTemplate，针对jedis客户端中大量api进行了归类封装，将同一类型操作封装为operation接口

* ValueOperations：简单K-V操作
* SetOperations：set类型数据操作
* ZSetOperations：zset类型数据操作
* HashOperations：针对map类型的数据操作
* ListOperations：针对list类型的数据操作

1. 引入org.springframework.boot的spring-boot-starter-data-redis

2. yml配置Redis（只要到了依赖，作了yml配置，就可以直接注入RedisTemplate）

```yaml
spring:
  # Redis相关配置
  redis:
    host: localhost
    port: 6379
    # password: 1234 redis默认没有密码，有密码则需要配置
    database: 0 # Redis默认提供了16个数据库，默认使用0号数据库，每个数据库数据相互独立
    jedis:
      # redis连接池配置
      pool:
        max-active: 0 # 最大连接数
        max-wait: 1ms # 连接池最大阻塞等待时间
        max-idle: 4 # 连接池中的最大控线连接
        min-idle: 0 # 连接池中的最小空闲连接
```

```java
@Autowired
private RedisTemplate redisTemplate;

@Test
public void testString(){
    // 直接操作，插入数据
    redisTemplate.opsForValue().set("city", "beijing");
}
```

**注意RedisTemplate在操作Redis时，对key和value作了序列化**

**对key的序列化做修改方便观察，而value的序列化一般不用修改**

**对RedisTemplate的key序列化器作修改修改默认的key序列化器，这样key就是正常的字符串了**

```java
public class RedisConfig extends CachingConfigureSupport{
    @Bean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory           connectionFactory){
        RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
        
        // 默认的key序列化器为：JdkSerializationRedisSerializer
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        
        redisTemplate.setConnectionFactory(connectionFactory);
        return redisTemplate;
    }
}
```



#### Spring Cache

Spring Cache实现了基于注解的缓存功能，只需要简单的加一个注解，就能实现缓存功能。

Spring Cache提供了一层抽象，底层可以切换不同的cache实现。具体就是通过CacheManager来同一不同缓存技术。

CacheManager是Spring提供的各种缓存技术抽象接口

针对不同的缓存技术需要实现不同的CacheManager:

* EhCacheCacheManager：使用EhCache作为缓存技术
* GuavaCacheManager：使用Google的GuavaCache作为缓存技术
* RedisCacheManager：使用Redis作为缓存技术

**Spring Cache常用注解：**

* @EnableCaching：开启缓存注解功能（放在启动类上）
* @Cacheable：在方法执行前spring先查看缓存中是否有数据，如果有则直接返回缓存数据。若没有数据，调用方法并将方法返回值放到缓存中
* @CachePut：将方法的返回值放到缓存中
* @CacheEvict：将一条或多条数据从缓存中删除

在spring boot项目中，使用缓存技术只需在项目中导入相关缓存技术的依赖包，并在启动类上使用@EnableCaching开启缓存支持即可

例如，使用Redis作为缓存技术，只需要导入Spring data Redis的Maven坐标即可

```java
@Autowired
private CacheManager cacheManager;
// CachePut：将方法返回值放入缓存
// value：缓存的名称，每个缓存名称下可以有多个key
// key：缓存的key
@CachePut(value="userCache", key="#user.id")
@PostMapping
public User save(User user){
    userService.save(user);
    return user;
}
```

