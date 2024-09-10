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
