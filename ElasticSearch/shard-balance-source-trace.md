# Elasticsearch 分片 Balance 源码跟踪

> 分析基线：`d781f5026b34`<br>
> 当前分支：`codex/uncompressed-stored-fields-codec-design`<br>
> 分析范围：`server` 中分片分配与均衡主链路，以及本仓库的 DiDi region 定制<br>
> 说明：文中的行号对应上述提交，后续代码变更可能造成偏移。

## 1. 结论先行

当前仓库默认使用的分片分配器是 `desired_balance`，不是直接使用 `balanced`：

1. `ClusterModule` 创建 `DesiredBalanceShardsAllocator`。
2. `DesiredBalanceShardsAllocator` 内部仍以 `BalancedShardsAllocator` 作为 delegate。
3. delegate 在后台的模拟 `RoutingAllocation` 上反复执行，计算完整的目标节点集合，也就是 `DesiredBalance`。
4. `DesiredBalanceReconciler` 回到 master 更新线程，根据最新的真实集群状态和实时 `AllocationDecider` 决策，逐步把真实路由表推进到目标布局。
5. 真实的 `RoutingNodes` 变更最终由 `AllocationService` 转成新的 `RoutingTable`、`Metadata` 和 `ClusterState`，再由调用它的 master task 发布。

因此，理解本仓库的 balance 必须同时跟踪两条链：

- **目标计算链**：`DesiredBalanceComputer -> BalancedShardsAllocator -> balanceByWeights`。
- **真实落地链**：`DesiredBalanceReconciler -> RoutingNodes -> AllocationService -> ClusterState`。

还有一个重要的 fork 差异：`cluster.routing.master_enhance.enable` 默认是 `true`。开启后，`BalancedShardsAllocator` 的强制搬迁和权重均衡只处理随机选择的一个 region，并把没有 region 属性的 `default` 节点并入该 region。这一优化改变了单次 reroute 的处理范围，也使部分上游均衡测试在本仓库被跳过。

## 2. 概念边界

源码中三个动作经常都被笼统称为“balance”，但语义不同：

| 动作 | 触发条件 | 主要方法 | 结果 |
| --- | --- | --- | --- |
| 分配未分配分片 | 分片处于 `UNASSIGNED` | `allocateUnassigned()` | `UNASSIGNED -> INITIALIZING` |
| 强制搬迁 | 当前节点不再允许保留该分片，`canRemain == NO` | `moveShards()` | `STARTED -> RELOCATING`，目标副本为 `INITIALIZING` |
| 纯均衡搬迁 | 分片可以继续留在当前节点，但放到目标节点可改善权重或符合 desired assignment | `balance()/balanceByWeights()` | 同样创建 relocation |

`BalancedShardsAllocator.allocate()` 和 `DesiredBalanceReconciler.run()` 都严格按照以下顺序执行：

1. 分配未分配分片；
2. 搬走不能继续留在当前节点的分片；
3. 再执行纯均衡搬迁。

这个顺序保证“恢复可用性”和“满足硬约束”的优先级高于追求更漂亮的权重分布。

## 3. 总体架构

~~~mermaid
flowchart TD
    A["集群状态变化 / reroute 请求"] --> B["AllocationService.reroute"]
    B --> C["构造可变 RoutingAllocation"]
    C --> D["ExistingShardsAllocator 先处理已有副本"]
    D --> E{"cluster.routing.allocation.type"}
    E -->|"balanced"| F["BalancedShardsAllocator 直接修改真实 RoutingNodes"]
    E -->|"desired_balance（默认）"| G["DesiredBalanceShardsAllocator"]
    G --> H["立即按旧的/current DesiredBalance 协调一步"]
    G --> I["generic 线程异步计算新 DesiredBalance"]
    I --> J["BalancedShardsAllocator 在模拟 RoutingNodes 上迭代"]
    J --> K["URGENT master task"]
    K --> L["DesiredBalanceReconciler 按新目标协调一步"]
    F --> M["RoutingTable.of"]
    H --> M
    L --> M
    M --> N["Metadata / RestoreInProgress 更新"]
    N --> O["新 ClusterState"]
~~~

默认模式的线程切换如下：

~~~mermaid
sequenceDiagram
    participant M as "MasterService 更新线程"
    participant A as "DesiredBalanceShardsAllocator"
    participant G as "generic 线程"
    participant B as "BalancedShardsAllocator"
    participant Q as "URGENT master task queue"
    participant R as "DesiredBalanceReconciler"

    M->>A: allocate(real RoutingAllocation, listener)
    A->>A: 生成递增 input index，登记 listener
    A->>G: onNewInput(immutable simulation snapshot)
    A->>R: reconcile(currentDesiredBalance, real allocation)
    A-->>M: 当前 reroute 返回一步真实变更

    G->>B: delegate.allocate(simulated allocation)
    B-->>G: 模拟分配/搬迁结果
    G->>G: start 所有模拟 INITIALIZING 分片并继续迭代
    G->>Q: submit latest DesiredBalance

    Q->>R: reconcile(latest DesiredBalance, fresh real allocation)
    R-->>Q: 修改真实 RoutingNodes
    Q-->>M: 返回并发布新 ClusterState
    M-->>A: publication success 后完成相应 listener
~~~

关键边界是：generic 线程只生成目标，不直接发布集群状态。所有真实路由写入仍经由 master 更新线程串行完成。

## 4. 分配器的创建与选择

入口位于 [ClusterModule.java](../server/src/main/java/org/elasticsearch/cluster/ClusterModule.java)：

- `ClusterModule:101-109` 定义 `balanced`、`desired_balance` 和 `cluster.routing.allocation.type`。
- 默认值为 `desired_balance`。
- 该设置是 node-scope，并被标记为 deprecated；源码中的 `@UpdateForV9` 注释说明后续版本只保留一种 allocator。

`createShardsAllocator()` 在 `ClusterModule:379-417` 完成实例化：

~~~text
balanced
  -> BalancedShardsAllocator

desired_balance
  -> DesiredBalanceShardsAllocator
       delegateAllocator -> BalancedShardsAllocator
~~~

所以两种模式共享同一套权重函数和 `balanceByWeights()`。差异不在“用什么权重”，而在：

| 维度 | `balanced` | `desired_balance` |
| --- | --- | --- |
| 计算位置 | master reroute 当前调用栈 | generic 后台线程 |
| 输入 | 当前真实 `RoutingAllocation` | 当前状态的模拟快照 |
| 目标 | 直接决定本次可以执行的动作 | 尽量计算完整目标布局 |
| 限流影响 | 当前调用直接受限 | 目标计算可模拟未来完成状态，真实协调仍受限 |
| 路由修改 | delegate 直接修改真实 `RoutingNodes` | `DesiredBalanceReconciler` 修改真实 `RoutingNodes` |
| 收敛方式 | 多次外部 reroute | 后台内部迭代求目标，再通过多次协调落地 |

## 5. 从 `AllocationService.reroute()` 进入

### 5.1 构造 `RoutingAllocation`

正常入口位于 [AllocationService.java](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/AllocationService.java) `:392-397`：

~~~text
AllocationService.reroute(clusterState, reason, listener)
  -> executeWithRoutingAllocation(...)
       -> shardsAllocator.j(routingAllocation, listener)
~~~

`executeWithRoutingAllocation()` 在 `:409-416` 执行：

1. `adaptAutoExpandReplicas()` 先同步 auto-expand replica 数量；
2. `createRoutingAllocation()` 创建本次可变计算上下文；
3. 调用私有 `reroute()`；
4. 如果既没有 auto-expand 变化，也没有 `RoutingNodes` 变化，则复用旧 `ClusterState`；
5. 否则调用 `buildResultAndLogHealthChange()`。

`createRoutingAllocation()` 在 `:619-627` 注入：

- `clusterState.mutableRoutingNodes()`；
- 全部 `AllocationDeciders`；
- 当前 `ClusterInfo`；
- snapshot shard size；
- 当前 monotonic time。

### 5.2 已有副本恢复先于普通 allocator

私有 `reroute()` 位于 `AllocationService:523-531`：

~~~text
removeDelayMarkers
  -> allocateExistingUnassignedShards
  -> selected ShardsAllocator
~~~

`allocateExistingUnassignedShards()` 在 `:534-560` 中先处理 primary，再调用 allocator hook，然后处理 replica。这里服务于 gateway/store 恢复语义：

- 已有 primary 的有效 shard copy 选择不由普通权重 allocator 抢先决定；
- replica 也先给 `ExistingShardsAllocator` 机会；
- 被 gateway allocator 标为 fetching、delayed 或 out-of-scope 的分片，会被后续 desired-balance 计算保留为 ignored，而不是错误地当成普通新分片。

这解释了为什么跟踪 balance 时不能从 `BalancedShardsAllocator.allocate()` 单独开始。

### 5.3 真实状态的物化

`buildResultAndLogHealthChange()` 位于 `AllocationService:156-180`：

1. `RoutingTable.of(oldVersion, newRoutingNodes)` 把可变模型转成不可变路由表；
2. `updateMetadataWithRoutingChanges()` 同步 in-sync allocation ID 等 metadata；
3. 校验新路由表与 metadata 一致；
4. 更新 `RestoreInProgress`；
5. 构造新 `ClusterState`。

这里返回的是新状态对象，真正的 cluster-state publication 由外层 master task 完成。

## 6. 默认 `desired_balance` 的异步控制层

源码位于 [DesiredBalanceShardsAllocator.java](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/DesiredBalanceShardsAllocator.java)。

### 6.1 核心状态

`:50-61` 的关键字段如下：

| 字段 | 作用 |
| --- | --- |
| `delegateAllocator` | 实际为 `BalancedShardsAllocator`，只负责模拟目标计算 |
| `desiredBalanceComputer` | 驱动多轮模拟直至收敛 |
| `desiredBalanceReconciler` | 把真实布局推进到目标布局 |
| `desiredBalanceComputation` | 单 worker、只保留最新输入的后台计算器 |
| `queue` | 按 input index 保存等待完成的 listener |
| `indexGenerator` | 从 `-1` 开始递增标识 reroute 输入 |
| `pendingDesiredBalanceMoves` | 暂存显式 move command，供下一次目标计算吸收 |
| `masterServiceTaskQueue` | 名为 `reconcile-desired-balance` 的 URGENT 队列 |
| `currentDesiredBalance` | 最新目标或中间目标，初始为 `DesiredBalance.INITIAL` |

### 6.2 一次 `allocate()` 做了什么

listener 版本的 `allocate()` 位于 `:178-194`：

1. 断言运行在 master update/test 线程；
2. 递增 input index；
3. 将 listener 放入 `PendingListenersQueue`；
4. 通过 `DesiredBalanceInput.create()` 快照当前 allocation；
5. 把快照交给 `ContinuousComputation.onNewInput()`；
6. 不等待后台结果，立刻对 `currentDesiredBalance` 执行一次真实协调。

第 6 步非常重要。真实集群可能因为 recovery throttle 尚未达到旧目标；每次普通 reroute 都继续向旧目标前进一步，而不是在新目标计算完成前完全停住。

无 listener 的同步 `allocate(RoutingAllocation)` 在 `:173-176` 直接抛出 `UnsupportedOperationException`，防止误走 legacy 同步调用方式。

### 6.3 `ContinuousComputation` 如何合并输入

源码位于 [ContinuousComputation.java](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/ContinuousComputation.java)：

- `onNewInput()` 使用 `AtomicReference.getAndSet()` 保存最新对象；
- 只有从 `null` 变为非空时才向 executor 提交 worker；
- `isFresh(input)` 使用对象身份判断当前 input 是否仍是最新输入；
- worker 结束时 CAS 清空 input；
- 如果 CAS 失败，说明计算期间又来了新输入，于是同一个 worker 再次入队。

效果是：

- 同一时刻最多一个后台计算线程；
- 尚未开始的旧输入可以被直接覆盖；
- 已开始的旧计算至少完成一轮模拟，产出中间目标后尽快切换到最新输入；
- burst reroute 不会为每个旧状态堆积一次完整计算。

### 6.4 新结果如何回到 master

`processInput()` 位于 `DesiredBalanceShardsAllocator:114-138`：

1. 以 `currentDesiredBalance` 作为初始目标；
2. 调用 `DesiredBalanceComputer.compute()`；
3. 即使输入已经 stale，也先保存它产生的中间 `DesiredBalance`，供下一轮计算复用；
4. 只有 input 仍 fresh 时才提交 reconciliation task。

`ReconcileDesiredBalanceExecutor` 位于 `:321-360`：

- 同一 batch 中选择 `lastConvergedIndex` 最大的任务；
- 在 batch 的最新真实 `ClusterState` 上重新创建 `RoutingAllocation`；
- 调用 `DesiredBalanceReconciler`；
- publication success 后才完成不大于该 `lastConvergedIndex` 的 listener；
- 其余被 supersede 的 task 标记成功但不重复协调。

因此，API listener 的“完成”不只是后台算完，而是一个 fresh、已收敛 input 对应的协调结果已经成功发布。

### 6.5 失去 master 时

`onNoLongerMaster()` 位于 `:290-300`，会：

- 把 index 重置为 `-1`；
- 把目标重置为 `INITIAL`；
- 以 not-master 失败所有 pending listener；
- 清空 move command；
- 清空 reconciliation ordering 和相关统计。

这避免旧 master 的目标或等待者泄漏到下一届 master 任期。

## 7. `DesiredBalanceComputer`：目标布局如何计算

源码位于 [DesiredBalanceComputer.java](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/DesiredBalanceComputer.java)，主方法是 `compute()`，位于 `:73-361`。

### 7.1 复制模拟上下文

第一步调用：

~~~text
desiredBalanceInput.routingAllocation().mutableCloneForSimulation()
~~~

之后所有 `initializeShard()`、`relocateShard()` 和 `startShard()` 都作用于模拟副本。`ClusterInfoSimulator` 同步模拟 shard start 后的磁盘占用。

如果没有 data node，直接返回当前 input index 对应的空 assignment。

### 7.2 假设正在恢复的分片最终成功

`:93-101` 遍历当前 `INITIALIZING` 分片：

1. 更新模拟 cluster info；
2. 调用 `RoutingNodes.startShard()` 把它们视为 `STARTED`。

目标计算关心的是恢复完成后的稳态，不会把当前瞬时 recovery 状态永久固化进 desired layout。

### 7.3 尊重 gateway allocator 的职责边界

`:103-128` 把 unassigned 和 assigned routing 按 `ShardId` 分组。

被 `ExistingShardsAllocator` 声明为 ignored 的 existing primary 不由 desired allocator 擅自接管；replica 也只有在 gateway allocator 放弃后才进入普通分配。仍然存活的 `lastAllocatedNodeId` 会被保留为候选起点。

### 7.4 以前一次 desired assignment 稳定本次计算

`:130-192` 先读取 `previousDesiredBalance`：

- 移除已经离开集群的目标节点；
- 加回 unassigned shard 的有效 `lastAllocatedNodeId`；
- 已经位于旧目标节点的 shard copy 原地保留；
- 多余的已分配 copy 优先搬到旧目标中的缺失节点；
- 仍有旧目标空位时，再初始化 unassigned copy。

目标是减少计算抖动：如果拓扑和约束没有迫使布局变化，新一轮不应该随意换一套等价节点。

模拟预热阶段对 target decision 的要求是“不是 `NO`”。也就是说，`THROTTLE` 不被当成永久禁止，因为计算的是未来稳态。

### 7.5 primary 先于 replica

`:194-232` 分两轮初始化：

1. 先 primary；
2. 再 replica；
3. 如果对应 primary 被标记为 out-of-scope，则不接管它的 replica。

每次初始化后立即在模拟模型中 `startShard()`。

### 7.6 吸收显式 move command

`:234-250` drain `pendingDesiredBalanceMoves`，把 `MoveAllocationCommand` 应用到模拟 allocation。

命令失败只记录 debug 日志，不会使整个目标计算失败。这样，显式 move 不只是一次真实路由操作，也能进入后续 desired target，避免普通均衡马上把它搬回去。

### 7.7 反复运行 delegate 直到没有新动作

核心循环位于 `:257-331`：

~~~text
repeat:
    如果不是首轮:
        reset ignored unassigned shards
        再次排除真正 out-of-scope 的 shards

    注入最新 simulated ClusterInfo
    BalancedShardsAllocator.allocate(simulated allocation)

    把本轮所有 INITIALIZING shard 立即模拟为 STARTED

    如果没有任何 INITIALIZING:
        收敛，退出

    如果 input 已经 stale:
        保留至少一轮的中间结果，退出并让最新 input 重算
~~~

为什么必须循环：一次 `BalancedShardsAllocator.allocate()` 创建的 recovery/relocation target 处于 `INITIALIZING`。只有模拟它完成后，下一轮才能基于新的稳态继续放 replica 或执行下一批搬迁。

### 7.8 `lastConvergedIndex` 的语义

循环结束后，`:334-361` 收集每个 `ShardId` 的：

- desired node ID 集合；
- copy 总数；
- unassigned 数量；
- ignored 数量。

若计算真正稳定，则：

~~~text
lastConvergedIndex = current input index
~~~

若因 stale input 或尚缺 decider 信息而只得到中间结果，则：

~~~text
lastConvergedIndex = previousDesiredBalance.lastConvergedIndex
~~~

因此它不是“最近执行过的 input”，而是“最近完整收敛并可用于完成 listener 的 input”。

## 8. `BalancedShardsAllocator` 的核心算法

源码位于 [BalancedShardsAllocator.java](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java)。

### 8.1 每次调用的固定顺序

`allocate()` 位于 `:180-198`：

~~~text
无节点 -> fail new primary allocation
有节点 -> 创建 WeightFunction
       -> 创建 Balancer/model
       -> allocateUnassigned()
       -> moveShards()
       -> balance()
~~~

若它是 `desired_balance` 的 delegate，这些操作发生在模拟 routing nodes；若直接选择 `balanced`，这些操作发生在真实 reroute allocation。

### 8.2 模型保存什么

`ModelNode` 位于 `:1343-1436`，为每个节点维护：

- 总 shard copy 数；
- 每个 index 的 shard copy 集合与数量；
- forecast write load；
- actual/forecast disk bytes；
- 对应的 `RoutingNode`。

`addShard()` 和 `removeShard()` 同时更新以上四类指标。`RELOCATING` source 在建模时被跳过，因为它已经有一个位于目标节点的 `INITIALIZING` shadow copy；这样不会把一次 relocation 双重计数。

`NodeSorter` 位于 `:1484-1552`：

- 对指定 index 重算每个节点的 weight；
- 按 weight 从小到大排序；
- 第一个是最轻节点，最后一个是最重节点；
- `delta()` 为最大 weight 减最小 weight。

## 9. 权重公式

### 9.1 动态设置与本仓库默认值

设置定义在 `BalancedShardsAllocator:90-124`：

| 设置 | 默认值 | 含义 |
| --- | ---: | --- |
| `cluster.routing.allocation.balance.index` | `0.55` | 同一 index 在节点间均匀分布 |
| `cluster.routing.allocation.balance.shard` | `0.45` | 全部 shard copy 的节点总量均匀分布 |
| `cluster.routing.allocation.balance.write_load` | `0.0` | forecast write load |
| `cluster.routing.allocation.balance.disk_usage` | `2e-20` | actual/forecast disk bytes |
| `cluster.routing.allocation.balance.threshold` | `1.0` | 触发搬迁所需的最小收益倍率 |

前四项会除以总和归一化。不能全部为 `0`，否则构造 `WeightFunction` 时抛出异常。

threshold 小于 `1.0` 时，`ensureValidThreshold()` 会记录 critical deprecation warning，并把实际值钳制到 `1.0`。

本仓库提交 `df896a16122` 把默认值：

~~~text
write_load: 10.0 -> 0.0
disk_usage: 2e-11 -> 2e-20
~~~

提交说明中的目标是避免扩容时大量 index shard 集中分配到一台机器。实际含义是默认更强调 shard count 与 per-index count；write load 默认完全不参与，disk bytes 仅以很小系数参与。

### 9.2 归一化公式

设原始因子总和为：

~~~text
S = shard + index + write_load + disk_usage
~~~

则：

~~~text
θ0 = shard / S
θ1 = index / S
θ2 = write_load / S
θ3 = disk_usage / S
~~~

对节点 `n` 和索引 `i`：

~~~text
weight(n, i)
  = θ0 * (nodeShardCount(n) - globalAvgShardCount)
  + θ1 * (indexShardCount(n, i) - avgShardCountForIndex(i))
  + θ2 * (nodeWriteLoad(n) - globalAvgWriteLoad)
  + θ3 * (nodeDiskBytes(n) - globalAvgDiskBytes)
~~~

weight 越小，节点对该 index 越“轻”。

这里所有项都减去平均值。减平均值不改变同一 index 下节点间的相对排序，但使 weight 更容易解释为相对负载。

### 9.3 单个 shard 的最小 weight 单位

`minWeightDelta()` 位于 `:322-324`：

~~~text
unit(i)
  = θ0
  + θ1
  + θ2 * forecastWriteLoadPerShard(i)
  + θ3 * maxShardSizeBytes(i)
~~~

本地阈值：

~~~text
localThreshold(i) = unit(i) * balance.threshold
~~~

移动一个 shard 后，source 减少一个 unit，target 增加一个 unit。若移动前两端差值为 `delta`，要让移动后的绝对差更小，至少需要 `delta > unit`。这就是默认 threshold 为 `1.0` 时“只做有净收益搬迁”的含义。

`lessThan()` 还额外加入 `0.001` 浮点容差：

~~~text
delta <= localThreshold + 0.001
~~~

满足该条件就认为无需搬迁，以避免极小的浮点误差造成反复 relocation。

### 9.4 默认值示例

忽略极小的 disk 因子，默认约为 `θ0=0.45`、`θ1=0.55`。

假设对 index A：

- 节点 X 的总 shard 数比平均值多 2，A 的 shard 数比平均值多 1；
- 节点 Y 的总 shard 数比平均值少 1，A 的 shard 数恰好等于平均值。

则：

~~~text
weight(X, A) = 0.45 * 2 + 0.55 * 1 = 1.45
weight(Y, A) = 0.45 * -1 + 0.55 * 0 = -0.45
delta = 1.90
unit(A) ≈ 0.45 + 0.55 = 1.00
~~~

`1.90 > 1.00 + 0.001`，所以权重条件允许尝试从 X 向 Y 搬一个 A 的 shard。是否真正执行还要通过 shard-level `canRebalance` 和 target-level `canAllocate`。

## 10. 未分配 shard 的选择

`allocateUnassigned()` 位于 `BalancedShardsAllocator:1086-1191`。

### 10.1 排序和公平性

unassigned shard 排序规则：

1. primary 在 replica 前；
2. 同一 index 按 shard ID；
3. index 之间使用 `PriorityComparator`。

算法使用 primary/secondary 两个数组。分配某个 replica 后，同一 `ShardId` 的其他等价 replica 被推迟到下一轮，使不同 shard set 交错分配，避免连续副本偏向相同的一组节点。

### 10.2 最小权重节点

`decideAllocateUnassigned()` 位于 `:1199-1287`：

1. 先执行 shard-level `canAllocate(shard, allocation)`；若为 `NO`，直接结束；
2. 遍历所有 model node；
3. 已有相同 shard copy 的节点跳过；
4. 计算该节点针对当前 index 的 weight；
5. 调用 target-level `canAllocate(shard, node, allocation)`；
6. 在 `YES` 或 `THROTTLE` 节点中选择 weight 最小者。

相同 weight 时：

1. `YES` 优先于 `THROTTLE`；
2. 再用节点上该 index 的最高 primary shard ID 做环形 tie-break，使新 index 的 primary/replica 形成更均匀的交错布局。

### 10.3 `THROTTLE` 的模型语义

若最优节点返回 `YES`：

- 调用 `RoutingNodes.initializeShard()`；
- 真实/模拟 routing nodes 都出现 `INITIALIZING`；
- model node 加入该 shard。

若最优节点返回 `THROTTLE`：

- 当前真实 shard 仍放入 ignored unassigned；
- 但 `BalancedShardsAllocator` 会把一个只存在于 model 的 initialized shard 加到该节点；
- 后面的权重计算会把这份“未来即将分配的负载”算进去。

这避免限流时所有后续 shard 都错误地选择同一个暂时最轻节点。

需要区分“model 预测”和“最终 desired assignment”：`DesiredBalanceComputer` 最后从模拟 `RoutingNodes` 收集 assignment，而不是从 `ModelNode` 收集。因此，仅存在于 model 中的 `THROTTLE` 预测不会自动变成持久的 desired node；它主要影响当前 delegate pass 的后续选择。只有已经进入模拟 routing nodes 的布局才会出现在 `DesiredBalance` 中。

## 11. 强制搬迁 `moveShards()`

`moveShards()` 位于 `BalancedShardsAllocator:902-937`。

它按节点交错遍历 `STARTED` shard，先调用：

~~~text
canRemain(shard, currentNode, allocation)
~~~

- 非 `NO`：继续保留；
- `NO`：必须寻找目标。

候选节点按当前 index weight 从小到大检查，选第一个 `canAllocate == YES` 的最低权重节点。`THROTTLE` 会被记录，用于 fork 的提前退出优化，但不会真正启动 relocation。

若源节点处于 replacement shutdown，普通 `canAllocate` 找不到目标时，还会尝试 `canForceAllocateDuringReplace`，用于把 shard 强制迁往 replacement node。

强制搬迁不要求它改善权重，因为硬约束已经不允许 shard 留在当前节点。

## 12. 纯均衡 `balanceByWeights()`

### 12.1 进入条件

`balance()` 位于 `BalancedShardsAllocator:533-557`。以下任一条件成立时直接跳过：

- 仍有异步 shard/store fetch；
- 全局 `AllocationDeciders.canRebalance(allocation)` 不是 `YES`；
- 全部 model node 少于 2 个。

等待 store fetch 是为了避免信息回来后又把刚移动的 shard 搬回去。

region 模式下这里检查的是完整 `nodes` map，不是 region sorter 的长度；某个 index 在所选 region 中实际少于两个 relevant node 时，会在 `balanceByWeights()` 内被跳过。

### 12.2 index 处理顺序

`buildWeightOrderedIndices()` 位于 `:828-890`：

1. 对每个 index 计算当前最重与最轻节点的 weight delta；
2. delta 从大到小排序；
3. 先处理最不均衡的 index。

这样可降低新增节点时某一个 index 先占满新节点、后续又反向搬回的中间抖动。

### 12.3 relevant node 压缩

`balanceByWeights()` 位于 `:693-813`。

对每个 index，只有以下节点进入 relevant window：

- 已经持有该 index 的 shard；或
- index-level `canAllocate(indexMetadata, node, allocation)` 不是 `NO`。

少于两个 relevant node 时，该 index 无法均衡。

### 12.4 双端窗口算法

对 relevant node 按 weight 升序排序：

~~~text
low  -> 最轻节点
high -> 最重节点
~~~

核心过程可概括为：

~~~text
for index in most-unbalanced-first:
    sort relevant nodes by weight
    low = 0
    high = last

    while low < high:
        if weight(high) - weight(low) 不超过 localThreshold:
            stop，或处理 decider 导致的特殊窗口

        if 可以从 high 搬一个该 index 的 shard 到 low:
            更新 source/target model
            重新排序
            low/high 回到两端
        else:
            先从 MIN 向 MAX 缩小窗口
            再从 MAX 向 MIN 缩小窗口
~~~

双向缩窗是为了处理 awareness/filter 等约束：最重节点可能不能向绝对最轻节点搬，但可能可以向第二轻节点搬；反之也可能必须放弃当前最重节点，检查次重节点。

### 12.5 具体选择哪个 shard

`tryRelocateShard()` 位于 `:1296-1340`：

1. 只看 max node 上该 index 的 `STARTED` shard；
2. shard ID 降序，保证选择确定性；
3. shard-level `canRebalance` 不能是 `NO`；
4. min target 的 `canAllocate` 不能是 `NO`；
5. 合并两个 decision。

若合并结果为 `YES`：

~~~text
RoutingNodes.relocateShard(...)
~~~

若为 `THROTTLE`：

- 不修改 routing nodes；
- 只在内部 model 中模拟 source 减少、target 增加；
- 让本轮后续选择预见该潜在搬迁，避免过度规划同一目标。

### 12.6 `decideRebalance()` 不是生产 balance 主循环

`decideRebalance()` 位于 `:564-680`，主要供 cluster allocation explain API 判断单个 shard。

真正批量均衡调用的是：

~~~text
balance() -> balanceByWeights() -> tryRelocateShard()
~~~

阅读调用链时若把 `decideRebalance()` 当成真实执行入口，会误解生产路径。

## 13. `DesiredBalanceReconciler`：目标如何真实落地

源码位于 [DesiredBalanceReconciler.java](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/DesiredBalanceReconciler.java)。

### 13.1 每次只基于最新真实状态前进一步

`reconcile()` 位于 `:129-134`。它先从公平排序器中移除已离群节点，再创建 `Reconciliation`。

`run()` 位于 `:153-187`：

1. 设置 reconciling flag；
2. 无 data node 时把新 primary 的 allocation status 标成 `DECIDERS_NO`，使健康状态保持 red；
3. desired assignment 为空时等待后台目标；
4. `allocateUnassigned()`；
5. `moveShards()`；
6. `balance()`。

### 13.2 分配 unassigned shard

`allocateUnassigned()` 位于 `:250-361`，排序和双数组公平策略与 legacy allocator 相同，但候选节点来自 `ShardAssignment.nodeIds()`。

真实协调只接受 `YES`：

- `YES`：调用 `RoutingNodes.initializeShard()`；
- `THROTTLE`：保持 unassigned，状态为 `DECIDERS_THROTTLED`；
- `NO`：继续检查其他 desired node，最后为 `DECIDERS_NO`；
- assignment 本身标记 ignored：状态为 `NO_ATTEMPT`。

primary 还有两个特殊路径：

- shrink/split/clone 等操作可能通过 `getForcedInitialShardAllocationToNodes()` 强制初始节点；
- desired node 暂时都不可用且没有发生 throttle 时，可 fallback 到全集群其他节点，优先恢复 primary 可用性。

### 13.3 必须搬迁的 shard

`moveShards()` 位于 `:441-489`：

1. 只处理 `STARTED`；
2. 当前节点已经在 desired node set 中则跳过；
3. shard-level `canAllocate` 必须为 `YES`；
4. `canRemain == NO` 才在这一阶段搬迁；
5. 目标必须属于 desired set，且 target-level decision 为 `YES`；
6. replacement node 场景可尝试 force allocation。

`OrderedShardsIterator` 和 `moveOrdering` 使受限 recovery 能公平地从多个 source node 启动，而不是总从同一节点先搬满。

### 13.4 普通 desired rebalance

`balance()` 位于 `:491-561`：

- 全局 `canRebalance` 必须为 `YES`；
- 当前不在 desired set 的 shard 计为 undesired allocation；
- shard-level `canRebalance` 必须为 `YES`；
- shard-level 和 target-level `canAllocate` 都必须为 `YES`；
- 找到 desired target 后调用 `RoutingNodes.relocateShard()`。

注意它不再重新比较权重。权重优化已经在后台生成 desired assignment 时完成；协调阶段只回答“当前能否向目标走一步”。

这正是异步设计能保持安全的原因：目标可能基于稍旧快照，但每个真实动作都重新通过当前 decider 和当前节点存在性检查。

## 14. `AllocationDecider` 如何参与

`ClusterModule.createAllocationDeciders()` 在 `ClusterModule:339-369` 注册 18 个内置 decider，再追加插件 decider：

| 类别 | 主要 decider | 约束 |
| --- | --- | --- |
| 重试与恢复来源 | `MaxRetryAllocationDecider`、`ResizeAllocationDecider` | 失败次数、resize 初始节点 |
| primary/replica 时序 | `ReplicaAfterPrimaryActiveAllocationDecider`、`RebalanceOnlyWhenActiveAllocationDecider` | primary 激活、所有 copy 激活 |
| 全局 rebalance | `ClusterRebalanceAllocationDecider`、`ConcurrentRebalanceAllocationDecider` | rebalance 开关和并发数 |
| allocation/rebalance 开关 | `EnableAllocationDecider` | cluster/index allocation 与 rebalance mode |
| 兼容性 | `NodeVersionAllocationDecider` | 节点版本兼容 |
| snapshot/restore | `SnapshotInProgressAllocationDecider`、`RestoreInProgressAllocationDecider` | snapshot/restore 中的移动限制 |
| 节点生命周期 | `NodeShutdownAllocationDecider`、`NodeReplacementAllocationDecider` | shutdown/replacement |
| 放置规则 | `FilterAllocationDecider`、`SameShardAllocationDecider`、`AwarenessAllocationDecider` | include/exclude、同 shard copy、机架/zone |
| 容量和限流 | `DiskThresholdDecider`、`ThrottlingAllocationDecider`、`ShardsLimitAllocationDecider` | 磁盘水位、recovery 并发、shard 数限制 |

聚合逻辑位于 [AllocationDeciders.java](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/AllocationDeciders.java) `:197-229`：

- 非 explain 模式遇到第一个 `NO` 立即返回；
- 没有 `NO` 但至少一个 `THROTTLE`，最终返回 `THROTTLE`；
- 全部允许才返回 `YES`；
- explain 模式按 debug level 收集多个 decision。

可以把三个结果理解为：

| Decision | 目标计算 | 真实协调 |
| --- | --- | --- |
| `NO` | 永久/当前硬约束，不能把它当目标 | 不执行 |
| `THROTTLE` | 多数场景可作为未来可能位置或模型负载 | 本次不执行 |
| `YES` | 可进入目标 | 可真实初始化或 relocation |

## 15. `RoutingNodes` 状态变化

源码位于 [RoutingNodes.java](../server/src/main/java/org/elasticsearch/cluster/routing/RoutingNodes.java)。

### 15.1 初始分配

`initializeShard()` 位于 `:432-450`：

~~~text
UNASSIGNED
  -> ShardRouting.initialize(nodeId, allocationId, expectedSize)
  -> 加入目标 RoutingNode
  -> 增加 inactive/recovery 计数
  -> 加入 assignedShards
  -> 通知 RoutingChangesObserver.shardInitialized
~~~

### 15.2 relocation

`relocateShard()` 位于 `:459-474`：

~~~text
source STARTED
  -> source RELOCATING
  -> 创建 target INITIALIZING shadow
  -> target 加入目标 RoutingNode
  -> 增加 relocating/recovery 计数
  -> 通知 relocationStarted
~~~

这段时间同一个 shard copy 在路由模型中有 source 和 target 两个 routing entry，但它们共享 relocation 关系，不表示多了一份独立副本。

### 15.3 recovery 完成

`startShard()` 位于 `:487-542`：

- 普通 initializing shard 转为 `STARTED`；
- relocation target 启动后删除 source；
- 通知 `relocationCompleted`；
- 若完成的是 primary relocation，还会重建进行中的 replica recovery，因为它们的 recovery source 已改变。

后台 `DesiredBalanceComputer` 调用相同 API，但对象来自 `mutableCloneForSimulation()`；真实协调调用这些 API 后，才由 `AllocationService` 物化新 cluster state。

## 16. DiDi fork 的 region 感知与 master 优化

主要变更来自提交 `873d6fbf752` 和后续修复 `240d050df23`。

### 16.1 开关

[EnableAllocationDecider.java](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/EnableAllocationDecider.java) `:83-113` 定义：

~~~text
cluster.routing.master_enhance.enable = true
~~~

它是 dynamic、node-scope 设置，并保存在 static volatile 字段中。

关闭时，`BalancedShardsAllocator` 回到全节点 `NodeSorter` 和全 index 处理方式。

### 16.2 region 数据来源

节点 region 来自 node attribute：

~~~text
node.attr.region
~~~

源码使用常量：

~~~text
INDEX_SETTING_REGION_ATTRIBUTE = "region"
INDEX_SETTING_DEFAULT_REGION_NAME = "default"
~~~

没有 region 属性的节点归入 `default`。

index 是否属于某 region 则读取 routing include filter 中的 region 值，也就是 `index.routing.allocation.include.region` 对应的配置。

### 16.3 单次 Balancer 如何选择节点

`Balancer` 构造器位于 `BalancedShardsAllocator:345-365`：

1. `buildModelFromAssigned()` 仍为全部节点建立 `nodes` model；
2. 同时建立 `region -> ModelNode list`；
3. 若只有一个 region，直接选它；
4. 若有多个 region，随机选择一个非 `default` region；
5. 把 `default` region 节点并入所选 region；
6. `NodeSorter` 只包含这个节点子集。

影响范围：

| 阶段 | 是否 region 限定 |
| --- | --- |
| `allocateUnassigned()` | 否；它遍历完整 `nodes.values()`，最终仍由 decider/filter 控制 |
| `moveShards()` | 是；只交错扫描所选 region 与 default 节点 |
| `balanceByWeights()` | 是；sorter 只含所选 region 与 default 节点 |
| index 列表 | 是；只处理未设置 region filter 的 index，或 include 当前 region 的 index |

### 16.4 平均值仍是全局平均

虽然 sorter 是 region 子集，但以下平均值仍按全部 routing nodes 和全部 metadata 计算：

- global shards per node；
- per-index shards per node；
- write load per node；
- disk usage per node。

因此当前实现是“用全局平均值评价某个 region 子集”，不是“在 region 内重新计算局部平均值”。这是理解 region 模式下 weight 的关键。

### 16.5 限流提前退出

`moveShards()` 记录所有返回 `THROTTLE` 的 candidate model node。开启 master enhance 时，如果：

~~~text
throttledNodes.size() >= selectedRegionNodeIds.size()
~~~

就停止扫描剩余 shard，降低大集群 master 上重复 decider 检查的成本。

`decideMove()` 也会提前跳过：

- 已知 throttled node；
- 已经持有同一 shard copy 的 node。

### 16.6 对默认 `desired_balance` 的影响

`DesiredBalanceComputer` 每一轮都会调用一次 delegate，delegate 每次又创建新的 `Balancer` 并随机选择 region：

- 如果本轮产生 initializing/relocation，模拟 start 后下一轮可能随机处理另一个 region；
- 如果本轮所选 region 已无动作，`hasChanges == false` 时目标计算可能结束，即使另一个未选 region 仍有改善空间；
- 后续外部 reroute 和新一轮 desired computation 会再次随机选择 region。

所以 fork 的收敛粒度更接近“跨多次 reroute 分批处理 region”，而不是保证一次目标计算遍历所有 region。

### 16.7 两个需要明确记录的实现观察

1. [BalancedShardsAllocatorTests.java](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocatorTests.java) `:103` 的 `testBalanceByForecastWriteLoad()` 被 `@AwaitsFix(bugUrl = "fork-skipped")` 跳过，注释为“Fork 增加了 region 感知均衡逻辑”。这意味着 region 定制与 forecast write-load 均衡组合缺少现行自动化覆盖。
2. `RoutingNodes.selectNodeIDByRegion()` 在 `RoutingNodes:1390-1399` 直接取得 `nodesPerRegionNames` 中保存的 list，再对它执行 `addAll(defaultNodes)`。重复调用可能向缓存 list 追加重复 default node ID。当前 iterator 主要做 membership check，因此不一定重复扫描 shard，但 `nodeIdsByRegion.size()` 会影响“所有节点均 throttle”的提前退出条件，值得后续单独验证。

## 17. 并发、一致性和收敛保证

### 17.1 为什么旧目标仍然安全

普通 reroute 会立刻协调 `currentDesiredBalance`，它可能不是基于最新 cluster state 计算的。但是协调器会重新检查：

- desired node 是否仍存在；
- 当前节点是否仍持有该 shard；
- 当前 `canRemain`；
- 当前全局和 shard-level `canRebalance`；
- 当前 shard-level 和 target-level `canAllocate`；
- 当前 recovery throttle。

所以 stale target 最多导致“本次不动”或向仍然合法的旧目标前进一步，不会绕过实时硬约束。

### 17.2 为什么目标计算可接受 `THROTTLE`

目标回答的是“限流解除后的稳定位置”，协调回答的是“现在能不能执行”。把二者分开后：

- 目标不会因为瞬时并发恢复数而不断漂移；
- 真实集群不会突破 recovery 限制；
- 每次 recovery 完成触发 reroute 后可继续向同一目标推进。

### 17.3 为什么 listener 可能长期等待

如果计算只能得到中间 desired balance，`lastConvergedIndex` 不前进。即使后台执行完成并提交过 reconciliation task，也不会完成较新的 listener。

后续缺失信息到达、限流解除或新 reroute 触发完整收敛后，index 才前进并批量完成 listener。

### 17.4 delayed allocation 的特殊处理

后台结果触发的 `createReconcileAllocationAction()` 故意不删除 delayed marker。原因是该 desired balance 计算时 delayed allocation 可能尚未过期：

- 协调时突然删除 marker，也没有相应目标可分配它；
- 同时可能失去负责后续触发分配的 reroute。

普通 reroute 仍会按正常策略处理 delay expiration。

## 18. 测试证据与覆盖地图

### 18.1 `BalancedShardsAllocatorTests`

文件：[BalancedShardsAllocatorTests.java](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocatorTests.java)

| 测试 | 验证点 |
| --- | --- |
| `testDecideShardAllocation` | explain 与实际 allocate 选择同一目标 |
| `testBalanceByForecastDiskUsage` | forecast shard size 参与 disk balance |
| `testBalanceByActualDiskUsage` | actual shard size 参与 disk balance |
| `testBalanceByActualAndForecastDiskUsage` | actual/forecast 合并 |
| `testRebalanceImprovesTheBalanceOfTheShards` | relocation 不把 index 布局变差 |
| `testThresholdLimit` | threshold 小于 1 被钳制并告警 |
| `testShardSizeDiscrepancyWithinIndex` | 同 index 不同 shard size 的处理 |
| `testBalanceByForecastWriteLoad` | 当前被 fork 跳过，是已知覆盖缺口 |

### 18.2 `DesiredBalanceComputerTests`

文件：[DesiredBalanceComputerTests.java](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/allocator/DesiredBalanceComputerTests.java)

| 测试 | 验证点 |
| --- | --- |
| `testComputeBalance` | 基本目标 assignment |
| `testStopsComputingWhenStale` | stale 后至少执行一轮并返回中间目标，下一轮继续 |
| `testAssignShardsToTheirPreviousLocationIfAvailable` | 复用旧目标降低抖动 |
| `testRespectsAssignmentByGatewayAllocators` | 不接管 existing shard allocator 范围 |
| `testSimulatesAchievingDesiredBalanceBeforeDelegating` | 调 delegate 前先恢复旧 desired locations |
| `testAppliesMoveCommands` | move command 进入目标 |
| `testDesiredBalanceShouldConvergeInABigCluster` | 大集群收敛 |
| `testComputeConsideringShardSizes` | shard size 进入目标计算 |
| `testShouldLogComputationIteration` | 长计算进度日志 |

### 18.3 `DesiredBalanceReconcilerTests`

文件：[DesiredBalanceReconcilerTests.java](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/allocator/DesiredBalanceReconcilerTests.java)

重点覆盖：

- primary 优先、replica 交错和 index priority；
- ignored shard 与 allocation status；
- desired node、fallback 和 forced initial allocation；
- filter 变化后的强制 move；
- global/shard `canRebalance` 和 `canAllocate` 阻断；
- recovery throttle 下 source-node 公平性；
- 节点在计算后离群时不向它搬迁；
- replacement shutdown 的 force move；
- undesired allocation 指标与 warning。

### 18.4 `DesiredBalanceShardsAllocatorTests`

文件：[DesiredBalanceShardsAllocatorTests.java](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/allocator/DesiredBalanceShardsAllocatorTests.java)

重点覆盖：

- gateway allocator 优先级；
- listener 只在 fresh input 对应协调完成后回调；
- 多个快速 cluster-state update 合并为一次 reconciliation；
- 失去 master 后 listener 失败；
- desired balance reset；
- 后台 reconciliation 不移除 delay marker。

### 18.5 设置级测试

[BalanceConfigurationTests.java](../server/src/test/java/org/elasticsearch/cluster/routing/allocation/BalanceConfigurationTests.java) 覆盖：

- index balance；
- replica balance；
- 动态设置持久化和更新。

## 19. 诊断与观测入口

### 19.1 私有 desired balance API

REST spec 定义：

~~~http
GET /_internal/desired_balance
DELETE /_internal/desired_balance
~~~

它们是 private、experimental diagnostics API，不应作为业务稳定接口。

GET 由 [TransportGetDesiredBalanceAction.java](../server/src/main/java/org/elasticsearch/action/admin/cluster/allocation/TransportGetDesiredBalanceAction.java) 构造：

- `DesiredBalanceStats`；
- 当前/目标 cluster balance stats；
- 根据 assignment 构造的目标 routing table；
- cluster info。

DELETE 会 reset 当前 desired balance 并触发 reroute，适合诊断“旧目标是否影响当前协调”，但属于有状态操作，生产环境使用前应评估 relocation 影响。

### 19.2 关键统计

`DesiredBalanceShardsAllocator.getStats()` 暴露：

- submitted/executed/converged computation 数；
- 累计 iteration 数；
- 计算出的 shard movement 数；
- computation/reconciliation 时间；
- unassigned、total allocation、undesired allocation 数；
- 当前是否仍在计算。

### 19.3 日志定位

建议按以下顺序看日志：

1. `Executing allocate for [index]`：master 提交了哪个 input；
2. `Starting desired balance computation`：后台开始计算；
3. `still not converged`：是否发生长迭代；
4. `interrupted ... newer cluster state received`：是否因 stale input 中断；
5. `scheduling reconciliation`：fresh 结果已进入 master queue；
6. `Reconciling desired balance`：真实协调开始；
7. `Assigning/Moving/Rebalancing shard`：实际动作；
8. `DECIDERS_THROTTLED` 或 allocation explain：判断为何只前进一步。

## 20. 推荐的源码阅读顺序

按以下顺序阅读最容易建立完整心智模型：

1. [ClusterModule.java](../server/src/main/java/org/elasticsearch/cluster/ClusterModule.java) `:101-109, 339-417`
   看 allocator 默认值、delegate 组装和 decider 列表。
2. [AllocationService.java](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/AllocationService.java) `:392-416, 523-560, 619-627, 156-180`
   看 reroute、existing shard 优先级和 cluster state 物化。
3. [DesiredBalanceShardsAllocator.java](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/DesiredBalanceShardsAllocator.java) `:111-165, 178-194, 216-263, 290-360`
   看线程切换、listener 和 master task。
4. [ContinuousComputation.java](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/ContinuousComputation.java) `:21-100`
   看输入合并与 stale 识别。
5. [DesiredBalanceComputer.java](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/DesiredBalanceComputer.java) `:73-361`
   看模拟预热、迭代和收敛 index。
6. [BalancedShardsAllocator.java](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java) `:180-198, 296-324, 533-557, 693-813, 902-1039, 1086-1340`
   看权重、未分配、强制搬迁和纯均衡。
7. [DesiredBalanceReconciler.java](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/DesiredBalanceReconciler.java) `:129-187, 250-408, 441-625`
   看目标如何受实时 decider 约束并落地。
8. [RoutingNodes.java](../server/src/main/java/org/elasticsearch/cluster/routing/RoutingNodes.java) `:432-542, 1355-1399`
   看状态机和 region iterator。
9. [AllocationDeciders.java](../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/AllocationDeciders.java) `:68-110, 181-229`
   看 decision 聚合。

## 21. 最终调用链速查

### 21.1 默认模式

~~~text
AllocationService.reroute
  -> executeWithRoutingAllocation
    -> AllocationService.reroute(private)
      -> allocateExistingUnassignedShards
      -> DesiredBalanceShardsAllocator.allocate
        -> ContinuousComputation.onNewInput
        -> DesiredBalanceReconciler.reconcile(current target)
          -> allocateUnassigned
          -> moveShards
          -> balance
        -> AllocationService.buildResultAndLogHealthChange

generic thread:
  DesiredBalanceComputer.compute
    -> mutableCloneForSimulation
    -> restore previous desired placement
    -> BalancedShardsAllocator.allocate
      -> allocateUnassigned
      -> moveShards
      -> balance
        -> balanceByWeights
          -> tryRelocateShard
    -> simulate start
    -> repeat until converged/stale
    -> DesiredBalance
    -> submit URGENT reconcile task

master task:
  AllocationService.executeWithRoutingAllocation
    -> DesiredBalanceReconciler.reconcile(new target)
    -> RoutingNodes.initializeShard/relocateShard
    -> RoutingTable.of
    -> new ClusterState
    -> publication success
    -> complete pending listeners
~~~

### 21.2 直接 `balanced` 模式

~~~text
AllocationService.reroute
  -> allocateExistingUnassignedShards
  -> BalancedShardsAllocator.allocate(real allocation)
    -> allocateUnassigned
    -> moveShards
    -> balance
      -> balanceByWeights
  -> RoutingTable.of
  -> new ClusterState
~~~

## 22. 核心认识

1. 默认 `desired_balance` 是异步控制层，真正的优化器仍是 `BalancedShardsAllocator`。
2. weight 只负责给合法候选排序和判断搬迁收益，`AllocationDecider` 才是最终安全边界。
3. `THROTTLE` 在目标/模型中可代表未来负载，但在真实协调中绝不会当作 `YES` 执行。
4. 目标计算通过模拟 start recovery 看到未来稳态；真实集群通过多次 reroute 逐步收敛。
5. `RoutingNodes` 是本次计算的可变状态，`RoutingTable`/`ClusterState` 才是最终可发布状态。
6. 本仓库的 region 优化使单次 balance 只覆盖一个随机 region 与 default 节点，并保留全局平均值作为权重基准。
7. region 与 forecast write-load 的组合当前存在明确的测试跳过，`selectNodeIDByRegion()` 也有缓存 list 被原地追加的实现细节，后续修改相关逻辑时应优先补充针对性测试。
