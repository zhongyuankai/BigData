# `AllocationDecider` 内置决策器源码跟踪

> 仓库：`elasticsearch-didi`
>
> 基线提交：`db41ec0c4e78`
>
> 跟踪范围：`ClusterModule#createAllocationDeciders()` 直接注册的 18 个内置决策器
>
> 注册入口：[`ClusterModule.java`](../server/src/main/java/org/elasticsearch/cluster/ClusterModule.java#L339-L370)
>
> 聚合入口：[`AllocationDeciders.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/AllocationDeciders.java#L30-L241)

## 1. 范围说明

本文只跟踪下列 18 个内置决策器：

1. `MaxRetryAllocationDecider`
2. `ResizeAllocationDecider`
3. `ReplicaAfterPrimaryActiveAllocationDecider`
4. `RebalanceOnlyWhenActiveAllocationDecider`
5. `ClusterRebalanceAllocationDecider`
6. `ConcurrentRebalanceAllocationDecider`
7. `EnableAllocationDecider`
8. `NodeVersionAllocationDecider`
9. `SnapshotInProgressAllocationDecider`
10. `RestoreInProgressAllocationDecider`
11. `NodeShutdownAllocationDecider`
12. `NodeReplacementAllocationDecider`
13. `FilterAllocationDecider`
14. `SameShardAllocationDecider`
15. `DiskThresholdDecider`
16. `ThrottlingAllocationDecider`
17. `ShardsLimitAllocationDecider`
18. `AwarenessAllocationDecider`

不展开 `ClusterPlugin#createAllocationDeciders()` 注入的 X-Pack、CCR、归档索引和 searchable snapshot 决策器。为了说明上述 18 个类如何真正生效，本文仍会跟踪：

- 抽象契约 `AllocationDecider`；
- 聚合器 `AllocationDeciders`；
- `Decision.YES`、`Decision.NO`、`Decision.THROTTLE` 的合并规则；
- `BalancedShardsAllocator`、`DesiredBalanceReconciler` 和 gateway allocator 的主要调用入口；
- `EnableAllocationDecider` 中 DiDi 自定义的 `cluster.routing.master_enhance.enable` 的实际使用位置。

## 2. 结论先行

这 18 个决策器不是 18 个依次执行的“分配步骤”，而是同一个候选操作上的 18 个约束条件。allocator 先提出问题，例如“这个 shard 能否放到这个节点”，`AllocationDeciders` 再用同一个问题依次调用所有决策器并合并结果。

最终优先级固定为：

~~~text
NO > THROTTLE > YES
~~~

- 任意一个决策器返回 `NO`，整个候选操作就是 `NO`。
- 没有 `NO`、但至少一个决策器返回 `THROTTLE`，整体是 `THROTTLE`。
- 所有决策器均为 `YES`，整体才是 `YES`。

18 个决策器大致分成六层约束：

| 约束层 | 决策器 | 核心问题 |
| --- | --- | --- |
| 失败保护 | `MaxRetryAllocationDecider` | 失败次数是否达到重试上限 |
| 恢复拓扑 | `ResizeAllocationDecider`、`ReplicaAfterPrimaryActiveAllocationDecider` | resize 的源 shard 在哪里、replica 的 primary 是否 active |
| rebalance 闸门 | `RebalanceOnlyWhenActiveAllocationDecider`、`ClusterRebalanceAllocationDecider`、`ConcurrentRebalanceAllocationDecider`、`EnableAllocationDecider` | 当前能否开始 rebalance、某个 shard 是否允许 rebalance |
| 版本和快照生命周期 | `NodeVersionAllocationDecider`、`SnapshotInProgressAllocationDecider`、`RestoreInProgressAllocationDecider` | recovery 是否版本兼容、是否正在 snapshot/restore |
| 节点生命周期 | `NodeShutdownAllocationDecider`、`NodeReplacementAllocationDecider` | shutdown/replacement 节点能否接收或保留 shard |
| 节点容量与拓扑 | `FilterAllocationDecider`、`SameShardAllocationDecider`、`DiskThresholdDecider`、`ThrottlingAllocationDecider`、`ShardsLimitAllocationDecider`、`AwarenessAllocationDecider` | 属性、同副本、磁盘、恢复并发、shard 数和故障域是否满足约束 |

需要特别注意四点：

1. `THROTTLE` 是“当前暂缓”，不是永久不可分配。典型来源是恢复并发、rebalance 并发和正在运行的 snapshot。
2. `canAllocate()`、`canRemain()`、`canRebalance()` 是不同问题。一个节点允许 shard 保留，不代表允许新 shard 进入；cluster 允许 rebalance，也不代表每个 shard 都允许移动。
3. 强制分配不是无条件绕过所有约束。各决策器可以通过覆盖 `canForceAllocatePrimary()` 或 `canForceAllocateDuringReplace()` 保留自身的硬约束。
4. 注册顺序会决定非 debug 模式首先暴露哪个 `NO`，但不会改变最终的 `NO > THROTTLE > YES` 语义。

## 3. 注册与对象生命周期

### 3.1 注册顺序

[`ClusterModule#createAllocationDeciders()`](../server/src/main/java/org/elasticsearch/cluster/ClusterModule.java#L339-L370) 使用 `LinkedHashMap<Class<?>, AllocationDecider>` 保存决策器：

~~~java
Map<Class<?>, AllocationDecider> deciders = new LinkedHashMap<>();
addAllocationDecider(deciders, new MaxRetryAllocationDecider());
// ...其余内置决策器...
addAllocationDecider(deciders, new AwarenessAllocationDecider(settings, clusterSettings));
~~~

这同时保证了：

- 迭代顺序与注册顺序一致；
- 同一个实现类不能注册两次；
- 后续插件决策器只能追加，不能用相同 class 替换内置实例。

[`ClusterModule#addAllocationDecider()`](../server/src/main/java/org/elasticsearch/cluster/ClusterModule.java#L372-L376) 以具体 class 为 key；发现重复 class 时抛出 `IllegalArgumentException`。

### 3.2 动态设置如何进入决策器

构造器接收 `ClusterSettings` 的决策器通常通过 `initializeAndWatch()` 或 `addSettingsUpdateConsumer()`：

~~~text
节点启动时读取当前 setting
  -> 写入 volatile 字段
  -> cluster setting 动态更新
  -> update consumer 更新 volatile 字段
  -> 下一次 allocation 立即读取新值
~~~

因此 `ClusterRebalanceAllocationDecider`、`ConcurrentRebalanceAllocationDecider`、`EnableAllocationDecider`、`SameShardAllocationDecider`、`ThrottlingAllocationDecider`、`ShardsLimitAllocationDecider` 等不需要重建 `AllocationService` 就能响应动态设置。

`MaxRetryAllocationDecider` 等 index-scoped 决策器则直接从本次 `RoutingAllocation.metadata()` 中读取 index settings。

## 4. `AllocationDecider` 的完整契约

抽象基类位于 [`AllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/AllocationDecider.java#L26-L155)。除强制分配的特殊语义外，未覆盖的方法默认返回 `Decision.ALWAYS`，也就是无标签、无解释的恒定 `YES`。

| 方法 | 回答的问题 | 默认行为 |
| --- | --- | --- |
| `canAllocate(ShardRouting, RoutingAllocation)` | 这个 shard 在当前状态下是否可以分配到任何节点 | `ALWAYS` |
| `canAllocate(ShardRouting, RoutingNode, RoutingAllocation)` | 这个 shard 是否能分配到指定节点 | `ALWAYS` |
| `canAllocate(IndexMetadata, RoutingNode, RoutingAllocation)` | 该 index 的 shard 是否能进入指定节点 | `ALWAYS` |
| `canRemain(...)` | 已在节点上的 shard 能否继续留在该节点 | `ALWAYS` |
| `canRebalance(RoutingAllocation)` | cluster 当前能否执行任何 rebalance | `ALWAYS` |
| `canRebalance(ShardRouting, RoutingAllocation)` | 指定 started shard 当前能否 rebalance | `ALWAYS` |
| `shouldAutoExpandToNode(...)` | auto-expand replicas 计算是否应包含指定节点 | `ALWAYS` |
| `canForceAllocatePrimary(...)` | 有本地副本的 unassigned primary 能否被强制分配 | 重新调用普通 `canAllocate()`；普通结果为 `NO` 时转换为 `YES` |
| `canForceAllocateDuringReplace(...)` | replacement source 上的 shard 能否强制迁到 target | 默认直接 `YES` |
| `canAllocateReplicaWhenThereIsRetentionLease(...)` | 有 retention lease 的旧 replica copy 能否恢复 | 默认复用普通 `canAllocate()` |
| `getForcedInitialShardAllocationToNodes(...)` | 初次分配是否只能落入某个 node-id 集合 | 默认 `Optional.empty()`，表示没有附加集合约束 |

### 4.1 `canForceAllocatePrimary()` 的反转语义

基类先执行普通 `canAllocate()`：

~~~text
普通结果 NO       -> 默认转换成 YES，允许强制恢复 primary
普通结果 THROTTLE -> 保持 THROTTLE
普通结果 YES      -> 保持 YES
~~~

所以需要在强制 primary 时仍然生效的硬约束，必须覆盖 `canForceAllocatePrimary()`。本范围内覆盖它的类是：

- `MaxRetryAllocationDecider`
- `ResizeAllocationDecider`
- `RestoreInProgressAllocationDecider`
- `SameShardAllocationDecider`

前三个直接复用普通结果，不允许基类把 `NO` 翻成 `YES`；`SameShardAllocationDecider` 只保留“同一节点不能有同 shard 的另一 copy”约束，不检查可选的 same-host 约束。

### 4.2 replacement 强制分配的 opt-out 模型

`canForceAllocateDuringReplace()` 默认 `YES`。因此 replacement 搬迁默认可以绕过普通 allocation `NO`。需要保留硬约束的类会显式覆盖该方法。

本范围内显式保留约束的包括：

- retry 上限；
- resize 源节点；
- primary-before-replica；
- node version；
- snapshot/restore 生命周期；
- replacement source-target 对应关系；
- same shard；
- 磁盘不能超过物理容量；
- recovery throttling；
- awareness 故障域。

而 `EnableAllocationDecider`、`FilterAllocationDecider`、`ShardsLimitAllocationDecider` 等没有覆盖 replacement 强制入口，它们的普通 `NO` 可以在合法 replacement 流程中被绕过。

## 5. `AllocationDeciders` 如何聚合 18 个结果

### 5.1 普通模式：遇到第一个 `NO` 短路

[`AllocationDeciders#withDeciders()`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/AllocationDeciders.java#L201-L229) 在 `DebugMode.OFF` 下的逻辑等价于：

~~~java
Decision result = Decision.YES;
for (AllocationDecider decider : deciders) {
    Decision decision = action(decider);
    if (decision.type() == NO) {
        return decision;
    }
    if (result.type() == YES && decision.type() == THROTTLE) {
        result = decision;
    }
}
return result;
~~~

效果是：

- 第一个 `NO` 立即返回，后面的决策器不执行；
- 第一个 `THROTTLE` 暂存，但循环继续，因为后面仍可能出现更强的 `NO`；
- 后续 `YES` 不会覆盖已出现的 `THROTTLE`。

### 5.2 debug 模式：收集 `Decision.Multi`

`DebugMode.ON` 不短路，收集除 `Decision.ALWAYS` 外的全部结果；`DebugMode.EXCLUDE_YES_DECISIONS` 只收集 `NO` 和 `THROTTLE`。

`Decision.Multi#type()` 再遍历内部结果：

~~~text
发现 NO       -> 立即得到 NO
否则发现 THROTTLE -> 最终至少为 THROTTLE
否则          -> YES
~~~

这就是 allocation explain 能同时展示多个 decider 解释的基础。debug 与非 debug 模式应产生相同的最终 type，只是执行量和解释信息不同。

### 5.3 ignored shard-node 在 18 个决策器之前短路

对于带目标节点的 `canAllocate()`、`canRemain()`、`canForceAllocatePrimary()` 和 retention-lease replica 检查，聚合器先调用：

~~~java
allocation.shouldIgnoreShardForNode(shardId, nodeId)
~~~

如果该 shard-node 组合因本轮更早的失败被临时标记 ignored，直接返回标签为 `ignored_shards_for_node` 的 `NO`，18 个决策器都不会执行。

### 5.4 初始节点集合使用交集

[`getForcedInitialShardAllocationToNodes()`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/AllocationDeciders.java#L231-L240) 不使用 `YES/NO/THROTTLE` 聚合，而是把所有非空 node-id 集合求交集：

~~~text
Resize 限定集合 ∩ Filter 初始恢复限定集合 ∩ 其他扩展限定集合
~~~

- 所有决策器都返回 `Optional.empty()`：没有预筛选限制。
- 交集为非空集合：gateway/allocator 只需考察这些节点。
- 交集为空集合：当前不存在满足全部初始拓扑约束的节点。

## 6. allocator 在什么时候问这些问题

~~~mermaid
flowchart TD
    A["AllocationService 创建 RoutingAllocation"] --> B["Gateway allocator 处理已有副本"]
    B --> C["canAllocate / canForceAllocatePrimary"]
    C --> D["ShardsAllocator 处理普通分配"]
    D --> E["unassigned shard"]
    D --> F["cannot-remain shard"]
    D --> G["rebalance shard"]
    E --> E1["canAllocate(shard, allocation)"]
    E1 --> E2["canAllocate(shard, node, allocation)"]
    F --> F1["canRemain(shard, currentNode, allocation)"]
    F1 --> F2["canAllocate 或 canForceAllocateDuringReplace"]
    G --> G1["canRebalance(allocation)"]
    G1 --> G2["canRebalance(shard, allocation)"]
    G2 --> G3["canAllocate(shard, allocation)"]
    G3 --> G4["canAllocate(shard, targetNode, allocation)"]
~~~

主要源码入口如下：

| 场景 | 聚合方法 | 主要调用位置 |
| --- | --- | --- |
| gateway primary copy 选择 | `canAllocate()` / `canForceAllocatePrimary()` | [`PrimaryShardAllocator.java`](../server/src/main/java/org/elasticsearch/gateway/PrimaryShardAllocator.java#L388-L404) |
| gateway replica copy 选择 | `canAllocate()` / retention-lease 特殊入口 | [`ReplicaShardAllocator.java`](../server/src/main/java/org/elasticsearch/gateway/ReplicaShardAllocator.java#L280-L295) |
| balanced 分配 unassigned | shard-level 后 node-level `canAllocate()` | [`BalancedShardsAllocator.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L1200-L1245) |
| balanced 强制搬离 | `canRemain()` 后选择普通或 replacement force | [`BalancedShardsAllocator.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L947-L1045) |
| balanced rebalance | global 和 shard-level `canRebalance()` | [`BalancedShardsAllocator.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L540-L580) |
| desired balance 落地 | 同一组 `canAllocate/canRemain/canRebalance` | [`DesiredBalanceReconciler.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/DesiredBalanceReconciler.java#L450-L535) |
| 手工 move/allocate replica | node-level `canAllocate()` | [`MoveAllocationCommand.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/command/MoveAllocationCommand.java#L145-L157)、[`AllocateReplicaAllocationCommand.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/command/AllocateReplicaAllocationCommand.java#L128-L143) |

## 7. 覆盖方法总表

`●` 表示该类覆盖对应入口；空白表示继承基类默认行为。

| 决策器 | shard `canAllocate` | node `canAllocate` | `canRemain` | global `canRebalance` | shard `canRebalance` | auto-expand | force primary | force replace | retention lease | forced initial nodes |
| --- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| `MaxRetryAllocationDecider` | ● | ● |  |  |  |  | ● | ● |  |  |
| `ResizeAllocationDecider` | ● | ● |  |  |  |  | ● | ● |  | ● |
| `ReplicaAfterPrimaryActiveAllocationDecider` | ● | ● |  |  |  |  |  | ● |  |  |
| `RebalanceOnlyWhenActiveAllocationDecider` |  |  |  |  | ● |  |  |  |  |  |
| `ClusterRebalanceAllocationDecider` |  |  |  | ● | ● |  |  |  |  |  |
| `ConcurrentRebalanceAllocationDecider` |  |  |  | ● | ● |  |  |  |  |  |
| `EnableAllocationDecider` | ● | ● |  | ● | ● |  |  |  |  |  |
| `NodeVersionAllocationDecider` |  | ● |  |  |  |  |  | ● |  |  |
| `SnapshotInProgressAllocationDecider` |  | ● |  |  | ● |  |  | ● |  |  |
| `RestoreInProgressAllocationDecider` | ● | ● |  |  |  |  | ● | ● |  |  |
| `NodeShutdownAllocationDecider` |  | ● | ● |  |  | ● |  |  |  |  |
| `NodeReplacementAllocationDecider` |  | ● | ● |  |  | ● |  | ● | ● |  |
| `FilterAllocationDecider` |  | ● | ● |  |  | ● |  |  |  | ● |
| `SameShardAllocationDecider` |  | ● |  |  |  |  | ● | ● |  |  |
| `DiskThresholdDecider` |  | ● | ● |  |  |  |  | ● |  |  |
| `ThrottlingAllocationDecider` |  | ● |  |  |  |  |  | ● |  |  |
| `ShardsLimitAllocationDecider` |  | ● | ● |  |  |  |  |  |  |  |
| `AwarenessAllocationDecider` |  | ● | ● |  |  |  |  | ● |  |  |

下文按注册顺序逐个跟踪。这个顺序也是 `DebugMode.OFF` 时实际执行和首次 `NO` 暴露的顺序。

## 8. `MaxRetryAllocationDecider`：失败重试上限

源码：[`MaxRetryAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/MaxRetryAllocationDecider.java#L26-L128)

### 8.1 设置与状态来源

| 设置 | 默认值 | 作用域 | 含义 |
| --- | ---: | --- | --- |
| `index.allocation.max_retries` | `5` | dynamic、index | allocation 或 relocation 连续失败达到该值后停止自动重试 |

该设置带 `NotCopyableOnResize`，resize 目标 index 不直接继承源 index 的失败重试配置。

决策器不自己累加次数，而是读取：

- unassigned shard 的 `UnassignedInfo#getNumFailedAllocations()`；
- 已分配 shard 的 `RelocationFailureInfo#failedRelocations()`。

### 8.2 `canAllocate()` 决策流程

~~~text
isSimulating?
  -> YES，desired-balance 模拟忽略历史失败

numFailedAllocations > 0?
  -> >= maxRetries: NO
  -> <  maxRetries: YES

numFailedRelocations > 0?
  -> >= maxRetries: NO
  -> <  maxRetries: YES

都没有失败
  -> YES
~~~

allocation failure 优先于 relocation failure 检查。达到上限的判断是 `>=`，不是 `>`。

debug 解释会提示通过：

~~~text
POST /_cluster/reroute?retry_failed&metric=none
~~~

手工触发失败 shard 的再次尝试。

### 8.3 强制分配语义

`canForceAllocatePrimary()` 和 `canForceAllocateDuringReplace()` 都直接复用普通 `canAllocate()`，所以：

- retry 上限不能被 force-primary 自动反转；
- replacement 也不能绕过 retry 上限。

这是一个明确的防止故障热循环的硬闸门。

## 9. `ResizeAllocationDecider`：split/clone 的源 shard 约束

源码：[`ResizeAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/ResizeAllocationDecider.java#L25-L103)

该决策器只处理同时满足以下条件的 shard：

~~~text
存在 UnassignedInfo
AND recoverySource.type == LOCAL_SHARDS
~~~

也就是从本机其他 shard 构造目标 shard 的 resize 初始恢复。

### 9.1 source index 不存在

从目标 `IndexMetadata#getResizeSourceIndex()` 找源 index；metadata 中找不到时直接 `NO`。

### 9.2 shrink 与 split/clone 的分工

~~~text
目标 shard 数 < 源 shard 数
  -> shrink
  -> 本决策器返回 ALWAYS
  -> 由 FilterAllocationDecider 的 initial recovery filter 约束节点

目标 shard 数 == 源 shard 数
  -> clone
  -> selectCloneShard() 找源 shard

目标 shard 数 > 源 shard 数
  -> split
  -> selectSplitShard() 找源 shard
~~~

对 clone/split：

1. `routingNodes.activePrimary(sourceShardId)` 必须存在，否则 `NO`。
2. shard-level `canAllocate()` 只要求源 primary active。
3. node-level `canAllocate()` 还要求候选 node 正是源 primary 所在 node。

### 9.3 初始节点预筛选

`getForcedInitialShardAllocationToNodes()` 对 clone/split 返回：

~~~text
Optional.of({ sourcePrimary.currentNodeId })
~~~

源 index 不存在或源 primary 未 active 时返回 `Optional.of(emptySet)`，让上层在进入逐节点 decider 前就知道没有可行节点。

### 9.4 强制入口

force-primary 和 force-replace 均复用普通判断，因此不能把 resize shard 放到不持有对应 active source primary 的节点。

## 10. `ReplicaAfterPrimaryActiveAllocationDecider`：先 primary，后 replica

源码：[`ReplicaAfterPrimaryActiveAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/ReplicaAfterPrimaryActiveAllocationDecider.java#L18-L43)

逻辑非常直接：

~~~text
shard 是 primary
  -> YES

shard 是 replica
  -> routingNodes.activePrimary(shardId) 存在: YES
  -> 不存在: NO
~~~

它回答的是“replica 的数据源是否已经可用”，不是候选 node 是否合适，所以 shard-level 和 node-level 入口使用同一判断。

force-replace 仍复用普通判断，replacement 流程不能在 primary 尚未 active 时强行分配 replica。

注册顺序上它早于 `NodeVersionAllocationDecider`。因此非 debug 模式下 primary 不 active 时会在这里先返回 `NO`，不会再进入 replica 的版本兼容检查。

## 11. `RebalanceOnlyWhenActiveAllocationDecider`：单 replication group 完整性

源码：[`RebalanceOnlyWhenActiveAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/RebalanceOnlyWhenActiveAllocationDecider.java#L17-L38)

它只覆盖 shard-level `canRebalance()`：

~~~java
allocation.routingNodes().allShardsActive(shardId, metadata)
~~~

- primary 与全部 replica 都 active：`YES`。
- 任意 copy unassigned 或 inactive：`NO`。

它只约束当前 shard 的 replication group；全 cluster 是否具备 rebalance 条件由下一个 `ClusterRebalanceAllocationDecider` 决定。

## 12. `ClusterRebalanceAllocationDecider`：cluster 健康闸门

源码：[`ClusterRebalanceAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/ClusterRebalanceAllocationDecider.java#L39-L180)

### 12.1 设置

| 设置 | 默认值 | 可选值 |
| --- | --- | --- |
| `cluster.routing.allocation.allow_rebalance` | `indices_all_active` | `always`、`indices_primaries_active`、`indices_all_active` |

设置动态更新到 volatile `type`。兼容解析同时接受旧式 camelCase：`indicesPrimariesActive`、`indicesAllActive`。

### 12.2 三种模式

| 模式 | `NO` 条件 | `YES` 条件 |
| --- | --- | --- |
| `always` | 无 | 总是允许 cluster rebalance |
| `indices_primaries_active` | 存在 unassigned primary，或存在 inactive primary | 所有 primary 已分配且 active |
| `indices_all_active` | 存在任意 unassigned shard，或存在任意 inactive shard | 所有 primary/replica 都 active |

代码先检查 unassigned，再检查 inactive。inactive 特别覆盖了 initializing 但不是 relocation target 的 shard。

global 和 shard-level `canRebalance()` 最终都走同一个 cluster-wide 判断，所以该约束会在 allocator 的 global gate 和单 shard gate 中重复出现；第一次 global `NO` 通常已使 rebalance 阶段提前结束。

## 13. `ConcurrentRebalanceAllocationDecider`：同时 rebalance 的 relocation 上限

源码：[`ConcurrentRebalanceAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/ConcurrentRebalanceAllocationDecider.java#L31-L94)

### 13.1 设置

| 设置 | 默认值 | 特殊值 |
| --- | ---: | --- |
| `cluster.routing.allocation.cluster_concurrent_rebalance` | `2` | `-1` 表示无限制 |

### 13.2 决策流程

先读取 `routingNodes.getRelocatingShardCount()`：

~~~text
simulation && relocatingShards >= 2
  -> THROTTLE

setting == -1
  -> YES

relocatingShards >= setting
  -> THROTTLE

否则
  -> YES
~~~

simulation 的 `>= 2` 检查位于 `-1` 之前，因此 desired-balance 模拟即使配置无限并发，也最多按两个并发 relocation 建模。源码注释说明这是为了避免 `BalancedShardsAllocator` 在较高并发值下产生不必要移动。

它返回 `THROTTLE` 而非 `NO`：现有 relocation 完成后，后续 reroute 可以继续 rebalance。

## 14. `EnableAllocationDecider`：allocation/rebalance 总开关

源码：[`EnableAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/EnableAllocationDecider.java#L50-L279)

### 14.1 设置与优先级

| 设置 | 默认值 | 作用域 | 可选值 |
| --- | --- | --- | --- |
| `cluster.routing.allocation.enable` | `all` | cluster | `none`、`new_primaries`、`primaries`、`all` |
| `index.routing.allocation.enable` | `all` | index | 同上 |
| `cluster.routing.rebalance.enable` | `all` | cluster | `none`、`primaries`、`replicas`、`all` |
| `index.routing.rebalance.enable` | `all` | index | 同上 |
| `cluster.routing.master_enhance.enable` | `true` | cluster | DiDi 自定义 boolean |

如果 index setting 显式存在，它覆盖 cluster setting；不是简单取两者中更严格的值。

### 14.2 allocation 开关

`canAllocate(shard, node, allocation)` 忽略 node，直接委托 shard-level 入口。

前置短路：

1. `allocation.ignoreDisable()` 为 true：`YES`。该标志用于 reroute API 的显式手工命令。
2. `allocation.isSimulating()` 为 true：`YES`。desired-balance 计算不因 allocation enable 开关停止构造长期目标。

普通分支：

| 值 | 结果 |
| --- | --- |
| `all` | 所有 shard `YES` |
| `none` | 所有 shard `NO` |
| `primaries` | primary `YES`，replica `NO` |
| `new_primaries` | 仅从非 `EXISTING_STORE` 恢复的、尚未 active 的 primary 为 `YES` |

`new_primaries` 不等于“所有 unassigned primary”：从已有磁盘 copy 恢复的 primary 会被拒绝。

### 14.3 global rebalance 开关

global `canRebalance(allocation)` 同样先处理 `ignoreDisable` 和 simulation。

当 cluster 值为 `none` 时，它扫描全部 index：

- 至少一个 index 显式配置为非 `none`：global 返回 `YES`，把精确判断留给 shard-level 入口。
- 没有任何 index 放开：global 返回 `NO`，整个 rebalance 阶段可提前结束。

cluster 值不是 `none` 时 global 直接 `YES`，即使某些 index 显式为 `none`，也由 shard-level 逐个拒绝。

### 14.4 shard-level rebalance

shard-level `canRebalance()` 再按 index-over-cluster 优先级处理：

| 值 | primary | replica |
| --- | --- | --- |
| `all` | `YES` | `YES` |
| `none` | `NO` | `NO` |
| `primaries` | `YES` | `NO` |
| `replicas` | `NO` | `YES` |

一个重要细节：该方法没有 `allocation.isSimulating()` 短路。因此模拟时：

- global rebalance gate 被放开；
- 具体 shard 仍尊重 index/cluster rebalance enable 设置。

### 14.5 DiDi `master_enhance` 的真实作用

`cluster.routing.master_enhance.enable` 被保存在 static volatile 字段，但没有参与本类的 `canAllocate()` 或 `canRebalance()` 返回值。它在其他位置控制：

- `BalancedShardsAllocator.Balancer` 是否按 region 构造候选节点；
- index 排序是否只处理当前 region 匹配的 index；
- `decideMove()` 是否跳过已知 throttled/已持有相同 shard 的候选节点；
- `moveShards()` 是否在 region 内所有节点都 throttled 后提前结束；
- `AllocationService` 是否记录 cluster health 变化日志。

所以它是 allocator 性能/region 行为开关，不是第 19 个 allocation policy。

## 15. `NodeVersionAllocationDecider`：rolling upgrade 的恢复方向

源码：[`NodeVersionAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/NodeVersionAllocationDecider.java#L25-L140)

### 15.1 unassigned primary

| recovery source | 判断 |
| --- | --- |
| snapshot | target `maxIndexVersion` 必须不早于 snapshot 的 index version |
| empty store / existing store / 其他 | `YES`，视为新 primary 或节点上已有 copy |

### 15.2 relocating primary

目标 node version 必须 `onOrAfter(sourceNode.version)`：

~~~text
旧节点 -> 同版本或新节点：YES
新节点 -> 旧节点：NO
~~~

这避免已被新版本写过的 primary 回迁到旧版本节点。

### 15.3 allocating replica

先找 active primary。存在时，replica target version 必须不早于 primary node version，保证 peer recovery 和 Lucene 格式兼容。

active primary 不存在时本类返回 `YES`，因为更早注册的 `ReplicaAfterPrimaryActiveAllocationDecider` 会负责返回 `NO`。

### 15.4 replacement

force-replace 显式复用版本判断，node replacement 不能绕过版本兼容性。

## 16. `SnapshotInProgressAllocationDecider`：snapshot 期间暂缓移动 primary

源码：[`SnapshotInProgressAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/SnapshotInProgressAllocationDecider.java#L22-L117)

`canRebalance()`、node-level `canAllocate()` 和 force-replace 都进入 `canMove()`。

### 16.1 快速放行

以下情况返回 `YES`：

- simulation；
- shard 是 replica，因为 snapshot 只从 primary 执行；
- cluster 没有 `SnapshotsInProgress`；
- shard 当前未分配节点；
- snapshot entry 是 clone；
- entry 没有 `INIT` 状态的 shard snapshot；
- 当前 snapshot 不包含该 shard；
- shard snapshot 已完成；
- snapshot 实际执行 node 不是 shard 当前 node。

### 16.2 精确阻塞条件

只有同时命中以下条件才返回 `THROTTLE`：

~~~text
当前 shard 是 primary
AND shard 已分配
AND 存在真实 snapshot entry
AND 该 entry 正在 snapshot 当前 shard
AND shard snapshot 尚未完成
AND snapshot nodeId == shard.currentNodeId
~~~

返回 `THROTTLE` 是因为 snapshot 完成后移动自然可继续，不是永久拓扑冲突。

已完成的 shard snapshot 不再阻塞。`ABORTED` 本身的 `completed` 标志为 false；代码依赖更早的 `entry.hasShardsInInitState() == false` 分支，在整个 snapshot entry 已无任何 `INIT` shard 时放行，而不是单凭当前 shard 为 `ABORTED` 就无条件放行。

## 17. `RestoreInProgressAllocationDecider`：失败 restore 的自动分配熔断

源码：[`RestoreInProgressAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/RestoreInProgressAllocationDecider.java#L21-L72)

### 17.1 非 API restore

以下情况直接 `YES`：

- recovery source 不是 snapshot；
- `restoreUUID == NO_API_RESTORE_UUID`。

后者表示恢复不是由 snapshot restore API 管理，不依赖 `RestoreInProgress` cluster custom。

### 17.2 API restore

从 cluster state 的 `RestoreInProgress` 以 `restoreUUID` 查 entry，再按 shard id 查状态：

- entry 和 shard status 存在，且状态未 completed：`YES`，restore 正常进行。
- entry/status 不存在，或状态已经 completed 但 shard 仍需分配：`NO`，视为 restore 已失败。

失败解释会提示关闭/删除 index 后重试 restore，或通过 reroute API 强制分配 empty primary。

### 17.3 force 仍不能复活失败 restore

force-primary 与 force-replace 都复用普通结果，防止把一个仍携带失败 snapshot recovery source 的 shard 当作成功 restore 自动恢复。

## 18. `NodeShutdownAllocationDecider`：shutdown 节点的通用限制

源码：[`NodeShutdownAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/NodeShutdownAllocationDecider.java#L24-L87)

它读取 `Metadata.nodeShutdowns()`，同时约束：

- 新分配 `canAllocate()`；
- 原地保留 `canRemain()`；
- auto-expand 的候选 node。

### 18.1 shutdown type 矩阵

| type | 普通 allocate/remain | auto-expand |
| --- | --- | --- |
| `REMOVE` | `NO` | `NO` |
| `SIGTERM` | `NO` | `NO` |
| `RESTART` | `YES` | `YES` |
| `REPLACE` | `NO` | replacement target 尚未加入时，source 可暂时 `YES`；target 加入后 `NO` |

`RESTART` 预期节点仍会回到 cluster，因此不主动驱逐 shard。

`REPLACE` 的精细 source-target 流向由下一个 `NodeReplacementAllocationDecider` 管理；本类负责通用 shutdown 禁止逻辑。

本类没有覆盖 force-primary 或 force-replace，因此合法强制路径使用基类默认语义。

## 19. `NodeReplacementAllocationDecider`：source 到指定 target 的封闭通道

源码：[`NodeReplacementAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/NodeReplacementAllocationDecider.java#L23-L257)

该类同时区分：

- replacement source：以 node id 标识；
- replacement target：shutdown metadata 中配置的是 node name；
- target 实际加入后的 node id。

混淆 node id 与 node name 会直接导致误判源码含义。

### 19.1 普通 `canAllocate()`

按顺序执行：

1. 没有 replacement：`YES`。
2. shard 当前 source 与候选 target 正好构成 replacement 对：`YES`。
3. shard 当前在 replacement source，但候选不是 target：正常模式 `NO`。
4. 候选 node 自己是 replacement source：`NO`，不允许新数据进入正在腾空的节点。
5. 候选 node 是 replacement target，但 shard 并非来自对应 source：`NO`，防止无关数据占据 target、阻碍 source 腾空。
6. replacement 与该 shard/node 无关：`YES`。

### 19.2 reconciliation 特例

`allocation.isReconciling()` 时有两处放宽：

- source 上的 shard 可以移动到 desired node，即使 desired node 不是 replacement target；
- 已分配 shard 可以移动到 target，以便 desired-balance reconciliation 调整已有数据。

但 unassigned shard 仍不能在 reconciliation 中随意进入 replacement target，因为它可能消耗用于接收 source 数据的空间。

### 19.3 `canRemain()`

- replacement source 上的 shard：`NO`，必须被搬走。
- 其他节点：`YES`。

这会让 allocator 的 `moveShards()` 把 replacement source 视为 cannot-remain source，不要求移动改善 balance weight。

### 19.4 auto-expand

对 target：只有该 index 已在 source 或 target 上有 shard，才允许 auto-expand 到 target。

对 source：只有 source 已有该 index shard且 target 尚未加入，才临时允许 auto-expand；target 到达后拒绝。

### 19.5 force-replace

只有以下精确关系返回 `YES`：

~~~text
shard.currentNodeId 是 replacement source
AND candidate.node.name 是该 source 配置的 replacement target name
~~~

其他组合全部 `NO`。这个结果与其他显式覆盖 force-replace 的决策器继续做 `AND` 聚合。

### 19.6 retention lease 特例

候选 node 是 replacement target 时直接 `YES`，允许曾在该处存在并带 retention lease 的 replica copy 恢复；否则回到普通 `canAllocate()`。

## 20. `FilterAllocationDecider`：require/include/exclude 属性过滤

源码：[`FilterAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/FilterAllocationDecider.java#L60-L246)

### 20.1 cluster settings

| setting prefix | 组合语义 | 不匹配时 |
| --- | --- | --- |
| `cluster.routing.allocation.require.*` | `AND` | `NO` |
| `cluster.routing.allocation.include.*` | `OR` | `NO` |
| `cluster.routing.allocation.exclude.*` | `OR` | 匹配时 `NO` |

index 级对应设置来自 `IndexMetadata`：

- `index.routing.allocation.require.*`
- `index.routing.allocation.include.*`
- `index.routing.allocation.exclude.*`

### 20.2 判断顺序

~~~text
cluster require
  -> cluster include
  -> cluster exclude
  -> index require
  -> index include
  -> index exclude
  -> YES
~~~

cluster 层先于 index 层，任一层返回 `NO` 就结束本决策器。

所有 filter 都先经过 `DiscoveryNodeFilters.trimTier()`。这意味着 `_tier` 属性不由本内置决策器处理，而是留给插件侧的 data-tier 决策逻辑；本文不展开后者。

### 20.3 shrink 初始恢复

对于 unassigned 且 recovery source 为 `LOCAL_SHARDS` 的 shard，先检查 index 的 internal initial recovery filter。候选节点必须持有 shrink 所需的全部源 shard copy，否则 `NO`。

`getForcedInitialShardAllocationToNodes()` 还会把匹配 initial recovery filter 的 node ids 作为预筛选集合返回，与 `ResizeAllocationDecider` 的集合求交集。

### 20.4 三种入口

- shard-node `canAllocate()`：额外包含 shrink initial recovery 检查。
- index-node `canAllocate()` 与 `canRemain()`：执行普通 cluster/index filter。
- `shouldAutoExpandToNode()`：同样执行普通 filter。

本类没有覆盖 force-primary 和 force-replace，所以显式 force-primary 或合法 replacement force 可以绕过普通 filter `NO`。

## 21. `SameShardAllocationDecider`：同一 shard copy 隔离

源码：[`SameShardAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/SameShardAllocationDecider.java#L35-L151)

### 21.1 永久规则：同 node 不得有另一 copy

`routingNodes.assignedShards(shardId)` 返回同 replication group 已分配的全部 copy。只要候选 `nodeId` 已持有其中任一 copy，就返回 `NO`。

这是 primary/replica 不共置以及两个 replica 不共置的基础硬约束。

### 21.2 可选规则：同 host 不得有另一 copy

| 设置 | 默认值 |
| --- | --- |
| `cluster.routing.allocation.same_shard.host` | `false` |

为 true 时，除了 node id，还比较 `DiscoveryNode#getHostAddress()`。同一台物理机运行多个 ES node 时，可禁止相同 shard copy 落在相同 host。

index 设置 auto-expand replicas 到 all nodes 时，same-host 限制被忽略，否则 auto-expand 无法覆盖同 host 的所有 node；同-node 限制仍然先执行并保持有效。

### 21.3 强制入口

- force-primary：只调用 `decideSameNode()`，因此绝不允许同 node 两个 copy，但可以忽略 same-host 限制。
- force-replace：复用完整普通判断，因此同-node 和已启用的 same-host 都不能绕过。

## 22. `DiskThresholdDecider`：磁盘水位与 relocation 预估

源码：[`DiskThresholdDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/DiskThresholdDecider.java#L66-L570)

设置容器：[`DiskThresholdSettings.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/DiskThresholdSettings.java#L34-L535)

### 22.1 相关设置

| 设置 | 默认值 | 本决策器中的作用 |
| --- | --- | --- |
| `cluster.routing.allocation.disk.threshold_enabled` | `true` | false 时 fail-open，直接 `YES` |
| `cluster.routing.allocation.disk.watermark.low` | `85%` | 新 shard 是否允许进入 node |
| `cluster.routing.allocation.disk.watermark.low.max_headroom` | `200GB`，显式配置 low 后默认为 `-1` | 百分比 low 的最大空闲空间要求 |
| `cluster.routing.allocation.disk.watermark.high` | `90%` | 已有 shard 是否必须移走，以及加入 shard 后的上限 |
| `cluster.routing.allocation.disk.watermark.high.max_headroom` | `150GB`，显式配置 high 后默认为 `-1` | 百分比 high 的最大空闲空间要求 |
| `cluster.routing.allocation.disk.watermark.enable_for_single_data_node` | `true` | deprecated；当前只允许 true，构造器读取它以产生弃用提示 |
| `index.routing.allocation.disk.watermark.ignore` | `false` | private index setting；true 时该 index 忽略水位 |

`flood_stage`、frozen flood stage 和 reroute interval 也由 `DiskThresholdSettings` 保存，但 `DiskThresholdDecider#canAllocate/canRemain` 的直接判断只使用 enable、low 和 high。flood stage 主要由 disk monitor 用于 index block 等后续动作。

### 22.2 百分比与绝对字节

watermark 可全部使用百分比/ratio，或全部使用绝对 free bytes，不能混用。校验关系为：

~~~text
百分比 used：low <= high <= flood
绝对 free bytes：low >= high >= flood
~~~

百分比模式还可用 max headroom 限制超大磁盘需要保留的绝对空闲空间。

### 22.3 磁盘数据选择

`canAllocate()` 使用 `getNodeMostAvailableDiskUsages()`，选择候选 node 最宽裕的数据路径；`canRemain()` 使用 `getNodeLeastAvailableDiskUsages()`，关注 node 最拥挤的数据路径。

如果目标 node 没有单独 usage，但其他节点有数据，使用其他节点平均 usage。整个 usage map 为空时 fail-open 返回 `YES`，避免 stats 暂缺导致全 cluster 停止分配。

### 22.4 relocation 空间修正

`DiskUsageWithRelocations.freeBytes` 的本质是：

~~~text
stats.freeBytes
- reserved/incoming/unaccounted shard size
+ 可选的 leaving shard size
~~~

其中 `sizeOfUnaccountedShards()`：

- 优先计入 `ClusterInfo.ReservedSpace`；
- 对 stats/reserved space 尚未覆盖的 initializing shard，加入 expected shard size；
- 加入未被 cluster info 统计到的 searchable snapshot shard size；
- `subtractShardsMovingAway=true` 时减去正在迁出的 shard size。

`canAllocate()` 传 false，迁出数据在真正释放前仍算占用，取保守值；`canRemain()` 传 true，因为判断的是迁出完成后的稳态。

### 22.5 `canAllocate()` 完整顺序

~~~text
disk threshold disabled?
  -> YES
usage map empty?
  -> YES
index ignore watermark?
  -> YES
修正后 freeBytes < 0?
  -> NO
freeBytes 低于 low?
  -> 普通 shard: NO
  -> 从 EMPTY_STORE 首次创建的 primary:
       高于 high 所需 free bytes: YES
       否则: NO
预计加入 shard 后低于 high 所需 free bytes?
  -> NO
否则
  -> YES
~~~

所以全新 empty primary 可以在 low 与 high 之间启动，以免轻微磁盘压力造成新 index primary 永久 unassigned；但不能突破 high。

第二次 high 检查使用 `getExpectedShardSize()` 计算 `freeBytesAfterShard`，防止“当前尚未超 high，但加入这个大 shard 后会超 high”。

### 22.6 `canRemain()`

只有 shard 所在 data path 正好是 node 的最拥挤路径时才检查 high；否则返回 `YES_NOT_MOST_UTILIZED_DISK`。

对最拥挤路径：

- 修正后 free bytes 为负：`NO`；
- 低于 high 所需 free bytes：`NO`，要求 shard 搬离；
- 否则 `YES`。

注意 `canRemain()` 不使用 low。low 控制新流入，high 才触发已有 shard 的强制迁出，这形成回滞区间，减少水位附近反复搬迁。

### 22.7 replacement 强制分配

force-replace 不再检查 low/high，只计算加入 shard 后是否小于 0 bytes：

- 会超过物理容量 100%：`NO`。
- 仍有非负空间：返回基类 force-replace `YES`。

replacement 因而可以突破常规 watermarks，但不能规划一个物理上放不下的 shard。

## 23. `ThrottlingAllocationDecider`：recovery 并发限流

源码：[`ThrottlingAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/ThrottlingAllocationDecider.java#L43-L236)

### 23.1 设置

| 设置 | 默认值 | 含义 |
| --- | ---: | --- |
| `cluster.routing.allocation.node_concurrent_recoveries` | `2` | incoming/outgoing 的兼容总默认值 |
| `cluster.routing.allocation.node_concurrent_incoming_recoveries` | 继承上项 | 每个目标 node 同时接收的 peer recovery 数 |
| `cluster.routing.allocation.node_concurrent_outgoing_recoveries` | 继承上项 | 每个 primary node 同时输出的 peer recovery 数 |
| `cluster.routing.allocation.node_initial_primaries_recoveries` | `4` | 每个 node 同时执行的本地/快照等 initial primary recovery 数 |

### 23.2 unassigned primary：initial recovery

当 shard 同时为 primary 和 unassigned 时，预计的 initializing shard recovery source 必须不是 `PEER`。

它遍历候选 node 的 initializing shard，只统计：

~~~text
shard.primary() && shard.relocatingNodeId() == null
~~~

也就是 initial primary recovery，不把 primary relocation target 计入该配额。

- simulation：直接 `YES`，避免模拟被初始 primary 恢复速度限制。
- 当前数 `>= node_initial_primaries_recoveries`：`THROTTLE`。
- 否则：`YES`。

### 23.3 其他情况：peer recovery

其他 shard 预计 recovery source 为 `PEER`，依次检查：

1. target node incoming recovery 数 `>=` incoming limit：`THROTTLE`。
2. 找不到 active primary：`NO`。
3. primary node outgoing recovery 数 `>=` outgoing limit：`THROTTLE`。
4. 两侧均低于限制：`YES`。

这里没有为 simulation 放行 peer recovery；desired-balance 模拟仍会建模 peer recovery 并发。

### 23.4 `initializingShard()` 的作用

传入 `canAllocate()` 的 routing 可能是 unassigned、initializing、started 或 relocating。helper 把各种状态投影为“如果把它分到候选 node，会形成哪个 initializing target”，再通过 recovery source 断言区分 initial recovery 与 peer recovery。

force-replace 复用完整 throttling 结果，因此 replacement 搬迁也受 incoming/outgoing recovery 并发控制。

## 24. `ShardsLimitAllocationDecider`：每节点 shard 数上限

源码：[`ShardsLimitAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/ShardsLimitAllocationDecider.java#L42-L155)

### 24.1 设置

| 设置 | 默认值 | 含义 |
| --- | ---: | --- |
| `cluster.routing.allocation.total_shards_per_node` | `-1` | cluster 维度每 node shard 总数，负数不限 |
| `index.routing.allocation.total_shards_per_node` | `-1` | 单 index 在每 node 上的 shard 数，负数不限 |

### 24.2 allocate 与 remain 的边界差异

`canAllocate()` 使用：

~~~java
count >= limit
~~~

因为再加入一个 shard 会突破上限。

`canRemain()` 使用：

~~~java
count > limit
~~~

所以恰好等于 limit 的已有 shard 可以保留；动态下调 limit 后，只有当前计数真正大于新值才要求搬离。

判断先检查 cluster total，再检查当前 index total。两者都禁用时直接 `YES`。

本类没有覆盖 force-primary 或 force-replace，因此这些特殊路径可绕过 shard-count soft limit。

## 25. `AwarenessAllocationDecider`：跨 zone/rack/host 分散 copy

源码：[`AwarenessAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/AwarenessAllocationDecider.java#L71-L290)

### 25.1 设置

| 设置 | 示例 | 含义 |
| --- | --- | --- |
| `cluster.routing.allocation.awareness.attributes` | `zone,rack` | 依次检查的 awareness 属性 |
| `cluster.routing.allocation.awareness.force.<attr>.values` | `zone1,zone2,zone3` | 即使某些 value 当前没有节点，也把它计入故障域总数 |

forced awareness group 只允许子 key `values`，出现其他 key 会在 setting 校验阶段抛出异常。

### 25.2 快速放行

- awareness attribute 列表为空：`YES`。
- index auto-expand replicas 到 all nodes：`YES`，忽略 awareness。

### 25.3 每个 attribute 的算法

对 `awareness.attributes` 中每个属性：

1. 候选 node 必须带该属性，否则 `NO`。
2. 读取 cluster 当前实际存在的 attribute values。
3. 统计同一 shard id 已经位于目标 attribute value 上的 active/initializing copy 数。
4. 如果是在评估移动到目标 node，加入移动后的增量：
    - unassigned shard：加 1；
    - assigned shard 从不同 value 搬入：加 1；
    - shard 已在相同 value 内移动：不加，因为该 value 的 copy 总数不变。
5. value 总数取“实际 values 与 forced values 的去重并集”。
6. 计算每个 value 最大 copy 数：

~~~text
shardCount = numberOfReplicas + 1
valueCount = distinct(actualValues ∪ forcedValues)
maximum = ceil(shardCount / valueCount)
        = (shardCount + valueCount - 1) / valueCount
~~~

7. 目标 value 上的预计 copy 数大于 maximum：`NO`；所有属性都通过：`YES`。

只统计 `STARTED` 和 `INITIALIZING` routing。relocation target 是 initializing，会作为新位置计数；relocation source 是 relocating，不重复计数。

### 25.4 `canRemain()`

`canRemain()` 使用同一 `underCapacity()`，但 `moveToNode=false`，不额外加一次候选 copy。动态修改 awareness 或 forced values 后，如果现有 value 上的 copy 数超过新上限，返回 `NO` 触发迁出。

### 25.5 replacement

force-replace 仍调用普通 `canAllocate()`。源码注释明确要求 replacement 不能破坏 zone/rack 等故障域，否则可能把同一 shard 的全部 copy 聚集到单一故障域。

## 26. 典型组合场景

### 26.1 分配一个普通 unassigned replica

~~~text
MaxRetry：历史失败是否超限
  -> Resize：通常不适用
  -> ReplicaAfterPrimaryActive：primary 是否 active
  -> Enable：replica allocation 是否启用
  -> NodeVersion：target 是否不旧于 primary
  -> NodeShutdown/Replacement：target 是否处于特殊生命周期
  -> Filter：属性过滤
  -> SameShard：target/node host 是否已有 copy
  -> DiskThreshold：当前和加入后磁盘是否安全
  -> Throttling：target incoming、primary outgoing 是否有额度
  -> ShardsLimit：cluster/index shard 数是否超限
  -> Awareness：目标 zone/rack copy 数是否超上限
~~~

任何一步 `NO` 都拒绝该节点；只有 throttling 相关约束时可能得到 `THROTTLE` 并等待后续 reroute。

### 26.2 rebalance 一个 started shard

~~~text
global canRebalance:
  ClusterRebalance + ConcurrentRebalance + Enable

shard canRebalance:
  RebalanceOnlyWhenActive
  + ClusterRebalance
  + ConcurrentRebalance
  + Enable
  + SnapshotInProgress

target canAllocate:
  再运行与普通 node allocation 相关的决策器
~~~

因此 rebalance 需要同时通过三层：cluster gate、shard gate、target node gate。

### 26.3 搬走 `canRemain == NO` 的 shard

能产生 `canRemain == NO` 的内置类主要是：

- `NodeShutdownAllocationDecider`
- `NodeReplacementAllocationDecider`
- `FilterAllocationDecider`
- `DiskThresholdDecider`
- `ShardsLimitAllocationDecider`
- `AwarenessAllocationDecider`

这类移动是纠正非法/不再合规的现状，不需要证明 balance weight 改善。allocator 只需找到一个通过 target allocation 的节点。

### 26.4 replacement 强制迁移

~~~text
NodeReplacement：必须是配置的 source -> target
AND MaxRetry：失败次数未超限
AND Resize：满足源 shard 位置
AND ReplicaAfterPrimaryActive：replica 的 primary active
AND NodeVersion：版本兼容
AND Snapshot/Restore：生命周期允许
AND SameShard：不能与已有 copy 冲突
AND DiskThreshold：不能超过物理容量
AND Throttling：recovery 有并发额度
AND Awareness：不破坏故障域
~~~

普通 allocation enable、filter、常规 disk watermark、shard-count limit 等可被 replacement force 放宽。

## 27. 设置速查

| 决策器 | setting | 默认值 |
| --- | --- | --- |
| `MaxRetryAllocationDecider` | `index.allocation.max_retries` | `5` |
| `ClusterRebalanceAllocationDecider` | `cluster.routing.allocation.allow_rebalance` | `indices_all_active` |
| `ConcurrentRebalanceAllocationDecider` | `cluster.routing.allocation.cluster_concurrent_rebalance` | `2` |
| `EnableAllocationDecider` | `cluster.routing.allocation.enable` | `all` |
| `EnableAllocationDecider` | `index.routing.allocation.enable` | `all` |
| `EnableAllocationDecider` | `cluster.routing.rebalance.enable` | `all` |
| `EnableAllocationDecider` | `index.routing.rebalance.enable` | `all` |
| `EnableAllocationDecider` | `cluster.routing.master_enhance.enable` | `true` |
| `FilterAllocationDecider` | `cluster.routing.allocation.require.*` | 空 |
| `FilterAllocationDecider` | `cluster.routing.allocation.include.*` | 空 |
| `FilterAllocationDecider` | `cluster.routing.allocation.exclude.*` | 空 |
| `SameShardAllocationDecider` | `cluster.routing.allocation.same_shard.host` | `false` |
| `DiskThresholdDecider` | `cluster.routing.allocation.disk.threshold_enabled` | `true` |
| `DiskThresholdDecider` | `cluster.routing.allocation.disk.watermark.low` | `85%` |
| `DiskThresholdDecider` | `cluster.routing.allocation.disk.watermark.high` | `90%` |
| `DiskThresholdDecider` | `index.routing.allocation.disk.watermark.ignore` | `false` |
| `ThrottlingAllocationDecider` | `cluster.routing.allocation.node_concurrent_recoveries` | `2` |
| `ThrottlingAllocationDecider` | `cluster.routing.allocation.node_concurrent_incoming_recoveries` | 继承 concurrent |
| `ThrottlingAllocationDecider` | `cluster.routing.allocation.node_concurrent_outgoing_recoveries` | 继承 concurrent |
| `ThrottlingAllocationDecider` | `cluster.routing.allocation.node_initial_primaries_recoveries` | `4` |
| `ShardsLimitAllocationDecider` | `cluster.routing.allocation.total_shards_per_node` | `-1` |
| `ShardsLimitAllocationDecider` | `index.routing.allocation.total_shards_per_node` | `-1` |
| `AwarenessAllocationDecider` | `cluster.routing.allocation.awareness.attributes` | 空 |
| `AwarenessAllocationDecider` | `cluster.routing.allocation.awareness.force.<attr>.values` | 空 |

`ResizeAllocationDecider`、`ReplicaAfterPrimaryActiveAllocationDecider`、`RebalanceOnlyWhenActiveAllocationDecider`、`NodeVersionAllocationDecider`、`SnapshotInProgressAllocationDecider`、`RestoreInProgressAllocationDecider`、`NodeShutdownAllocationDecider` 和 `NodeReplacementAllocationDecider` 主要依赖 routing/cluster metadata，而不是独立 allocation setting。

## 28. 排障时如何读 allocation explain

建议先按 decision type，再按 label 分类：

| label | 常见含义 | 首要检查 |
| --- | --- | --- |
| `max_retry` | allocation/relocation 失败次数达上限 | `UnassignedInfo`、失败 details、是否需要 `retry_failed` |
| `resize` | source index/primary 不存在或候选 node 不持有源 primary | resize source metadata、源 primary routing |
| `replica_after_primary_active` | replica 的 primary 尚未 active | primary unassigned 原因 |
| `rebalance_only_when_active` | replication group 仍有 inactive copy | 同 shard id 的全部 routing |
| `cluster_rebalance` | cluster 健康不满足 rebalance 设置 | unassigned/inactive primary 或 replica |
| `concurrent_rebalance` | relocation 并发达到限制 | relocating shard count、动态设置 |
| `enable` | allocation/rebalance 被 cluster/index setting 关闭 | index override 是否存在 |
| `node_version` | rolling upgrade 方向或 snapshot index version 不兼容 | source/primary/target version |
| `snapshot_in_progress` | primary 正在当前 node 上执行 snapshot | snapshot shard status；通常等待即可 |
| `restore_in_progress` | API restore 已失败或 cluster restore 状态丢失 | restore UUID、shard restore status、details |
| `node_shutdown` | node 正在 remove/sigterm/replace | node shutdown metadata |
| `node_replacement` | source-target 不匹配或 target 被无关数据占用 | source node id、target node name |
| `filter` | cluster/index 属性不匹配 | require/include/exclude 及 node attributes |
| `same_shard` | node/host 已有相同 shard copy | assigned shard copies、same-host 设置 |
| `disk_threshold` | low/high 或预估 shard size 导致空间不足 | most/least available path、relocation accounting |
| `throttling` | initial/incoming/outgoing recovery 达上限 | initializing/recovery 计数；这是暂缓而非永久失败 |
| `shards_limit` | node 总 shard 或单 index shard 达上限 | owning shard count 与动态设置 |
| `awareness` | 属性缺失或目标 value 上 copy 过多 | actual/forced values、replica 数、ceil 上限 |

如果 explain 只显示一个 `NO`，不能推断其他决策器一定为 `YES`：普通非 debug 路径会在第一个 `NO` 短路，而 explain 是否包含全部结果取决于 `RoutingAllocation.DebugMode`。

## 29. 关键测试入口

| 主题 | 测试源码 |
| --- | --- |
| 聚合、debug 与 forced node 交集 | [`AllocationDecidersTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/decider/AllocationDecidersTests.java) |
| retry 上限 | [`MaxRetryAllocationDeciderTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/MaxRetryAllocationDeciderTests.java) |
| resize | [`ResizeAllocationDeciderTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/ResizeAllocationDeciderTests.java) |
| rebalance group active | [`RebalanceOnlyWhenActiveAllocationDeciderTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/decider/RebalanceOnlyWhenActiveAllocationDeciderTests.java) |
| cluster rebalance | [`ClusterRebalanceRoutingTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/ClusterRebalanceRoutingTests.java) |
| concurrent rebalance | [`ConcurrentRebalanceRoutingTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/ConcurrentRebalanceRoutingTests.java) |
| enable 与 DiDi short-circuit | [`EnableAllocationTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/decider/EnableAllocationTests.java)、[`EnableAllocationShortCircuitTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/decider/EnableAllocationShortCircuitTests.java) |
| node version | [`NodeVersionAllocationDeciderTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/NodeVersionAllocationDeciderTests.java) |
| snapshot/restore | [`SnapshotInProgressAllocationDeciderTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/decider/SnapshotInProgressAllocationDeciderTests.java)、[`RestoreInProgressAllocationDeciderTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/decider/RestoreInProgressAllocationDeciderTests.java) |
| shutdown/replacement | [`NodeShutdownAllocationDeciderTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/decider/NodeShutdownAllocationDeciderTests.java)、[`NodeReplacementAllocationDeciderTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/decider/NodeReplacementAllocationDeciderTests.java) |
| filter/same shard | [`FilterAllocationDeciderTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/decider/FilterAllocationDeciderTests.java)、[`SameShardRoutingTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/SameShardRoutingTests.java) |
| disk | [`DiskThresholdDeciderUnitTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/decider/DiskThresholdDeciderUnitTests.java)、[`DiskThresholdDeciderTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/decider/DiskThresholdDeciderTests.java) |
| recovery throttling | [`ThrottlingAllocationTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/ThrottlingAllocationTests.java) |
| shard limit | [`ShardsLimitAllocationTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/ShardsLimitAllocationTests.java) |
| awareness | [`AwarenessAllocationTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/AwarenessAllocationTests.java) |

## 30. 最终心智模型

可以把一次分配理解为如下布尔表达式，但其中还存在 `THROTTLE` 第三态：

~~~text
可分配 =
    未超过失败重试上限
AND 恢复拓扑成立
AND allocation/rebalance 开关允许
AND 版本与 snapshot/restore 生命周期允许
AND 节点生命周期允许
AND 属性过滤通过
AND 不与相同 shard copy 冲突
AND 磁盘允许
AND recovery/rebalance 并发有额度
AND shard 数量未超限
AND awareness 故障域容量允许
~~~

其中：

- `NO` 表示当前候选违反至少一个硬条件；
- `THROTTLE` 表示条件有望随正在执行的 recovery、relocation 或 snapshot 完成而自动解除；
- `YES` 只表示 policy 允许，allocator 仍可能因为权重、desired assignment、gateway copy 质量或阶段优先级而不选择该节点。

最后一点最重要：`AllocationDecider` 决定的是“能不能”，`BalancedShardsAllocator`/`DesiredBalanceReconciler` 决定的是“在所有能放的位置中，这一轮实际放哪里”。
