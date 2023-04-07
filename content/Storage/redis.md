# redis

## 单机（Standalone)

通过自带的哨兵集群保证高可用。

Sentinel 会通过 Gossip 协议进行故障检测，确认宕机后会通过一个简化的 Raft 协议来提升 Slave 成为新的 Master

通常情况我们仅使用 1 个 Slave 节点进行冷备，如果有读写分离请求，可以建立多个 Read only slave 来进行读写分离

**注意点**

- 只读 Slave 节点可以按照需求**设置 slave-priority 参数为 0**，防止故障切换时选择了只读节点而不是热备 Slave 节点
- Sentinel 进行故障切换后会执行 CONFIG REWRITE 命令将 SLAVEOF 配置落地，如果 Redis 配置中禁用了 CONFIG 命令，切换时会发生错误，可以通过修改 Sentinel 代码来替换 CONFIG 命令；
- Sentinel Group 监控的节点不宜过多，实测超过 500 个切换过程偶尔会进入 TILT 模式，导致 Sentinel 工作不正常，推荐部署多个 Sentinel 集群并保证每个集群监控的实例数量小于 300 个；
- Master 节点应与 Slave 节点跨机器部署，有能力的使用方可以跨机架部署，不推荐跨机房部署 Redis 主从实例；
- Sentinel 切换功能主要依赖 down-after-milliseconds 和 failover-timeout 两个参数，down-after-milliseconds 决定了 Sentinel 判断 Redis 节点宕机的超时，知乎使用 30000 作为阈值。而 failover-timeout 则决定了两次切换之间的最短等待时间，如果对于切换成功率要求较高，可以适当缩短 failover-timeout 到秒级保证切换成功，具体详见 Redis 官方文档；
- 单机网络故障等同于机器宕机，但如果机房全网发生大规模故障会造成主从多次切换，此时资源发现服务可能更新不够及时，需要人工介入

## 集群-客户端分片

[知乎客户端分片](https://github.com/zhihu/redis-shard)

#### 客户端分片优点

- 快、没有中间件，仅需要客户端进行一次哈希计算，不需要经过代理，没有官方集群方案的 MOVED/ASK 转向
- 不需要多余的 Proxy 机器，不用考虑 Proxy 部署与维护
- 可以自定义更适合生产环境的哈希算法。

#### 缺点

- 需要每种语言都实现一遍客户端逻辑。语言多时，**维护成本高**。
- 无法正常使用 MSET、MGET 等多种同时操作多个 Key 的命令，需要使用 Hash tag 来保证多个 Key 在同一个分片上
- **升级麻烦**，升级客户端需要所有业务升级更新重启，业务规模变大后无法推动
- **扩容困难**，存储需要停机使用脚本 Scan 所有的 Key 进行迁移，缓存只能通过传统的翻倍取模方式进行扩容；
- 由于每个客户端都要与所有的分片建立池化连接，客户端基数过大时会造成 Redis 端连接数过多，Redis 分片过多时会造成 Python 客户端负载升高。

## 集群之Twemproxy 集群方案

[Twitter集群方案](https://github.com/twitter/twemproxy)

#### 优点

- 性能很好且足够稳定，自建内存池实现 Buffer 复用，代码质量很高；

- 支持 fnv1a_64、murmur、md5 等多种哈希算法；

- 支持一致性哈希（ketama），取模哈希（modula）和随机（random）三种分布式算法。



