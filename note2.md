### 一. 数据量达到多少要开始分表

结合业务场景和系统架构考虑

1. **单表的数据量**：如果单表的数据量很大，例如超过百万级别，就需要分表
2. **数据库性能**：当单个数据库的性能无法满足业务需求时，需要考虑分库
3. **数据的访问频率**：如果某些表的数据访问评率非常高，单个数据库节点无法满足高并发需求，就需要把这些表分到不同的数据库节点上，去提高整个数据库的IO性能
4. **业务拆分**：当系统业务越来越复杂，不同业务之间的数据耦合度越来越低，就可以考虑对系统进行拆分，以便管理和扩展

``` java
// 单独讨论数据量决定分库分表是不严谨的，决定分库分表的因素很多，比如
// 1.查询性能下降或业务解耦
// 2.数据行的大小（列数）即宽表
// 3.索引大小会决定B+树的层高，层高又间接决定不同数据量级的一个查询性能
// 4.硬件性能配置（带宽，内存，磁盘）
// 5.表结构设计不合理
```



### 二、HashMap什么时候扩容，怎么扩容？

1. 当集合容量达到某个阈值时，进行扩容，HashMap扩容为原来大小的**两倍**（临界值=负载因子*容量大小， **负载因子默认为0.75**）
2. 在集合初始化时，**最好指定容量大小**避免频繁扩容影响性能。
3. 扩容因子为0.75的原因：扩容因子表示Hash表的填充程度，扩容因子越大，整体空间利用率越高，但是Hash冲突的概率也会增加。扩容因子的值是冲突的概率和空间利用率之间的一个平衡。

4. **HashMap采用链式寻址的方式来解决Hash冲突，为了避免链表过长导致时间复杂度增加，当链表长度>8时，就会转换为红黑树提高检索效率。当扩容因子为0.75时，链表长度达到8的可能性几乎为0，比较好的平衡了时间成本和空间成本**



### 三、解决Hash冲突的方法

1. 再hash法：当某个hash函数产生了冲突，就用另一个hash进行计算
2. 开放寻址法：就是直接从冲突的位置寻找下一个空的数组下标
3. 建立公共溢出区：把存在冲突的key统一放在一个公共溢出区里



### 四、HashMap中的hash方法为什么要无符号右移十六位并且进行异或？

**主要是为了要hash值的散列度更高，尽可能减少hash冲突，从而提升数据的查找性能**

原理：hash表的位置i=hash值%(n-1)，一般情况下n的值一般是小于2^16，也就意味着i的值始终是使用hash值的低十六位与(n-1)进行取模运算，这样就会造成key的散列度不是很高，导致大量的key集中存储在一个固定的几个数组位置上，显然影响数据的查找性能。为了提升key的散列度：**将hashCode右移16位，相当于把高位和低位的特征进行了一个组合**，这样得到的i的散列度一定会更高。



### 五、HashMap和HashTable的区别

1. HashTable是**线程安全**的，HashMap不是
2. HashMap性能比HashTable好(HashTable使用了全局同步锁保证线程安全性，对性能较大)
3. HashTable使用数组+链表实现，HashMap使用**数组+链表+红黑树**
4. HashMap初始容量是16，HashTable初始容量是**11**
5. **HashMap可以使用null作为key**(HashMap会把null转化为一个0进行存储)，而HashTable不允许
6. **散列算法不同**，HashTable直接使用hashCode对数组长度取模，而HashMap对hashCode作了二次散列从而避免key的分布不均匀影响到性能



### 六、kafka如何避免重复消费

kafka上的消息通过offset来维护，消费者每消费消费一批数据，kafka就会更新offset，来避免重复消费

**重复消费的原因：**

1. kafka的自动提交有个默认5s的间隔，在5s之后下一次向offset获取消息时，来实现offset的自动提交。如果消费者在消费过程中，被强制**宕机或kill**掉时，会导致offset未提交，从而导致重复消费
2. kafka的**rebalance机制**，如果消费者在默认的5min内没有处理完这批消息时，就会触发kafka的rebalance机制，从而导致offset自动提交失败。在rebalance后，消费者还是会在提交offset前，进行消费，从而导致重复消费

**解决方法：**

1. 提高消费端处理性能**避免触发rebalance**。（a）用异步方式处理消息，缩短单个消息处理时长    (b)   调整消息处理时长，把时间拉长一点，例如5min拉到10min    （c）减少一次性从broker上拉去消息的条数
2. 针对每一条消息**生成一个md5值**，保存在数据库或者Redis里。在处理消息之前先去判断数据库或Redis里是否已经存在相同的md值，如果存在就不需要再去消费了。（利用幂等性思想）



### 七、kafka如何保证消息不丢失

三个方面

##### 生产者：

1. 把异步发送改为同步发送，这样生产者能实时知道消息发送的结果
2. 添加异步回调函数来监听消息发送的结果，如果发送失败，可以再回调中重试
3. producer本身提供了一个参数retries，如果因为网络问题或者broker故障导致发送失败，producer会自动重试

##### broker:（确保producer过来发送的消息不会丢失 —> 持久化到磁盘中）

kafka为了提升性能采用异步批量刷盘的实现机制（按照一定的消息量和时间间隔去刷盘）

如果刷盘前，操作系统崩溃就会导致数据丢失

1. Partition副本机制：leader负责事务请求，follower负责同步leader中的数据
2. acks参数：producer可以设置acks参数结合broker副本机制，共同保障数据的可靠性

（1） acks=0，producer不需要等待broker的响应

（2） acks=1，leader收到消息后，不等待其他follower同步，就和producer返回确认（如果leader挂了，就会存在数据丢失）

（3） acks=-1，等待ISR列表中的所有follower同步完返回确认

##### 消费者：保证消费者必须能够消费消息

不太可能出现不会消费到消息的情况，除非消费者没有消费完，就提交了offset。如果出现这种情况，可以重新调整offset的值



### 八、kafka的零拷贝原理

当需要将磁盘中的某个内容发送到远程服务器上时，会经过**四次拷贝**的过程

磁盘 ——> 内核缓冲区 ——> 用户空间缓冲区 ——> 内核空间的Socket Buffer ——> 网卡缓冲区NIC Buffer

1. 从磁盘读取内容，拷贝到内核缓冲区
2. CPU把内核缓冲区的数据拷贝到用户空间缓冲区
3. 在应用程序中调用write()方法，把用户空间缓冲区拷贝到内核空间的Socket Buffer
4. 把内核空间的Socket Buffer中的数据赋值到网卡
5. 最后网卡缓冲区在把数据传输到目标服务器上

**有两次拷贝是浪费的**，1）从内核空间拷贝到用户空间    2）从用户空间再次拷贝到内核空间

其次，由于内核空间和用户空间的切换，会带来CPU的上下文切换，对CPU的性能也会造成影响

**所谓的零拷贝就是把这两次多余的拷贝忽略掉**。应用程序可以直接把磁盘中的数据，从内核中，直接传输到Socket中，而不需要再次经过应用程序所在的用户空间。零拷贝通过**DMA技术**，把文件内容复制到内核空间的Socket Buffer中，然后把包含信息长度和位置的信息文件描述符加载到Socket Buffer中，DMA引擎可以直接把数据从内核空间传递到网卡设备。这个过程中数据只经历了两次拷贝



### 九、什么是ISR，为什么要引入ISR？

**ISR是一个集合列表，里面保存的是和leader Partition节点数据最接近的所有的follower Partition节点。如果某个follower节点落后leader节点太多，那么这个节点就会被踢出ISR集合。ISR列表里同步的数据一定是最新的，后续选举leader只需要从ISR里筛选就好了**

1. 尽可能的保证数据同步的效率，因为同步效率不高的节点会被踢出ISR列表
2. 避免数据的丢失，因为ISR列表里的节点数据是和leader最接近的



### 十、kafka如何保证消息消费的顺序性

**存在无序消费的原因：**

kafka用Partition的分区机制，来实现消息的物理存储，也就是在同一个topic里可以有多个分区，消费时的顺序可能就不会是按发送时的顺序来实现。

**解决方法：**

1. 自定义消息分区的路由算法，把指定的key都发送到同一个Partition里，然后再指定一个去消费该分区的数据就保证了消息的顺序消费。
2. 当消费端采用异步线程的方式时，因每个线程的处理效率不一样，也可能导致乱序消费。所以在消息消费端采用一个阻塞队列，把获取到的消息先保存在阻塞队列里。就可以实现顺序消费。



### 十一、kafka消息队列如何保证Exactly Once

**MQ消息投递中的三种语义：**

1. At Most Once：消息至多投递一次，可能会丢失，但不会重复
2. At Least Once：消息至少投递一次，可能会重复，但不会丢失
3. Exactly Once:消息投递正好一次，不会出现重复也不会丢失



1. 生产者采用事务消息的方式，事务可以支持多分区的数据完整性，原子性
2. 消费端采用幂等性的机制来避免重试带来重复消费的问题

顺序消费：kafka中每个partition中的消息是按顺序存储的，只需针对某个topic设置一个partition，就可以实现消息的顺序处理。



### 十二、kafka中partition的leader选举机制

kafka会**从ISR集合中选取**，ISR集合中的副本与leader同步最接近，切leader的网络延迟通信最小。如果**ISR没有可用副本，kafka会从所有副本中选择一个具有最新数据的副本**。（由于新leader和老leader数据延迟较大，这种选取会导致数据的丢失）

