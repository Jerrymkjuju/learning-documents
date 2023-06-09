## RocketMq初印象

- Java 实现，方便修改和定制

- 当前版本为5.1.0
  - 移除了producerGroup。本意是为了能够找到事务消息状态确认和回滚对应的处理方法。但是在实际的生产中，很少存在同一个topic对应不同的处理方式（该场景一般会使用不同的topic）。所以5.0版本以上做了匿名处理，不再需要进行配置
  
- 支持事务消息、定时消息、延时消息、顺序消息

- 支持消息过滤，通过Tag进行配置，在RocketMq服务端进行过滤，而非在消费者侧进行过滤

  > 还支持SQL过滤，通过生产者设置Property，消费者订阅时制定Filter规则进行过滤

  <img src="RocketMq.assets/messagefilter0-ad2c8360f54b9a622238f8cffea12068.png" alt="消息过滤" style="zoom: 50%;" />

- 下表显示了RocketMQ、ActiveMQ和Kafka之间的比较 

## MQ 比较

| Messaging Product|Client SDK| Protocol and Specification | Ordered Message  | Scheduled Message | Batched Message |BroadCast Message| Message Filter|Server Triggered Redelivery|Message Storage|Message Retroactive|Message Priority|High Availability and Failover|Message Track|Configuration|Management and Operation Tools|
| -------|--------|--------|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|
| ActiveMQ|Java, .NET, C++ etc. |Push model, support OpenWire, STOMP, AMQP, MQTT, JMS|Exclusive Consumer or Exclusive Queues can ensure ordering|Supported|Not Supported|Supported|Supported|Not Supported|Supports very fast persistence using JDBC along with a high performance journal，such as levelDB, kahaDB|Supported|Supported|Supported, depending on storage,if using levelDB it requires a ZooKeeper server|Not Supported|The default configuration is low level, user need to optimize the configuration parameters|Supported|
| Kafka      | Java, Scala etc.|Pull model, support TCP|Ensure ordering of messages within a partition|Not Supported|Supported, with async producer|Not Supported|Supported, you can use Kafka Streams to filter messages|Not Supported|High performance file storage|Supported offset indicate|Not Supported|Supported, requires a ZooKeeper server|Not Supported|Kafka uses key-value pairs format for configuration. These values can be supplied either from a file or programmatically.|Supported, use terminal command to expose core metrics|
| RocketMQ      |Java, C++, Go |Pull model, support TCP, JMS, OpenMessaging|Ensure strict ordering of messages,and can scale out gracefully|Supported|Supported, with sync mode to avoid message loss|Supported|Supported, property filter expressions based on SQL92|Supported|High performance and low latency file storage|Supported timestamp and offset two indicates|Not Supported|Supported, Master-Slave model, without another kit|Supported|Work out of box,user only need to pay attention to a few configurations|Supported, rich web and terminal command to expose core metrics|



## 5.0 新特性

- POP消费
- StaticTopic
- BatchConsumeQueue
- 自动主从切换
- BrokerContainer
- SlaveActingMaster模式
- Grpc Proxy

## 关键知识点

### 5.0 架构图

<img src="RocketMq.assets/76af2963ec644582b0c71f987d7c95d3~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.png" alt="img" style="zoom:50%;" />

### 消费者

- DefaultMQPushConsumer

  - **实现方式**：并不是真正的push，而是通过long-polling实现，所以逻辑中大部分是pullRequest

  - **流量控制**：每个messageQueue有一个ProcessQueue去做流量控制（客户端在每次pullRequestqianyu会判断获取但还未处理的消息个数，消息总大小，offset的跨度，任意一个超过阈值的话就会间隔一段时间再拉取）

  - **启动检测**：会检查各种配置信息，但是无法连接nameserver的情况并不会退出（而是不断重连）

    > *如果希望可以启动就暴露问题，可以在start后，调用fetchSubscribeMessageQueues，此时配置错误会抛出MQClientException*

- DefaultLitePullConsumer

  - 旧版本为DefaultMQPullConsumer
  - 主动权在consumer手中，需要consumer自己维护offset

### 生产者

- 延迟消息仅支持预设值的时间长度

- 多个MessageQueue时，一般是轮流发送，消费者也会根据负载均衡策略进行分配（具体发到哪里是未知的）

  - 可以在发送消息时指定MessageQueueSelector和args来选在发到哪一个messageQueue

- 事务消息是使用二阶段提交的方式来实现

  > RocketMq是将数据顺序写到磁盘来提升性能的，commit或者rollback需要更改一阶段的状态，这样会造成磁盘catch的脏页过多，降低系统性能。所以4.x的版本中这部分功能被去除，用户可根据实际需求实现自己的事务功能
  >
  > *TODO 确认当前最新的事务方式是如何处理的*

### Offset

- DefaultMQPushConsumer
  - 集群模式下，是Broker代存，RemoteBrokerOffsetStore
  - 广播模式下，是本地文件类型，LocalFileOffsetStore
  
- PullConsumer
  - 自己处理offsetStore（本地持久化）
  
- ConsumeFromWhere
  - LAST/FIRST/TIMESTAMP

- Log

  > 旧版本  org.apache.rocketmq.Client.Log ClientLogger
  
  新版本的日志封装到了rocketmq-slf4j-api-1.0.1.jar，各个module中配置相关的logback配置文件，如：rmq.client.logback.xml
  
  <img src="RocketMq.assets/image-20230425133910971.png" alt="image-20230425133910971" style="zoom:50%;" />

### NameServer

- 是无状态的，其中的Broker、Topic等状态信息不会持久存储，而且各个角色定时上报并存储在内存中的（支持配置持久化，但是基本用不到）
- 是一个非常简单的Topic路由注册中心，其角色类似Dubbo中的zookeeper，支持Broker的动态注册与发现。
- 主要包括两个功能：
  - **Broker管理**，NameServer接受Broker集群的注册信息并且保存下来作为路由信息的基本数据。然后提供心跳检测机制，检查Broker是否还存活；
  - **路由信息管理**，每个NameServer将保存**关于Broker集群的整个路由信息和用于客户端查询的队列信息**。然后Producer和Consumer通过NameServer就可以知道整个Broker集群的路由信息，从而进行消息的投递和消费。
- **高可用**
  - NameServer通常也是集群的方式部署，各实例间相互不进行信息通讯。Broker是向每一台NameServer注册自己的路由信息，所以每一个NameServer实例上面都保存一份完整的路由信息。当某个NameServer因某种原因下线了，Broker仍然可以向其它NameServer同步其路由信息，Producer和Consumer仍然可以动态感知Broker的路由的信息。
- **为何不用ZooKeeper**
  ZooKeeper的功能很强大，包括自动Master选举等，RocketMQ的架构设计决定了它不需要进行Master选举，用不到这些复杂的功能，只需要一个轻量级的元数据服务器就足够了。
  中间件对稳定性要求很高，RocketMQ的NameServer只有很少的代码，容易维护，所以不需要再依赖另一个中间件，从而减少整体维护成本。

### 消息的存储和发送

- 通过使用mmap的方式，可以省去向用户态的内存复制，提高速度

- 高可用：

  - Consumer：broker的master/slave来支持

  - Producer：不同name的多个broker支持

  - ~~*暂不支持slave自动转成master*~~ 新版本broker已经支持自动主从切换 [自动主从切换介绍](https://juejin.cn/post/7157730173620584478)

  - 异步刷盘，同步刷盘，异步复制，同步复制

    > 通常情况下，应该把Master和Save配置成ASYNC_FLUSH的刷盘方式，主从之间配置成SYNC_MASTER的复制方式，这样即使有一台机器出故障，仍然能保证数据不丢，是个不错的选择。

- 可靠性
  - 顺序消息问题
    - 全局顺序：消息队列、生产者、消费者都是1（消除并发、全部单线程）
    - 部分顺序：根据业务属性，需要顺序处理的消息要发到同一个messageQueue中，消费者消费是不能并发处理
  - 重复消费问题
    - 一定投递”和“不重复投递”是很难做到的。
    - RocketMQ选择了确保一定投递，保证消息不丢失，但有可能造成消息重复
      - 网络波动，生产者的重试会造成重复投递
      - 消费者需要保证消费逻辑幂等性或者判断消息是否消费过