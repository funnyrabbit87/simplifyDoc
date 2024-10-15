# Concepts
- simple and lightweight client library
- no external dependencies on systems other than Apache
- fault-tolerant local state Kafka itself
- exactly-once
- one-record-at-a-time processing
- high-level Streams DSL and a low-level Processor API.
## Stream Processing Topology
 Topology就是若干processor 组成的拓扑图用来处理数据。 A stream processor is a node in the processor topology; it represents a processing step to transform data in streams by receiving one input record at a time,produce one or more output records to its downstream processors.
- Source Processor: A source processor is a special type of stream processor that does not have any upstream processors. It produces an input stream to its topology from one or multiple Kafka topics by consuming records from these topics and forwarding them to its down-stream processors.
- Sink Processor: A sink processor is a special type of stream processor that does not have down-stream processors. It sends any received records from its up-stream processors to a specified Kafka topic.  

## Time
- Event time  an event or data record occurred , originally created "at the source".
- Processing time the event or data record happens to be processed by the stream processing application.
- Ingestion time an event or data record is stored in a topic partition by a Kafka broker.
使用event-time or ingestion-time 取决于Kafka的配置log.message.timestamp.type（CreateTime（生产者提供的时间戳），LogAppendTime（到达 Kafka broker 的时间戳）） Kafka Streams assigns a timestamp to every data record via the TimestampExtractor interface.通过这个接口来自定义时间戳的计算和获取    
- new output records are generated via processing some input record like context.forward(): Inherits timestamp from the input record.
-  new output records are generated via periodic functions  like Punctuator#punctuate(): Uses the current internal timestamp (context.timestamp()).
- Aggregations: Uses the maximum timestamp of all contributing input records.
change the default behavior in the Processor API by assigning timestamps to output records explicitly when calling #forward()
- For joins (stream-stream, table-table) that have left and right input records, the timestamp of the output record is assigned max(left.ts, right.ts).
- For stream-table joins, the output record is assigned the timestamp from the stream record.
- For aggregations, Kafka Streams also computes the max timestamp over all records, per key, either globally (for non-windowed) or per-window.
- For stateless operations, the input record timestamp is passed through （passed through是指透传，使用输入数据的时间）. 
- For flatMap and siblings that emit multiple records, all output records inherit the timestamp from the corresponding input record.
## Duality of Streams and Tables
流可以认为是表的变更日志，表是流的快照。
## Aggregations
aggregation operation takes one input stream or table, and yields a new table by combining multiple input records into a single output record.输出是一个表。
## Windowing
control how to group records that have the same key for stateful operations such as aggregations or joins into so-called windows.  specify a grace period for the window，wait for out-of-order data records for a given window.当前流时间大于窗口结束时间加上宽限期，则记录将被丢弃
## States
state stores, Every task in Kafka Streams embeds one or more state stores that can be accessed via APIs to store and query data required for processing. These state stores can either be a persistent key-value store, an in-memory hashmap, or another convenient data structure. Kafka Streams offers fault-tolerance and automatic recovery for local state stores. 类型 **Key-Value Store**  **Session Store** **Window Store**  
**本地存储** 默认情况下使用 RocksDB 作为嵌入式数据库来存储状态，In-Memory 的不会被持久化，而是存储在内存中。不会被备份  
**容错性** 将状态定期备份到 Kafka 主题（称为 Changelog Topic）  
**状态恢复** 从 Changelog Topic 中重新构建 State Store，从而实现故障恢复和高可用性。

## Processing Guarantees
processing.guarantee config value (default value is at_least_once) to StreamsConfig.EXACTLY_ONCE_V2 
## Out-of-Order Handling
Within a topic-partition record's timestamp 的增长和offsets的增长是不一致的。时间晚的记录先到达Kafka broker.（处理方式可以用时间戳提取器TimestampExtractor， 事件时间窗口）  
Within a stream task that may be processing multiple topic-partitions.不等所有分区数据到达就处理。 （处理方式可以用等待所有分区，时间戳提取器TimestampExtractor， 事件时间窗口）   
For Stream-Stream joins, all three types (inner, outer, left) handle out-of-order records correctly.  
对于 Stream-Table 连接，如果不使用版本化存储，则不会处理无序记录（即 Streams 应用程序不会检查无序记录，而只是按偏移顺序处理所有记录），因此可能会产生不可预测的结果。使用版本化存储，通过在表中执行基于时间戳的查找，可以正确处理流端无序数据。表端无序数据仍未得到处理。  
对于表-表连接，如果不使用版本化存储，则不会处理无序记录（即，Streams 应用程序不会检查无序记录，而只是按偏移顺序处理所有记录）。但是，连接结果是变更日志流，因此最终将是一致的。使用版本化存储，表-表连接语义从基于偏移量的语义变为基于时间戳的语义，并相应地处理无序记录。

# Architecture
## Stream Partitions and Tasks
stream partition is a totally ordered sequence of data records and maps to a Kafka topic partitio
data record in the stream maps to a Kafka message from that topic  
keys of data records determine the partitioning of data in both Kafka and Kafka Streams  
Kafka Streams uses the StreamsPartitionAssignor class and doesn't let you change to a different assignor. If you try to use a different assignor, Kafka Streams ignores it.
## Threading Model
启动更多流线程或更多应用程序实例仅相当于复制拓扑并让其处理不同的 Kafka 分区子集，从而有效地并行化处理  
## Local State Stores

## Fault Tolerance
For each state store, it maintains a replicated changelog Kafka topic in which it tracks any state updates.  Log compaction is enabled on the changelog topics so that old data can be purged safely to prevent the topics from growing indefinitely. To minimize this restoration time, users can configure their applications to have standby replicas of local states **num.standby.replicas** **rack.aware.assignment.tags**  
There is also a client config client.rack which can set the rack for a Kafka consumer. If brokers also have their rack set via broker.rack, then rack aware task assignment can be enabled via rack.aware.assignment.strategy

# Windowing
## Tumbling time window
固定时间窗口， 窗口大小固定，不重叠  
```
// A tumbling time window with a size of 5 minutes (and, by definition, an implicit
// advance interval of 5 minutes), and grace period of 1 minute.
Duration windowSize = Duration.ofMinutes(5);
Duration gracePeriod = Duration.ofMinutes(1);
TimeWindows.ofSizeAndGrace(windowSize, gracePeriod);

// The above is equivalent to the following code:
TimeWindows.ofSizeAndGrace(windowSize, gracePeriod).advanceBy(windowSize);
```
![](/kafka_img/streams-time-windows-tumbling.png)
## Hopping time window
hopping window is defined by two properties: the window’s size and its advance interval.The advance interval specifies by how much a window moves forward relative to the previous one
```
Duration windowSize = Duration.ofMinutes(5);
Duration advance = Duration.ofMinutes(1);
TimeWindows.ofSizeWithNoGrace(windowSize).advanceBy(advance);
```
![](/kafka_img/streams-time-windows-hopping.png)
## Sliding time window
sliding windows are used for join operations, specified by using the JoinWindows class, and windowed aggregations, specified by using the SlidingWindows class.
滑动窗口模拟一个在时间轴上连续滑动的固定大小窗口 两个数据记录的时间戳差在窗口大小范围内，则称这两个数据记录包含在同一窗口中。当滑动窗口沿时间轴移动时，记录可能会落入滑动窗口的多个快照中，但每个唯一的记录组合仅出现在一个滑动窗口快照中。
边界与进入的data record 时间对齐，可以理解为数据触发窗口滑动。
```
// A sliding time window with a time difference of 10 minutes and grace period of 30 minutes
Duration timeDifference = Duration.ofMinutes(10);
Duration gracePeriod = Duration.ofMinutes(30);
SlidingWindows.ofTimeDifferenceAndGrace(timeDifference, gracePeriod);
```
![](/kafka_img/streams-sliding-windows.png)
## Session Windows
Session windows are used to aggregate key-based events into so-called sessions, Sessions represent a period of activity separated by a defined gap of inactivity (or “idleness”).
all windows are tracked independently across keys – e.g. windows of different keys typically have different start and end times  
their window sizes sizes vary – even windows for the same key typically have different sizes  
```
// A session window with an inactivity gap of 5 minutes.
SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(5));
```
grace period 让窗口延迟关闭，用于应对数据时间在窗口内，但是因为延迟到达时间比后发的数据晚而乱序out of order 数据。让结果更准确。