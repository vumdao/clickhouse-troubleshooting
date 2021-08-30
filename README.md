<p align="center">
  <a href="https://dev.to/vumdao">
    <img alt="Clickhouse Server - Troubleshooting" src="https://github.com/vumdao/clickhouse-troubleshooting/blob/master/img/clickhouse.png?raw=true" width="800" />
  </a>
</p>
<h1 align="center">
  <div><b>Clickhouse Server - Troubleshooting</b></div>
</h1>

## Abstract
- When we get clickhouse performance issue like high CPU usage, in order to investigate what is the real problem and how to solve or provide workaround, we need to understand Clickhouse system/user config attributes.

## Table Of Contents
 * [max_part_loading_threads](#max_part_loading_threads)
 * [max_part_removal_threads](#max_part_removal_threads)
 * [number_of_free_entries_in_pool_to_execute_mutation](#number_of_free_entries_in_pool_to_execute_mutation)
 * [background_pool_size](#background_pool_size)
 * [background_schedule_pool_size](#background_schedule_pool_size)
 * [max_threads](#max_threads)
 * [Get tables size](#Get-tables-size)
 * [Understand clickhouse](#Understand-clickhouse-Compression)
 * [Enable allow_introspection_functions for query profiling](#Enable-allow_introspection_functions-for-query-profiling)


## 🚀 **max_part_loading_threads** <a name="max_part_loading_threads"></a>
The maximum number of threads that read parts when ClickHouse starts.

- Possible values:
  Any positive integer.
  Default value: auto (number of CPU cores).

- During startup ClickHouse reads all parts of all tables (reads files with metadata of parts) to build a list of all parts in memory. In some systems with a large number of parts this process can take a long time, and this time might be shortened by increasing max_part_loading_threads (if this process is not CPU and disk I/O bound).

- Query check
```
SELECT *
FROM system.merge_tree_settings
WHERE name = 'max_part_loading_threads'

Query id: 5f8c7c7a-5dec-4e89-88dc-71f06d800e04

┌─name─────────────────────┬─value─────┬─changed─┬─description──────────────────────────────────────────┬─type───────┐
│ max_part_loading_threads │ 'auto(4)' │       0 │ The number of threads to load data parts at startup. │ MaxThreads │
└──────────────────────────┴───────────┴─────────┴──────────────────────────────────────────────────────┴────────────┘

1 rows in set. Elapsed: 0.003 sec. 
```

## 🚀 max_part_removal_threads <a name="max_part_removal_threads"></a>
The number of threads for concurrent removal of inactive data parts. One is usually enough, but in ‘Google Compute Environment SSD Persistent Disks’ file removal (unlink) operation is extraordinarily slow and you probably have to increase this number (recommended is up to 16).

## 🚀 number_of_free_entries_in_pool_to_execute_mutation <a name="number_of_free_entries_in_pool_to_execute_mutation"></a>
- This attribute must be align with `background_pool_size`, its values must be <= value of `background_pool_size`

```
SELECT *
FROM system.merge_tree_settings
WHERE name = 'number_of_free_entries_in_pool_to_execute_mutation'

┌─name───────────────────────────────────────────────┬─value─┬─changed─┬─description──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ number_of_free_entries_in_pool_to_execute_mutation │ 10    │       0 │ When there is less than specified number of free entries in pool, do not execute part mutations. This is to leave free threads for regular merges and avoid "Too many parts" │
└────────────────────────────────────────────────────┴───────┴─────────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## 🚀 background_pool_size <a name="background_pool_size"></a>
- [background_pool_size](https://clickhouse.tech/docs/en/operations/settings/settings/#background_pool_size)
  Sets the number of threads performing background operations in table engines (for example, merges in MergeTree engine tables). This setting is applied from thedefault profile at the ClickHouse server start and can’t be changed in a user session. By adjusting this setting, you manage CPU and disk load. Smaller pool sizeutilizes less CPU and disk resources, but background processes advance slower which might eventually impact query performance.

- Before changing it, please also take a look at related MergeTree settings, such as number_of_free_entries_in_pool_to_lower_max_size_of_merge andnumber_of_free_entries_in_pool_to_execute_mutation.

- Possible values:
  Any positive integer.
  Default value: 16.

- Start log

```
2021.08.29 04:22:30.824446 [ 12372 ] {} <Information> BackgroundSchedulePool/BgSchPool: Create BackgroundSchedulePool with 16 threads
2021.08.29 04:22:47.891697 [ 12363 ] {} <Information> Application: Available RAM: 15.08 GiB; physical cores: 4; logical cores: 8.
```

- How to update this value eg. 5

  Update `config.xml`
  ```
      <merge_tree>
        <number_of_free_entries_in_pool_to_execute_mutation>5</number_of_free_entries_in_pool_to_execute_mutation>
      </merge_tree>
  ```
  Update `users.xml`
  ```
    <profiles>
        <default>
            <background_pool_size>5</background_pool_size>
        </default>
    </profiles>
  ```

## 🚀 background_schedule_pool_size <a name="background_schedule_pool_size"></a>
- [background_schedule_pool_size](https://clickhouse.tech/docs/en/operations/settings/settings/#background_schedule_pool_size)
  Sets the number of threads performing background tasks for replicated tables, Kafka streaming, DNS cache updates. This setting is applied at ClickHouse server start and can’t be changed in a user session.

- Possible values:
    Any positive integer.
    Default value: 128.

- How to update this value? - At user profile -> update `users.xml` (disable `background_schedule_pool_size` if we don't use `ReplicatedMergeTree` engine)
  ```
    <profiles>
      <default>
          <background_schedule_pool_size>0</background_schedule_pool_size>
      </default>
    </profiles>
  ```

- Get pool size
```
SELECT
    name,
    value
FROM system.settings
WHERE name LIKE '%pool%'

┌─name─────────────────────────────────────────┬─value─┐
│ connection_pool_max_wait_ms                  │ 0     │
│ distributed_connections_pool_size            │ 1024  │
│ background_buffer_flush_schedule_pool_size   │ 16    │
│ background_pool_size                         │ 100   │
│ background_move_pool_size                    │ 8     │
│ background_fetches_pool_size                 │ 8     │
│ background_schedule_pool_size                │ 0     │
│ background_message_broker_schedule_pool_size │ 16    │
│ background_distributed_schedule_pool_size    │ 16    │
└──────────────────────────────────────────────┴───────┘
```

- Get background pool task
```
SELECT
    metric,
    value
FROM system.metrics
WHERE metric LIKE 'Background%'

┌─metric──────────────────────────────────┬─value─┐
│ BackgroundPoolTask                      │     0 │
│ BackgroundFetchesPoolTask               │     0 │
│ BackgroundMovePoolTask                  │     0 │
│ BackgroundSchedulePoolTask              │     0 │
│ BackgroundBufferFlushSchedulePoolTask   │     0 │
│ BackgroundDistributedSchedulePoolTask   │     0 │
│ BackgroundMessageBrokerSchedulePoolTask │     0 │
└─────────────────────────────────────────┴───────┘
```

- Get BgSchPool
```
# ps H -o 'tid comm' $(pidof -s clickhouse-server) |  tail -n +2 | awk '{ printf("%s\t%s\n", $1, $2) }' | grep BgSchPool
7346    BgSchPool/D
```

- [Viewing cluster](https://support.huaweicloud.com/intl/en-us/cmpntguide-mrs/mrs_01_2398.html)
```
SELECT 
    cluster, 
    shard_num, 
    replica_num, 
    host_name
FROM system.clusters

┌─cluster───────────────────────────┬─shard_num─┬─replica_num─┬─host_name─┐
│ test_cluster_two_shards           │         1 │           1 │ 127.0.0.1 │
│ test_cluster_two_shards           │         2 │           1 │ 127.0.0.2 │
│ test_cluster_two_shards_localhost │         1 │           1 │ localhost │
│ test_cluster_two_shards_localhost │         2 │           1 │ localhost │
│ test_shard_localhost              │         1 │           1 │ localhost │
│ test_shard_localhost_secure       │         1 │           1 │ localhost │
│ test_unavailable_shard            │         1 │           1 │ localhost │
│ test_unavailable_shard            │         2 │           1 │ localhost │
└───────────────────────────────────┴───────────┴─────────────┴───────────┘
```

## 🚀 max_threads <a name="max_threads"></a>
[max_threads](https://clickhouse.tech/docs/en/operations/settings/settings/#settings-max_threads)
  - The maximum number of query processing threads, excluding threads for retrieving data from remote servers (see the ‘max_distributed_connections’ parameter).

  - This parameter applies to threads that perform the same stages of the query processing pipeline in parallel.
  For example, when reading from a table, if it is possible to evaluate expressions with functions, filter with WHERE and pre-aggregate for GROUP BY in parallel using at least ‘max_threads’ number of threads, then ‘max_threads’ are used.

  - Default value: the number of physical CPU cores.

  - For queries that are completed quickly because of a LIMIT, you can set a lower ‘max_threads’. For example, if the necessary number of entries are located in every block and max_threads = 8, then 8 blocks are retrieved, although it would have been enough to read just one.

  - The smaller the max_threads value, the less memory is consumed.

  - Update this value at user profile

## 🚀 Get tables size <a name="Get-tables-size"></a>
[clickhouse-get-tables-size.sql](https://gist.github.com/sanchezzzhak/511fd140e8809857f8f1d84ddb937015)
```
select concat(database, '.', table)                         as table,
       formatReadableSize(sum(bytes))                       as size,
       sum(rows)                                            as rows,
       max(modification_time)                               as latest_modification,
       sum(bytes)                                           as bytes_size,
       any(engine)                                          as engine,
       formatReadableSize(sum(primary_key_bytes_in_memory)) as primary_keys_size
from system.parts
where active
group by database, table
order by bytes_size desc
```

- For table detail of database
```
select parts.*,
       columns.compressed_size,
       columns.uncompressed_size,
       columns.ratio
from (
         select table,
                formatReadableSize(sum(data_uncompressed_bytes))          AS uncompressed_size,
                formatReadableSize(sum(data_compressed_bytes))            AS compressed_size,
                sum(data_compressed_bytes) / sum(data_uncompressed_bytes) AS ratio
         from system.columns
         where database = currentDatabase()
         group by table
         ) columns
         right join (
    select table,
           sum(rows)                                            as rows,
           max(modification_time)                               as latest_modification,
           formatReadableSize(sum(bytes))                       as disk_size,
           formatReadableSize(sum(primary_key_bytes_in_memory)) as primary_keys_size,
           any(engine)                                          as engine,
           sum(bytes)                                           as bytes_size
    from system.parts
    where active and database = currentDatabase()
    group by database, table
    ) parts on columns.table = parts.table
order by parts.bytes_size desc;
```

## 🚀 Understand clickhouse Compression <a name="Understand-clickhouse-Compression"></a>
- [Compression in ClickHouse](https://altinity.com/blog/2017/11/21/compression-in-clickhouse)

## 🚀 Enable allow_introspection_functions for query profiling <a name="Enable-allow_introspection_functions-for-query-profiling"></a>
- [Introspection Functions](https://clickhouse.tech/docs/en/sql-reference/functions/introspection/). Update at user profile
```
        <default>
            <allow_introspection_functions>1</allow_introspection_functions>
        </default>
```

- Get thread stack trace
```
WITH arrayMap(x -> demangle(addressToSymbol(x)), trace) AS all
SELECT
    thread_id,
    query_id,
    arrayStringConcat(all, '\n') AS res
FROM system.stack_trace
WHERE res LIKE '%SchedulePool%'

┌─thread_id─┬─query_id─┬─res──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│      7346 │          │ pthread_cond_wait
DB::BackgroundSchedulePool::delayExecutionThreadFunction()

ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>)

start_thread
clone │
└───────────┴──────────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## 🚀 parts_to_throw_insert
- [parts_to_throw_insert](https://clickhouse.tech/docs/en/operations/settings/merge-tree-settings/#parts-to-throw-insert)
  If the number of active parts in a single partition exceeds the parts_to_throw_insert value, INSERT is interrupted with the Too many parts (N). Merges are processing significantly slower than inserts exception.

  Possible values:
    Any positive integer.
    Default value: 300.

  To achieve maximum performance of SELECT queries, it is necessary to minimize the number of parts processed, see Merge Tree.

  You can set a larger value to 600 (1200), this will reduce the probability of the Too many parts error, but at the same time SELECT performance might degrade. Also in case of a merge issue (for example, due to insufficient disk space) you will notice it later than it could be with the original 300.

- [Facing issue?](Ref: https://github.com/ClickHouse/ClickHouse/issues/3174)
```
2021.08.30 11:30:44.526367 [ 7369 ] {} <Error> void DB::SystemLog<DB::MetricLogElement>::flushImpl(const std::vector<LogElement> &, uint64_t) [LogElement = DB::MetricLogElement]: Code: 252, e.displayText() = DB::Exception: Too many parts (300). Parts cleaning are processing significantly slower than inserts, Stack trace (when copying this message, always include the lines below):
```

- And you decide to increase `parts_to_throw_insert` -> Update `config.xml`
```
    <merge_tree>
         <parts_to_throw_insert>600</parts_to_throw_insert>
    </merge_tree>
```

---

<h3 align="center">
  <a href="https://dev.to/vumdao">:stars: Blog</a>
  <span> · </span>
  <a href="https://github.com/vumdao/clickhouse-troubleshooting/">Github</a>
  <span> · </span>
  <a href="https://stackoverflow.com/users/11430272/vumdao">stackoverflow</a>
  <span> · </span>
  <a href="https://www.linkedin.com/in/vu-dao-9280ab43/">Linkedin</a>
  <span> · </span>
  <a href="https://www.linkedin.com/groups/12488649/">Group</a>
  <span> · </span>
  <a href="https://www.facebook.com/CloudOpz-104917804863956">Page</a>
  <span> · </span>
  <a href="https://twitter.com/VuDao81124667">Twitter :stars:</a>
</h3>
