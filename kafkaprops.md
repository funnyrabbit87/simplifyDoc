- key.serializer
- value.serializer  
  StringSerializer.class
- bootstrap.servers  
  host1:port1,host2:port2,

### Producer  
- acks  
0: not wait for any acknowledgment from the server at all.  
1: leader will write the record to its local log but will respond without awaiting full acknowledgement from all followers
-1(all):  leader will wait for the full set of in-sync replicas to acknowledge the record 
- buffer.memory  
  The total bytes of memory the producer can use to buffer records waiting to be sent to the server. If records are sent faster than they can be delivered to the server the producer will block for max.block.ms after which it will throw an exception.
This setting should correspond roughly to the total memory the producer will use, but is not a hard bound since not all memory the producer uses is used for buffering. Some additional memory will be used for compression (if compression is enabled) as well as for maintaining in-flight requests.
定义了生产者可以用于缓冲待发送到服务器的记录（消息）的内存总字节数。生产者在发送记录的速度超过它们能够交付到服务器的速度时，会在达到这个内存限制后发生阻塞，最多阻塞一段时间（由 "max.block.ms" 参数指定），超过这段时间后，生产者将抛出异常。 max.block.ms（最大阻塞时间）：这是一个与 "buffer.memory" 参数
- compression.type  
  [none, gzip, snappy, lz4, zstd]
- retries  
  Setting a value greater than zero will cause the client to resend any record whose send fails with a potentially transient error. Note that this retry is no different than if the client resent the record upon receiving the error. Produce requests will be failed before the number of retries has been exhausted if the timeout configured by delivery.timeout.ms expires first before successful acknowledgement. Users should generally prefer to leave this config unset and instead use delivery.timeout.ms to control retry behavior.
Enabling idempotence requires this config value to be greater than 0. If conflicting configurations are set and idempotence is not explicitly enabled, idempotence is disabled.
- batch.size  
  每当多个记录发送到同一分区时，生产者将尝试将记录一起批处理为更少的请求。 这有助于提高客户端和服务器的性能。 此配置控制默认批量大小（以字节为单位）。  
  The producer will attempt to batch records together into fewer requests whenever multiple records are being sent to the same partition. This helps performance on both the client and the server. This configuration controls the default batch size in bytes.  
If we have fewer than this many bytes accumulated for this partition, we will ‘linger’ for the linger.ms time waiting for more records to show up. This linger.ms setting defaults to 0, which means we’ll immediately send out a record even the accumulated batch size is under this batch.size setting.
- client.id  
  An id string to pass to the server when making requests. The purpose of this is to be able to track the source of requests beyond just ip/port by allowing a logical application name to be included in server-side request logging
- delivery.timeout.ms  
  An upper bound on the time to report success or failure after a call to send() returns.
  The value of this config should be greater than or equal to the sum of request.timeout.ms and linger.ms
- linger.ms  
  This setting gives the upper bound on the delay for batching: once we get batch.size worth of records for a partition it will be sent immediately regardless of this setting, however if we have fewer than this many bytes accumulated for this partition we will ‘linger’ for the specified time waiting for more records to show up
- max.request.size  
  This setting will limit the number of record batches the producer will send in a single request to avoid sending huge requests
- partitioner.class  
  A class to use to determine which partition to be send to when produce the records.  
  If not set, the default partitioning logic is used. This strategy will try sticking to a partition until at least batch.size bytes is produced to the partition. It works with the strategy:If no partition is specified but a key is present, choose a partition based on a hash of the keyIf no partition or key is present, choose the sticky partition that changes when at least batch.size bytes are produced to the partition.  
有key用hash 无key等发完batch.size bytes
- request.timeout.ms
  The configuration controls the maximum amount of time the client will wait for the response of a request. If the response is not received before the timeout elapses the client will resend the request if necessary or fail the request if retries are exhausted. This should be larger than replica.lag.time.max.ms (a broker configuration) to reduce the possibility of message duplication due to unnecessary producer retries.
- enable.idempotence
enabled by default. enabling idempotence requires max.in.flight.requests.per.connection to be less than or equal to 5 (with message ordering preserved for any allowable value), retries to be greater than 0, and acks must be ‘all’
- max.in.flight.requests.per.connection  
  The maximum number of unacknowledged requests the client will send on a single connection before blocking. Note that if this configuration is set to be greater than 1 and enable.idempotence is set to false, there is a risk of message reordering after a failed send due to retries (i.e., if retries are enabled); if retries are disabled or if enable.idempotence is set to true, ordering will be preserved. Additionally, enabling idempotence requires the value of this configuration to be less than or equal to 5. If conflicting configurations are set and idempotence is not explicitly enabled, idempotence is disabled.
- partitioner.adaptive.partitioning.enable  
  When set to ‘true’, the producer will try to adapt to broker performance and produce more messages to partitions hosted on faster brokers. If ‘false’, producer will try to distribute messages uniformly. Note: this setting has no effect if a custom partitioner is used
- partitioner.availability.timeout.ms  
  If a broker cannot process produce requests from a partition for partitioner.availability.timeout.ms time, the partitioner treats that partition as not available. If the value is 0, this logic is disabled. Note: this setting has no effect if a custom partitioner is used or partitioner.adaptive.partitioning.enable is set to ‘false’
- retry.backoff.ms  
  The amount of time to wait before attempting to retry a failed request to a given topic partition. This avoids repeatedly sending requests in a tight loop under some failure scenarios
- transaction.timeout.ms  
  The maximum amount of time in milliseconds that a transaction will remain open before the coordinator proactively aborts it.
- transactional.id  
  This enables reliability semantics which span multiple producer sessions since it allows the client to guarantee that transactions using the same TransactionalId have been completed prior to starting any new transactions.
  因为它允许客户端保证使用相同 TransactionalId 的事务在开始任何新事务之前已完成
- ssl.protocol
- ssl.keystore.location
- ssl.keystore.password
- ssl.key.password
- ssl.truststore.location
- ssl.truststore.password  
### Consumer  
- fetch.min.bytes  
  The minimum amount of data the server should return for a fetch request. If insufficient data is available the request will wait for that much data to accumulate before answering the request.
- group.id
  A unique string that identifies the consumer group this consumer belongs to.
- heartbeat.interval.ms  
  The expected time between heartbeats to the consumer coordinator when using Kafka’s group management facilities. Heartbeats are used to ensure that the consumer’s session stays active and to facilitate rebalancing when new consumers join or leave the group. The value must be set lower than session.timeout.ms, but typically should be set no higher than 1/3 of that value. It can be adjusted even lower to control the expected time for normal rebalances.
- max.partition.fetch.bytes  
  The maximum amount of data per-partition the server will return. Records are fetched in batches by the consumer. If the first record batch in the first non-empty partition of the fetch is larger than this limit, the batch will still be returned to ensure that the consumer can make progress. The maximum record batch size accepted by the broker is defined via message.max.bytes (broker config) or max.message.bytes (topic config). See fetch.max.bytes for limiting the consumer request size.
- session.timeout.ms  
  The timeout used to detect client failures when using Kafka’s group management facility. The client sends periodic heartbeats to indicate its liveness to the broker. If no heartbeats are received by the broker before the expiration of this session timeout, then the broker will remove this client from the group and initiate a rebalance. Note that the value must be in the allowable range as configured in the broker configuration by group.min.session.timeout.ms and group.max.session.timeout.ms.
- auto.offset.reset
earliest,latest  
none/anything else: throw exception to the consumer if no previous offset is found for the consumer’s group
- default.api.timeout.ms
  Specifies the timeout (in milliseconds) for client APIs. This configuration is used as the default timeout for all client operations that do not specify a timeout parameter.
- enable.auto.commit
- fetch.max.bytes
  The maximum amount of data the server should return for a fetch request.
- group.instance.id  
  A unique identifier of the consumer instance provided by the end user. Only non-empty strings are permitted. If set, the consumer is treated as a static member, which means that only one instance with this ID is allowed in the consumer group at any time. This can be used in combination with a larger session timeout to avoid group rebalances caused by transient unavailability (e.g. process restarts). If not set, the consumer will join the group as a dynamic member, which is the traditional behavior.
- max.poll.interval.ms  
  The maximum delay between invocations of poll() when using consumer group management. This places an upper bound on the amount of time that the consumer can be idle before fetching more records. If poll() is not called before expiration of this timeout, then the consumer is considered failed and the group will rebalance in order to reassign the partitions to another member. For consumers using a non-null group.instance.id which reach this timeout, partitions will not be immediately reassigned. Instead, the consumer will stop sending heartbeats and partitions will be reassigned after expiration of session.timeout.ms. This mirrors the behavior of a static consumer which has shutdown.
- max.poll.records  
  The maximum number of records returned in a single call to poll(). Note, that max.poll.records does not impact the underlying fetching behavior. The consumer will cache the records from each fetch request and returns them incrementally from each poll.
- partition.assignment.strategy
  RangeAssignor: Assigns partitions on a per-topic basis.
  RoundRobinAssignor: Assigns partitions to consumers in a round-robin fashion.
  StickyAssignor: Guarantees an assignment that is maximally balanced while preserving as many existing partition assignments as possible.
  CooperativeStickyAssignor: Follows the same StickyAssignor logic, but allows for cooperative rebalancing.
  The default assignor is [RangeAssignor, CooperativeStickyAssignor], which will use the RangeAssignor by default, but allows upgrading to the CooperativeStickyAssignor with just a single rolling bounce that removes the RangeAssignor from the list.

Implementing the org.apache.kafka.clients.consumer.ConsumerPartitionAssignor interface allows you to plug in a custom assignment strategy