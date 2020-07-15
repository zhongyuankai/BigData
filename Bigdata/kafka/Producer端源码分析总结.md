# Producer 端源码分析总结

消息发送的流程图：

![](img\Producer 发送模型.png)

源码体现：

```java
KafkaProduce{
send(){
	// 对消息进行拦截处理
	ProducerRecord<K, V> interceptedRecord = this.interceptors.onSend(record);
	doSend(){
		// 1. 确认数据要发送到的 topic 的 metadata 是可用的
		ClusterAndWaitTime clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), maxBlockTimeMs);
		Cluster cluster = clusterAndWaitTime.cluster;
		// 2. 序列化key和value
		serializedKey = keySerializer.serialize(record.topic(), record.key());
		serializedValue = valueSerializer.serialize(record.topic(), record.value());
		// 3. 计算分区
		int partition = partition(record, serializedKey, serializedValue, cluster);
		// 4. 计算序列化后的大小，record 的字节超出限制或大于内存限制时,就会抛出 RecordTooLargeException 异常
		int serializedSize = Records.LOG_OVERHEAD + Record.recordSize(serializedKey, serializedValue);
		ensureValidRecordSize(serializedSize); 
		// 5. 向 accumulator 中追加数据
		RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey, serializedValue, interceptCallback, remainingWaitMs);
		// 6. 如果 batch 已经满了,唤醒 sender 线程发送数据
		if (result.batchIsFull || result.newBatchCreated) {
			this.sender.wakeup();
		}
	}
	}
}
```

在 `dosend()` 方法的实现上，一条 `Record `数据的发送，可以分为以下五步：

1. 确认数据要发送到的` topic `的 `metadata` 是可用的（如果该`partition`的`leader` 存在则是可用的，如果开启权限时，`client` 有相应的权限），如果没有`topic`的 `metadata` 信息，就需要获取相应的  `metadata`；
2. 序列化 `record` 的` key` 和 `value`；
3. 确保序列化后的消息的大小不超出限制。
4. 获取该 `record` 要发送到的 `partition`（可以指定，也可以根据算法计算）；
5. 向 `accumulator` 中追加 `record` 数据，数据会先进行缓存；
6. 如果追加完数据后，对应的 `RecordBatch` 已经达到了 `batch.size` 的大小（或者`batch` 的剩余空间不足以添加下一条 `Record`），则唤醒 `sender` 线程发送数据。



下面是sender被唤醒，开始发送数据：

```java
Sender{
run(){
sendProducerData(){
	// 1. 获取元数据
	Cluster cluster = metadata.fetch();
	// 2. 获取已经可以发送的 RecordBatch 对应的 nodes
	RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);
	// 如果有 topic-partition 的 leader 是未知的,就强制 metadata 更新
	if (!result.unknownLeaderTopics.isEmpty()) {
		this.metadata.requestUpdate();
	}
	// 3. 获取 node 对应的所有可以发送的 RecordBatch，并将 RecordBatch 从对应的 queue 中移除
	Map<Integer, List<ProducerBatch>> batches = this.accumulator.drain(cluster, result.readyNodes, this.maxRequestSize, now);
	// 4. 发送 RecordBatch
    sendProduceRequests(batches, now){
		for (Map.Entry<Integer, List<ProducerBatch>> entry : collated.entrySet()){
			sendProduceRequest(now, entry.getKey(), acks, requestTimeoutMs, entry.getValue()){
			// 5. 封装client请求
			ClientRequest clientRequest = client.newClientRequest(nodeId, requestBuilder, now, acks != 0,requestTimeoutMs, callback);
			// 6. KafkaClient 进行发送
			client.send(clientRequest, now);
			}
		}
	}
}
// 更新元数据
client.poll(pollTimeout, now);
}
}
```

上述就是`Sender`线程发送数据的核心步骤。总结起来就是将`accumulator`中缓存的`RecordBatch`按照`node`进行分组，将一组数据发往指定的`node`。

## Metadata 更新过程

### 更新机制

1.  `KafkaProducer` 第一次发送消息时强制更新，其他时间周期性更新，它会通过 `Metadata` 的 `lastRefreshMs`和 `lastSuccessfulRefreshMs` 这2个字段来实现；
2.  强制更新： 调用 `Metadata.requestUpdate()` 将 `needUpdate` 置成了 `true` 来强制更新。

### Metadata 内容

```java
// 这个类被 client 线程和后台 sender 所共享,它只保存了所有 topic 的部分数据,当我们请求一个它上面没有的 topic meta 时,它会通过发送 metadata update 来更新 meta 信息,
// 如果 topic meta 过期策略是允许的,那么任何 topic 过期的话都会被从集合中移除,
// 但是 consumer 是不允许 topic 过期的因为它明确地知道它需要管理哪些 topic
public final class Metadata {
    private static final Logger log = LoggerFactory.getLogger(Metadata.class);

    public static final long TOPIC_EXPIRY_MS = 5 * 60 * 1000;
    private static final long TOPIC_EXPIRY_NEEDS_UPDATE = -1L;

    private final long refreshBackoffMs; // metadata 更新失败时,为避免频繁更新 meta,最小的间隔时间,默认 100ms
    private final long metadataExpireMs; // metadata 的过期时间, 默认 60,000ms
    private int version; // 每更新成功1次，version自增1,主要是用于判断 metadata 是否更新
    private long lastRefreshMs; // 最近一次更新时的时间（包含更新失败的情况）
    private long lastSuccessfulRefreshMs; // 最近一次成功更新的时间（如果每次都成功的话，与前面的值相等, 否则，lastSuccessulRefreshMs < lastRefreshMs)
    private Cluster cluster; // 集群中一些 topic 的信息
    private boolean needUpdate; // 是都需要更新 metadata
    /* Topics with expiry time */
    private final Map<String, Long> topics; // topic 与其过期时间的对应关系
    private final List<Listener> listeners; // 事件监控者
    private final ClusterResourceListeners clusterResourceListeners; //当接收到 metadata 更新时, ClusterResourceListeners的列表
    private boolean needMetadataForAllTopics; // 是否强制更新所有的 metadata
    private final boolean topicExpiryEnabled; // 默认为 true, Producer 会定时移除过期的 topic,consumer 则不会移除
}
```

关于 `topic` 的详细信息（`leader` 所在节点、`replica` 所在节点、`isr` 列表）都是在 `Cluster` 实例中保存的。

```java
// 并不是一个全集,metadata的主要组成部分
public final class Cluster {
    // 从命名直接就看出了各个变量的用途
    private final boolean isBootstrapConfigured;
    private final List<Node> nodes; // node 列表
    private final Set<String> unauthorizedTopics; // 未认证的 topic 列表
    private final Set<String> internalTopics; // 内置的 topic 列表
    private final Map<TopicPartition, PartitionInfo> partitionsByTopicPartition; // partition 的详细信息
    private final Map<String, List<PartitionInfo>> partitionsByTopic; // topic 与 partition 的对应关系
    private final Map<String, List<PartitionInfo>> availablePartitionsByTopic; //  可用（leader 不为 null）的 topic 与 partition 的对应关系
    private final Map<Integer, List<PartitionInfo>> partitionsByNode; // node 与 partition 的对应关系
    private final Map<Integer, Node> nodesById; // node 与 id 的对应关系
    private final ClusterResource clusterResource;
}

// org.apache.kafka.common.PartitionInfo
// topic-partition: 包含 topic、partition、leader、replicas、isr
public class PartitionInfo {
    private final String topic;
    private final int partition;
    private final Node leader;
    private final Node[] replicas;
    private final Node[] inSyncReplicas;
}
```

分析Producer发送消息的时候，两次调用了waitOnMedata方法来更新元数据，接下来就来看看到到底是如何更新元数据的。

```java
waitOnMetadata{
    waitOnMetadata(String topic, Integer partition, long maxWaitMs){
	Cluster cluster = metadata.fetch();
	// add时topic不存在。则会将needUpdate标志改为true
	metadata.add(topic);
	// 获取该topic对应的分区数量
	Integer partitionsCount = cluster.partitionCountForTopic(topic);
	// 当前 metadata 中如果已经有这个 topic 的 meta 的话,就直接返回
    if (partitionsCount != null && (partition == null || partition < partitionsCount))
        return new ClusterAndWaitTime(cluster, 0);
	do {
	int version = metadata.requestUpdate();
	metadata.add(topic);
	// 将needUpdate标志改为true，并返回当前的版本号
	int version = metadata.requestUpdate();
	// 唤醒sender进行元数据的更新
	sender.wakeup();
	// 等待元数据的更新
	metadata.awaitUpdate(version, remainingWaitMs){
		while ((this.version <= lastVersion) && !isClosed()) {
			if (remainingWaitMs != 0)
                wait(remainingWaitMs);
			long elapsed = System.currentTimeMillis() - begin;
				// 超时了则抛出异常
			if (elapsed >= maxWaitMs)
                throw new TimeoutException();
			remainingWaitMs = maxWaitMs - elapsed;
		}
	}
	// 获取最新的元数据
	cluster = metadata.fetch();
	// 判断是否超时
	elapsed = time.milliseconds() - begin;
    if (elapsed >= maxWaitMs) throw new TimeoutException();
	// 该topic是用户是否有权限
	if (cluster.unauthorizedTopics().contains(topic)) throw new TopicAuthorizationException(topic);
	// 该topic是否有效
	if (cluster.invalidTopics().contains(topic)) throw new InvalidTopicException(topic);
	remainingWaitMs = maxWaitMs - elapsed;
	// 从最新的元数据中获取分区个数
    partitionsCount = cluster.partitionCountForTopic(topic);
	// 一直获取，直到获取到或异常发生跳出
	} while (partitionsCount == null || (partition != null && partition >= partitionsCount));
}
}
```

如果 `metadata` 中不存在这个 `topic` 的 `metadata`，那么就请求更新 `metadata`，如果 `metadata` 没有更新的话，方法就一直处在 `do ... while` 的循环之中，在循环之中，主要做以下操作：

1. `metadata.requestUpdate()` 将 `metadata` 的 `needUpdate` 变量设置为 `true`（强制更新），并返回当前的版本号（`version`），通过版本号来判断 `metadata` 是否完成更新；
2. `sender.wakeup()` 唤醒 sender 线程，sender 线程又会去唤醒 `NetworkClient` 线程，`NetworkClient` 线程进行一些实际的操作；
3. `metadata.awaitUpdate(version, remainingWaitMs)` 等待 `metadata` 的更新。



唤醒了sender线程来更新元数据，那么接着就来看看是怎么来发送metadata请求并更新本地的元数据。

sender线程发送数据的最后调用`client.poll(pollTimeout, now);`这个。没错这就是更新元数据。

```java
poll(){
	// 判断是否需要更新metadata
	long metadataTimeout = metadataUpdater.maybeUpdate(now)
	{
	// metadata 下次更新的时间（需要判断是强制更新还是 metadata 过期更新,前者是立马更新,后者是计算 metadata 的过期时间）
	long timeToNextMetadataUpdate = metadata.timeToNextUpdate(now);
	// 如果一条 metadata 的 fetch 请求还未从 server 收到,那么时间设置为 waitForMetadataFetch（默认30s）
    long waitForMetadataFetch = this.metadataFetchInProgress ? requestTimeoutMs : 0;
	long metadataTimeout = Math.max(timeToNextMetadataUpdate, waitForMetadataFetch);
    if (metadataTimeout > 0) {// 时间未到时,直接返回下次应该更新的时间
        return metadataTimeout;
    }
	// 选择一个连接数最小的节点
	Node node = leastLoadedNode(now);
	// 可以发送 metadata 请求的话,就发送 metadata 请求
	return maybeUpdate(now, node) 
	{// 判断是否可以发送请求,可以的话将 metadata 请求加入到发送列表中
		String nodeConnectionId = node.idString();
		// 通道已经 ready 并且支持发送更多的请求
		if (canSendRequest(nodeConnectionId)) {
			// 准备开始发送数据,将 metadataFetchInProgress 置为 true
			this.metadataFetchInProgress = true; 
			// 创建 metadata 请求
			MetadataRequest.Builder metadataRequest; 
			// 强制更新所有 topic 的 metadata（虽然默认不会更新所有 topic 的 metadata 信息，但是每个 Broker 会保存所有 topic 的 meta 信息）
			if (metadata.needMetadataForAllTopics())
				metadataRequest = MetadataRequest.Builder.allTopics();
			else // 只更新 metadata 中的 topics 列表（列表中的 topics 由 metadata.add() 得到）
				metadataRequest = new MetadataRequest.Builder(new ArrayList<>(metadata.topics()));
			
			// 发送 metadata 请求
			sendInternalMetadataRequest(metadataRequest, nodeConnectionId, now);
		}
	}
	}
	List<ClientResponse> responses = new ArrayList<>();
	// 通过 selector 中获取 Server 端的 response
    handleCompletedSends(responses, updatedNow);
	// 在返回的 handler 中，会处理 metadata 的更新
    handleCompletedReceives(responses, updatedNow){
	for (NetworkReceive receive : this.selector.completedReceives()) {
		InFlightRequest req = inFlightRequests.completeNext(source);
		// 判断响应的类型
		if (req.isInternalRequest && body instanceof MetadataResponse)
			// 处理metadata的响应
            metadataUpdater.handleCompletedMetadataResponse(req.header, now, (MetadataResponse) body){
				if (response.brokers().isEmpty()) {
					this.metadata.failedUpdate(now, null);
				} else {
					// 具体的处理更新
					this.metadata.update(response, now);
				}
			}
        else if (req.isInternalRequest && body instanceof ApiVersionsResponse)
            handleApiVersionsResponse(responses, req, now, (ApiVersionsResponse) body);
        else
            responses.add(req.completed(body, now));
	}
	}
    handleDisconnections(responses, updatedNow);
    handleConnections();
    handleInitiateApiVersionRequests(updatedNow);
    handleTimedOutRequests(responses, updatedNow);
    completeResponses(responses);
}

```

`Metadata` 会在下面两种情况下进行更新

1. `KafkaProducer` 第一次发送消息时强制更新，其他时间周期性更新，它会通过 `Metadata` 的 `lastRefreshMs`和`lastSuccessfulRefreshMs` 这2个字段来实现；
2. 强制更新： 调用 `Metadata.requestUpdate()` 将 `needUpdate` 置成了 `true` 来强制更新。

在 `NetworkClient` 的 `poll()` 方法调用时，就会去检查这两种更新机制，只要达到其中一种，就行触发更新操作。

## producer 端需要注意的地方

### acks：是一个影响消息吞吐量的一个关键参数。

主要有 `[all、-1, 0, 1]` 这几个选项，默认为 1。

当 `acks = all/-1` 时：会确保所有的 `follower` 副本都完成数据的写入才会返回。这样可以保证消息不会丢失，但是性能和吞吐量是最低的。

当 `acks = 0` 时：`producer` 不会等待副本的任何响应，这样最容易丢失消息但同时性能却是最好的！

当 `acks = 1` 时：会等待副本 `Leader` 响应，但不会等到 `follower` 的响应。一旦 `Leader` 挂掉消息就会丢失。但性能和消息安全性都得到了一定的保证。

### batch.size：内部缓存区的大小限制

对他适当的调大可以提高吞吐量。但也不能极端，调太大会浪费内存。小了也发挥不了作用，也是一个典型的时间和空间的权衡。

### retries：该参数主要是来做重试使用，当发生一些网络抖动都会造成重试。

- 因为是重发所以消息顺序可能不会一致，这也是上文提到就算是一个分区消息也不会是完全顺序的情况。
- 还是由于网络问题，本来消息已经成功写入了但是没有成功响应给 producer，进行重试时就可能会出现**消息重复**。这种只能是消费者进行幂等处理。

### 高效的发送方式

如果消息量真的非常大，同时又需要尽快的将消息发送到 `Kafka`。一个 `producer` 始终会收到缓存大小等影响。

那是否可以创建多个 `producer` 来进行发送呢？

- 配置一个最大 producer 个数。
- 发送消息时首先获取一个 `producer`，获取的同时判断是否达到最大上限，没有就新建一个同时保存到内部的 `List` 中，保存时做好同步处理防止并发问题。
- 获取发送者时可以按照默认的分区策略使用轮询的方式获取（保证使用均匀）。

这样在大量、频繁的消息发送场景中可以提高发送效率减轻单个 `producer` 的压力。







