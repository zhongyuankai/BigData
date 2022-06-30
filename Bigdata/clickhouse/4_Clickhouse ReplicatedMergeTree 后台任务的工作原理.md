# 1.BackgroundPool
在ReplicatedMergeTree中所有的后台任务都需要在全局的后台线程池中调度执行，后台线程池在ck有两种实现，分别是BackgroundProcessingPool和BackgroundSchedulePool。

1. BackgroundProcessingPool内部会维护一个任务队列，该任务队列会基于下一次执行的时间进行优先级排序，内部线程会根据顺序取任务执行，任务最大的并发数会等于pool_size。
2. BackgroundSchedulePool内部会维护一个消息队列，内部线程会监听这个队列，等待消息，当获取到消息时就开始执行消息中封装的任务，如果新来消息中的任务已经在执行了，则会抛弃这条消息，即在该线程池中执行的任务最大并发为1。

ReplicatedMergeTree中会用3个后台线程池。
![在这里插入图片描述](https://img-blog.csdnimg.cn/0797d92126254a5a9a8e9501648a84bf.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzMTE1NjA2,size_16,color_FFFFFF,t_70#pic_center)



# 2. mutationsUpdatingTask

调度执行是交给SchedulePool来处理的。

工作原理：

1. 该任务会监听zk_path/mutations节点，当节点变化时，会触发该任务的执行。
2. 拉取新增的MutationEntry，同时会清除mutations_by_partition中已经完成的mutation任务，mutations_by_partition这个数据结构里面会维护一个MutationStatus，MutationStatus对应着system下的mutations表一行记录。
3. 处理MutationEntry将其添加入到mutations_by_partition中。
触发mergeSelectingTask和mutationsFinalizingTask执行。

MutationEntr用于封装/mutations的子节点信息，对应的操作是ALTER DELETE和ALTER UPDATE。

# 3. mergeSelectingTask
ReplicatedMergeTree构造的时候创建，并会在确认成为leader之后，激活并调度执行，当退出成为leader后，会成为非active状态，不允许执行，任务的调度执行是交给SchedulePool来处理的。

工作原理：

1. 首先会判断当前正在执行的merge和mutation的任务总数是否超过max_replicated_merges_in_queue，默认值16，超过则直接结束。
2. 从所有的part中选出可以进行merge的part集合，如果不为空则将part集合封装成一个MERGE_PARTS的任务，并上传到zk的zk_path/logs下，等待执行，具体选择part的规则可以参考 MergeTree Merge选择过程分析。
3. 如果没有part可以merge，接着会遍历所有的part，判断part所在的partition是否在mutations_by_partition中，即该part需要是否要执行mutation操作，如果需要则会将该part封装成一个MUTATE_PART的任务，并上传到zk的zk_path/logs下，等待执行。
4. 最后会判断任务是否创建成功，成功则继续触发自己执行，失败则延迟触发，默认是5s。同时当执行commitPart、fetchPart、mergePart等操作之后都会触发mergeSelectingTask的执行。

一个merge任务可以同时merge多个part，但一个mutate任务只能操作一个part，所以mutation任务的执行周期会比较长。

# 6. queueUpdatingTask
调度执行是交给SchedulePool来处理的。
工作原理：

1. 该任务会监听zk的zk_path/logs节点，当节点发生变化，会触发该任务的执行。
2. 拉取新增的LogEntry并更新replica_path/log_pointer，将其指向最新的日志下标。
3. 拉取到的LogEntry，并不会立即执行，而是将其转为任务对象放至本地任务队列和zk中的replica_path/queue/中。

LogEntry用于封装/logs的子节点信息，操作类型有DROP_RANGE、REPLACE_RANGE、GET_PART、MERGE_PARTS、MUTATE_PART、ALTER_METADATA。

# 5. queueHandleTask
任务会由BackgroudPool执行。

工作原理：

1. 遍历本地任务队列，筛选出符合条件的LogEntry，比如MERGE_PARTS任务中涉及的part正在另一个任务中执行，那么该任务是不能被执行的。
2. 开始执行选择出来的LogEntry，根据LogEntry任务类型调用相应的函数来处理。
3. 执行成功后，会删除本地任务队列和replica_path/queue/的LogEntry。

# 6. mutationsFinalizingTask
任务会由SchedulePool执行。

工作原理：该任务主要是遍历所有的MutationStatus，将完成了的mutation任务标记为done，并更新本地和zk上的mutation_pointer。

# 7. cleanupTask
任务会由SchedulePool执行。

工作原理：

1. 清除生命周期超过8分钟的非active的part，需要将zk该路径replica_path/parts/part_name的znode清除、本地的part信息删除和磁盘中删除相应的part文件。
2. 清除过期的WAL日志。
3. 清除tmp开头并且超过生命周期的的文件，生命周期默认是一天。
4. 清理zk中zk_path/logs过期的日志，当日志数量小于10时不清理，大于10的时候会将小于log_pointer的znode删除。
5. 清除中zk_path/blocks 下过期的block，记录的block的作用是副本间重复写入的去重，block生命周期7天。
6. 清除zk_path/mutations 下过期的mutation，会将完成的mutation节点超过100的部分移除。

# 8. partCheckTask

任务会由SchedulePool执行。

内部会维护一个PartsToCheckQueue，会保存待检查的part，当处理part的两段提交过程中如果zk失联，就会将该part添加到PartsToCheckQueue，等待检查。

工作原理：

1. 首先从PartsToCheckQueue选择一个part，检查本地的part的columns、checksums与zk中的是否一致。
2. 检查本地的part的columns、checksums与对应磁盘中columns、checksums是否一致。
3. 如果出现不一致，则删除该part在zk上对应的znode，并且发起一个GET_PART请求，同时移除本地的part信息，将磁盘的part文件加上broken前缀，移到detach目录下。
4. 在PartsToCheckQueue中移除该part。

# 9. restartingTask

任务执行也是由SchedulePool调度执行。

工作原理：

1. 第一次运行的时候会注册自己成为leader，并激活启动后台任务。
2. 负责在zk上的心跳注册管理，每隔60秒会触发自己执行，防止zk session过期，当session过期时，会停止所有的后台任务。
# 10. movePartsHandleTask
在ReplicatedMergeTree表启动的时候判断是否创建启动后台移动part的任务，依据的条件是该表设置的存储策略，是否配置了多个Volume，或则配置了多块盘。满足条件会将创建movePartsHandleTask，并交给BackgroundMovePool调度执行。

工作原理：

1. 遍历所有的盘，当该盘的可用空间小于移动的阈值（total_space * move_factor）时，则该盘需要移动了，会找到该盘下所有的part，总大小不超过移动的阈值，并按照part大小进行排序。
2. 遍历需要移动的盘下需要移动的part，找预留空间大于该part的盘即为目标盘，没有空间该part则不移动。
3. 将所有需要移动的part，从源磁盘移动到目标磁盘，并将移动的log记录到part_log表中。
# 11. 总结
通过mutation操作全流程，讲上述后台任务都串起来。

![在这里插入图片描述](https://img-blog.csdnimg.cn/0a60243fa42b4662b4b918c59d1c1e02.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzMTE1NjA2,size_16,color_FFFFFF,t_70#pic_center)