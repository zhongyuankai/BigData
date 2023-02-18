# Kafka核心组件和流程

[TOC]

## 控制器

`Kafka`的集群由n个的`broker`所组成，每个`broker`就是一个kafka的实例或者称之为`kafka`的服务。其实控制器也是一个`broker`，控制器也叫`leader broker`。他除了具有一般`broker`的功能外，还负责分区`leader`的选取，也就是负责选举`partition`的`leader  replica`。控制器是`kafka`核心中的核心，需要重点学习和理解。

### 控制器选举

`kafka`每个`broker`启动的时候，都会实例化一个`KafkaController`，并将`broker`的`id`注册到`zookeeper`。集群在启动过程中，通过选举机制选举出其中一个`broker`作为`leader`，也就是控制器。

包括集群启动在内，有三种情况触发控制器选举：

1、集群启动

2、控制器所在代理发生故障

3、`zookeeper`心跳感知，控制器与自己的`session`过期

集群启动时，控制器选举过程:

假设此集群有三个broker，同时启动。

1. 3个`broker`从`zookeeper`获取`/controller`临时节点信息。`/controller`存储的是选举出来的`leader`信息。此举是为了确认是否已经存在`leader`。

2. 如果还没有选举出`leader`，那么此节点是不存在的，返回`-1`。如果返回的不是`-1`，则是`leader`的`json`数据，那么说明已经有`leader`存在，选举结束。

3. 三个`broker`发现返回`-1`，说明到目前没有`leader`，于是均会触发向临时节点`/controller`写入自己的信息。最先写入的就会成为`leader`。

4. 假设`broker  0`的速度最快，他先写入了`/controller`节点，那么他就成为了`leader`。而`broker1`、`broker2`因为晚了一步，他们在写`/controller`的过程中会抛出`ZkNodeExistsException`，也就是`zk`告诉他们，此节点已经存在了。

经过以上四步，`broker 0`成功写入`/controller`节点，其它`broker`写入失败了，所以`broker 0`成功当选`leader`。

此外`zk`中还有`controller_epoch`节点，存储了`leader`的变更次数，初始值为`0`，以后`leader`每变一次，该值`+1`。所有向控制器发起的请求，都会携带此值。

- 如果控制器和自己内存中比较，请求值小，说明`kafka`集群已经发生了新的选举，此请求过期，此请求无效。-  
- 如果请求值大于控制器内存的值，说明已经有新的控制器当选了，自己已经退位，请求无效。`kafka`通过`controller_epoch`保证集群控制器的唯一性及操作的一致性。

由此可见，`Kafka`控制器选举就是看谁先争抢到`/controller`节点写入自身信息。

### 控制器初始化

控制器的初始化，其实是初始化控制器所用到的组件及监听器，准备元数据。

每个`broker`都会实例化并启动一个`KafkaController`。`KafkaController`和他的组件关系，以及各个组件的介绍如下图：

![](img\KafkaController.png)

图中箭头为组件层级关系，组件下面还会再初始化其他组件。可见控制器内部还是有些复杂的，主要有以下组件：

1、`ControllerContext`，此对象存储了控制器工作需要的所有上下文信息，包括存活的代理、所有主题及分区分配方案、每个分区的`AR`、`leader`、`ISR`等信息。

2、一系列的`listener`，通过对`zookeeper`的监听，触发相应的操作，黄色的框的均为`listener`

3、分区和副本状态机，管理分区和副本。

4、当前代理选举器`ZookeeperLeaderElector`，此选举器有上位和退位的相关回调方法。

5、分区`leader`选举器，`PartitionLeaderSelector`

6、主题删除管理器，`TopicDeletetionManager`

7、`leader`向`broker`批量通信的`ControllerBrokerRequestBatch`。缓存状态机处理后产生的`request`，然后统一发送出去。

8、控制器平衡操作的`KafkaScheduler`，仅在`broker`作为`leader`时有效。

对于理解`kafkaController`的整体有帮助，需要反复回来理解此图，思考组件所处的位置，对整体的作用。

### 故障迁移

故障转移其实就是`leader`所在`broker`发生故障，`leader`转移为其他的`broker`。转移的过程就是重新选举`leader`的过程。

重新选举`leader`后，需要为该`broker`注册相应权限，调用的是`ZookeeperLeaderElector`的`onControllerFailover()`方法。在这个方法中初始化和启动了一系列的组件来完成`leader`的各种操作。具体如下，其实和控制器初始化有很大的相似度。

1. 注册分区管理的相关监听器

   监听名称监听`zookeeper`节点作用`PartitionsReassignedListener/admin/reassign_partitions`节点变化将会引发分区重分配，`IsrChangeNotificationListener/isr_change_notification`处理分区的`ISR`发生变化引发的操作，`PreferredReplicaElectionListener/admin/preferred_replica_election`将优先副本选举为`leader`副本

2. 注册主题管理的相关监听

   监听名称监听`zookeeper`节点作用`TopicChangeListener/brokers/topics`监听主题发生变化时进行相应操作，`DeleteTopicsListener/admin/delete_topics`完成服务器端删除主题的相应操作。否则客户端删除主题仅仅是表示删除

3. 注册代理变化监听器

   监听名称监听`zookeeper`节点作用`BrokerChangeListener/brokers/ids`代理发生增减的时候进行相应的处理

4. 重新初始化`ControllerContext`

5. 启动控制器和其他代理之间通信的`ControllerChannelManager`

6. 创建用于删除主题的`TopicDeletionManager`对象,并启动。

7. 启动分区状态机和副本状态机

8. 轮询每个主题，添加监听分区变化的`PartitionModificationsListener`

9. 如果设置了分区平衡定时操作，那么创建分区平衡的定时任务，默认300秒检查并执行。

除了这些组件的启动外，`onControllerFailover`方法中还做了如下操作：

1. `/controller_epoch`值+1，并且更新到`ControllerContext`

2. 检查是否出发分区重分配，并做相关操作

3. 检查需要将优先副本选为`leader`，并做相关操作

4. 向`kafka`集群所有代理发送更新元数据的请求。

下面来看代理下线的方法`onControllerResignation`

1. 该方法中注销了控制器的权限。取消在`zookeeper`中对于分区、副本感知的相应监听器的监听。

2. 关闭启动的各个组件

3. 最后把`ControllerContext`中记录控制器版本的数值清零，并设置当前`broker`为`RunnignAsBroker`，变为普通的`broker`。

通过对控制器启动过程的学习，我们应该已经对`kafka`工作的原理有了了解，核心是监听`zookeeper`的相关节点，节点变化时触发相应的操作。其它的处理流程都是相类似的。

### 代理上下线

有新的`broker`加入集群时，称为代理上线。反之，当`broker`关闭，推出集群时，称为代理下线。

#### 代理上线：

1. 新代理启动时向`/brokers/ids`写数据

2. `BrokerChangeListener`监听到变化。对新上线节点调用`controllerChannelManager.addBroker()`，完成新上线代理网络层初始化

3. 调用`KafkaController.onBrokerStartup()`处理
   1. 通过向所有代理发送`UpdateMetadataRequest`，告诉所有代理有新代理加入
   2. 根据分配给新上线节点的副本集合，对副本状态做变迁。对分区也进行处理。
   3. 触发一次`leader`选举，确认新加入的是否为分区`leader`
   4. 轮询分配给新`broker`的副本，调用`KafkaController.onPartitionReassignment()`，执行分区副本分配
   5. 恢复因新代理上线暂停的删除主题操作线程

#### 代理下线

1. 查找下线节点集合

2. 轮询下线节点，调用`controllerChannelManager.removeBroker()`，关闭每个下线节点网络连接。清空下线节点消息队列，关闭下线节点request请求

3. 轮询下线节点，调用`KafkaController.onBrokerFailure`处理
   1. 处理leader副本在下线节点上上的分区，重新选出leader副本，发送`updateMetadataRequest`请求。
   2. 处理下线节点上的副本集合，做下线处理，从`ISR`集合中删除，不再同步，发送`updateMetadataRequest`请求。

4. 向集群全部存活代理发送`updateMetadataRequest`请求

### 主题管理

通过分区状态机及副本状态机来进行主题管理

1. **创建主题**

`/brokers/topics`下创建主题对应子节点

`TopicChangeListener`监听此节点

变化时获取重入锁`ReentrantLock`,调用`handleChildChange`方法进行处理。

通过对比`zookeeper`中`/brokers/topics`存储的主题集合及控制器的`ControllerContext`中缓存的主题集合的差集，得到新增的主题。反过来求差集，得到删除的主题。

接下来遍历新增的主题集合，进行主题操作的实质性操作。之前仅仅是在`zookeeper`中添加了主题。新增主题涉及的操作有分区、副本状态的转化、分区leader的分配、分区存储日志的创建等。

2. **删除主题**

`/admin/delete_topics`创建删除主题的子节点

`DeleteTopicsListener`监听此节点，

变化时获取重入锁`ReentrantLock`,进行处理

具体的删除逻辑再次就不再详述。

### 分区管理

1. **分区自动平衡**

`onControllerFailover`方法中启动分区自动平衡任务。定时检查是否失去平衡。

自动平衡的操作就是把优先副本选为分区`leader`，`AR`中第一个副本为优先副本。

先查出所有可用副本，以分区`AR`头节点分组。

轮询代理节点，判断分区不平衡率是否超过`10%``(leader`为非优先副本的分区/该代理分区总数)，则调用`onPreferredReplicaElection()`，让优先副本成为`leader`。达到自动平衡。

2. **分区重分配**

当`zk`节点`/admin/reassign_partitions`变化时，触发分区重分配操作。该节点存储分区重分配的方案。

通过计算主题分区原`AR`（`OAR`）和重新分配后的`AR`（`RAR`），分别做相应处理：

1. `OAR+RAR`：更新到该主题分区`AR`，并通知副本节点同步。`leader_epoch+1`

2. `RAR-OAR`：副本设为`NewReplica`。

3. （`OAR+RAR`）- `RAR`：需要下线的副本，做下线操作

## 协调器

协调器负责协调工作，是用来协调消费者工作分配的。简单点说，就是消费者启动后，到可以正常消费前，这个阶段的初始化工作。消费者能够正常运转起来，全有赖于协调器。

主要的协调器有如下两个：

1、消费者协调器（`ConsumerCoordinator`）

2、组协调器（`GroupCoordinator`）

此外还有任务管理协调器（`WorkCoordinator`），用作`kafka connect`的`works`管理， 这里不做讲解。

`kafka`引入协调器有其历史过程，原来`consumer`信息依赖于`zookeeper`存储，当代理或消费者发生变化时，引发消费者平衡，此时消费者之间是互不透明的，每个消费者和`zookeeper`单独通信，容易造成羊群效应和脑裂问题。

为了解决这些问题，`kafka`引入了协调器。**服务端引入组协调器（`GroupCoordinator`），消费者端引入消费者协调器（`ConsumerCoordinator`）**。每个`broker`启动的时候，都会创建`GroupCoordinator`实例，管理部分消费组（集群负载均衡）和组下每个消费者消费的偏移量（`offset`）。每个`consumer`实例化时，同时实例化一个`ConsumerCoordinator`对象，负责同一个消费组下各个消费者和服务端组协调器之前的通信。

如下图：

![](img\KafkaCoordinator.png)

### 消费者协调器

消费者很多操作通过消费者协调器进行处理。

消费者协调器主要负责如下工作：

1. 更新消费者缓存的`MetaData`

2. 向组协调器申请加入组

3. 消费者加入组后的相应处理

4. 请求离开消费组

5. 向组协调器提交偏移量

6. 通过心跳，保持组协调器的连接感知。

7. 被组协调器选为`leader`的消费者的协调器，负责消费者分区分配。分配结果发送给组协调器。

8. 非`leader`的消费者，通过消费者协调器和组协调器同步分配结果。

消费者协调器主要依赖的组件和说明见下图：

[![](img\ConsumerGroupCoordinator.png)]()



### 组协调器

组协调器负责处理消费者协调器发过来的各种请求。它主要提供如下功能：

- 在与之连接的消费者中选举出消费者`leader`
- 下发`leader`消费者返回的消费者分区分配结果给所有的消费者
- 管理消费者的消费偏移量提交，保存在`kafka`的内部主题中
- 和消费者心跳保持，知道哪些消费者已经死掉，组中存活的消费者是哪些。

组协调器在`broker`启动的时候实例化，每个组协调器负责一部分消费组的管理。

它主要依赖的组件见下图：

![](img\GroupCoordinator.png)

### 消费者入组过程

下图展示了消费者启动选取`leader`、入组的过程。

![](img\group.png)

消费者入组的过程，很好的展示了消费者协调器和组协调器之间是如何配合工作的。`leader consumer`会承担分区分配的工作，这样`kafka`集群的压力会小很多。同组的`consumer`通过组协调器保持同步。消费者和分区的对应关系持久化在`kafka`内部主题。

### 消费偏移量管理

消费者消费时，会在本地维护消费到的位置（`offset`），就是偏移量，这样下次消费才知道从哪里开始消费。如果整个环境没有变化，这样做就足够了。但一旦消费者平衡操作或者分区变化后，消费者不再对应原来的分区，而每个消费者的`offset`也没有同步到服务器，这样就无法接着前任的工作继续进行了。

因此只有把消费偏移量定期发送到服务器，由`GroupCoordinator`集中式管理，分区重分配后，各个消费者从`GroupCoordinator`读取自己对应分区的`offset`，在新的分区上继续前任的工作；

开始时，`consumer 0`消费`partition 0 `和`1`，后来由于新的`consumer 2`入组，分区重新进行了分配。`consumer  0`不再消费`partition2`，而由`consumer 2`来消费`partition  2`，但由于`consumer`之间是不能通讯的，所有`consumer2`并不知道从哪里开始自己的消费。

因此`consumer`需要定期提交自己消费的`offset`到服务端，这样在重分区操作后，每个`consumer`都能在服务端查到分配给自己的`partition`所消费到的`offset`，继续消费；

由于`kafka`有高可用和横向扩展的特性，当有新的分区出现或者新的消费入组后，需要重新分配消费者对应的分区，所以如果偏移量提交的有问题，会重复消费或者丢消息。偏移量提交的时机和方式要格外注意！！

下面两种情况分别会造成重复消费和丢消息：

- 如果提交的偏移量小于消费者最后一次消费的偏移量，那么再均衡后，两个`offset`之间的消息就会被重复消费
- 如果提交的偏移量大于消费者最后一次消费的偏移量，那么再均衡后，两个`offset`之间的消息就会丢失





## 副本管理器

副本机制使得kafka整个集群中，只要有一个代理存活，就可以保证集群正常运行。这大大提高了Kafka的可靠性和稳定性。Kafka中代理的存活，需要满足以下两个条件：

- 存活的节点要维持和zookeeper的session连接，通过zookeeper的心跳机制实现
- Follower副本要与leader副本保持同步，不能落后太多。

满足以上条件的节点在ISR中，一旦宕机，或者中断时间太长，Leader就会把同步副本从ISR中踢出。

所有节点中，leader节点负责接收客户端的读写操作，follower节点从leader复制数据。

副本管理器负责对副本管理。由于副本是分区的副本，所以对副本的管理体现在对分区的管理。

介绍两个重要的概念，LEO和HW。

- LEO是Log End Offset缩写。表示每个分区副本的最后一条消息的位置，也就是说每个副本都有LEO。
- HW是Hight Watermark缩写，他是一个分区所有副本中，最小的那个LEO。

![](img\LEO.png)

分区test-0有三个副本，每个副本的LEO就是自己最后一条消息的offset。可以看到最小的LEO是Replica2的，等于3，也就是说HW=3。这代表offset=4的消息还没有被所有副本复制，是无法被消费的。而offset<=3的数据已经被所有副本复制，是可以被消费的。

副本管理器所承担的职责如下：

- 副本过期检查
- 追加消息
- 拉取消息
- 副本同步过程
- 副本角色转换
- 关闭副本