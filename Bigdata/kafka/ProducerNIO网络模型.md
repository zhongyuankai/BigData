# Kafka 源码解析之 Producer NIO 网络模型

## Producer 的网络模型

`KafkaProducer` 通过 `Sender` 进行相应的 `IO` 操作，而 `Sender` 又调用 `NetworkClient` 来进行 `IO` 操作，`NetworkClient` 底层是对 `Java NIO` 进行相应的封装。

![](img\producer-network.png)

## Producer 整体流程

![](img\producer-nio-flow.png)

首先这图我觉得画得特别好，可以先了解整体的流程，之后再深入每一个方法，最后再回来看这张图就会比较清晰。

### KafkaProducer.dosend()

主要干两件事

1. `waitOnMetadata()`：请求更新 tp（topic-partition） meta，中间会调用 `sender.wakeup()`；
2. `accumulator.append()`：将 msg 写入到其 tp 对应的 deque 中，如果该 tp 对应的 deque 新建了一个 Batch，最后也会调用 `sender.wakeup()`。

### sender.wakeup() 方法

具体实现：

```java
// org.apache.kafka.clients.producer.internals.Sender
/**
* Wake up the selector associated with this send thread
*/
public void wakeup() {
    this.client.wakeup();
}

// org.apache.kafka.clients.NetworkClient
/**
* Interrupt the client if it is blocked waiting on I/O.
*/
@Override
public void wakeup() {
    this.selector.wakeup();
}

// org.apache.kafka.common.network.Selector
/**
* Interrupt the nioSelector if it is blocked waiting to do I/O.
*/
//note: 如果 selector 是阻塞的话,就唤醒
@Override
public void wakeup() {
    this.nioSelector.wakeup();
}
```

调用过程：

Sender -> NetworkClient -> Selector(Kafka 封装的) -> Selector(Java NIO)

跟上面两张图中 KafkaProducer 的总体调用过程大概一致，它的作用就是将 Sender 线程从 `select()` 方法的阻塞中唤醒，`select()` 方法的作用是轮询注册在多路复用器上的 Channel，它会一直阻塞在这个方法上，除非满足下面条件中的一个：

- at least one channel is selected;
- this selector’s {@link #wakeup wakeup} method is invoked;
- the current thread is interrupted;
- the given timeout period expires.

否则 `select()` 将会一直轮询，阻塞在这个地方，直到条件满足。

分析到这里，KafkaProducer 中 `dosend()` 方法调用 `sender.wakeup()` 方法作用就很明显的，作用就是：当有新的 RecordBatch 创建后，旧的 RecordBatch 就可以发送了（或者此时有 Metadata 请求需要发送），如果线程阻塞在 `select()` 方法中，就将其唤醒，Sender 重新开始运行 `run()` 方法，在这个方法中，旧的 RecordBatch （或相应的 Metadata 请求）将会被选中，进而可以及时将这些请求发送出去。

### Sender.run()

具体实现：

```java
//note: Sender 线程每次循环具体执行的地方
void run(long now) {
    Cluster cluster = metadata.fetch();
    //note: Step1 获取那些已经可以发送的 RecordBatch 对应的 nodes
    RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);

    //note: Step2  如果有 topic-partition 的 leader 是未知的,就强制 metadata 更新
    if (!result.unknownLeaderTopics.isEmpty()) {
        for (String topic : result.unknownLeaderTopics)
            this.metadata.add(topic);
        this.metadata.requestUpdate();
    }

    //note: 如果与node 没有连接（如果可以连接,会初始化该连接）,暂时先移除该 node
    Iterator<Node> iter = result.readyNodes.iterator();
    long notReadyTimeout = Long.MAX_VALUE;
    while (iter.hasNext()) {
        Node node = iter.next();
        if (!this.client.ready(node, now)) {//note: 没有建立连接的 broker,这里会与其建立连接
            iter.remove();
            notReadyTimeout = Math.min(notReadyTimeout, this.client.connectionDelay(node, now));
        }
    }

    //note: Step3  返回该 node 对应的所有可以发送的 RecordBatch 组成的 batches（key 是 node.id,这些 batches 将会在一个 request 中发送）
    Map<Integer, List<RecordBatch>> batches = this.accumulator.drain(cluster,
                                                                     result.readyNodes,
                                                                     this.maxRequestSize,
                                                                     now);
    //note: 保证一个 tp 只有一个 RecordBatch 在发送,保证有序性
    //note: max.in.flight.requests.per.connection 设置为1时会保证
    if (guaranteeMessageOrder) {
        // Mute all the partitions draine
        for (List<RecordBatch> batchList : batches.values()) {
            for (RecordBatch batch : batchList)
                this.accumulator.mutePartition(batch.topicPartition);
        }
    }

    //note: 将由于元数据不可用而导致发送超时的 RecordBatch 移除
    List<RecordBatch> expiredBatches = this.accumulator.abortExpiredBatches(this.requestTimeout, now);
    for (RecordBatch expiredBatch : expiredBatches)
        this.sensors.recordErrors(expiredBatch.topicPartition.topic(), expiredBatch.recordCount);

    sensors.updateProduceRequestMetrics(batches);

    long pollTimeout = Math.min(result.nextReadyCheckDelayMs, notReadyTimeout);
    if (!result.readyNodes.isEmpty()) {
        log.trace("Nodes with data ready to send: {}", result.readyNodes);
        pollTimeout = 0;
    }
    //note: Step4 发送 RecordBatch
    sendProduceRequests(batches, now);

    //note: 如果有 partition 可以立马发送数据,那么 pollTimeout 为0.
    //note: Step5 关于 socket 的一些实际的读写操作
    this.client.poll(pollTimeout, now);
}
```

`Sender.run()` 的大概流程总共有以下五步：

1. `accumulator.ready()`：遍历所有的 tp（topic-partition），如果其对应的 RecordBatch 可以发送（大小达到 `batch.size` 大小或时间达到 `linger.ms`），就将其对应的 leader 选出来，最后会返回一个可以发送 Produce request 的 `Set`（实际返回的是 `ReadyCheckResult` 实例，不过 `Set` 是最主要的成员变量）；
2. 如果发现有 tp 没有 leader，那么这里就调用 `requestUpdate()` 方法更新 metadata，实际上还是在第一步对 tp 的遍历中，遇到没有 leader 的 tp 就将其加入到一个叫做  `unknownLeaderTopics` 的 set 中，然后会请求这个 tp 的 meta
3. `accumulator.drain()`：遍历每个 leader （第一步中选出）上的所有 tp，如果该 tp  对应的 RecordBatch 不在 backoff 期间（没有重试过，或者重试了但是间隔已经达到了 retryBackoffMs  ），并且加上这个 RecordBatch 其大小不超过 maxSize（一个 request 的最大限制，默认为 1MB），那么就把这个  RecordBatch 添加 list 中，最终返回的类型为 `Map>`，key 为 leader.id，value 为要发送的 RecordBatch 的列表；
4. `sendProduceRequests()`：发送 Produce 请求，从图中，可以看出，这个方法会调用 `NetworkClient.send()` 来发送 clientRequest；
5. `NetworkClient.poll()`：关于 socket 的 IO 操作都是在这个方法进行的，它还是调用 Selector 进行的相应操作，而 Selector 底层则是封装的 Java NIO 的相关接口，这个下面会详细讲述。

在第三步中，可以看到，如果要向一个 leader 发送 Produce 请求，那么这 leader 对应 tp，如果其 RecordBatch 没有达到要求（`batch.size` 或 `linger.ms` 都没达到）还是可能会发送，这样做的好处是：可以减少 request 的频率，有利于提供发送效率。

### NetworkClient.poll()

1. 如果需要更新 Metadata，那么就发送 Metadata 请求；

2. 调用 Selector 进行相应的 IO 操作；

3. 处理 Server 端的 response 及一些其他的操作。

```java
public List<ClientResponse> poll(long timeout, long now) {
    //note: Step1 判断是否需要更新 meta,如果需要就更新（请求更新 metadata 的地方）
    long metadataTimeout = metadataUpdater.maybeUpdate(now);
    //note: Step2 调用 Selector.poll() 进行 socket 相关的 IO 操作
    try {
        this.selector.poll(Utils.min(timeout, metadataTimeout, requestTimeoutMs));
    } catch (IOException e) {
        log.error("Unexpected error during I/O", e);
    }

    //note: Step3 处理完成后的操作
    long updatedNow = this.time.milliseconds();
    List<ClientResponse> responses = new ArrayList<>();
    handleAbortedSends(responses);
    //note: 处理已经完成的 send（不需要 response 的 request,如 send）
    handleCompletedSends(responses, updatedNow);//note: 通过 selector 中获取 Server 端的 response
    //note: 处理从 server 端接收到 Receive（如 Metadata 请求）
    handleCompletedReceives(responses, updatedNow);//note: 在返回的 handler 中，会处理 metadata 的更新
    //note: 处理连接失败那些连接,重新请求 meta
    handleDisconnections(responses, updatedNow);
    //note: 处理新建立的那些连接（还不能发送请求,比如:还未认证）
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

这个方法大致分为三步：

1. `metadataUpdater.maybeUpdate()`：如果 Metadata 需要更新，那么就选择连接数最小的 node，发送 Metadata 请求；
2. `selector.poll()`：进行 socket IO 相关的操作，下面会详细讲述；
3. process completed actions：在一个 select() 过程之后的相关处理。
   - `handleAbortedSends(responses)`：处理那么在发送过程出现 `UnsupportedVersionException` 异常的 request；
   - `handleCompletedSends(responses, updatedNow)`：处理那些已经完成的 request，如果是那些不需要 response 的 request 的话，这里直接调用 `request.completed()`，标志着这个 request 发送处理完成；
   - `handleCompletedReceives(responses, updatedNow)`：处理那些从 Server 端接收的 Receive，metadata 更新就是在这里处理的（以及 `ApiVersionsResponse`）；
   - `handleDisconnections(responses, updatedNow)`：处理连接失败那些连接,重新请求 metadata；
   - `handleConnections()`：处理新建立的那些连接（还不能发送请求,比如:还未认证）；
   - `handleInitiateApiVersionRequests(updatedNow)`：对那些新建立的连接，发送 apiVersionRequest（默认情况：第一次建立连接时，需要向 Broker 发送 ApiVersionRequest 请求）；
   - `handleTimedOutRequests(responses, updatedNow)`：处理 timeout 的连接，关闭该连接，并刷新 Metadata。

### Selector.poll()

Selector 类是 Kafka 对 Java NIO 相关接口的封装，socket IO 相关的操作都是这个类中完成的，这里先看一下 `poll()` 方法，主要的操作都是这个方法中调用的，其代码实现如下：

```java
public void poll(long timeout) throws IOException {
    if (timeout < 0)
        throw new IllegalArgumentException("timeout should be >= 0");

    //note: Step1 清除相关记录
    clear();

    if (hasStagedReceives() || !immediatelyConnectedKeys.isEmpty())
        timeout = 0;

    /* check ready keys */
    //note: Step2 获取就绪事件的数
    long startSelect = time.nanoseconds();
    int readyKeys = select(timeout);
    long endSelect = time.nanoseconds();
    this.sensors.selectTime.record(endSelect - startSelect, time.milliseconds());

    //note: Step3 处理 io 操作
    if (readyKeys > 0 || !immediatelyConnectedKeys.isEmpty()) {
        pollSelectionKeys(this.nioSelector.selectedKeys(), false, endSelect);
        pollSelectionKeys(immediatelyConnectedKeys, true, endSelect);
    }

    //note: Step4 将处理得到的 stagedReceives 添加到 completedReceives 中
    addToCompletedReceives();

    long endIo = time.nanoseconds();
    this.sensors.ioTime.record(endIo - endSelect, time.milliseconds());

    // we use the time at the end of select to ensure that we don't close any connections that
    // have just been processed in pollSelectionKeys
    //note: 每次 poll 之后会调用一次
    //TODO: 连接虽然关闭了,但是 Client 端的缓存依然存在
    maybeCloseOldestConnection(endSelect);
}
```

`Selector.poll()` 方法会进行四步操作:

#### clear()

`clear()` 方法是在每次 `poll()` 执行的第一步，它作用的就是清理上一次 poll 过程产生的部分缓存。

```java
//note: 每次 poll 调用前都会清除以下缓存
private void clear() {
    this.completedSends.clear();
    this.completedReceives.clear();
    this.connected.clear();
    this.disconnected.clear();
    // Remove closed channels after all their staged receives have been processed or if a send was requested
    for (Iterator<Map.Entry<String, KafkaChannel>> it = closingChannels.entrySet().iterator(); it.hasNext(); ) {
        KafkaChannel channel = it.next().getValue();
        Deque<NetworkReceive> deque = this.stagedReceives.get(channel);
        boolean sendFailed = failedSends.remove(channel.id());
        if (deque == null || deque.isEmpty() || sendFailed) {
            doClose(channel, true);
            it.remove();
        }
    }
    this.disconnected.addAll(this.failedSends);
    this.failedSends.clear();
}
```

#### select()

Selector 的 `select()` 方法在实现上底层还是调用 Java NIO 原生的接口，这里的 `nioSelector` 其实就是 `java.nio.channels.Selector` 的实例对象，这个方法最坏情况下，会阻塞 ms 的时间，如果在一次轮询，只要有一个 Channel 的事件就绪，它就会立马返回。

```java
private int select(long ms) throws IOException {
    if (ms < 0L)
        throw new IllegalArgumentException("timeout should be >= 0");

    if (ms == 0L)
        return this.nioSelector.selectNow();
    else
        return this.nioSelector.select(ms);
}
```

#### pollSelectionKeys()

这部分是 socket IO 的主要部分，发送 Send 及接收 Receive 都是在这里完成的，在 `poll()` 方法中，这个方法会调用两次：

1. 第一次调用的目的是：处理已经就绪的事件，进行相应的 IO 操作；
2. 第二次调用的目的是：处理新建立的那些连接，添加缓存及传输层（Kafka 又封装了一次）的握手与认证。

```java
private void pollSelectionKeys(Iterable<SelectionKey> selectionKeys,
                                   boolean isImmediatelyConnected,
                                   long currentTimeNanos) {
    Iterator<SelectionKey> iterator = selectionKeys.iterator();
    while (iterator.hasNext()) {
        SelectionKey key = iterator.next();
        iterator.remove();
        KafkaChannel channel = channel(key);

        // register all per-connection metrics at once
        sensors.maybeRegisterConnectionMetrics(channel.id());
        if (idleExpiryManager != null)
            idleExpiryManager.update(channel.id(), currentTimeNanos);

        try {

            /* complete any connections that have finished their handshake (either normally or immediately) */
            //note: 处理一些刚建立 tcp 连接的 channel
            if (isImmediatelyConnected || key.isConnectable()) {
                if (channel.finishConnect()) {//note: 连接已经建立
                    this.connected.add(channel.id());
                    this.sensors.connectionCreated.record();
                    SocketChannel socketChannel = (SocketChannel) key.channel();
                    log.debug("Created socket with SO_RCVBUF = {}, SO_SNDBUF = {}, SO_TIMEOUT = {} to node {}",
                            socketChannel.socket().getReceiveBufferSize(),
                            socketChannel.socket().getSendBufferSize(),
                            socketChannel.socket().getSoTimeout(),
                            channel.id());
                } else
                    continue;
            }

            /* if channel is not ready finish prepare */
            //note: 处理 tcp 连接还未完成的连接,进行传输层的握手及认证
            if (channel.isConnected() && !channel.ready())
                channel.prepare();

            /* if channel is ready read from any connections that have readable data */
            if (channel.ready() && key.isReadable() && !hasStagedReceive(channel)) {
                NetworkReceive networkReceive;
                while ((networkReceive = channel.read()) != null)//note: 知道读取一个完整的 Receive,才添加到集合中
                    addToStagedReceives(channel, networkReceive);//note: 读取数据
            }

            /* if channel is ready write to any sockets that have space in their buffer and for which we have data */
            if (channel.ready() && key.isWritable()) {
                Send send = channel.write();
                if (send != null) {
                    this.completedSends.add(send);//note: 将完成的 send 添加到 list 中
                    this.sensors.recordBytesSent(channel.id(), send.size());
                }
            }

            /* cancel any defunct sockets */
            //note: 关闭断开的连接
            if (!key.isValid())
                close(channel, true);

        } catch (Exception e) {
            String desc = channel.socketDescription();
            if (e instanceof IOException)
                log.debug("Connection with {} disconnected", desc, e);
            else
                log.warn("Unexpected error from {}; closing connection", desc, e);
            close(channel, true);
        }
    }
}
```

### addToCompletedReceives()

这个方法的目的是处理接收到的 Receive，由于 Selector 这个类在 Client 和 Server 端都会调用，这里分两种情况讲述一下：

1. 应用在 Server 端时，Server 为了保证消息的时序性，在 Selector 中提供了两个方法：`mute(String id)` 和 `unmute(String id)`，对该 KafkaChannel 做标记来保证同时只能处理这个 Channel 的一个 request（可以理解为排它锁）。当 Server 端接收到 request 后，先将其放入 `stagedReceives` 集合中，此时该 Channel 还未 mute，这个 Receive 会被放入 `completedReceives` 集合中。Server 在对 `completedReceives` 集合中的 request 进行处理时，会先对该 Channel mute，处理后的 response 发送完成后再对该 Channel unmute，然后才能处理该 Channel 其他的请求；
2. 应用在 Client 端时，Client 并不会调用 Selector 的 `mute()` 和 `unmute()` 方法，client 的时序性而是通过 `InFlightRequests` 和 RecordAccumulator 的 `mutePartition` 来保证的（下篇文章会讲述），因此对于 Client 端而言，这里接收到的所有 Receive 都会被放入到 `completedReceives` 的集合中等待后续处理。

这个方法只有配合 Server 端的调用才能看明白其作用，它统一 Client 和 Server 调用的 api，使得都可以使用 Selector 这个类。

```java
/**
 * checks if there are any staged receives and adds to completedReceives
 */
private void addToCompletedReceives() {
    if (!this.stagedReceives.isEmpty()) {//note: 处理 stagedReceives
        Iterator<Map.Entry<KafkaChannel, Deque<NetworkReceive>>> iter = this.stagedReceives.entrySet().iterator();
        while (iter.hasNext()) {
            Map.Entry<KafkaChannel, Deque<NetworkReceive>> entry = iter.next();
            KafkaChannel channel = entry.getKey();
            if (!channel.isMute()) {
                Deque<NetworkReceive> deque = entry.getValue();
                addToCompletedReceives(channel, deque);
                if (deque.isEmpty())
                    iter.remove();
            }
        }
    }
}

private void addToCompletedReceives(KafkaChannel channel, Deque<NetworkReceive> stagedDeque) {
    NetworkReceive networkReceive = stagedDeque.poll();
    this.completedReceives.add(networkReceive); //note: 添加到 completedReceives 中
    this.sensors.recordBytesReceived(channel.id(), networkReceive.payload().limit());
}
```

## Network.send() 方法

至此，文章的主要内容已经讲述得差不多了，第二张图中最上面的那个调用关系已经讲述完，下面讲述一下另外一个小分支，也就是从 `Sender.run()` 调用 `NetworkClient.send()` 开始的那部分，其调用过程如下：

```
Sender.run()
Sender.sendProduceRequests()
NetworkClient.send()
NetworkClient.dosend()
Selector.send()
KafkaChannel.setSend()
```

### NetworkClient.dosend()

Producer 端的请求都是通过 `NetworkClient.dosend()` 来发送的，其作用就是：

- 检查版本信息，并根据 `apiKey()` 构建 Request；
- 创建 `NetworkSend` 实例；
- 调用 `Selector.send` 发送该 Send。

```java
//note: 发送请求
private void doSend(ClientRequest clientRequest, boolean isInternalRequest, long now) {
    String nodeId = clientRequest.destination();
    if (!isInternalRequest) {
        // If this request came from outside the NetworkClient, validate
        // that we can send data.  If the request is internal, we trust
        // that that internal code has done this validation.  Validation
        // will be slightly different for some internal requests (for
        // example, ApiVersionsRequests can be sent prior to being in
        // READY state.)
        if (!canSendRequest(nodeId))
            throw new IllegalStateException("Attempt to send a request to node " + nodeId + " which is not ready.");
    }
    AbstractRequest request = null;
    AbstractRequest.Builder<?> builder = clientRequest.requestBuilder();
    //note: 构建 AbstractRequest, 检查其版本信息
    try {
        NodeApiVersions versionInfo = nodeApiVersions.get(nodeId);
        // Note: if versionInfo is null, we have no server version information. This would be
        // the case when sending the initial ApiVersionRequest which fetches the version
        // information itself.  It is also the case when discoverBrokerVersions is set to false.
        if (versionInfo == null) {
            if (discoverBrokerVersions && log.isTraceEnabled())
                log.trace("No version information found when sending message of type {} to node {}. " +
                        "Assuming version {}.", clientRequest.apiKey(), nodeId, builder.version());
        } else {
            short version = versionInfo.usableVersion(clientRequest.apiKey());
            builder.setVersion(version);
        }
        // The call to build may also throw UnsupportedVersionException, if there are essential
        // fields that cannot be represented in the chosen version.
        request = builder.build();//note: 当为 Produce 请求时,转化为 ProduceRequest,Metadata 请求时,转化为 Metadata 请求
    } catch (UnsupportedVersionException e) {
        // If the version is not supported, skip sending the request over the wire.
        // Instead, simply add it to the local queue of aborted requests.
        log.debug("Version mismatch when attempting to send {} to {}",
                clientRequest.toString(), clientRequest.destination(), e);
        ClientResponse clientResponse = new ClientResponse(clientRequest.makeHeader(),
                clientRequest.callback(), clientRequest.destination(), now, now,
                false, e, null);
        abortedSends.add(clientResponse);
        return;
    }
    RequestHeader header = clientRequest.makeHeader();
    if (log.isDebugEnabled()) {
        int latestClientVersion = ProtoUtils.latestVersion(clientRequest.apiKey().id);
        if (header.apiVersion() == latestClientVersion) {
            log.trace("Sending {} to node {}.", request, nodeId);
        } else {
            log.debug("Using older server API v{} to send {} to node {}.",
                header.apiVersion(), request, nodeId);
        }
    }
    //note: Send是一个接口，这里返回的是 NetworkSend，而 NetworkSend 继承 ByteBufferSend
    Send send = request.toSend(nodeId, header);
    InFlightRequest inFlightRequest = new InFlightRequest(
            header,
            clientRequest.createdTimeMs(),
            clientRequest.destination(),
            clientRequest.callback(),
            clientRequest.expectResponse(),
            isInternalRequest,
            send,
            now);
    this.inFlightRequests.add(inFlightRequest);
    //note: 将 send 和对应 kafkaChannel 绑定起来，并开启该 kafkaChannel 底层 socket 的写事件
    selector.send(inFlightRequest.send);
}
```

### Selector.send()

这个方法就比较容易理解了，它的作用就是获取该 Send 对应的 KafkaChannel，调用 `setSend()` 向 KafkaChannel 注册一个 `Write` 事件。

```java
//note: 发送请求
public void send(Send send) {
    String connectionId = send.destination();
    if (closingChannels.containsKey(connectionId))
        this.failedSends.add(connectionId);
    else {
        KafkaChannel channel = channelOrFail(connectionId, false);
        try {
            channel.setSend(send);
        } catch (CancelledKeyException e) {
            this.failedSends.add(connectionId);
            close(channel, false);
        }
    }
}
```

### KafkaChannel.setSend()

`setSend()` 方法需要配合 `write()`（该方法是在 `Selector.poll()` 中调用的） 方法一起来看

- `setSend()`：将当前 KafkaChannel 的 Send 赋值为要发送的 Send，并注册一个 `OP_WRITE` 事件；
- `write()`：发送当前的 Send，发送完后删除注册的 `OP_WRITE` 事件。

```java
//note: 每次调用时都会注册一个 OP_WRITE 事件
public void setSend(Send send) {
    if (this.send != null)
        throw new IllegalStateException("Attempt to begin a send operation with prior send operation still in progress.");
    this.send = send;
    this.transportLayer.addInterestOps(SelectionKey.OP_WRITE);
}

//note: 调用 send() 发送 Send
public Send write() throws IOException {
    Send result = null;
    if (send != null && send(send)) {
        result = send;
        send = null;
    }
    return result;
}

//note: 发送完成后,就删除这个 WRITE 事件
private boolean send(Send send) throws IOException {
    send.writeTo(transportLayer);
    if (send.completed())
        transportLayer.removeInterestOps(SelectionKey.OP_WRITE);

    return send.completed();
}
```

最后，简单总结一下，可以回过头再看一下第一张图，对于 KafkaProducer 而言，其直接调用是 Sender，而 Sender  底层调用的是 NetworkClient，NetworkClient 则是通过 Selector 实现，Selector 则是对 Java  NIO 原生接口的封装。























