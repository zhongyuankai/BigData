# BigData

## Java 基础
- [Java 基础常考知识点](Java/Basis/JavaBasis.md)
- [ArrayList/Vector](Java/Basis/ArrayList.md)
- [LinkedList](Java/Basis/LinkedList.md)
- [HashMap](Java/Basis/HashMap.md)
- [HashSet](Java/Basis/HashSet.md)
- [LinkedHashMap](Java/Basis/LinkedHashMap.md)
- [浅析I/O模型及NIO的实现](Java/Basis/NIO.md)
- [Java8 新特性](Java/Basis/Java8.md)

## Java 多线程
- [Java 多线程常见问题](Java/Thread/Thread-common-problem.md)
- [锁的种类与介绍](Java/Thread/java-lock.md)
- [volatile 关键字详解](Java/Thread/volatile.md)
- [CAS 底层原理](Java/Thread/CAS.md)
- [synchronized 关键字原理](Java/Thread/Synchronize.md)
- [ReentrantLock 实现原理](Java/Thread/ReentrantLock.md)
- [ThreadLocal 实现原理](Java/Thread/ThreadLocal.md)
- [CopyOnWrite容器 源码分析](Java/Thread/CopyOnWrite.md)
- [ConcurrentHashMap 源码分析](Java/Thread/ConcurrentHashMap.md)
- [CountDownLatch、CyclicBarrier和Semaphore用法](Java/Thread/CountDownLatch.md)
- [阻塞队列](Java/Thread/BlockingQueue.md)
- [线程池底层工作原理](Java/Thread/ThreadPool.md)
- [多线程代码实践-大厂面试题](Java/Thread/ThreadCode.md)

## JVM
- [Java 运行时的内存划分](Java/Jvm/MemoryAllocation.md)
- [对象的创建、内存分配和四大引用](Java/Jvm/newObject.md)
- [类加载机制](Java/Jvm/ClassLoad.md)
- [OOM 分析](Java/Jvm/OOMAnalysis.md)
- [垃圾回收](Java/Jvm/GarbageCollection.md)
- [常用工具+实战](Java/Jvm/tools.md)

## C++


## 计算机网络原理

- [计算机网络](Basis/计算机网络.md)

## 数据结构与算法

- [二分法](DataStructureAndAlgorithm/Binary.md)
- [动态规划](DataStructureAndAlgorithm/Dynamicprogramming.md)
- [贪心算法](DataStructureAndAlgorithm/greedy.md)
- [哈希表](DataStructureAndAlgorithm/HashTable.md)
- [广度优先搜索](DataStructureAndAlgorithm/BreadthFirstSearch.md)

## 设计模式

- [常见设计模式的对比](DesignPattern/常见设计模式的对比.md)

## Mysql 数据库

- [数据库水平垂直拆分](Mysql/数据库水平垂直拆分.md)
- [MySQL 索引原理](Mysql/MySQL索引原理.md)
- [事务的实现](Mysql/事务的实现.md)

## Redis 缓存数据库

- [Redis 基础](Redis/Redis基础.md)
- [缓存雪崩、击穿、穿透](Redis/缓存雪崩、击穿、穿透.md)

## Elasticsearch

- [`ReplicationTracker` 源码跟踪：从副本加入到 global checkpoint 推进](ElasticSearch/replication-tracker-source-trace.md)
- [Retention Lease 理解及源码跟踪](<ElasticSearch/Retention Lease理解及源码跟踪.md>)
- [`NodeStats` 源码跟踪](ElasticSearch/node-stats-source-trace.md)
- [`IndicesStats` 源码跟踪](ElasticSearch/indices-stats-source-trace.md)
- [`ClusterInfo` 源码跟踪：从统计采集到分片分配消费](ElasticSearch/cluster-info-source-trace.md)
- [`AllocationDecider` 内置决策器源码跟踪](ElasticSearch/allocation-deciders-source-trace.md)
- [`DiskThresholdDecider` 源码跟踪：从磁盘统计到分配决策](ElasticSearch/disk-threshold-decider-source-trace.md)
- [`allocateExistingUnassignedShards` 源码跟踪](ElasticSearch/allocate-existing-unassigned-shards-source-trace.md)
- [Elasticsearch 分片 Balance 源码跟踪](ElasticSearch/shard-balance-source-trace.md)
- [`DesiredBalanceShardsAllocator.allocate` 源码跟踪](ElasticSearch/desired-balance-shards-allocator-allocate-source-trace.md)
- [`BalancedShardsAllocator.allocate` 源码跟踪](ElasticSearch/balanced-shards-allocator-allocate-source-trace.md)
- [SLM 源码跟踪详解](ElasticSearch/slm-source-trace.md)
- [社区 Elasticsearch 基于 S3 冷备的源码跟踪](ElasticSearch/s3-cold-backup-source-trace.md)

## Kafka 引擎

- [kafka 的数据存储](Kafka/kafka的数据存储.md)
- [Producer 发送模型](Kafka/Producer发送模型.md)
- [Producer Metadata 更新机制](Kafka/ProducerMetadata更新机制.md)
- [Producer 端源码分析总结](Kafka/Producer端源码分析总结.md)
- [ProducerNIO网络模型](Kafka/ProducerNIO网络模型.md)
- [KafkaProducer内存管理](Kafka/KafkaProducer内存管理.md)
- [Topic 创建过程](Kafka/Topic创建过程.md)
- [Kafka 核心组件和流程](Kafka/Kafka核心组件和流程.md)

## Spark 引擎

- [Yarn 集群模式部署流程源码分析](Spark/Yarn集群模式部署流程源码分析.md)
- [通讯架构源码解析](Spark/通讯架构源码解析.md)

## Clickhouse 引擎

- [源码阅读Clickhouse Server是如何启动](Clickhouse/1_源码阅读Clickhouse-server是如何启动的.md)
- [Clickhouse 分布式表的写入原理](Clickhouse/2_Clickhouse分布式表的写入原理.md)
- [两行代码节省30%的CPU消耗](Clickhouse/3_两行代码节省30%的CPU消耗.md)
- [ReplicatedMergeTree 后台任务的工作原理](Clickhouse/4_ReplicatedMergeTree后台任务的工作原理.md)
- [Clickhouse-copier 源码剖析](Clickhouse/5_Clickhouse-copier源码剖析.md)
- [MergeTree 的写入流程](Clickhouse/6_MergeTree的写入流程.md)

## 源码分析工具

- [源码跟踪与文档发布 Skill](skills/trace-source-code/SKILL.md)
