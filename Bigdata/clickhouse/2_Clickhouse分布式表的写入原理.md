# Distributed 表引擎的介绍

`Distributed` 表引擎是一种特殊的表引擎，自身不会存储任何数据，而是通过读取或写入其他远端节点上的表进行数据处理的表引擎，该表引擎需要依赖各个节点的本地表来创建。

```SQL
Distributed(cluster, database, table[, sharding_key][, storage_policy])
```

`Distributed` 表引擎有5个参数，前三个参数是指定本地表所在集群的数据库和表名：

- `sharding_key` ：可选参数，默认为`null`，仅集群只有一个`shard`时，可以正常工作。在数据写入的过程中，分布式表会依据分片键的规则，将数据分布到各个节点的本地表。一般可以指定`rand()` 来随机分配数据，或者可以指定为主键列的`hash`值，如`xxHash32(order_id)`，这样可以保证主键相同的数据落在同一个`shard`上，可以用于去重或聚合；
- `storage_policy`：可选参数，默认为`default`，用于异步写入时，临时存储的本地磁盘的路径；

# Distributed 表写入原理

这里略过 SQL 的词法解析、语法解析等步骤，直接从输出流`DistributedBlockOutputStream`的`write`方法开始。

```c++
void DistributedBlockOutputStream::write(const Block & block)
{
    Block ordinary_block{ block };
  
    if (insert_sync)
        writeSync(ordinary_block);
    else
        writeAsync(ordinary_block);
}
```

在`Clickhouse`中数据都是由`Block`来组织的，调用`write`方法写一批数据。上面可以看到写入的过程中分为同步写和异步写，通过`insert_sync`来控制，该参数的值是由`insert_distributed_sync`配置的，默认为`false`。接下来分别介绍同步写和异步写。

## 同步写

同步写主要分为以下四步：

1. 首先在`initWritingJobs`方法中会初始化每个shard的副本任务(`JobReplica`)。
2. 根据`sharding_key`创建选择器`Selector`，遍历所有的行，根据`Selector`划分每个`shard`任务的数据。
3. 通过线程池`pool`执行每个`shard`上的所有副本任务。
4. 等待所有副本任务执行完成。

```c++
void DistributedBlockOutputStream::writeSync(const Block & block)
{
    if (!pool)
    {
        initWritingJobs(block, start, end);
    }
  
    if (num_shards > 1)
    {
        auto current_selector = createSelector(block);
				/// 拆分数据
        for (size_t i = 0; i < block.rows(); ++i)
            per_shard_jobs[current_selector[i]].shard_current_block_permutation.push_back(i);
    }
  	/// 执行每个shard所有的副本任务
    for (size_t shard_index : collections::range(0, shards_info.size()))
        for (JobReplica & job : per_shard_jobs[shard_index].replicas_jobs)
            pool->scheduleOrThrowOnError(runWritingJob(job, block, num_shards));

    waitForJobs();
}
```
下面将详细分析同步写的前三步。


### 如何初始化任务

在`initWritingJobs`方法初始化副本任务过程中会判断`shard_info.hasInternalReplication()` ，其实就是对应着配置文件中的 `internal_replication` 参数，如果为`true`，则每个`shard`只会有一个副本任务，如果为`false`，则会为每个`shard`创建所有的副本任务，即为`ture`写操作只选一个正常的副本写入数据，为`false`写操作会将数据写入所有的副本，如果分布式表的底表是复制表(`*ReplicaMergeTree`)，需要配置为`true`，将数据的复制工作交给实际需要写入数据的表而不是分布式表。

```c++
void DistributedBlockOutputStream::initWritingJobs(const Block & first_block, size_t start, size_t end)
{
    per_shard_jobs.resize(shards_info.size());
    for (size_t shard_index : collections::range(start, end))
    {
        const auto & shard_info = shards_info[shard_index];
        auto & shard_jobs = per_shard_jobs[shard_index];
        const auto & replicas = addresses_with_failovers[shard_index];

        for (size_t replica_index : collections::range(0, replicas.size()))
        {
            shard_jobs.replicas_jobs.emplace_back(shard_index, replica_index, false, first_block);
            /// 如果internal_replication为true，只添加了一个副本任务
            if (shard_info.hasInternalReplication())
                break;
        }
    }
}
```
### 如何根据sharding_key拆分数据

拆分数据是在上面的第二步，首先看一下创建`Selector`的过程。

```c++
IColumn::Selector DistributedBlockOutputStream::createSelector(const Block & source_block) const
{
    Block current_block_with_sharding_key_expr = source_block;
    storage.getShardingKeyExpr()->execute(current_block_with_sharding_key_expr); // 执行sharding_key
    const auto & key_column = current_block_with_sharding_key_expr.getByName(storage.getShardingKeyColumnName());
  
    const auto & slot_to_shard = cluster->getSlotToShard();
    const auto total_weight = slots.size();
    size_t num_rows = column.size();
    IColumn::Selector selector(num_rows);
    using UnsignedT = make_unsigned_t<T>;

    if (isColumnConst(column))    // 区分常量列还是非常量列
    {
        const auto data = assert_cast<const ColumnConst &>(column).getValue<T>();
        const auto shard_num = slots[static_cast<UnsignedT>(data) % total_weight];
        selector.assign(num_rows, shard_num);
    } else {
        using TUInt32Or64 = std::conditional_t<sizeof(UnsignedT) <= 4, UInt32, UInt64>;
        libdivide::divider<TUInt32Or64> divider(total_weight);
        const auto & data = typeid_cast<const ColumnVector<T> &>(column).getData();

        for (size_t i = 0; i < num_rows; ++i)
            selector[i] = slots[static_cast<TUInt32Or64>(data[i]) - (static_cast<TUInt32Or64>(data[i]) / divider) * total_weight];
    }
    return selector
}
```

1. 获取到`sharding_key`表达式，将表达式应用到所有的行，得到一列`key_column`，保存着每一行表达式计算的结果。
2. 获取`shard`插槽(`slot_to_shard`)，这里为什么不直接拿`shard_info`列表，原因是为了处理权重，`slot_to_shard`保存的是`shard`的索引，当`metrika.xml` 的集群配置中第一`shard`的权重为10，那么就会往`slot_to_shard`添加10个0，第二`shard`的权重为5，则会往`slot_to_shard`中添加5个1。
3. 判断`sharing_key`的计算结果是否是常量列(即这一列的值都是相同的)，如果是常量列直接取第一个值模于`slot_to_shard`长度得到`slot_to_shard`的索引，通过索引获取`shard`索引，并填充长度为`num_rows`的`selector`，即得到的一个保存了每一行数据该划分到那个`shard`的索引列表。如果非常量列，则遍历所有的行，计算得到对应的`shard`索引。

当拿到`selector`后，就可以遍历所有行，将每一行的索引添加到对应的`shard`任务上，即完成了数据的拆分。



### 如何执行副本任务

执行副本任务的核心逻辑在`runWritingJob`方法。

```c++
ThreadPool::Job
DistributedBlockOutputStream::runWritingJob(DistributedBlockOutputStream::JobReplica & job, const Block & current_block, size_t num_shards)
{
    auto thread_group = CurrentThread::getGroup();
    return [this, thread_group, &job, ¤t_block, num_shards]()
    {
        const auto & shard_info = cluster->getShardsInfo()[job.shard_index];
        auto & shard_job = per_shard_jobs[job.shard_index];

        if (shard_block.rows() == 0)
            return;

        if (!job.is_local_job || !settings.prefer_localhost_replica)
        {
            if (!job.stream) {
                job.stream = std::make_shared<RemoteBlockOutputStream>(
                    *job.connection_entry, timeouts, query_string, settings, context->getClientInfo());
            }
            job.stream->write(adopted_shard_block);
        } else { // local
            if (!job.stream) {
                auto copy_query_ast = query_ast->clone();
                InterpreterInsertQuery interp(copy_query_ast, job.local_context, allow_materialized);
                auto block_io = interp.execute();
                job.stream = block_io.out;
            }
            for (size_t i = 0; i < shard_info.getLocalNodeCount(); ++i)
                job.stream->write(adopted_block);
        }
    };
}
```

从上面的源码可以看到主要分为三步：

1. 通过`shard_index`获取到`shard_job`后，紧接着会判断当前`Block`的行数是否为0，为0直接结束了。
2. 如果当前`job`是本地任务（即数据直接写到当前节点），则解析语法树获取到底表对应的`BlockOutputStream`，调用输出流的`write`方法，将数据写入到对应的存储，`ReplicatedMergeTree`引擎对应的输出流为`ReplicatedMergeTreeBlockOutputStream`。
3. 如果不是本地任务，即数据需要通过网络写入到远端节点，这时需要创建`RemoteBlockOutputStream`，调用输出流的`write`方法将数据写到对应的节点上。



# 异步写入

上面对于同步写入进行了详细分析，对于异步写入思路上也差不多，首先根据`sharding_key`为每个`shard`切分出一个`Block`，遍历所有的`shard`，如果是本地`shard`，那么直接将`Block`写入到当前节点，如果是远端`shard`，则将`Block`写入到当前节点本地文件中，最后让`StorageDistributedDirectoryMonitor`调度后台线程任务的执行，将数据同步到远端节点。



## 数据如何写入本地节点

一般情况下`Distributed` 表都是基于 `ReplicatedMergeTree` 系列表进行创建的，大多数场景下是异步写入，数据会先写入本地再由后台线程分发到远端节点。那么写入分布式表的数据是如何保证正在写入的过程中就把不完整的数据发送给远端其他节点呢？

写本地文件的逻辑在`writeToShard`方法中，如下：
```c++
void DistributedBlockOutputStream::writeToShard(const Block & block, const std::vector<std::string> & dir_names)
{
    auto it = dir_names.begin();
    {
        {
            WriteBufferFromFile out{first_file_tmp_path};
            CompressedWriteBuffer compress{out, compression_codec};
            NativeBlockOutputStream stream{compress, DBMS_TCP_PROTOCOL_VERSION, block.cloneEmpty()};
          
            WriteBufferFromOwnString header_buf;
            writeStringBinary(query_string, header_buf);  /// 写入insert语句（values前的内容）
            writeStringBinary(header_buf.stringRef(), out);	
            stream.write(block);		/// 写入Block数据
        }
        // Create hardlink here to reuse increment number
        const std::string block_file_path(fs::path(path) / file_name);
        createHardLink(first_file_tmp_path, block_file_path);
    }
    ++it;
    fs::remove(first_file_tmp_path);

    /// Notify
    auto sleep_ms = context->getSettingsRef().distributed_directory_monitor_sleep_time_ms;
    for (const auto & dir_name : dir_names)
    {
        StorageDistributedDirectoryMonitor & directory_monitor = storage.requireDirectoryMonitor(disk, dir_name, /* startup= */ false);
        directory_monitor.addAndSchedule(file_size, sleep_ms.totalMilliseconds());
    }
}
```

从代码中可以看到，分布式表在写本地文件的时候会将sql和数据一起写到临时路径的临时文件中，然后通过硬链接的方式将临时文件链接到正式路径上，删除临时文件，最后通过`directory_monitor`唤醒后台任务的执行。

分布式表写入本地的文件是`sql`文件，并不是`part`文件，分发到远端节点后需要重新语法解析等操作，原因是分布式表的底表可以是各种表引擎，不一定都是以`part`文件来存储的。同时使用硬链接链接临时文件的方式可以解决上面的问题避免数据文件在没写完就被发送到远端。

## 数据如何分发到各个节点

在上面 `writeToShard` 方法的最后会调用 `requireDirectoryMonitor`方法，这个方法就会注册监听上面分布式表的存储目录，并通过 `StorageDistributedDirectoryMonitor`  来实现数据文件的分发。

```c++
StorageDistributedDirectoryMonitor::StorageDistributedDirectoryMonitor(
    StorageDistributed & storage_,
    const DiskPtr & disk_,
    const std::string & relative_path_,
    ConnectionPoolPtr pool_,
    ActionBlocker & monitor_blocker_,
    BackgroundSchedulePool & bg_pool)
  ......
{
    task_handle = bg_pool.createTask(getLoggerName() + "/Bg", [this]{ run(); });
    task_handle->activateAndSchedule();
}
```

在`StorageDistributedDirectoryMonitor`的构造函数中，创建了一个后台的任务`task_handle`，该任务的核心逻辑在`run()`上，会将数据文件分发到对应的节点上，这里就不在深究，同时将`task_handle`添加到`DistributedSchedulePool`调度执行，线程池大小默认是16，`Clickhouse`所有的表的分布式调度任务都会由该线程池执行，有关更多`Clickhouse`后台线程池的介绍可以参考 《[ReplicatedMergeTree 后台任务的工作原理][1]》。

# 总结
经过上面的分析可以看到，一批数据写入分布式表会被拆分成多份小批量的数据写入`Clickhouse`集群，大量的小`part`文件会增加集群后台`merge`线程池的压力，当`merge`的处理能力小于写入能力时，`Clickhouse`会禁止写入，所以写入`Clickhouse`期望是**频率低批次大**。


目前写入分布式表都是异步写入的，要分发到远端节点的数据都需要先落盘到本地，然后由`DistributedSchedulePool`调度执行将数据分发到远端节点，当`DistributedSchedulePool`的处理能力小于写入能力时，就会造成分布式表堆积，这时用户是查不到堆积的数据的，当堆积到一定的程度，集群是比较难恢复的，需要清除分布式表堆积的数据，用户是会丢失数据的，出现这种问题的原因其中一个原因是用户在`StreamSql`任务中配置的**并发太高间隔太短**，写入太猛造成的。


还有一个造成分布式表堆积的原因是用户在修改表结构的时候未停写，就可能会出现分布式表中还堆积着修改表结构之前的数据，当分发的时候，由于远端的表结构已经修改，就会分发失败，`Clickhouse`就会无限次重试，导致后续的数据也无法处理，从而堆积。


从上面的分析可以看出，分布式表写入还是比较影响性能，并且还会存在潜在的问题。目前物化的视图的写入需要借助分布式哈希的能力，即物化视图都是分布式表写入的，但是仍然存在一部分用户分不清分布式表还是本地表，就可能出现写非物化视图也是直接写分布式表的情况，针对这个问题，正在开发写分布式表直接路由到本节点底表的功能，避免数据拆分和异步分发的问题，用户也不需要在感知本地表存在。


[1]: http://way.xiaojukeji.com/article/32148