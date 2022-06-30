clickhouse-copier 用于集群间数据的迁移，也可以用于集群内数据的均衡。
接下来会针对源码进行深度剖析，如果觉得对你有帮助别忘记点赞关注。
## 工具参数
```bash
daemon — 后台运行copier工具，进程将在后台启动。
 
config — zookeeper.xml的存放路径，用来连接zookeeper集群。
 
task-path — zookeeper上的任务节点路径，该路径中的内容用来存储任务，以及多个copier进程间的协调信息，同一任务的不同copier进程要保持一致的配置路径。
 
task-file — 指向配置了任务的配置文件，如：task-config.xml文件内容会上传到zookeeper的/clickhouse-copier/db/tb/description节点。
 
task-upload-force — 若设置为true,那么将根据task-file文件的内容，强制刷新覆盖上个参数提到的zookeeper的description节点。
 
base-dir — 会存储一些日志以及相关的辅助型文件，copier工具进程启动后，会在$base-dir创建copier_YYYYMMHHSS_<PID>格式的子目录（日志文件会在该子目录下，以及辅助型分布式表的相关信息在data目录下），若没有传该参数，则在copier运行的当前目录创建。
```
## 配置参数
**Zookeeper.xml**

```xml
<yandex>
    <logger>
        <level>trace</level>
        <size>100M</size>   // 达到100M进行滚动
        <count>3</count>    // 文件数量达到3会清除老的文件
    </logger>
    <zookeeper>
        <node index="1">
            <host>127.0.0.1</host>
            <port>2181</port>
        </node>
    </zookeeper>
</yandex>
```
**task-config.xml**

```xml
<yandex>
    <!-- 配置源集群和目标集群 -->
    <remote_servers>
        <source_cluster>
            <shard>
                <internal_replication>false</internal_replication>
                    <replica>
                        <host>127.0.0.1</host>
                        <port>9000</port>
                    </replica>
            </shard>
        </source_cluster>
        <destination_cluster>
        ...
        </destination_cluster>
    </remote_servers>
 
    <!-- clickhouse-copier最大工作的进程数 -->
    <max_workers>2</max_workers>
     
    <!-- 执行select和insert的参数配置 -->
    <settings_pull>
        <readonly>1</readonly>
    </settings_pull>
    <settings_push>
        <readonly>0</readonly>
    </settings_push>
 
    <!-- select和insert公共配置 -->
    <settings>
        <connect_timeout>3</connect_timeout>
        <insert_distributed_sync>1</insert_distributed_sync>
    </settings>
 
    <!-- 表迁移任务-->
    <tables>
        <table>
            <cluster_pull>source_cluster</cluster_pull>
            <database_pull>test</database_pull>
            <table_pull>hits</table_pull>
 
            <cluster_push>destination_cluster</cluster_push>
            <database_push>test</database_push>
            <table_push>hits2</table_push>
             
            <!-- 当目的表不存在时，会根据该engine配置创建表，以及创建临时辅助表。 -->
            <engine>
            ENGINE=ReplicatedMergeTree('/clickhouse/tables/{cluster}/{shard}/hits2', '{replica}')
            PARTITION BY toMonday(date)
            ORDER BY (CounterID, EventDate)
            </engine>
            <sharding_key>intHash64(UserID)</sharding_key>
 
            <where_condition>CounterID != 0</where_condition>
            <enabled_partitions>
                <partition>'2018-02-26'</partition>
                ...
            </enabled_partitions>
        </table>
        ...
    </tables>
</yandex>
```
## 启动
```bash
clickhouse-copier  --config /home/clickhouse/clickhouse-copier/zookeeper.xml --task-path /clickhouse-copier/db/tb --base-dir /home/clickhouse/clickhouse-copier/db/tb --task-file /home/clickhouse/clickhouse-copier/db/tb/task-config.xml --task-upload-force=true
```
## 原理解析
### 任务模型的设计
copier启动之后首先会加载配置文件进行初始化，将<tables>下面的任务统统加载进来，然后根据<table>中的配置去访问src_cluster的各个shard(这些shards会根据hostname与copier所在的机器的hostname进行diff，匹配越多的字符说明离的越近，按照从近到远进行排序)下的各个src_table，加载这些table的partitions（如果指定了enabled_partitions,那么就enabled_partitions中配置的，没有的话，则查找所有partitions），然后将这些partition存进任务队列，每个分区任务还会进行分片（默认分成10片），以一个片为最小任务粒度进行处理。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210405144259624.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzMTE1NjA2,size_16,color_FFFFFF,t_70#pic_center)
控制一个分区分多少片是由number_of_splits参数控制的(它和cluster_pull标签同级): 默认是10，第一个优点是：可以增加并行度，多个copier可以同时处理多个pieces；第二个优点则是单个pieceTask失败后的重试粒度变小了。
### 整体设计
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210405144337568.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzMTE1NjA2,size_16,color_FFFFFF,t_70#pic_center)
1. 任务划分之后，循环遍历任务队列中的partitionTask, 然后根据num_of_splits的配置将partitionTask切分成3片，接下来开始处理每一个最小粒度的pieceTask，这些pieceTask同时只能被一个copier来进行处理，状态通过zookeeper来同步。
2. 处理pieceTask时会根据该pieceTask所属的shard来在本地内存数据库中创建读分布式表.read_shard_0.cluster_name.database.src_table，该分布式表配置的本地表名就是src_table，以及该分布式表所在的虚拟集群.read_shard_0.cluster_name，该虚拟集群仅有一个shard，就是当前处理pieceTask所属的shard，由近到远，前期处理的应该都是本地的shard。
3. 创建完读分布式表后，接着创建写分布式表.split.cluster_name.database.des_table_piece_0，该分布式表中的创建的配置则不是虚拟集群，而是真正的dest_cluster做为参数传入，本地表则是根据<engine>的配置来创建的辅助临时表des_table.piece_0，如果引擎是Replicated*则会去掉Replicated，避免副本同步，并且目标集群所有的shards上都会创建，该写分布式表的shard_key则是我们在copy-job.xml中配置的，根据这个shard_key来选择数据该发往哪个shard。
4. 辅助表准备好之后，读取.read_shard_0.cluster_name.database.src_table分布式表，过滤出当前分区的数据当前piece的数据，即根据src_table的分区键、cityHash64(primaryKey) % num_of_splits分配后形成select查询语句来查询读分布式表，获取到一个输入流；写入是通过INSERT INTO .split.cluster_name.database.dest_table_piece_0分布式表，拿到输出流，然后输入流和输出流对接进行数据的迁移。

```sql
SELECT
    *
FROM
    _local.`.read_shard_0.destination_cluster.wujie.dm_cx_trd_multi_order_di_local`
WHERE
    (dt = ('2021-02-13' AS partition_key))
    AND ((cityHash64(stat_time) % 10) = 0)
 
 
INSERT INTO _local.`.split.destination_cluster.wujie.dwm_cx_trd_sub_order_di_local_piece_0` VALUES
```
5. 当该partition的所有pieces都己经完成数据copy后，接下来会进行movePartition操作，将这些临时辅助表的数据attach到des_table中，下面这个SQL会在目标节点上执行。

```sql
ALTER TABLE wujie.dm_cx_trd_multi_order_di_local ATTACH PARTITION '2021-02-13' FROM wujie.dm_cx_trd_multi_order_di_local_piece_0
```
6. 当整个tableTask的所有partitionTasks处理完成以后，会进行删除临时辅助表的操作；如果所有的tableTasks都处理完以后，copier进程就会自动退出。
## 代码主要实现

```cpp
mianImpl()
{
	context->makeGlobalContext();
	context->setCurrentDatabase(default_database); // _local

	auto copier = std::make_unique<ClusterCopier>(task_path, host_id, default_database, *context);
	copier->setExperimentalUseSampleOffset(experimental_use_sample_offset); // false

	// task-file不为空则会去更新zk路径上的任务配置，task_path + "/description"
	auto task_file = config().getString("task-file", "");
	if (!task_file.empty())
        copier->uploadTaskDescription(task_path, task_file, config().getBool("task-upload-force", false));

    copier->init()	// 初始化TaskCluster
    {
    	task_cluster = std::make_unique<TaskCluster>(task_zookeeper_path, working_database_name);
    	reloadTaskDescription();//监控zk的description节点，并获取配置
    	task_cluster->loadTasks(*task_cluster_initial_config);	// 解析配置，保存 TasksTable table_tasks;
    	for (auto & task_table : task_cluster->table_tasks)
    	{

        	task_table.cluster_pull = context.getCluster(task_table.cluster_pull_name);
        	task_table.cluster_push = context.getCluster(task_table.cluster_push_name);
        	task_table.initShards(task_cluster->random_engine); // 初始化TaskShard
    	}
    }

    copier->process{
    	for (TaskTable & task_table : task_cluster->table_tasks){

    		discoverTablePartitions(timeouts, task_table);//封装partition——task（Map<partition_name, ShardPartition>），用的是多线程

    		for (const TaskShardPtr & task_shard : task_table.all_shards){
    			for (const auto & partition_elem : task_shard->partition_tasks){
    				task_table.cluster_partitions.emplace(partition_name, ClusterPartition{});//初始集群分区的迁移状态
    			}
    		}

            for (auto & partition_elem : task_table.cluster_partitions){	// 遍历所有的分区
                for (const TaskShardPtr & task_shard : task_table.all_shards)
                    task_shard->checked_partitions.emplace(partition_name);		// 每个task_shard里面保存所有的partitition_name

                task_table.ordered_partition_names.emplace_back(partition_name);	// 生成partition顺序
            }

            // max_table_tries = 1000 
            tryProcessTable(timeouts, task_table) // 开始处理当前的表任务
            { 
            	for (const String & partition_name : task_table.ordered_partition_names)
            	{
            		for (const TaskShardPtr & shard : task_table.all_shards) 
            		{
            			checkShardHasPartition()
            			checkPresentPartitionPiecesOnCurrentShard();

            			// max_shard_partition_tries = 600
            			task_status = tryProcessPartitionTask(timeouts, partition, is_unprioritized_task) // partition (ShardPartition)
            			{ 
            				res = iterateThroughAllPiecesInPartition(timeouts, task_partition, is_unprioritized_task)
            				{
            					// 分区内按片进行迁移
            					for (size_t piece_number = 0; piece_number < total_number_of_pieces; piece_number++)
            					{
            						// max_shard_partition_tries = 600
            						res = processPartitionPieceTaskImpl(timeouts, task_partition, piece_number, is_unprioritized_task)
            						{
            							/// Load balancing  这里会限制max_worker，即超过了会sleep
    									createTaskWorkerNodeAndWaitIfNeed(zookeeper, current_task_piece_status_path, is_unprioritized_task);

    									// 创建读取的分布式表、写的分布表表、写的local表
            							createShardInternalTables(timeouts, task_shard, true);

            							context.setCluster(shard_read_cluster_name, cluster_pull_current_shard);


            							// 检查当前这个任务是不是干净的，如果是干净的则可以继续运行，否则删除zk dirty节点
            							checkPartitionPieceIsClean(zookeeper, clean_state_clock, piece_status_path);

            							// 创建 task_piece 的active节点，如果创建失败则该任务正在运行，退出
            							zkutil::EphemeralNodeHolder::create(current_task_piece_is_active_path, *zookeeper, host_id);

            							// 检查当前任务是都处理完成
            							if (zookeeper->tryGet(current_task_piece_status_path, status_data)){...}

            							// 当前shard上没有这个piece，会exit
            							if (partition_piece.is_absent_piece){
            								auto res = zookeeper->tryCreate(current_task_piece_status_path, state_finished, zkutil::CreateMode::Persistent);
            							}


            							// 在目标集群上每个shard上创建接收数据的本地表


            							// 获取输入输出流
            							BlockInputStreamPtr input = InterpreterFactory::get(query_select_ast, *context_select)->execute().getInputStream();
                						BlockOutputStreamPtr output = InterpreterFactory::get(query_insert_ast, *context_insert)->execute().out;

                						//两个流对接
                						copyData(*input, *output, cancel_check, update_stats);

                						// 创建目标表如果不存在

            						}
            					}
            				}
            			}
            		}

            	}

            	// 检查分区的拷贝是否否完成
            	bool partition_copying_is_done = checkAllPiecesInPartitionAreDone(task_table, partition_name, expected_shards);

            	if (partition_copying_is_done){
            		auto res = tryMoveAllPiecesToDestinationTable(task_table, partition_name){
            			for (size_t current_piece_number = 0; current_piece_number < task_table.number_of_splits; ++current_piece_number)
    					{

    						// ALTER TABLE db.tb ATTACH PARTITION partition_name FROM db.tb_piece_num;
    						size_t num_nodes = executeQueryOnCluster(task_table.cluster_push,query_alter_ast_string,settings_push,PoolMode::GET_MANY,ClusterExecutionMode::ON_EACH_NODE);

    					}
            		}
            	}

            }
            dropHelpingTables(task_table);
    	}
    }
```

目前集群迁移并未使用这种方式，原因是速度太慢了，必须将速度提上去才能投入使用。如果觉得对你有帮助别忘记点赞关注。