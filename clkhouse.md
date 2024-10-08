# Observability
## Schema design
#### Materialized views
Extract columns from string blobs. Querying these will be faster than using string operations at query time.  
Extract keys from maps. The default schema places arbitrary attributes into columns of the Map type.   
#### Materialized views
 allow users to shift the cost of computation from query time to insert time. A ClickHouse Materialized View is just a trigger that runs a query on blocks of data as they are inserted into a table. The results of this query are inserted into a second "target" table. 生成独立的表，增加插入时间换取更快查询。持有独立的数据副本，预先执行了计算和聚合等操作

 #### Using Aliases
 Map方式查询慢，可以用这个解决

 #### Accelerating queries
 加速查询的办法有 Materialized views, Projections
 Projections allow users to specify multiple ORDER BY clauses for a table.相当于多张不完整的表，This will slow inserts and consume more disk space.


#### Choosing a primary (ordering) key
Select columns that align with your common filters and access patterns. If users typically start Observability investigations by filtering by a specific column e.g. pod name, this column will be used frequently in WHERE clauses. Prioritize including these in your key over those which are used less frequently.  
Prefer columns which help exclude a large percentage of the total rows when filtered, thus reducing the amount of data which needs to be read. Service names and status codes are often good candidates - in the latter case only if users filter by values which exclude most rows e.g. filtering by 200s will in most systems match most rows, in comparison to 500 errors which will correspond to a small subset.    
Prefer columns that are likely to be highly correlated with other columns in the table. This will help ensure these values are also stored contiguously, improving compression.  
GROUP BY and ORDER BY operations for columns in the ordering key can be made more memory efficient.


# Advanced Guides
## TTL (Time To Live)
支持TTL到期删除行，列。2 settings that trigger TTL events:  
**merge_with_ttl_timeout**: the minimum delay in seconds before repeating a merge with delete TTL. The default is 14400 seconds (4 hours).
**merge_with_recompression_ttl_timeout**: the minimum delay in seconds before repeating a merge with recompression TTL (rules that roll up data before deleting). Default value: 14400 seconds (4 hours).

## Deduplication Strategies
Deduplication refers to the process of removing duplicate rows of a dataset， implemented  using table engines:  
**ReplacingMergeTree** table engine: with this table engine, duplicate rows with the same sorting key are removed during merges. ReplacingMergeTree is a good option for emulating upsert behavior (where you want queries to return the last row inserted).

**Collapsing rows**: the **CollapsingMergeTree** and **VersionedCollapsingMergeTree** table engines use a logic where an existing row is "canceled" and a new row is inserted. They are more complex to implement than ReplacingMergeTree, but your queries and aggregations can be simpler to write without worrying about whether or not data has been merged yet. These two table engines are useful when you need to update data frequently. VersionedCollapsingMergeTree 是应对多线程插入  
使用createAt这种带时间的会导致行重复失效，因为相同的数据创建时间不一样。 可以使用insert_deduplication_token 来让行重复
```
INSERT INTO test_table SETTINGS insert_deduplication_token = 'test' VALUES (2);
```

## Deduplicating Inserts on Retries
Only *MergeTree engines support deduplication on insertion.For *ReplicatedMergeTree engines, insert deduplication is enabled by default and is controlled by the replicated_deduplication_window and replicated_deduplication_window_seconds settings. For non-replicated *MergeTree engines, deduplication is controlled by the non_replicated_deduplication_window setting.  
For tables using *MergeTree engines, each block is assigned a unique block_id, which is a hash of the data in that block. This block_id is used as a unique key for the insert operation. If the same block_id is found in the deduplication log, the block is considered a duplicate and is not inserted into the table.  use the insert_deduplication_token setting to control the deduplication process.  
insert_deduplicate=1 is enabled, ClickHouse generates and stores block IDs for each block of data inserted into the table. These block IDs are used to identify and ignore duplicate blocks in subsequent insert attempts, 
**INSERT ... VALUES** queries, splitting the inserted data into blocks is deterministic and is determined by settings  
**INSERT ... SELECT** queries, it is important that the SELECT part of the query returns the same data in the same order for each operation. Note that this is hard to achieve in practical usage. To ensure stable data order on retries, define a precise ORDER BY section in the SELECT part of the query. Keep in mind that it is possible that the selected table could be updated between retries: the result data could have changed and deduplication will not occur. Additionally, in situations where you are inserting large amounts of data, it is possible that the number of blocks after inserts can overflow the deduplication log window, and ClickHouse won't know to deduplicate the blocks.  
**replicated_deduplication_window** **replicated_deduplication_window_seconds** 计算是否重复时用什么往前推多少条/秒 **non_replicated_deduplication_window** 类似，含义相反  

**max_block_size**：控制每次查询时处理的数据块的最大行数。  
**min_insert_block_size_rows** 和 **min_insert_block_size_bytes**：控制插入时最小的数据块大小，分别从行数和字节数的角度设定。  
**deduplicate_blocks_in_dependent_materialized_views**：控制在插入数据时，是否对物化视图中的数据块进行去重操作。
**_part** 是一个内部隐藏的虚拟列，用来表示数据所在的存储分区（part）的名称。每当数据插入到 MergeTree 或其派生的表引擎（如 ReplicatedMergeTree）时，ClickHouse 会将数据以分区的形式存储，并为每个分区分配一个唯一的名称。查看system.parts表。  

### Transactional (ACID) support
Case 1: INSERT into one partition, of one table, of the MergeTree* family  
Atomic：support  
Consistent：support  
Isolated: MVCC，使用事务的如果客户端在事务内（即使用事务进行操作），那么它们将使用快照隔离。如果客户端不在事务内（即没有显式使用事务），那么它们的隔离级别是“读取未提交”Durable: a successful INSERT is written to the filesystem before answering to the client, on a single replica or multiple replicas (controlled by the insert_quorum setting), and ClickHouse can ask the OS to sync the filesystem data on the storage media (controlled by the fsync_after_insert setting).
如果涉及物化视图，则可以使用一个语句将 INSERT 到多个表中（来自客户端的 INSERT 是针对具有关联物化视图的表）  
Case 2: INSERT into multiple partitions, of one table, of the MergeTree* family  
Same as Case 1 above, with this detail:  every partition is transactional  
Case 3: INSERT into one distributed table of the MergeTree* family  
 not transactional as a whole, while insertion into every shard is transactional
Case 4: Using a Buffer table  
 neither atomic nor isolated nor consistent nor durable  
 Case 5: Using async_insert  
 atomicity is ensured even if async_insert is enabled and wait_for_async_insert is set to 1 (the default), but if wait_for_async_insert is set to 0, then atomicity is not ensured.  
a graph. It highlights how ClickHouse is going to execute a query and what resources are going to be used.  


### Understanding Query Execution with the Analyzer

![解析执行过程](https://clickhouse.com/docs/assets/images/analyzer1-7854d7b203e30fc87a2bc62ba8e5e8e5.png "解析执行过程")
The main benefit of a query tree over an AST is that a lot of the components will be resolved, like the storage for instance.   
the query plan tells us how we will do it. Additional optimizations are going to be done as part of the query plan. You can use EXPLAIN PLAN or EXPLAIN to see the query plan

# Best Practices
## Sparse Primary Indexes
稀疏索引，一个索引指向一块数据（包含多行)  
创建含多列的主键，但是查询条件不是第一列时，第一列数据相似度越高（low cardinality），效率越高并且索引文件压缩率高，更小，提高IO效率。  
When a query is filtering **on at least one column** that is part of a compound key, and is the **first key column**, **binary search algorithm** over the key column's index marks.  

When a query is **filtering (only) on a column** that is part of a compound key, but is **no**t the first key column,**generic exclusion search algorithm** over the key column's index marks.  

**PRIMARY KEY(A, B) 用来创建索引，指定索引内顺序 。ORDER BY(A, B) 数据在存储时的物理排序顺序**
两者顺序不一致时会导致  
查询性能下降：由于数据存储顺序与索引顺序不匹配，稀疏索引无法有效工作，导致查询时需要扫描更多的数据块。  
稀疏索引失效：主键不能有效地跳过不相关的块。  
数据压缩效率降低：由于数据存储顺序的不一致，压缩效果也可能不如数据顺序匹配时好。    

| 内置表                         | 主要用途                                             |
|--------------------------------|------------------------------------------------------|
| `.bin` | 存储表的数据内容，按列分开。顺序时order by                              |
| `.idx` | 存储索引信息，key column values of the first row of granule x   |
| `.sidx` | 稀疏存储索引信息  |
| `.mrk` | block_offset  locating the block in the compressed column data file granule_offset  location of the granule within the uncompressed block data    |
| `.mrk2` | 同mrk，多了一列表示 行数    |

大致过程  
稀疏索引查找: 首先通过稀疏索引 (UserID.sidx) 查找 UserID=123 大致在哪些块中。  
主索引查找: 然后通过主索引 (UserID.idx) 精确确定包含 UserID=123 的数据块范围。  
标记文件: 读取标记文件 (UserID.mrk) 确定物理存储的块偏移量。  
读取数据: 从数据文件 (UserID.bin, URL.bin) 中读取对应的数据块，只读取满足 UserID=123 AND URL='example.com' 的数据。  
过滤和返回结果: 对数据进行过滤和进一步处理，返回最终查询结果。 
 

## Understanding ClickHouse Data Skipping Indexes
For MergeTree family of tables only. 4 arguments:  Index name,Index expression,TYPE,GRANULARITY  
skp_idx_{index_name}.idx, which contains the ordered expression values  
skp_idx_{index_name}.mrk2, which contains the corresponding offsets into the associated data column files.   
### Skip Index Types
**minmax**  lightweight index type requires no parameters,stores the minimum and maximum values of the index expression for each block  
**set** single parameter of the max_size of the value set per block (0 permits an unlimited number of discrete values).This index type works well with columns with low cardinality within each set of granules (essentially, "clumped together") but higher cardinality overall.(粒度单元内的数据集中分布，只有少数几个值,表中有很多不同的值，但它们分布在不同的粒度单元中). set(100) 理解为granules中最多能包含100个不同的值（超过时索引失效，索引存储的数据为空）  
**Bloom Filter** can be  be applied to arrays, where every value of the array is tested, and to maps, by converting either the keys or values to an array using the mapKeys or mapValues function.3 sub type bloom_filter ，tokenbf_v1 含3个参数(1) the size of the filter in bytes ，ngrambf_v1
**Bloom Filter** can be  be applied to arrays, where every value of the array is tested, and to maps, by converting either the keys or values to an array using the mapKeys or mapValues function.3 sub type  
bloom_filter
tokenbf_v1 含3个参数(1) the size of the filter in bytes, (2) number of hash functions applied (again, more hash filters reduce false positives), (3)the seed for the bloom filter hash functions.ngrambf_v1  
ngrambf_v1 the size of the ngrams to index.  
### Skip Index Settings
use_skip_indexes (0 or 1, default 1). force_data_skipping_indices (comma separated list of index names).  

## Asynchronous Inserts
**async_insert** 1为启用。异步插入默认情况下不支持自动去重  
buffer size has reached N bytes in size (N is configurable via async_insert_max_data_size)  
at least N ms has passed since the last buffer flush (N is configurable via async_insert_busy_timeout_ms)  
**wait_for_async_insert** **0** 数据进入in-memory buffer 就ACK. **1** flushing from buffer to part再ACK
```
ALTER USER default SETTINGS async_insert = 1
INSERT INTO YourTable SETTINGS async_insert=1, wait_for_async_insert=1 VALUES (...)
"jdbc:ch://HOST.clickhouse.cloud:8443/?user=default&password=PASSWORD&ssl=true&custom_http_params=async_insert=1,wait_for_async_insert=1"
```

## Avoid Nullable Columns
Nullable column (e.g. Nullable(String)) creates a separate column of UInt8 type. This additional column has to be processed every time a user works with a nullable column. This leads to additional storage space used and almost always negatively affects performance.  setting a default value for that column

## Choose a Low Cardinality Partitioning Key
 to minimize the number of write requests to the ClickHouse Cloud object storage, use a low cardinality partitioning key or avoid using any partitioning key for your table.

# Managing ClickHouse
## Performance and Optimizations
### Query Cache
缓存查询结果， SYSTEM DROP QUERY CACHE 清除查询缓存。缓存的内容显示在系统表 system.query_cache 中。默认user级别隔离，可改。  相同的查询可以用query_cache_tag来创建多个cache。部分不缓存dictGet()  now(), today(), yesterday() 等等，可以用 query_cache_nondeterministic_function_handling强制开启缓存
```
<query_cache>
    <max_size_in_bytes>1073741824</max_size_in_bytes>
    <max_entries>1024</max_entries>
    <max_entry_size_in_bytes>1048576</max_entry_size_in_bytes>
    <max_entry_size_in_rows>30000000</max_entry_size_in_rows>
</query_cache>
```

### Quotas
Quotas allow you to limit resource usage over a period of time or track the use of resources. Quotas are set up in the user config, which is usually ‘users.xml’.
```
<!-- Quotas -->
<quotas>
    <!-- Quota name. -->
    <default>
        <!-- Restrictions for a time period. You can set many intervals with different restrictions. -->
        <interval>
            <!-- Length of the interval. -->
            <duration>3600</duration>

            <!-- Unlimited. Just collect data for the specified time interval. -->
            <!-- The total number of requests.-->
            <queries>0</queries>
             <!-- The total number of select requests. -->
            <query_selects>0</query_selects>
            <!-- The total number of insert requests. -->
            <query_inserts>0</query_inserts>
            <!-- The number of queries that threw an exception. -->
            <errors>0</errors>
            <!-- The total number of rows given as a result. -->
            <result_rows>0</result_rows>
            <!-- The total number of source rows read from tables for running the query on all remote servers -->
            <read_rows>0</read_rows>
            <!-- The total query execution time, in seconds (wall time) -->
            <execution_time>0</execution_time>
        </interval>
    </default>
```
## More
### System Tables
# ClickHouse 内置表及用途简介

| 内置表                         | 主要用途                                             |
|--------------------------------|------------------------------------------------------|
| `asynchronous_insert_log`       | 记录异步插入的操作日志。                             |
| `asynchronous_inserts`          | 列出当前的异步插入任务。                             |
| `asynchronous_loader`           | 管理异步插入的数据加载。                             |
| `asynchronous_metric_log`       | 记录异步获取的系统指标日志。                         |
| `asynchronous_metrics`          | 显示异步获取的系统指标。                             |
| `backup_log`                    | 记录备份操作的日志。                                 |
| `blob_storage_log`              | 记录 Blob 存储操作日志。                             |
| `build_options`                 | 显示编译时使用的构建选项。                           |
| `clusters`                      | 显示集群中节点和分片的信息。                         |
| `columns`                       | 列出表的列信息。                                     |
| `contributors`                  | 列出 ClickHouse 的贡献者。                           |
| `crash_log`                     | 记录崩溃日志。                                       |
| `current_roles`                 | 显示当前会话的角色。                                 |
| `dashboards`                    | 显示系统中可用的仪表盘。                             |
| `data_skipping_indices`         | 列出数据跳过索引信息。                               |
| `data_type_families`            | 列出支持的数据类型族。                               |
| `database_engines`              | 列出支持的数据库引擎。                               |
| `databases`                     | 显示所有数据库的信息。                               |
| `detached_parts`                | 显示分离的分区信息。                                 |
| `detached_tables`               | 显示分离的表信息。                                   |
| `dictionaries`                  | 列出外部字典信息。                                   |
| `disks`                         | 显示系统中可用的磁盘。                               |
| `distributed_ddl_queue`         | 分布式DDL队列信息。                                  |
| `distribution_queue`            | 显示分布式表的队列任务。                             |
| `dns_cache`                     | 显示DNS缓存信息。                                    |
| `dropped_tables`                | 显示已删除的表信息。                                 |
| `dropped_tables_parts`          | 显示已删除表的分区信息。                             |
| `enabled_roles`                 | 显示已启用的角色。                                   |
| `error_log`                     | 记录系统错误日志。                                   |
| `errors`                        | 显示系统中的错误信息。                               |
| `events`                        | 记录系统事件。                                       |
| `functions`                     | 列出系统中的函数信息。                               |
| `grants`                        | 显示用户和角色的权限。                               |
| `graphite_retentions`           | Graphite数据保留规则。                               |
| `INFORMATION_SCHEMA`            | 提供信息模式的元数据。                               |
| `jemalloc_bins`                 | 显示jemalloc分配器的内存分配信息。                   |
| `kafka_consumers`               | 显示Kafka消费者的信息。                              |
| `licenses`                      | 显示系统中使用的许可证信息。                         |
| `merge_tree_settings`           | 显示MergeTree表的设置。                              |
| `merges`                        | 监控分区合并操作。                                   |
| `metric_log`                    | 记录系统指标日志。                                   |
| `metrics`                       | 提供系统实时指标。                                   |
| `moves`                         | 显示数据的移动操作。                                 |
| `mutations`                     | 显示表的突变操作。                                   |
| `numbers`                       | 生成连续数字序列。                                   |
| `numbers_mt`                    | 多线程生成连续数字序列。                             |
| `one`                           | 返回常数 `1`。                                       |
| `opentelemetry_span_log`        | 记录 OpenTelemetry 的 span 日志。                    |
| `part_log`                      | 记录分区的操作日志。                                 |
| `parts`                         | 显示表的分区信息。                                   |
| `parts_columns`                 | 显示分区的列信息。                                   |
| `processes`                     | 列出当前运行的查询。                                 |
| `processors_profile_log`        | 记录查询处理器的性能日志。                           |
| `projections`                   | 显示表的投影信息。                                   |
| `query_cache`                   | 显示查询缓存信息。                                   |
| `query_log`                     | 记录查询执行日志。                                   |
| `query_thread_log`              | 记录查询线程的执行日志。                             |
| `query_views_log`               | 记录查询视图的执行日志。                             |
| `quota_limits`                  | 显示配额的限制。                                     |
| `quota_usage`                   | 显示配额的使用情况。                                 |
| `quotas`                        | 显示系统中的配额。                                   |
| `quotas_usage`                  | 显示配额的详细使用情况。                             |
| `replicas`                      | 显示复制表的状态。                                   |
| `replicated_fetches`            | 监控从其他副本获取数据的操作。                       |
| `replication_queue`             | 复制队列的任务信息。                                 |
| `role_grants`                   | 显示角色分配的权限。                                 |
| `roles`                         | 显示系统中定义的角色。                               |
| `row_policies`                  | 显示行级别安全策略。                                 |
| `scheduler`                     | 显示调度任务的信息。                                 |
| `schema_inference_cache`        | 显示自动推断的表结构缓存。                           |
| `server_settings`               | 显示服务器的配置选项。                               |
| `session_log`                   | 记录用户会话日志。                                   |
| `settings`                      | 列出当前的系统设置。                                 |
| `settings_changes`              | 记录设置的变更历史。                                 |
| `settings_profile_elements`     | 显示设置配置文件的元素。                             |
| `settings_profiles`             | 显示系统中的设置配置文件。                           |
| `stack_trace`                   | 显示系统的堆栈跟踪信息。                             |
| `storage_policies`              | 显示存储策略信息。                                   |
| `symbols`                       | 显示系统中的符号表。                                 |
| `table_engines`                 | 列出支持的表引擎。                                   |
| `tables`                        | 显示数据库中表的信息。                               |
| `text_log`                      | 记录系统文本日志。                                   |
| `time_zones`                    | 显示支持的时区列表。                                 |
| `trace_log`                     | 记录系统跟踪日志。                                   |
| `user_processes`                | 显示用户的进程信息。                                 |
| `users`                         | 显示系统中定义的用户信息。                           |
| `view_refreshes`                | 显示视图刷新任务的信息。                             |
| `zookeeper`                     | 提供ZooKeeper的节点信息。                            |
| `zookeeper_connection`          | 显示ZooKeeper的连接信息。                            |
| `zookeeper_log`                 | 记录ZooKeeper的操作日志。                            |


# Data Types
### Integer types
UInt8, UInt16, UInt32, UInt64, UInt128, UInt256, Int8, Int16, Int32, Int64, Int128, Int256
Aliases:  
UInt8 — TINYINT UNSIGNED, INT1 UNSIGNED.  
UInt16 — SMALLINT UNSIGNED.  
UInt32 — MEDIUMINT UNSIGNED, INT UNSIGNED, INTEGER UNSIGNED  
UInt64 — UNSIGNED, BIGINT UNSIGNED, BIT, SET  
### Float types
Float32 — float.  Float64 — double.  
Aliases: Float32 — FLOAT, REAL, SINGLE. Float64 — DOUBLE, DOUBLE PRECISION.  
e.g. FLOAT(12), FLOAT(15, 22), DOUBLE(12), DOUBLE(4, 18)

### Decimal types
Decimal, Decimal(P), Decimal(P, S), Decimal32(S), Decimal64(S), Decimal128(S), Decimal256(S)  
P - precision. Valid range: [ 1 : 76 ]. Determines how many decimal digits number can have (including fraction). By default, the precision is 10.  
S - scale. Valid range: [ 0 : P ]. Determines how many decimal digits fraction can have.  

### String
Aliases:  
String — LONGTEXT, MEDIUMTEXT, TINYTEXT, TEXT, LONGBLOB, MEDIUMBLOB, TINYBLOB, BLOB, VARCHAR, CHAR, CHAR LARGE OBJECT, CHAR VARYING, CHARACTER LARGE OBJECT, CHARACTER VARYING, NCHAR LARGE OBJECT, NCHAR VARYING, NATIONAL CHARACTER LARGE OBJECT, NATIONAL CHARACTER VARYING, NATIONAL CHAR VARYING, NATIONAL CHARACTER, NATIONAL CHAR, BINARY LARGE OBJECT, BINARY VARYING
### FixedString(N)
A fixed-length string of N bytes (neither characters nor code points).
### Date
Supported range of values: [1970-01-01, 2149-06-06].
### Date32
Supports the date range same with DateTime64. Stored as a signed 32-bit integer in native byte order with the value representing the days since 1970-01-01 (0 represents 1970-01-01 and negative values represent the days before 1970).
### DateTime
Supported range of values: [1970-01-01 00:00:00, 2106-02-07 06:28:15].
Resolution: 1 second.
### DateTime64
DateTime64(precision, [timezone]) Tick size (precision): 10-precision seconds. Valid range: [ 0 : 9 ]. Typically, are used - 3 (milliseconds), 6 (microseconds), 9 (nanoseconds).
Supported range of values: [1900-01-01 00:00:00, 2299-12-31 23:59:59.99999999]
### Enum
Enum8 or Enum16 types to be sure in the size of storage.  
8-bit Enum. It can contain up to 256 values enumerated in the [-128, 127] range.  
16-bit Enum. It can contain up to 65536 values enumerated in the [-32768, 32767] range.  
### Bool
Type bool is internally stored as UInt8. Possible values are true (1), false (0).
### UUID
 a 16-byte value 
### LowCardinality(T)
LowCardinality(data_type) data_type — String, FixedString, Date, DateTime, and numbers excepting Decimal. LowCardinality is not efficient for some data types, see the allow_suspicious_low_cardinality_types setting description.
### Geo
Point is represented by its X and Y coordinates, stored as a Tuple(Float64, Float64).
### JSON
**Point**  Stores JavaScript Object Notation (JSON) documents in a single column.  
**Ring**   stored as an array of points: Array(Point).
```
CREATE TABLE geo_ring (r Ring) ENGINE = Memory();
INSERT INTO geo_ring VALUES([(0, 0), (10, 0), (10, 10), (0, 10)]);
```
**LineString** 
```
CREATE TABLE geo_linestring (l LineString) ENGINE = Memory();
INSERT INTO geo_linestring VALUES([(0, 0), (10, 0), (10, 10), (0, 10)]);
```

**MultiLineString**
```
CREATE TABLE geo_multilinestring (l MultiLineString) ENGINE = Memory();
INSERT INTO geo_multilinestring VALUES([[(0, 0), (10, 0), (10, 10), (0, 10)], [(1, 1), (2, 2), (3, 3)]]);
```

**Polygon**
```
CREATE TABLE geo_polygon (pg Polygon) ENGINE = Memory();
INSERT INTO geo_polygon VALUES([[(20, 20), (50, 20), (50, 50), (20, 50)], [(30, 30), (50, 50), (50, 30)]]);
SELECT pg, toTypeName(pg) FROM geo_polygon;
```

**MultiPolygon**
```
CREATE TABLE geo_multipolygon (mpg MultiPolygon) ENGINE = Memory();
INSERT INTO geo_multipolygon VALUES([[[(0, 0), (10, 0), (10, 10), (0, 10)]], [[(20, 20), (50, 20), (50, 50), (20, 50)],[(30, 30), (50, 50), (50, 30)]]]);
SELECT mpg, toTypeName(mpg) FROM geo_multipolygon;
```


## Database Engines
### Atomic
It supports non-blocking DROP TABLE and RENAME TABLE queries and atomic EXCHANGE TABLES queries. Atomic database engine is used by default.

### Replicated
based on the Atomic engine. It supports replication of metadata via DDL log being written to ZooKeeper and executed on all of the replicas for a given database.

### Table Engines
#### MergeTree Engine Family
####  MergeTree
high data ingest rates and huge data volumes. Insert operations create table parts which are merged by a background process with other table parts.  
Main features of MergeTree-family table engines.  
- The table's primary key determines the sort order within each table part (clustered index). The primary key also does not reference individual rows but blocks of 8192 rows called granules. This makes primary keys of huge data sets small enough to remain loaded in main memory, while still providing fast access to on-disk data.
- Tables can be partitioned using an arbitrary partition expression. Partition pruning ensures partitions are omitted from reading when the query allows it.
- Data can be replicated across multiple cluster nodes for high availability, failover, and zero downtime upgrades.
- support various statistics kinds and sampling methods to help query optimization
```
ENGINE MergeTree() PARTITION BY toYYYYMM(EventDate) ORDER BY (CounterID, EventDate, intHash32(UserID)) SAMPLE BY intHash32(UserID) SETTINGS index_granularity=8192
```
**ORDER_BY** A tuple of column names or arbitrary expressions If no primary key is defined (i.e. PRIMARY KEY was not specified), ClickHouse uses the the sorting key as primary key. If no sorting is required, you can use syntax ORDER BY tuple()  
**PARTITION BY** Optional. In most cases, you don't need a partition key Partitioning does not speed up queries For partitioning by month, use the toYYYYMM(date_column) expression, where date_column is a column with a date of the type Date. The partition names here have the "YYYYMM" format.  
**PRIMARY KEY**   it differs from the sorting key. Optional. Specifying a sorting key (using ORDER BY clause) implicitly specifies a primary key. It is usually not necessary to specify the primary key in addition to the sorting key.
**SAMPLE BY**  A sampling expression. Optional If specified, it must be contained in the primary key. The sampling expression must result in an unsigned integer.  
**TTL** A list of rules that specify the storage duration of rows and the logic of automatic parts movement between disks and volumes. Optional.Expression must result in a Date or DateTime, e.g. TTL date + INTERVAL 1 DAY.  
##### Data Storage
Data belonging to different partitions are separated into different parts. In the background, ClickHouse merges data parts for more efficient storage. Parts belonging to different partitions are not merged. The merge mechanism does not guarantee that all rows with the same primary key will be in the same data part.
Data parts format 
**Wide**  each column is stored in a separate file in a filesystem  
**Compact**  all columns are stored in one file. can be used to increase performance of small and frequent inserts.  
Data storing format is controlled by the **min_bytes_for_wide_part** and **min_rows_for_wide_part** settings of the table engine. If the number of bytes or rows in a data part is less then the corresponding setting's value, the part is stored in Compact format. Otherwise it is stored in Wide format. If none of these settings is set, data parts are stored in Wide format.  
data part is logically divided into granules， granule size is restricted by the **index_granularity** and **index_granularity_bytes** settings of the table engine. The number of rows in a granule lays in the [1, index_granularity] range, depending on the size of the rows. The size of a granule can exceed index_granularity_bytes if the size of a single row is greater than the value of the setting. In this case, the size of the granule equals the size of the row.

##### TTL for Columns and Tables
```
TTL date_time + INTERVAL 1 MONTH
ALTER TABLE tabMODIFY COLUMN c String TTL d + INTERVAL 1 DAY;

CREATE TABLE tab
(
    d DateTime,
    a Int
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(d)
ORDER BY d
TTL d + INTERVAL 1 MONTH DELETE,
    d + INTERVAL 1 WEEK TO VOLUME 'aaa',
    d + INTERVAL 2 WEEK TO DISK 'bbb';

ALTER TABLE tab MODIFY TTL d + INTERVAL 1 DAY;
```

##### Removing Expired Data
it performs an off-schedule merge. To control the frequency of such merges, you can set **merge_with_ttl_timeout**. If the value is too low, it will perform many off-schedule merges that may consume a lot of resources.  
If you perform the SELECT query between merges, you may get expired data. To avoid it, use the OPTIMIZE query before SELECT.  

#### Data Replication
ReplicatedMergeTree  
ReplicatedSummingMergeTree
ReplicatedReplacingMergeTree
ReplicatedAggregatingMergeTree
ReplicatedCollapsingMergeTree
ReplicatedVersionedCollapsingMergeTree
ReplicatedGraphiteMergeTree  
Compressed data for INSERT and ALTER queries is replicated,  CREATE, DROP, ATTACH, DETACH and RENAME queries are executed on a single server and are not replicated

#### Custom Partitioning Key
可多列，PARTITION BY (toMonday(StartDate), EventType). system.parts table to view the table parts and partitions. system.parts table **active** column shows the status of the part. 1 is active; 0 is inactive. The inactive parts are, for example, source parts remaining after merging to a larger part. The corrupted data parts are also indicated as inactive. Inactive parts will be deleted approximately 10 minutes after merging.  

#### Group By optimisation using partition key
让每个分区都包含不同的数据，每个线程搜索结果就不需要聚合  
- number of partitions involved in the query should be sufficiently large (more than max_threads / 2), otherwise query will under-utilize the machine
- partitions shouldn't be too small, so batch processing won't degenerate into row-by-row processing
- partitions should be comparable in size, so all threads will do roughly the same amount of work
#### ReplacingMergeTree
it removes duplicate entries with the same sorting key value (ORDER BY table section, not PRIMARY KEY). ReplacingMergeTree([ver [, is_deleted]])  
**ver** UInt*, Date, DateTime or DateTime64 类型，没有设置或者ver相同，就用part中最后一次插入的。
**is_deleted** Name of a column，data type — UInt8. 1 is a "deleted",0 is a "state" row(保留). is_deleted can only be enabled when ver is used. The row is deleted only when OPTIMIZE ... FINAL CLEANUP. This CLEANUP special keyword is not allowed by default unless allow_experimental_replacing_merge_with_cleanup MergeTree setting is enabled.
```
-- with ver and is_deleted
CREATE OR REPLACE TABLE myThirdReplacingMT
(
    `key` Int64,
    `someCol` String,
    `eventTime` DateTime,
    `is_deleted` UInt8
)
ENGINE = ReplacingMergeTree(eventTime, is_deleted)
ORDER BY key
SETTINGS allow_experimental_replacing_merge_with_cleanup = 1;

INSERT INTO myThirdReplacingMT Values (1, 'first', '2020-01-01 01:01:01', 0);
INSERT INTO myThirdReplacingMT Values (1, 'first', '2020-01-01 01:01:01', 1);

select * from myThirdReplacingMT final;

0 rows in set. Elapsed: 0.003 sec.

-- delete rows with is_deleted
OPTIMIZE TABLE myThirdReplacingMT FINAL CLEANUP;

INSERT INTO myThirdReplacingMT Values (1, 'first', '2020-01-01 00:00:00', 0);

select * from myThirdReplacingMT final;

┌─key─┬─someCol─┬───────────eventTime─┬─is_deleted─┐
│   1 │ first   │ 2020-01-01 00:00:00 │          0 │
└─────┴─────────┴─────────────────────┴────────────┘
```

#### SummingMergeTree
 replaces all the rows with the same primary key (or more accurately, with the same sorting key) with one row which contains summarized values for the columns with the numeric data type. If the sorting key is composed in a way that a single key value corresponds to large number of rows, this significantly reduces storage volume and speeds up data selection. ENGINE = SummingMergeTree([columns])  
 **columns** - a tuple with the names of columns where values will be summarized. Optional parameter. If columns is not specified, ClickHouse summarizes the values in all columns with a numeric data type that are not in the primary key.  
 数据定期做sum，所以查询时要用ggregate function，（SELECT) an aggregate function sum() and GROUP BY.
 ##### Common Rules for Summation
 If the values were 0 in all of the columns for summation, the row is deleted. If column is not in the primary key and is not summarized, an arbitrary value is selected from the existing ones(非主键且非汇总的字段：这些字段在数据合并时不会参与聚合，因此ClickHouse只会从重复的行数据中随意取一个值。).The values are not summarized for columns in the primary key.  
 ##### AggregatingMergeTree
 replaces all rows with the same primary key (or more accurately, with the same sorting key) with a single row (within a one data part) that stores a combination of states of aggregate functions.
 ```
 CREATE TABLE test.visits
 (
    StartDate DateTime64 NOT NULL,
    CounterID UInt64,
    Sign Nullable(Int32),
    UserID Nullable(Int32)
) ENGINE = MergeTree ORDER BY (StartDate, CounterID);
 ```
 #### CollapsingMergeTree
 CollapsingMergeTree asynchronously deletes (collapses) pairs of rows if all of the fields in a sorting key (ORDER BY) are equivalent except the particular field Sign, which can have 1 and -1 values. Rows without a pair are kept. ENGINE = CollapsingMergeTree(sign)
 **sign** Name of the column Int8.1 is a “state” row, -1 is a “cancel” row.  
 each group of consecutive rows with the same sorting key (ORDER BY) is reduced to not more than two rows, one with Sign = 1 (“state” row) and another with Sign = -1 (“cancel” row). In other words, entries collapse.
- The first “cancel” and the last “state” rows, if the number of “state” and “cancel” rows matches and the last row is a “state” row.
- The last “state” row, if there are more “state” rows than “cancel” rows.
- The first “cancel” row, if there are more “cancel” rows than “state” rows.
- None of the rows, in all other cases.
至多保留两条（“cancel”+“state”），当有多条记录时，state 记录保留最后一条，cancel 记录保留第一条。  
#### VersionedCollapsingMergeTree
Allows quick writing of object states that are continually changing. Deletes old object states in the background. This significantly reduces the volume of storage. ENGINE = VersionedCollapsingMergeTree(sign, version)
**sign**  Name of the column with the type of row: 1 is a “state” row, -1 is a “cancel” row.  
**version** Name of the column with the version of the object state. Type UInt*  
When ClickHouse merges data parts, it deletes each pair of rows that have the same primary key and version and different Sign. The order of rows does not matter.   
When ClickHouse inserts data, it orders rows by the primary key. If the Version column is not in the primary key, ClickHouse adds it to the primary key implicitly as the last field and uses it for ordering.  

#### Log Engine Family
for quickly write many small tables (up to about 1 million rows) and read them later as a whole.  
**TinyLog** stores each column in a separate file，不支持并行读。 **Log** uses a separate file for each column of the table 读效率最高 **StripeLog**  stores all the data in one file ，文件描述用的更少，读效率相对低。都支持并行读  