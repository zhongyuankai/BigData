# `BalancedShardsAllocator.allocate` 源码跟踪

> 仓库：`elasticsearch-didi`
>
> 基线提交：`db41ec0c4e78`
>
> 用户输入名称：`BalanceShardsAllocator.allocate`
>
> 实际目标方法：`BalancedShardsAllocator#allocate(RoutingAllocation)`
>
> 目标源码：[`BalancedShardsAllocator.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L180-L198)

## 1. 结论先行

`BalancedShardsAllocator.allocate()` 本身只有 19 行。它的职责不是在这 19 行里完成所有算法，而是：

1. 确认普通 allocation 阶段已经恢复 `ignoreDisable=false`。
2. 在没有 data node 时，把新 primary 的最后分配状态改成 `DECIDERS_NO`，然后结束。
3. 从本次调用读取到的四个动态 balance factor 构造归一化的 `WeightFunction`。
4. 基于当前 `RoutingAllocation` 构造一次性的 `Balancer` 和内存模型。
5. 严格按以下优先级执行三个阶段：

~~~text
allocateUnassigned()
  -> moveShards()
  -> balance()
~~~

三个阶段的语义分别是：

| 阶段 | 解决的问题 | 是否受 balance threshold 限制 |
| --- | --- | --- |
| `allocateUnassigned()` | shard 还没有节点 | 否 |
| `moveShards()` | shard 已在节点上，但 `canRemain == NO` | 否 |
| `balance()` | shard 可以继续留在原节点，但移动后权重更好 | 是 |

其中第二行“否”表示强制迁移只受 allocation decider 和 throttling 等约束，不要求移动能改善权重；权重只用于在允许的目标节点中优先选择较轻的节点。

同一次 `allocate()` 只创建一个 `Balancer`。前一个阶段对模型的修改会立即成为下一个阶段的输入：

~~~text
当前 routing state
  -> 构建 ModelNode / ModelIndex
  -> 分配 unassigned，并更新模型
  -> 搬走 cannot-remain shard，并更新模型
  -> 在更新后的模型上做可选 rebalance
~~~

## 2. 类名与分析边界

仓库中不存在 `BalanceShardsAllocator`，实际类名多一个 `d`：

~~~text
BalancedShardsAllocator
        ^
~~~

本文跟踪的是：

~~~java
public void allocate(RoutingAllocation allocation)
~~~

不把下面两个入口当成同一个方法：

- `ShardsAllocator#allocate(RoutingAllocation, ActionListener<Void>)`：带 listener 的接口默认方法。
- `BalancedShardsAllocator#decideShardAllocation(...)`：cluster allocation explain 使用的单 shard 决策入口。

不过这两个入口与目标方法共享算法，因此文中会在必要位置说明它们的关系。

## 3. 两种实际运行语境

当前仓库支持两种 allocator 类型：

| `cluster.routing.allocation.type` | `BalancedShardsAllocator` 的身份 | 修改的 routing state |
| --- | --- | --- |
| `balanced` | 顶层 allocator | 当前 master reroute 的真实可变 `RoutingNodes` |
| `desired_balance`，默认值 | `DesiredBalanceComputer` 的 delegate | generic 线程中的 simulation `RoutingNodes` |

创建逻辑位于 [`ClusterModule#createShardsAllocator`](../server/src/main/java/org/elasticsearch/cluster/ClusterModule.java#L379-L418)：

~~~java
allocators.put(
    BALANCED_ALLOCATOR,
    () -> new BalancedShardsAllocator(clusterSettings, writeLoadForecaster)
);

allocators.put(
    DESIRED_BALANCE_ALLOCATOR,
    () -> new DesiredBalanceShardsAllocator(
        clusterSettings,
        new BalancedShardsAllocator(clusterSettings, writeLoadForecaster),
        ...
    )
);
~~~

默认设置见 [`ClusterModule.java`](../server/src/main/java/org/elasticsearch/cluster/ClusterModule.java#L101-L109)：

~~~java
"cluster.routing.allocation.type" = "desired_balance"
~~~

所以生产默认路径不是由目标方法直接发布 cluster state，而是：

~~~text
DesiredBalanceComputer
  -> 在 simulation 上反复调用 BalancedShardsAllocator.allocate()
  -> 收集 DesiredBalance
  -> DesiredBalanceReconciler 在 master 线程推进真实 routing
~~~

只有显式选择旧 `balanced` allocator 时，目标方法才直接改变本轮 master reroute 的真实 `RoutingNodes`。

## 4. 总体时序

~~~mermaid
flowchart TD
    A["cluster state 变化或 reroute"] --> B["AllocationService"]
    B --> C["清理到期 delayed marker"]
    C --> D["ExistingShardsAllocator 优先处理已有副本"]
    D --> E{"allocator 类型"}

    E -->|"balanced"| F["BalancedShardsAllocator.allocate(real allocation)"]
    E -->|"desired_balance 默认"| G["DesiredBalanceShardsAllocator"]
    G --> H["generic 线程上的 DesiredBalanceComputer"]
    H --> I["BalancedShardsAllocator.allocate(simulation)"]

    F --> J["allocateUnassigned"]
    I --> J2["allocateUnassigned"]
    J --> K["moveShards"]
    J2 --> K2["moveShards"]
    K --> L["balance"]
    K2 --> L2["balance"]

    L --> M["本轮 RoutingNodes 变化"]
    L2 --> N["模拟 start shard 并继续迭代"]
    N --> O["DesiredBalance"]
    O --> P["DesiredBalanceReconciler"]
    P --> M
    M --> Q["AllocationService 构造 ClusterState"]
~~~

## 5. 从 `AllocationService` 进入

普通 reroute 通过：

~~~java
routingAllocation -> shardsAllocator.allocate(routingAllocation, listener)
~~~

调用 allocator。源码位置：

- [普通 reroute](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/AllocationService.java#L392-L397)
- [allocation commands 后 reroute](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/AllocationService.java#L363-L380)

进入 allocator 前，[`AllocationService#reroute`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/AllocationService.java#L523-L531) 已执行：

~~~java
rerouteStrategy.removeDelayMarkers(allocation);
allocateExistingUnassignedShards(allocation);
rerouteStrategy.execute(allocation);
~~~

因此目标方法看到的 unassigned shard 已经过 existing allocator：

- existing primary 可能已经从本地 store/gateway 恢复并进入 `INITIALIZING`。
- fetching、delayed 或仍由 existing allocator 负责的 shard 可能已经进入 ignored 列表。
- existing allocator 返回 `NOT_TAKEN` 的 shard 仍在普通 unassigned 列表中。

详细前置链路见 [`allocateExistingUnassignedShards()` 源码跟踪](allocate-existing-unassigned-shards-source-trace.md)。

### 5.1 `balanced` 模式的 listener

`BalancedShardsAllocator` 只重写单参数 `allocate()`，没有重写带 listener 的版本。因此会使用 [`ShardsAllocator` 默认实现](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/ShardsAllocator.java#L37-L51)：

~~~java
default void allocate(RoutingAllocation allocation, ActionListener<Void> listener) {
    allocate(allocation);
    listener.onResponse(null);
}
~~~

这表示 `balanced` 模式的 listener 在目标方法同步返回后立即成功，不等待 shard recovery 完成。

这与 `DesiredBalanceShardsAllocator` 的 listener 水位语义不同；后者需要等待异步 desired-balance 计算和 reconciliation task。

## 6. 目标方法完整源码

~~~java
@Override
public void allocate(RoutingAllocation allocation) {
    assert allocation.ignoreDisable() == false;

    if (allocation.routingNodes().size() == 0) {
        failAllocationOfNewPrimaries(allocation);
        return;
    }
    final WeightFunction weightFunction = new WeightFunction(
        indexBalanceFactor,
        shardBalanceFactor,
        writeLoadBalanceFactor,
        diskUsageBalanceFactor
    );
    final Balancer balancer = new Balancer(writeLoadForecaster, allocation, weightFunction, threshold);
    balancer.allocateUnassigned();
    balancer.moveShards();
    balancer.balance();
}
~~~

从控制流看，它只有一个 early return。只要存在 data node，就一定会依次调用三个阶段；每个阶段再根据当前状态自行快速返回。

## 7. `ignoreDisable` 断言

~~~java
assert allocation.ignoreDisable() == false;
~~~

显式 reroute command 执行时，[`ShardsAllocator#execute`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/ShardsAllocator.java#L56-L71) 会临时：

~~~java
allocation.ignoreDisable(true);
commands.execute(allocation, explain);
allocation.ignoreDisable(false);
~~~

手工 command 可以有意绕过 allocation disable 设置，但 command 之后的普通自动分配必须重新尊重：

- `cluster.routing.allocation.enable`
- `index.routing.allocation.enable`
- rebalance enable 设置

因此断言不是说 `BalancedShardsAllocator` 自己读取 disable flag；它确保后续 `AllocationDeciders` 看到的是正确的普通分配上下文。

## 8. 没有 data node 时的处理

~~~java
if (allocation.routingNodes().size() == 0) {
    failAllocationOfNewPrimaries(allocation);
    return;
}
~~~

[`failAllocationOfNewPrimaries()`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L223-L249) 遍历 unassigned shard，只处理：

~~~text
primary == true
AND lastAllocationStatus == NO_ATTEMPT
~~~

它保留 `UnassignedInfo` 中的：

- reason、message、failure
- failed allocation 次数
- unassigned 时间
- delayed 标记
- failed node IDs
- last allocated node ID
- recovery source

只把最后状态改成：

~~~text
NO_ATTEMPT -> DECIDERS_NO
~~~

目的不是伪造一次具体 decider 的 `NO`，而是在确实没有任何 data node 时让新 primary 表现为分配失败，从而让 cluster health 正确体现为 red。

replica 和已经有其他最后状态的 primary 不在这里修改。

## 9. 动态设置快照

外层 allocator 持有五个动态字段：

| 设置 | 当前默认值 | 用途 |
| --- | ---: | --- |
| `cluster.routing.allocation.balance.index` | `0.55` | 同一 index 在节点间的 shard 数均衡 |
| `cluster.routing.allocation.balance.shard` | `0.45` | 全集群 shard 总数均衡 |
| `cluster.routing.allocation.balance.write_load` | `0.0` | forecast write load 均衡 |
| `cluster.routing.allocation.balance.disk_usage` | `2e-20` | forecast/actual shard bytes 均衡 |
| `cluster.routing.allocation.balance.threshold` | `1.0` | 可选 rebalance 的收益门槛 |

定义见 [`BalancedShardsAllocator.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L90-L130)。

构造函数通过 `ClusterSettings.initializeAndWatch()` 注册动态更新。每次 `allocate()`：

1. 读取当时可见的四个 volatile factor。
2. 创建新的不可变 `WeightFunction`。
3. 读取当时可见的 threshold，传给新的 `Balancer`。

所以一次调用内部使用固定权重，但下一次 reroute 可以使用刚更新的动态设置。

### 9.1 threshold 的有效下限

设置定义允许输入 `0.0` 以上的值，但 [`ensureValidThreshold()`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L156-L178) 会：

- `threshold >= 1.0`：原样使用。
- `threshold < 1.0`：记录 critical deprecation warning，并按 `1.0` 使用。

因此当前版本的实际 threshold 不会小于 `1.0`。

### 9.2 factor 不能全部为零

`WeightFunction` 构造时计算：

~~~java
sum = index + shard + writeLoad + diskUsage;
~~~

如果 `sum <= 0`，会抛出：

~~~text
IllegalArgumentException:
Balance factors must sum to a value > 0
~~~

每个单独设置都允许为零，但组合起来至少一个必须大于零。

## 10. 权重公式

[`WeightFunction`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L296-L325) 先把四个 factor 归一化：

~~~text
sum    = shardFactor + indexFactor + writeFactor + diskFactor
theta0 = shardFactor / sum
theta1 = indexFactor / sum
theta2 = writeFactor / sum
theta3 = diskFactor / sum
~~~

注意源码中的变量映射：

| 字段 | 对应维度 |
| --- | --- |
| `theta0` | global shard count |
| `theta1` | per-index shard count |
| `theta2` | write load |
| `theta3` | disk usage |

某个节点 `node` 针对 index `i` 的权重是：

~~~text
weight(node, i)
  = theta0 * (node.totalShards - cluster.avgShardsPerNode)
  + theta1 * (node.shards(i) - cluster.avgShardsPerNode(i))
  + theta2 * (node.writeLoad - cluster.avgWriteLoadPerNode)
  + theta3 * (node.diskBytes - cluster.avgDiskBytesPerNode)
~~~

权重越小，节点在当前 index 视角下越“轻”；权重越大，越“重”。

平均值是中心化常量。比较两个节点时，平均值会抵消，但保留它们可以让权重表达“高于或低于平均水平”的方向。

### 10.1 当前默认权重

当前默认值下：

~~~text
theta0 ≈ 0.45
theta1 ≈ 0.55
theta2 = 0
theta3 ≈ 2e-20
~~~

因此默认主导因素仍是：

1. 单 index shard 数。
2. 节点总 shard 数。

write load 默认不参与；disk bytes 有极小非零权重。调高 write/disk factor 后，它们才会显著改变节点排序。

## 11. `minWeightDelta` 与 threshold

threshold 不是直接与裸权重差 `1.0` 比较。每个 index 先计算“一份 shard 对权重的最小贡献”：

~~~text
minWeightDelta(index)
  = theta0
  + theta1
  + theta2 * forecastedWriteLoadPerShard(index)
  + theta3 * maxShardSizeBytes(index)
~~~

然后：

~~~text
localThreshold = minWeightDelta(index) * configuredThreshold
~~~

`maxShardSizeBytes(index)` 取以下信息的最大值：

- index forecast shard size。
- 每个 shard ID 的实际 primary size。
- 每个 shard ID 的实际 replica size。

使用最大 shard size 而不是平均 shard size，是为了避免同一 index 内 shard 大小差异很大时产生无意义的来回搬迁。对应源码：

- [`WeightFunction#minWeightDelta`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L322-L324)
- [`Balancer#maxShardSizeBytes`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L452-L465)

比较时还通过：

~~~java
delta <= threshold + 0.001f
~~~

吸收浮点误差，避免接近临界值的抖动。

## 12. 构造 `Balancer`

[`Balancer` 构造函数](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L330-L365) 保存：

~~~text
RoutingAllocation
RoutingNodes
Metadata
WeightFunction
threshold
WriteLoadForecaster
~~~

并一次性计算：

~~~text
avgShardsPerNode
avgWriteLoadPerNode
avgDiskUsageInBytesPerNode
~~~

其中：

~~~text
avgShardsPerNode
  = metadata 中全部 shard copies / routingNodes.size()

avgWriteLoadPerNode
  = 所有 index 的 forecasted per-shard write load * copy 数 / node 数

avgDiskUsageInBytesPerNode
  = 所有 index 的估算总 bytes / node 数
~~~

这些平均值在本次 `allocate()` 内不会随着模型搬迁重新计算，因为：

- node 数不变。
- shard 总 copy 数不变。
- 总 forecast write load 不变。
- 总估算 disk bytes 不变。

### 12.1 disk 总量如何估算

[`getIndexDiskUsageInBytes()`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L416-L438) 对每个 shard ID：

1. primary size 取 forecast 与实际 primary size 的较大值。
2. replica size 取 forecast 与实际 replica size 的较大值，再乘 replica 数。
3. 如果只有部分 shard 有 size 信息，用已知平均值外推到 index 的全部 copies。
4. 完全没有信息时返回 `0`。

## 13. 构建内存模型

[`buildModelFromAssigned()`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L1046-L1080) 为每个 data node 创建一个 `ModelNode`。

模型保存：

- 节点当前建模的 shard 总数。
- 按 index 分组的 `ModelIndex`。
- forecast write load 总和。
- forecast/actual disk bytes 总和。
- 指向真实或模拟 `RoutingNode` 的引用。

构建时遍历已分配 shard：

~~~java
if (shard.state() != RELOCATING) {
    node.addShard(shard);
}
~~~

为什么跳过 source `RELOCATING`：

~~~text
source: RELOCATING
target: INITIALIZING
~~~

同一次 relocation 在 routing table 中同时有 source 与 target。模型保留 target `INITIALIZING`、跳过 source `RELOCATING`，从而只计算一次未来占位，并把它算在目标节点上。

这意味着模型表达的是“已有 relocation 完成后的布局”，而不是把一份正在迁移的 shard 计数两次。

## 14. 三阶段为什么必须保持顺序

### 14.1 先 `allocateUnassigned()`

未分配 shard 直接影响数据可用性和健康状态，优先级最高。

分配成功后 `ModelNode.addShard()` 立即增加：

- node shard count
- per-index shard count
- write load
- disk bytes

后续强制迁移与 rebalance 会看到这些新占位，避免继续把节点当成空节点。

### 14.2 再 `moveShards()`

当前节点如果违反 filter、tier、shutdown、disk 等硬约束，必须先撤离。该阶段不要求搬迁能改善 balance 权重。

成功 relocation 后模型同步执行：

~~~text
sourceModel.removeShard()
targetModel.addShard()
~~~

### 14.3 最后 `balance()`

纯均衡属于优化，不应抢在恢复可用性或满足硬约束之前消耗 recovery 并发额度。

它使用前两个阶段已经更新过的模型，所以评估的是本轮已计划动作之后的布局。

## 15. `allocateUnassigned()` 的输入排序

[`allocateUnassigned()`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L1082-L1191) 在 unassigned 为空时直接返回。

否则先 `drain()` 全部普通 unassigned shard，再按以下规则排序：

1. primary 在 replica 前。
2. 同一 index 内按 shard ID 升序。
3. 不同 index 之间使用 `PriorityComparator`，考虑 index priority、创建时间和名称等顺序。

伪代码：

~~~text
if primary 不同:
    primary first
else if index 相同:
    smaller shard id first
else:
    PriorityComparator
~~~

这保证 primary 可用性优先，同时保持 index 级 allocation priority。

ignored shard 不在 `drain()` 的普通队列中，因此不会被该阶段越权重新处理。

## 16. `decideAllocateUnassigned()` 如何选节点

[`decideAllocateUnassigned()`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L1193-L1287) 的核心目标是：

~~~text
在 decider 允许或限流的节点中，选择当前 index 权重最小的节点
~~~

具体顺序如下。

### 16.1 shard 级快速判定

先调用：

~~~java
allocation.deciders().canAllocate(shard, allocation)
~~~

如果结果为 `NO` 且不是 explain 模式，说明无论哪个节点都不允许分配，直接返回：

~~~text
DECIDERS_NO
~~~

explain 模式不能提前返回，因为需要收集逐节点解释。

### 16.2 遍历节点

普通非 explain 模式下会跳过：

- 已经持有同一 shard copy 的节点。
- 当前权重已经高于已找到最小权重的节点。

其余节点调用：

~~~java
allocation.deciders().canAllocate(shard, routingNode, allocation)
~~~

只有 `YES` 或 `THROTTLE` 能成为候选，`NO` 不能。

### 16.3 最小权重与同权重 tie-break

权重更小的候选直接替换当前 `minNode`。

权重相同则按两级规则：

1. `YES` 优先于 `THROTTLE`。
2. decision 类型相同时，参考节点上该 index 的最高 primary shard ID，构造 primary/replica 的环形错位。

源码注释给出的目标形态是：

~~~text
1 2 0
2 0 1
0 1 2
~~~

这不是另一个 balance factor，而是完全同权重时的确定性/分散性 tie-break。

### 16.4 explain 模式

`allocation.debugDecision()==true` 时：

- 不使用“已有更轻节点”的权重短路。
- 收集每个节点的 decider 结果。
- 最后按节点权重生成从 1 开始的 weight ranking。

因此 explain 路径比普通路径做更多工作，但应给出相同的目标节点。对应测试：

[`testDecideShardAllocation`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocatorTests.java#L70-L101)。

## 17. `YES`、`THROTTLE`、`NO` 的不同落地

### 17.1 `YES`

`RoutingNodes.initializeShard()` 真正执行：

~~~text
UNASSIGNED -> INITIALIZING
~~~

并把返回的新 shard 加进目标 `ModelNode`。

### 17.2 `THROTTLE`

真实 shard 不初始化，而是：

~~~java
unassigned.ignoreShard(shard, DECIDERS_THROTTLED, ...)
~~~

但只要存在被选中的 throttle target，模型会先加入一个模拟 `INITIALIZING`：

~~~java
minNode.addShard(
    shard.initialize(minNodeId, null, expectedShardSize)
);
~~~

这是非常关键的“真实状态与规划模型分离”：

| 层面 | `THROTTLE` 结果 |
| --- | --- |
| 真实 `RoutingNodes` | shard 仍 unassigned，并进入 ignored |
| 本轮 `Balancer` 模型 | 目标节点已经为它预留一个占位 |

这样后续 shard 不会全部因为同一个节点当前最轻而堆向该节点。

### 17.3 `NO`

没有候选节点时，shard 进入 ignored，并记录 `DECIDERS_NO`。模型不增加占位。

## 18. 多 replica 的轮转公平性

`allocateUnassigned()` 使用 `primary` 和 `secondary` 两个数组。

当某个 replica 成功分配时，与它完全相同的额外 replica copies 暂时移动到 `secondary`，先让其他 shard 获得一次分配机会。

例如排序结果：

~~~text
(0,P,index-a)
(0,P,index-b)
(0,R,index-a)
(0,R,index-a)
(0,R,index-b)
(0,R,index-b)
~~~

第一份 `index-a[0]` replica 分配后，第二份相同 replica 被推迟到下一轮；当前轮继续 `index-b[0]` replica。

如果第一份 replica 根本找不到节点，则后续完全相同的 replica 一起进入 ignored，因为相同约束下继续逐份扫描不会产生更好结果。

这个机制控制的是同一轮分配公平性，不改变 primary-first 和 index priority。

## 19. `moveShards()`：强制迁移阶段

[`moveShards()`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L892-L937) 处理已经 `STARTED`、但不能继续留在当前节点的 shard。

它按节点交错遍历：

~~~text
node-a 的第 1 个 shard
node-b 的第 1 个 shard
node-c 的第 1 个 shard
node-a 的第 2 个 shard
...
~~~

这样在 recovery throttling 存在时，不会总从某一个需要卸载的节点开始，把该节点的迁移额度先耗尽。

当前 DiDi 分支调用的是：

~~~java
regionNodeInterleavedShardIterator(
    nodeIdsByRegion,
    EnableAllocationDecider.isEnableMasterEnhance()
)
~~~

master-enhance 关闭时，iterator 仍遍历全部节点；开启时只遍历本轮选中 region 与 default region 的节点。region 细节见第 27 节。

该阶段维护一个 identity `throttledNodes` 集合，用于：

- 跳过本轮已知限流的目标节点。
- master-enhance 开启时判断当前 region 的节点是否全部限流，从而提前结束剩余 shard 扫描。

## 20. `decideMove()`：必须搬走时如何选目标

[`decideMove()`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L939-L1035) 的流程为：

~~~text
不是 STARTED
  -> NOT_TAKEN

canRemain != NO
  -> stay

canRemain == NO
  -> 按当前 index 权重从轻到重排序节点
  -> 找 canAllocate 的最佳目标
  -> 普通目标没有成功且 source 是 REPLACE shutdown
  -> 再尝试 canForceAllocateDuringReplace
~~~

### 20.1 `canRemain != NO` 为什么立即留下

`moveShards()` 只处理硬约束要求撤离的 shard。

即使其他节点权重更轻，只要 `canRemain` 不是 `NO`，也不会在这一阶段搬迁；它应留给最后的 `balance()` 判断收益和 threshold。

### 20.2 目标节点顺序

`sorter.reset(index)` 会重新计算当前 sorter 范围内每个节点对该 index 的权重，并按权重升序排列。

随后依次检查候选节点。decision 优先级为：

~~~text
YES > THROTTLE > NO
~~~

普通非 explain 模式下，找到第一个 `YES` 就停止，因为节点已经按权重从轻到重排序。

### 20.3 same-shard 与 throttled node

master-enhance 开启且不是 explain 模式时，代码会直接跳过：

~~~text
throttledNodes.contains(target)
OR target.containsShard(shard)
~~~

关闭 master-enhance 时，同 shard 约束主要由 `SameShardAllocationDecider` 返回 `NO`。

遇到 `THROTTLE` 时目标节点加入 `throttledNodes`，但 `MoveDecision` 只有得到 `YES` 时才带可立即执行的 target。

## 21. 强制 relocation 的真实状态变化

如果 `MoveDecision.forceMove()==true`，`moveShards()` 先更新模型：

~~~text
sourceModel.removeShard(shard)
~~~

然后调用：

~~~java
routingNodes.relocateShard(
    shard,
    targetNodeId,
    expectedShardSize,
    allocation.changes()
)
~~~

真实 routing 形成一对关联状态：

~~~text
source shard: STARTED -> RELOCATING
target shard:            INITIALIZING
~~~

最后把 target initializing shard 加入目标 `ModelNode`。

`RoutingNodes.relocateShard()` 同时维护 source/target 的 relocation 关系和 `AllocationId`。这里的 allocation ID 用于区分具体 shard copy 的本次分配/迁移尝试，后续 started、failed 和 stale event 校验都依赖它。

如果普通 `canAllocate` 找不到目标，并且 source node 正以 `REPLACE` 类型 shutdown，则再用：

~~~java
canForceAllocateDuringReplace(...)
~~~

尝试替换节点专用的强制撤离路径。

这个阶段不检查 balance threshold，也不要求目标权重改善；“必须离开当前节点”优先于“移动后是否更均衡”。

## 22. `balance()` 的三道总开关

[`balance()`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L529-L557) 在进入权重算法前依次检查：

### 22.1 pending async fetch

~~~java
if (allocation.hasPendingAsyncFetch()) {
    return;
}
~~~

gateway/store 信息仍在拉取时做 rebalance，可能等 fetch 返回后又把 shard 搬回去，因此完全跳过纯均衡。

这只跳过第三阶段：

- unassigned allocation 已经执行。
- cannot-remain 强制迁移已经执行。

### 22.2 cluster 级 `canRebalance`

~~~java
allocation.deciders().canRebalance(allocation).type() == YES
~~~

只有严格 `YES` 才继续。`NO` 和 `THROTTLE` 都跳过本轮纯均衡。

### 22.3 至少两个模型节点

~~~java
nodes.size() >= 2
~~~

只有一个 data node 时不存在节点间 balance。

三道开关都通过后才进入 `balanceByWeights()`。

## 23. 先处理最不均衡的 index

[`buildWeightOrderedIndices()`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L815-L890) 对每个候选 index：

1. `sorter.reset(index)`，计算并排序节点权重。
2. 计算 `maxWeight - minWeight`。
3. 按 delta 从大到小排序 index。

先处理最不均衡的 index，可减少全局 shard factor 在中间状态中压过 per-index factor、导致刚搬出的 shard 又需要搬回的情况。

### 23.1 relevant nodes

进入某个 index 后，[`balanceByWeights()`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L682-L813) 先筛出：

~~~text
已经持有该 index shard 的节点
OR
index 级 canAllocate(indexMetadata, node) != NO 的节点
~~~

只有 relevant node 不少于两个才可能均衡。

把完全不相关且 index 级 decider 明确拒绝的节点排除，可以：

- 缩小排序和搜索范围。
- 避免不可能接收 shard 的极轻节点扭曲 min/max delta。

最终搬具体 shard 时仍会调用 shard 级 `canAllocate`，因此 relevant-node 筛选不是最终授权。

## 24. `balanceByWeights()` 的双端窗口

对 relevant nodes 排序后：

~~~text
modelNodes[0]                 modelNodes[high]
最轻节点                           最重节点
    ^ lowIdx                         ^ highIdx
~~~

局部门槛：

~~~text
localThreshold = minWeightDelta(index) * threshold
~~~

主循环尝试：

~~~text
从 maxNode 选择该 index 的一个 shard
  -> 搬到 minNode
~~~

### 24.1 已在 threshold 内

如果：

~~~text
maxWeight - minWeight <= localThreshold + 0.001
~~~

通常停止处理该 index。

代码还处理 zone awareness 等特殊情形：最重节点无法搬向当前最轻节点时，其他节点对仍可能超过 threshold，因此不会简单认定整个 index 已平衡。

### 24.2 当前 min/max 无法迁移

搜索窗口按两个方向收缩：

1. `lowIdx++`：换下一个较轻节点作为 target。
2. low 侧用尽后，`highIdx--`：换下一个较重节点作为 source。

直到：

- 找到可迁移 shard；或
- 所有有意义的 source/target 对都无法迁移；或
- delta 已经进入 threshold。

### 24.3 成功一次后重新排序

每次 `tryRelocateShard()` 返回 true：

1. 重新计算 source/target 权重。
2. 对 relevant range 重新排序。
3. 把 low/high 重置到两端。

因为一次搬迁会同时改变：

- 两个节点的总 shard 数。
- 两个节点的该 index shard 数。
- 两个节点的 write load。
- 两个节点的 disk bytes。

继续使用旧排序会得到错误的下一步。

## 25. `tryRelocateShard()`：选择具体 shard

[`tryRelocateShard()`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L1289-L1340) 只查看 max node 上当前 index 的 shard。

候选条件：

1. shard 必须 `STARTED`。
2. 按 shard ID 降序检查，保证确定性。
3. shard 级 `canRebalance` 不能是 `NO`。
4. 对 min node 的 `canAllocate` 不能是 `NO`。

把两个 decision 合并：

~~~java
new Decision.Multi()
    .add(allocationDecision)
    .add(rebalanceDecision)
~~~

### 25.1 `YES`

真实 `RoutingNodes` 执行 relocation，同时模型把一个 shard 从 max node 移到 min node。

### 25.2 `THROTTLE`

真实 `RoutingNodes` 不执行 relocation，但模型仍：

~~~text
maxModel.removeShard()
minModel.addShard(simulated relocation)
~~~

这与 unassigned 阶段的 throttle 预占位相同：本轮模型按“该受限动作未来会发生”继续规划，避免反复为同一最优位置计划多份 shard。

### 25.3 一次只移动一个候选

找到第一个 `YES` 或 `THROTTLE` 候选后立即返回 true，让调用方更新权重并重新排序，再决定下一步。

这比在旧权重下连续搬多个 shard 更保守。

## 26. `ModelNode`、`ModelIndex` 与 `NodeSorter`

### 26.1 `ModelNode`

[`ModelNode`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L1343-L1436) 是节点的轻量规划视图：

| 字段 | 含义 |
| --- | --- |
| `numShards` | 当前模型中的 shard copy 总数 |
| `writeLoad` | 每份 shard 的 forecast write load 之和 |
| `diskUsageInBytes` | 每份 shard 的 forecast/actual bytes 之和 |
| `indices` | index name 到 `ModelIndex` |
| `routingNode` | 对应的真实或 simulation `RoutingNode` |

`addShard/removeShard` 会同步更新全部统计量。

### 26.2 `ModelIndex`

[`ModelIndex`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L1438-L1482) 使用 `HashSet<ShardRouting>` 保存一个节点上的同 index shard，并缓存 `highestPrimary`。

`highestPrimary` 只用于 unassigned 同权重 tie-break；模型增删 shard 时缓存重置为 `-1`。

### 26.3 `NodeSorter`

[`NodeSorter`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L1484-L1553) 同时维护：

~~~text
ModelNode[] modelNodes
float[] weights
~~~

`reset(index)` 会：

1. 针对指定 index 重算每个节点权重。
2. 用 Lucene `IntroSorter` 同步排序 node 与 weight 数组。
3. 保证 `weights[0]` 最小、最后一个最大。

它不是跨调用缓存。每个 `allocate()` 都随新的 `Balancer` 重建。

## 27. DiDi region/master-enhance 默认路径

当前分支增加动态设置：

~~~text
cluster.routing.master_enhance.enable = true
~~~

定义见 [`EnableAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/EnableAllocationDecider.java#L83-L114)。当前默认值是 `true`，因此以下逻辑默认生效。

### 27.1 节点如何分 region

`buildModelFromAssigned()` 读取 node attribute：

~~~text
region
~~~

没有该 attribute 的节点归入：

~~~text
default
~~~

常量定义见 [`IndexMetadata.java`](../server/src/main/java/org/elasticsearch/cluster/metadata/IndexMetadata.java#L213-L214)。

### 27.2 每次 `Balancer` 选择一个 region

`randomRegionModelNode()`：

- 没有 region：返回空列表和 `default`。
- 只有一个 region：使用该 region 全部节点。
- 多个 region：随机选择一个非 `default` region，并把 `default` 节点一并加入。

选中的节点列表用来构造 `NodeSorter`。

所以 master-enhance 开启时，一次 `Balancer` 不在所有 region 间同时做强制搬迁和纯均衡，而是缩小到：

~~~text
一个随机非 default region + default nodes
~~~

### 27.3 index 如何与 region 对应

`buildWeightOrderedIndices()` 读取：

~~~text
index.routing.allocation.include.region
~~~

兼容规则：

- index 未设置 region include：本轮仍处理。
- 当前选择的是 `default`：本轮处理全部 index。
- index 设置了 region include：只有包含当前选中 region 才进入 balance。

### 27.4 对三个阶段的实际影响

| 阶段 | region 是否缩小范围 |
| --- | --- |
| `allocateUnassigned()` | 它遍历 `nodes.values()`，仍考虑全部节点；最终范围主要由 allocation decider 限制 |
| `moveShards()` | source shard iterator 和按权重选择的 target 都限制在选中 region/default 节点 |
| `balance()` | index 列表和 sorter 节点都限制在选中 region/default 范围 |

`avgShardsPerNode`、avg write load、avg disk bytes 仍按全部 routing nodes 计算，而 sorter 可以只包含 region 子集。

### 27.5 限流提前退出

当：

~~~text
throttledNodes.size() >= nodeIdsByRegion.size()
~~~

`moveShards()` 认为 region 内所有节点都受限，提前停止扫描剩余 shard，减少 master CPU 消耗。

### 27.6 一个需要留意的实现细节

[`RoutingNodes#selectNodeIDByRegion`](../server/src/main/java/org/elasticsearch/cluster/routing/RoutingNodes.java#L1389-L1399) 直接取得 `nodesPerRegionNames` 中的可变 list，再把 default 节点 `addAll()` 进去。

因此在同一个 `RoutingNodes` 上重复调用时，非 default region 的内部 list 可能累积重复 default node ID。当前 iterator 用 `contains()` 判断范围，所以遍历集合语义不变；但提前退出条件使用 list `size()`，重复项可能使“全部节点限流”的判定推迟。

这是一项基于当前源码的实现观察，不影响本文对主流程的描述，但排查 desired-balance 多轮 simulation 的限流行为时值得检查。

## 28. 直接模式与 desired-balance delegate 模式

### 28.1 直接 `balanced` 模式

~~~mermaid
sequenceDiagram
    participant M as "master update thread"
    participant A as "AllocationService"
    participant B as "BalancedShardsAllocator"
    participant R as "real RoutingNodes"

    M->>A: "reroute(clusterState)"
    A->>R: "existing allocators"
    A->>B: "allocate(real RoutingAllocation)"
    B->>R: "initialize / relocate"
    B-->>A: "同步返回"
    A-->>M: "构造新 ClusterState"
    M->>M: "listener success"
~~~

特点：

- 目标方法通常运行在 master cluster-state update 上下文。
- 一次调用只执行一遍三个阶段。
- `YES` 决策立即修改真实 routing。
- listener 在方法返回后完成，但 recovery 仍是异步的。

### 28.2 默认 `desired_balance` 模式

~~~mermaid
sequenceDiagram
    participant G as "generic computation thread"
    participant C as "DesiredBalanceComputer"
    participant B as "BalancedShardsAllocator"
    participant S as "simulation RoutingNodes"

    G->>C: "compute(input snapshot)"
    loop "直到 simulation 无新 INITIALIZING 或 input stale"
        C->>B: "allocate(simulation)"
        B->>S: "allocate / move / balance"
        C->>S: "把 INITIALIZING 模拟为 STARTED"
        C->>S: "reset 可重试 ignored shards"
    end
    C-->>G: "DesiredBalance assignments"
~~~

相关循环位于 [`DesiredBalanceComputer.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/DesiredBalanceComputer.java#L257-L310)。

特点：

- 目标方法运行在 generic 线程。
- `RoutingAllocation.isSimulating()==true`。
- 真实 cluster state 不在这里改变。
- `DesiredBalanceComputer` 会把 initializing shard 模拟 start，再次调用 delegate，直到长期布局收敛。
- 每次 delegate 调用都会创建新的 `Balancer`；master-enhance 开启时也会重新选择 region。

最终 assignment 由 `DesiredBalanceReconciler` 回到 master 线程增量落地。

### 28.3 同一个 `THROTTLE` 的两层意义

| 场景 | `BalancedShardsAllocator` 内部 | 外层结果 |
| --- | --- | --- |
| 直接模式 | 模型预占位，真实 routing 暂不动作 | 等下一次真实 reroute |
| desired simulation | 模型预占位，simulation routing 暂不动作 | computer 后续 reset ignored 并再次迭代 |

## 29. `AllocationDecider` 调用地图

| 位置 | decider 调用 | 决定什么 |
| --- | --- | --- |
| unassigned 快速判断 | `canAllocate(shard, allocation)` | 是否所有节点都不可能 |
| unassigned 选节点 | `canAllocate(shard, node, allocation)` | 具体节点能否接收 |
| 强制迁移起点 | `canRemain(shard, currentNode, allocation)` | 是否必须离开 |
| 强制迁移目标 | `canAllocate(shard, target, allocation)` | 普通撤离目标 |
| replace fallback | `canForceAllocateDuringReplace(...)` | 替换节点专用目标 |
| balance 总开关 | `canRebalance(allocation)` | 本轮是否允许任何 rebalance |
| relevant node | `canAllocate(indexMetadata, node, allocation)` | index 是否原则上能使用该节点 |
| 具体 balance shard | `canRebalance(shard, allocation)` | 该 shard 是否可重平衡 |
| balance target | `canAllocate(shard, node, allocation)` | 该 shard 是否可落到目标 |

几个重要边界：

- threshold 不能绕过 decider。
- `moveShards()` 不调用 `canRebalance`，因为硬约束撤离不是普通 rebalance。
- `THROTTLE` 可以进入规划模型，但不会触发真实 initialize/relocate。
- explain 模式保留更多 node-level decision，不使用大部分短路。

在 simulation 中，[`EnableAllocationDecider`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/EnableAllocationDecider.java#L116-L213) 对 shard allocation 和 cluster 级 rebalance 有专门放行逻辑；但其他 decider 以及 shard 级 rebalance 判断仍会参与。真实 reconciliation 时会重新使用当时的 decider 结果。

## 30. routing 状态与模型状态对照

| 操作 | 真实/模拟 `RoutingNodes` | `Balancer` 模型 |
| --- | --- | --- |
| unassigned `YES` | `UNASSIGNED -> INITIALIZING` | target 增加 shard |
| unassigned `THROTTLE` | shard 进入 ignored | target 增加模拟 shard |
| cannot-remain `YES` | source `RELOCATING` + target `INITIALIZING` | source 减少、target 增加 |
| cannot-remain 无目标 | 不变 | 不变 |
| balance `YES` | source `RELOCATING` + target `INITIALIZING` | source 减少、target 增加 |
| balance `THROTTLE` | 真实 routing 不变 | source 减少、target 增加模拟 shard |

这里的模型不是最终输出对象，也不会跨 `allocate()` 保存。它只保证本次调用后续决策能看见已经执行或已经被 throttle 预留的动作。

所有真实 routing 修改都把 `allocation.changes()` 作为 observer 传下去，供 `AllocationService` 判断 routing 是否变化并构建新的 routing table。

## 31. 线程、生命周期与并发

### 31.1 外层 allocator

五个动态设置字段是 volatile，cluster settings 更新后对后续调用可见。

`writeLoadForecaster` 是 final。

外层类不保存某次 `Balancer` 的模型。

### 31.2 单次 `Balancer`

`Balancer`、`ModelNode`、`ModelIndex` 和 `NodeSorter` 都是单次调用对象，不设计为并发共享：

~~~text
allocate() 开始 -> 创建 -> 三阶段修改 -> allocate() 返回 -> 丢弃
~~~

### 31.3 调用线程取决于模式

| 模式 | 常见线程 |
| --- | --- |
| 顶层 `balanced` | master update/test thread |
| desired-balance delegate | generic computation thread |

目标方法自身不创建线程、future 或 callback。它是同步 CPU 计算；异步性来自外层 `DesiredBalanceShardsAllocator`。

## 32. 与 existing allocator 的职责边界

目标方法不是 existing shard recovery 的第一决策者。

每次普通 reroute 都先调用 `allocateExistingUnassignedShards()`。交接结果：

| existing allocator 结果 | `BalancedShardsAllocator` 看到的状态 |
| --- | --- |
| 已分配 | `INITIALIZING`，构建模型时计入目标节点 |
| fetching/delayed/仍负责 | ignored，不进入 `allocateUnassigned()` |
| `NOT_TAKEN` | 普通 unassigned，由目标方法处理 |

在 desired-balance 模式中，`DesiredBalanceInput` 还会复制 existing allocator 的 ignored shards，`DesiredBalanceComputer` 在每轮 reset ignored 后重新保护这些超出 delegate 职责的 shard。

## 33. 复杂度与热点

设：

~~~text
N = data node 数
U = 本轮普通 unassigned shard copy 数
A = assigned shard copy 数
I = index 数
M = canRemain == NO 的 started shard 数
~~~

### 33.1 模型构建

遍历节点和 assigned shard：

~~~text
O(N + A)
~~~

### 33.2 unassigned

~~~text
排序: O(U log U)
选节点: 最坏 O(U * N * deciderCost)
~~~

同副本轮转不会让每个 shard 被重复无限扫描；每个 drained shard 最终要么初始化，要么 ignored。

### 33.3 强制迁移

所有 assigned shard 至少经过一次交错遍历。只有必须迁移的 shard 才需要按 index 重算/排序节点并扫描目标：

~~~text
约 O(A * canRemainCost
   + M * (N log N + N * canAllocateCost))
~~~

master-enhance 通过 region 子集和“全部节点限流”提前退出降低单次处理量。

### 33.4 纯均衡

初始 index delta 排序约包含：

~~~text
I 次 node 权重计算与排序
~~~

每次成功规划 relocation 后又会重排 relevant nodes。具体成本取决于：

- 每个 index 的 relevant node 数。
- threshold 允许的移动数量。
- decider 拒绝导致的 source/target 窗口扫描。
- max node 上该 index 的 started shard 数。

因此 `balanceByWeights()` 是主要 CPU 热点，也是默认 desired-balance 架构把它放到 generic 线程的核心原因。

## 34. 关键测试映射

### 34.1 入口与 explain 一致性

[`BalancedShardsAllocatorTests#testDecideShardAllocation`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocatorTests.java#L70-L101) 验证：

- explain 开关不改变目标节点。
- 单 shard decision 与完整 `allocate()` 的实际分配一致。

### 34.2 write load 与 disk

[`BalancedShardsAllocatorTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocatorTests.java#L103-L271) 包含：

- `testBalanceByForecastWriteLoad`
- `testBalanceByForecastDiskUsage`
- `testBalanceByActualDiskUsage`
- `testBalanceByActualAndForecastDiskUsage`

其中 `testBalanceByForecastWriteLoad` 在当前 fork 上标记：

~~~java
@AwaitsFix(bugUrl = "fork-skipped")
~~~

注释明确指出原因是 fork 增加了 region 感知均衡逻辑。这是当前 region 定制改变上游测试假设的直接证据。

### 34.3 disk 总量估算

[`testGetIndexDiskUsageInBytes`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocatorTests.java#L333-L415) 覆盖：

- 无 size 信息返回 0。
- forecast 优先覆盖更小的实际值。
- 没 forecast 时回退实际 cluster info。
- size 信息不完整时按平均值外推。
- forecast 与 actual 取较大值。

### 34.4 threshold

[`testThresholdLimit`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocatorTests.java#L417-L436) 验证小于 1 的值被钳制并产生 critical warning。

[`testShardSizeDiscrepancyWithinIndex`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocatorTests.java#L438-L478) 验证 max shard size 会把 threshold 提高到足以阻止无意义搬迁。

### 34.5 index/global shard factor

[`BalanceConfigurationTests#testIndexBalance`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/BalanceConfigurationTests.java#L46-L95) 单独启用 index factor。

[`BalanceConfigurationTests#testReplicaBalance`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/BalanceConfigurationTests.java#L97-L143) 单独启用 global shard factor。

[`testPersistedSettings`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/BalanceConfigurationTests.java#L282-L314) 验证动态 cluster settings 会更新 allocator 字段。

### 34.6 单 shard rebalance

[`BalancedSingleShardTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/BalancedSingleShardTests.java#L48-L264) 覆盖：

- 非 started shard 不 rebalance。
- pending async fetch 返回 `AWAITING_INFO`。
- cluster/shard `canRebalance` 拒绝或限流。
- target `canAllocate` 拒绝。
- threshold 未达到。
- 单 shard explain 与完整 balance step 选择一致。
- node weight ranking。

### 34.7 当前测试空白

当前测试源码中没有直接命中以下两个内部 helper 的独立测试：

- `RoutingNodes#regionNodeInterleavedShardIterator`
- `RoutingNodes#selectNodeIDByRegion`

同时 forecast write-load 测试因 fork region 行为被 `@AwaitsFix` 跳过。阅读或修改 master-enhance 路径时，不能只依赖 upstream balance 测试推断覆盖充分。

## 35. 排障观察点

### 35.1 unassigned shard 一直不分配

检查顺序：

1. shard 是否仍在普通 unassigned，还是已经被 existing allocator 放进 ignored。
2. shard 级 `canAllocate(shard, allocation)` 是否直接 `NO`。
3. 逐节点 `canAllocate` 是 `NO` 还是 `THROTTLE`。
4. `UnassignedInfo.lastAllocationStatus` 是 `DECIDERS_NO`、`DECIDERS_THROTTLED` 还是 `NO_ATTEMPT`。
5. 是否根本没有 data node。

### 35.2 shard 违反 filter 却没有搬走

检查：

- shard 是否 `STARTED`。
- `canRemain` 是否确实返回 `NO`。
- 所有目标是否 `NO` 或 `THROTTLE`。
- source 是否是 `REPLACE` shutdown，force-allocate fallback 是否也失败。
- master-enhance 当前选择的 region 是否包含 source。
- region 内是否因所有目标限流提前退出。

### 35.3 集群不做纯均衡

检查三道总开关：

~~~text
hasPendingAsyncFetch == false
canRebalance(allocation) == YES
nodes.size() >= 2
~~~

再检查：

- index 是否进入当前 region 的 index 列表。
- relevant node 是否不少于两个。
- `delta > minWeightDelta * threshold + 0.001`。
- 是否存在同时通过 shard `canRebalance` 和 target `canAllocate` 的 started shard。

### 35.4 submitted shard 都偏向同一节点

重点确认：

- 自定义 factor 是否让某一维度完全压过其他维度。
- write/disk forecast 是否缺失或量纲过大。
- factor 是否全部被设成 0。
- throttle target 的模型预占位是否存在。
- region sorter 是否只包含部分节点。

### 35.5 desired balance 计算很慢

观察：

- node、index、shard 总量。
- `DesiredBalanceComputer` 迭代次数。
- 每轮被 throttle 后 reset ignored 的数量。
- balance 每次 relocation 后重新排序的次数。
- 高频新 input 是否让旧计算持续 stale。

## 36. 源码索引

| 主题 | 源码 |
| --- | --- |
| 目标方法与全部内部类 | [`BalancedShardsAllocator.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java) |
| allocator 接口与 listener 默认实现 | [`ShardsAllocator.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/ShardsAllocator.java) |
| reroute 编排 | [`AllocationService.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/AllocationService.java) |
| allocator 创建与默认类型 | [`ClusterModule.java`](../server/src/main/java/org/elasticsearch/cluster/ClusterModule.java) |
| desired-balance 迭代调用 | [`DesiredBalanceComputer.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/DesiredBalanceComputer.java) |
| desired-balance 控制层 | [`DesiredBalanceShardsAllocator.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/DesiredBalanceShardsAllocator.java) |
| region iterator 与 node 分组 | [`RoutingNodes.java`](../server/src/main/java/org/elasticsearch/cluster/routing/RoutingNodes.java) |
| master-enhance 设置 | [`EnableAllocationDecider.java`](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/EnableAllocationDecider.java) |
| region 常量与 index include 设置 | [`IndexMetadata.java`](../server/src/main/java/org/elasticsearch/cluster/metadata/IndexMetadata.java) |
| allocator 主测试 | [`BalancedShardsAllocatorTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocatorTests.java) |
| factor 配置测试 | [`BalanceConfigurationTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/BalanceConfigurationTests.java) |
| 单 shard rebalance 测试 | [`BalancedSingleShardTests.java`](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/BalancedSingleShardTests.java) |
| existing allocator 前置链路 | [`allocateExistingUnassignedShards` 跟踪](allocate-existing-unassigned-shards-source-trace.md) |
| 全局 balance 架构 | [分片 Balance 源码跟踪](shard-balance-source-trace.md) |
| desired allocator 入口 | [`DesiredBalanceShardsAllocator.allocate` 跟踪](desired-balance-shards-allocator-allocate-source-trace.md) |

## 37. 一句话心智模型

`BalancedShardsAllocator.allocate()` 为当前 `RoutingAllocation` 建立一个一次性的权重模型，先把未分配 shard 放到最轻的合法节点，再把违反硬约束的 shard 搬走，最后只在收益超过 index 局部门槛时从重节点向轻节点做 rebalance；`THROTTLE` 动作只推进模型、不推进真实 routing，而默认 desired-balance 模式会在 generic 线程上反复调用这套同步算法来计算长期目标。
