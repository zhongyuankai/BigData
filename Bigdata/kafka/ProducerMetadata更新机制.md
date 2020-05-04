## Kafka 源码解析之 Producer Metadata 更新机制

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

`Cluster` 实例主要是保存：

1.  `broker.id` 与 `node` 的对应关系；
2.  `topic` 与 `partition` （`PartitionInfo`）的对应关系；
3.  `node` 与 `partition` （`PartitionInfo`）的对应关系。

### Producer 的 Metadata 更新流程

`Producer` 在调用 `dosend()` 方法时，第一步就是通过 `waitOnMetadata` 方法获取该 `topic` 的 `metadata` 信息.

```java
// 等待 metadata 的更新
private ClusterAndWaitTime waitOnMetadata(String topic, Integer partition, long maxWaitMs) throws InterruptedException {
    // 在 metadata 中添加 topic 后,如果 metadata 中没有这个 topic 的 meta，那么 metadata 的更新标志设置为了 true
    metadata.add(topic);
    Cluster cluster = metadata.fetch();
    // 如果 topic 已经存在 meta 中,则返回该 topic 的 partition 数,否则返回 null
    Integer partitionsCount = cluster.partitionCountForTopic(topic);

    // 当前 metadata 中如果已经有这个 topic 的 meta 的话,就直接返回
    if (partitionsCount != null && (partition == null || partition < partitionsCount))
        return new ClusterAndWaitTime(cluster, 0);

    long begin = time.milliseconds();
    long remainingWaitMs = maxWaitMs;
    long elapsed;

    // 发送 metadata 请求,直到获取了这个 topic 的 metadata 或者请求超时
    do {
        log.trace("Requesting metadata update for topic {}.", topic);
        // 返回当前版本号,初始值为0,每次更新时会自增,并将 needUpdate 设置为 true
        int version = metadata.requestUpdate();
        // 唤起 sender，发送 metadata 请求
        sender.wakeup();
        try {
            // 等待 metadata 的更新
            metadata.awaitUpdate(version, remainingWaitMs);
        } catch (TimeoutException ex) {
            // Rethrow with original maxWaitMs to prevent logging exception with remainingWaitMs
            throw new TimeoutException("Failed to update metadata after " + maxWaitMs + " ms.");
        }
        cluster = metadata.fetch();
        elapsed = time.milliseconds() - begin;
        if (elapsed >= maxWaitMs)
            // 超时
            throw new TimeoutException("Failed to update metadata after " + maxWaitMs + " ms.");
        // 认证失败，对当前 topic 没有 Write 权限
        if (cluster.unauthorizedTopics().contains(topic))
            throw new TopicAuthorizationException(topic);
        remainingWaitMs = maxWaitMs - elapsed;
        partitionsCount = cluster.partitionCountForTopic(topic);
    } while (partitionsCount == null);// 不停循环,直到 partitionsCount 不为 null（即直到 metadata 中已经包含了这个 topic 的相关信息）

    if (partition != null && partition >= partitionsCount) {
        throw new KafkaException(
                String.format("Invalid partition given with record: %d is not in the range [0...%d).", partition, partitionsCount));
    }

    return new ClusterAndWaitTime(cluster, elapsed);
}
```

如果 `metadata` 中不存在这个 `topic` 的 `metadata`，那么就请求更新 `metadata`，如果 `metadata` 没有更新的话，方法就一直处在 `do ... while` 的循环之中，在循环之中，主要做以下操作：

1. `metadata.requestUpdate()` 将 `metadata` 的 `needUpdate` 变量设置为 `true`（强制更新），并返回当前的版本号（`version`），通过版本号来判断 `metadata` 是否完成更新；
2. `sender.wakeup()` 唤醒 sender 线程，sender 线程又会去唤醒 `NetworkClient` 线程，`NetworkClient` 线程进行一些实际的操作（后面详细介绍）；
3. `metadata.awaitUpdate(version, remainingWaitMs)` 等待 `metadata` 的更新。

```java
// 更新 metadata 信息（根据当前 version 值来判断）
public synchronized void awaitUpdate(final int lastVersion, final long maxWaitMs) throws InterruptedException {
    if (maxWaitMs < 0) {
        throw new IllegalArgumentException("Max time to wait for metadata updates should not be < 0 milli seconds");
    }
    long begin = System.currentTimeMillis();
    long remainingWaitMs = maxWaitMs;
    while (this.version <= lastVersion) {// 不断循环,直到 metadata 更新成功,version 自增
        if (remainingWaitMs != 0)
            wait(remainingWaitMs);// 阻塞线程，等待 metadata 的更新
        long elapsed = System.currentTimeMillis() - begin;
        if (elapsed >= maxWaitMs)// timeout
            throw new TimeoutException("Failed to update metadata after " + maxWaitMs + " ms.");
        remainingWaitMs = maxWaitMs - elapsed;
    }
}
```

在 `Metadata.awaitUpdate()` 方法中，线程会阻塞在 `while` 循环中，直到 `metadata` 更新成功或者 `timeout`。

从前面可以看出，此时 `Producer` 线程会阻塞在两个 `while` 循环中，直到 `metadata` 信息更新，那么 `metadata `是如何更新的呢？

根据前面的学习，主要是通过 `sender.wakeup()` 来唤醒 sender 线程，间接唤醒 `NetworkClient` 线程，`NetworkClient` 线程来负责发送 `Metadata` 请求，并处理 `Server`端的响应。

在 [Kafka 源码分析之 Producer 发送模型](Producer 发送模型) 中介绍 `Producer` 发送模型时，在第五步 `sender` 线程会调用 `NetworkClient.poll()` 方法进行实际的操作，其源码如下：

```java
public List<ClientResponse> poll(long timeout, long now) {
	// 判断是否需要更新 meta,如果需要就更新（请求更新 metadata 的地方）
    long metadataTimeout = metadataUpdater.maybeUpdate(now);
    try {
        this.selector.poll(Utils.min(timeout, metadataTimeout, requestTimeoutMs));
    } catch (IOException e) {
        log.error("Unexpected error during I/O", e);
    }

    // process completed actions
    long updatedNow = this.time.milliseconds();
    List<ClientResponse> responses = new ArrayList<>();
    handleAbortedSends(responses);
    // 通过 selector 中获取 Server 端的 response
    handleCompletedSends(responses, updatedNow);
    // 在返回的 handler 中，会处理 metadata 的更新
    handleCompletedReceives(responses, updatedNow);
    handleDisconnections(responses, updatedNow);
    handleConnections();
    handleInitiateApiVersionRequests(updatedNow);
    handleTimedOutRequests(responses, updatedNow);

    // invoke callbacks
    for (ClientResponse response : responses) {
        try {
            response.onComplete();
        } catch (Exception e) {
            log.error("Uncaught error in request completion:", e);
        }
    }
    return responses;
}
```

在这个方法中，主要会以下操作：

- `metadataUpdater.maybeUpdate(now)`：判断是否需要更新 `Metadata`，如果需要更新的话，先与 `Broker `建立连接，然后发送更新 `metadata` 的请求；
- 处理 `Server` 端的一些响应，这里主要讨论的是 `handleCompletedReceives(responses, updatedNow)` 方法，它会处理 `Server` 端返回的 `Metadata` 结果。

先看一下 `metadataUpdater.maybeUpdate()` 的具体实现：

```java
 public long maybeUpdate(long now) {
    // should we update our metadata? metadata 是否应该更新
    // metadata 下次更新的时间（需要判断是强制更新还是 metadata 过期更新,前者是立马更新,后者是计算 metadata 的过期时间）
    long timeToNextMetadataUpdate = metadata.timeToNextUpdate(now);
    // 如果一条 metadata 的 fetch 请求还未从 server 收到,那么时间设置为 waitForMetadataFetch（默认30s）
    long waitForMetadataFetch = this.metadataFetchInProgress ? requestTimeoutMs : 0;

    long metadataTimeout = Math.max(timeToNextMetadataUpdate, waitForMetadataFetch);
    if (metadataTimeout > 0) {// 时间未到时,直接返回下次应该更新的时间
        return metadataTimeout;
    }

    Node node = leastLoadedNode(now);// 选择一个连接数最小的节点
    if (node == null) {
        log.debug("Give up sending metadata request since no node is available");
        return reconnectBackoffMs;
    }

    return maybeUpdate(now, node); // 可以发送 metadata 请求的话,就发送 metadata 请求
}

/**
 * Add a metadata request to the list of sends if we can make one
 */
// 判断是否可以发送请求,可以的话将 metadata 请求加入到发送列表中
private long maybeUpdate(long now, Node node) {
    String nodeConnectionId = node.idString();

    if (canSendRequest(nodeConnectionId)) {// 通道已经 ready 并且支持发送更多的请求
        this.metadataFetchInProgress = true; // 准备开始发送数据,将 metadataFetchInProgress 置为 true
        MetadataRequest.Builder metadataRequest; // 创建 metadata 请求
        if (metadata.needMetadataForAllTopics())// 强制更新所有 topic 的 metadata（虽然默认不会更新所有 topic 的 metadata 信息，但是每个 Broker 会保存所有 topic 的 meta 信息）
            metadataRequest = MetadataRequest.Builder.allTopics();
        else // 只更新 metadata 中的 topics 列表（列表中的 topics 由 metadata.add() 得到）
            metadataRequest = new MetadataRequest.Builder(new ArrayList<>(metadata.topics()));


        log.debug("Sending metadata request {} to node {}", metadataRequest, node.id());
        sendInternalMetadataRequest(metadataRequest, nodeConnectionId, now);/ 发送 metadata 请求
        return requestTimeoutMs;
    }

    // If there's any connection establishment underway, wait until it completes. This prevents
    // the client from unnecessarily connecting to additional nodes while a previous connection
    // attempt has not been completed.
    if (isAnyNodeConnecting()) {// 如果 client 正在与任何一个 node 的连接状态是 connecting,那么就进行等待
        // Strictly the timeout we should return here is "connect timeout", but as we don't
        // have such application level configuration, using reconnect backoff instead.
        return reconnectBackoffMs;
    }

    if (connectionStates.canConnect(nodeConnectionId, now)) {// 如果没有连接这个 node,那就初始化连接
        // we don't have a connection to this node right now, make one
        log.debug("Initialize connection to node {} for sending metadata request", node.id());
        initiateConnect(node, now);// 初始化连接
        return reconnectBackoffMs;
    }
    return Long.MAX_VALUE;
}

 // 发送 Metadata 请求   
 private void sendInternalMetadataRequest(MetadataRequest.Builder builder,
                                         String nodeConnectionId, long now) {
    ClientRequest clientRequest = newClientRequest(nodeConnectionId, builder, now, true);// 创建 metadata 请求
    doSend(clientRequest, true, now);
}
```

所以，每次 `Producer` 请求更新 `metadata` 时，会有以下几种情况：

1. 如果 `node` 可以发送请求，则直接发送请求；
2. 如果该 `node` 正在建立连接，则直接返回；
3. 如果该 `node` 还没建立连接，则向 `broker` 初始化链接。

而 `KafkaProducer` 线程之前是一直阻塞在两个 `while` 循环中，直到 `metadata` 更新

1. `sender` 线程第一次调用 `poll()` 方法时，初始化与 `node` 的连接；
2. `sender` 线程第二次调用 `poll()` 方法时，发送 `Metadata` 请求；
3. `sender` 线程第三次调用 `poll()` 方法时，获取 `metadataResponse`，并更新 `metadata`。

经过上述 `sender` 线程三次调用 `poll()`方法，所请求的 `metadata` 信息才会得到更新，此时 `Producer` 线程也不会再阻塞，开始发送消息。

`NetworkClient` 接收到 `Server` 端对 `Metadata` 请求的响应后，更新 `Metadata` 信息。

```java
// 处理任何已经完成的接收响应
private void handleCompletedReceives(List<ClientResponse> responses, long now) {
    for (NetworkReceive receive : this.selector.completedReceives()) {
        String source = receive.source();
        InFlightRequest req = inFlightRequests.completeNext(source);
        AbstractResponse body = parseResponse(receive.payload(), req.header);
        log.trace("Completed receive from node {}, for key {}, received {}", req.destination, req.header.apiKey(), body);
        if (req.isInternalRequest && body instanceof MetadataResponse)// 如果是 meta 响应
            metadataUpdater.handleCompletedMetadataResponse(req.header, now, (MetadataResponse) body);
        else if (req.isInternalRequest && body instanceof ApiVersionsResponse)
            handleApiVersionsResponse(responses, req, now, (ApiVersionsResponse) body); // 如果是其他响应
        else
            responses.add(req.completed(body, now));
    }
}

// 处理 Server 端对 Metadata 请求处理后的 response
public void handleCompletedMetadataResponse(RequestHeader requestHeader, long now, MetadataResponse response) {
    this.metadataFetchInProgress = false;
    Cluster cluster = response.cluster();
    // check if any topics metadata failed to get updated
    Map<String, Errors> errors = response.errors();
    if (!errors.isEmpty())
        log.warn("Error while fetching metadata with correlation id {} : {}", requestHeader.correlationId(), errors);

    // don't update the cluster if there are no valid nodes...the topic we want may still be in the process of being
    // created which means we will get errors and no nodes until it exists
    if (cluster.nodes().size() > 0) {
        this.metadata.update(cluster, now);// 更新 meta 信息
    } else {// 如果 metadata 中 node 信息无效,则不更新 metadata 信息
        log.trace("Ignoring empty metadata response with correlation id {}.", requestHeader.correlationId());
        this.metadata.failedUpdate(now);
    }
}
```

## Producer Metadata 的更新策略

![](img\metadata.png)

`Metadata` 会在下面两种情况下进行更新

1. `KafkaProducer` 第一次发送消息时强制更新，其他时间周期性更新，它会通过 `Metadata` 的 `lastRefreshMs`和`lastSuccessfulRefreshMs` 这2个字段来实现；
2. 强制更新： 调用 `Metadata.requestUpdate()` 将 `needUpdate` 置成了 `true` 来强制更新。

在 `NetworkClient` 的 `poll()` 方法调用时，就会去检查这两种更新机制，只要达到其中一种，就行触发更新操作。

`Metadata` 的强制更新会在以下几种情况下进行：

1. `initConnect` 方法调用时，初始化连接；
2. `poll()` 方法中对 `handleDisconnections()` 方法调用来处理连接断开的情况，这时会触发强制更新；
3. `poll()` 方法中对 `handleTimedOutRequests()` 来处理请求超时时；
4. 发送消息时，如果无法找到 partition 的 leader；
5. 处理 Producer 响应（`handleProduceResponse`），如果返回关于 `Metadata` 过期的异常，比如：没有 `topic-partition` 的相关 `meta` 或者 `client` 没有权限获取其 `metadata`。

强制更新主要是用于处理各种异常情况。



参考文档：

- [Kafka源码深度解析－序列2 －Producer －Metadata的数据结构与读取、更新策略](http://blog.csdn.net/chunlongyu/article/details/52622422)；
- [kafka源码分析之Producer](http://luodw.cc/2017/05/02/kafka02/)。























