# 文件物理存储结构
文件夹topic+partitionN，包含segment(段)数据文件中。但每个段segment file消息数量不一定相等，这种特性方便old segment file快速被删除。
- segment file组成：由2大部分组成，分别为index file和data file，此2个文件一一对应，成对出现，后缀”.index”和“.log”分别表示为segment索引文件、数据文件.
- segment文件命名规则：partion全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值。数值最大为64位long大小，19位数字字符长度，没有数字用0填充。
![](/kafka_img/index-log-mapping.png)
以索引文件中元数据3,497为例，依次在数据文件中表示第3个message(在全局partiton表示第368772个message)、以及该消息的物理偏移地址为497。
查找过程，通过offset找到index和log文件，然后通过index找到最接近offset的记录，包含<offset,messageLength>, 然后从log文件中读取一批消息过滤到符合条件的消息。就是稀疏索引。
log.cleanup.policy 有compact， delete，前者是压缩相同key保留最新的。  
log.cleaner.min.compaction.lag.ms 日志清理器可以配置为保留最少量的未压缩日志“头部”。这可以通过设置压缩时间滞后来实现。  
log.cleaner.max.compaction.lag.ms 防止生产率较低的日志在无限时间内无法压缩
# Exactly Once
通过enable.idempotence 配置，保证消息发送的幂等性。max.in.flight.requests.per.connection 必须小于 5  retries 必须大于 0 acks 必须设置为 all。Kafka 将消息的序列号信息保存在分区维度的 .snapshot 文件中。该文件中保存了 ProducerId、ProducerEpoch 和 LastSequence。所以幂等的约束为：相同分区、相同 Producer（id 和 epoch） 发送的消息序列号需递增。即 Kafka 的生产者幂等性只在单连接、单分区生效，Producer 重启或消息发送到其他分区就失去了幂等性的约束。  
acks=0：生产者发了就算完了  
acks=1：消息写入主节点就算成功  
acks=-1或all：消息必须写入 ISR 中所有的副本才算成功  
# 复制
## Leader Epoch
包含两个值：
- Epoch：一个单调增加的版本号。每当 Leader 副本发生变更时，都会增加该版本号。Epoch 值较小的 Leader 被认为是过期 Leader，不能再行使 Leader 的权力；
- 起始位移（Start Offset）：Leader 副本在该 Epoch 值上写入首条消息的 Offset。
ISR 的全称叫做： In-Sync Replicas （同步副本集）, 可以理解为和 leader 保持同步的所有副本的集合。min.insync.replicas参数则用于控制最低同步副本数。踢出ISR的条件：
```
# 默认10000 即 10秒，Follower在过去的replica.lag.time.max.ms时间内，已经追赶上leader一次了的时间
replica.lag.time.max.ms
# 允许 follower 副本落后 leader 副本的消息数量，超过这个数量后，follower 会被踢出 ISR
replica.lag.max.messages 
```
LEO（Log End Offset，日志结束偏移量） 表示分区中 每个副本的日志中最高的 offset。每个 Kafka 分区副本都维护自己的 LEO 值，它标识了该副本当前写入数据的**下一个 offset** 的位置，也就是日志的末尾位置。HighWatermark 是指 ISR 中最小的LEO。某个分区上消费者能够读取的最高偏移量。是 Kafka分区中所有副本中已经成功提交的数据的最高 offset
# Producer
![](/kafka_img/kafka_producer_process.png)
# Consumer
拉模式，Consumer 在每次调用 Poll() 消费数据的时候，顺带一个 timeout 参数，当返回空数据的时候，会在 Long Polling 中进行阻塞，等待 timeout 再去消费，直到数据到达。 每个 Consumer Group 有一个或者多个 Consumer，Consumer Group 拥有一个公共且唯一的 Group ID。
3 种分区分配策略：RangeAssignor、RoundRobinAssignor 和 StickyAssignor。  
RangeAssignor 是 Kafka 默认的分区分配算法，对于每个 Topic，首先对 Partition 按照分区ID进行排序，然后对订阅这个 Topic 的 Consumer Group 的 Consumer 再进行排序，之后尽量均衡的按照范围区段将分区分配给 Consumer。此时可能会造成先分配分区的 Consumer 进程的任务过重（分区数无法被消费者数量整除）。  
RoundRobinAssignor 的分区分配策略是将 Consumer Group 内订阅的所有 Topic 的 Partition 及所有 Consumer 进行排序后按照顺序尽量均衡的一个一个进行分配。如果 Consumer Group 内，每个 Consumer 订阅都订阅了相同的Topic，那么分配结果是均衡的。如果订阅 Topic 是不同的，那么分配结果是不保证“尽量均衡”的，因为某些 Consumer 可能不参与一些 Topic 的分配。  
StickyAssignor 分区分配算法是 Kafka Java 客户端提供的分配策略中最复杂的一种，可以通过 partition.assignment.strategy 参数去设置，从 0.11 版本开始引入，目的就是在执行新分配时，尽量在上一次分配结果上少做调整，其主要实现了以下2个目标： 1 Topic Partition 的分配要尽量均衡。 2 当 
Rebalance(重分配，后面会详细分析) 发生时，尽量与上一次分配结果保持一致。  
max.poll.records 控制每次 poll() 调用可以返回的 最大记录数，主要用于控制消费者的处理负载和批次大小。  
fetch.max.bytes：指定 单次 fetch 请求（覆盖所有分区）可以拉取的最大字节数。主要用于流量控制，防止一次拉取过多数据导致内存压力和网络带宽问题

## Rebalance
触发条件  Consumer Group 组成员数量发生变化，订阅主题数量发生变化，订阅主题的分区数发生变化。  
通知机制就是靠 Consumer 端的心跳线程，它会定期发送心跳请求到 Broker 端的 Coordinator,当协调者决定开启 Rebalance 后，它会将“REBALANCE_IN_PROGRESS”封装进心跳请求的响应中发送给 Consumer ,当 Consumer 发现心跳响应中包含了“REBALANCE_IN_PROGRESS”，就知道 Rebalance 开始了。

## 位移提交机制
- 自动提交 enable.auto.commit = true (默认为true)  auto.commit.interval.ms Kafka 会保证在开始调用 Poll() 方法时，提交上一批消息的位移，再处理下一批消息, 因此它能保证不出现消费丢失的情况。但自动提交位移也有设计缺陷，那就是它可能会出现重复消费。就是在自动提交间隔之间发生 Rebalance 的时候，此时 Offset 还未提交，待 Rebalance 完成后， 所有 Consumer 需要将发生 Rebalance 前的消息进行重新消费一次。
- 手动提交 含同步提交API KafkaConsumer#commitSync() 异步提交API KafkaConsumer#commitAsync()
__consumer_offsets保存offset <Group ID，主题名，分区号 > 这个topic的创建涉及参数Broker 端参数 offsets.topic.num.partitions (默认值是50)和offsets.topic.replication.factor(默认值为3