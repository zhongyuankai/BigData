# `DiskThresholdDecider` 源码跟踪：从磁盘统计到分配决策

> 仓库：`elasticsearch-didi`
>
> 基线提交：`db41ec0c4e78`
>
> 核心源码：[`DiskThresholdDecider.java`](../../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/DiskThresholdDecider.java#L66-L570)
>
> 设置源码：[`DiskThresholdSettings.java`](../../server/src/main/java/org/elasticsearch/cluster/routing/allocation/DiskThresholdSettings.java#L27-L561)

## 1. 阅读主线

`DiskThresholdDecider` 不采集磁盘数据，也不亲自移动 shard。allocator 提出候选操作后，它只回答三个问题：

| 方法 | 问题 | 磁盘规则 |
| --- | --- | --- |
| `canAllocate()` | shard 能否进入目标节点 | 当前 free 不能低于 low，放入后不能低于 high |
| `canRemain()` | shard 能否留在当前节点 | 最拥挤 path 的 free 不能低于 high |
| `canForceAllocateDuringReplace()` | replacement 时能否强制进入 target | 可以越过 low/high，但不能超过物理容量 |

全文按源码真实执行顺序展开：

~~~text
watermark 配置
  -> 转换成 required free bytes
  -> ClusterInfo 提供原始磁盘统计
  -> getDiskUsage() 修正统计时间差
  -> canAllocate / canRemain / force replacement
  -> AllocationDeciders 聚合
  -> allocator 执行分配或迁移
~~~

### 1.1 贯穿全文的例子

后文始终使用同一组数据：

~~~text
total                           = 1TB
NodeStats 观测空闲 observedFree = 200GB
reserved space                  = 20GB
未被 reserved 覆盖的 incoming   = 10GB
统计滞后的 searchable snapshot  = 5GB
正在搬离当前 path 的 shard       = 30GB

lowRequiredFree                 = 150GB
highRequiredFree                = 100GB
候选 shard expected size        = 80GB
~~~

新分配不能提前使用 leaving space：

~~~text
allocateCorrectedFree = 200 - 20 - 10 - 5 = 165GB
~~~

判断是否还要继续驱逐时，可以认可已启动的搬离：

~~~text
remainCorrectedFree = 200 - 20 - 10 - 5 + 30 = 195GB
~~~

## 2. 注册与调用位置

### 2.1 注册为 `disk_threshold`

`ClusterModule#createAllocationDeciders()` 注册实例：

[`ClusterModule.java`](../../server/src/main/java/org/elasticsearch/cluster/ClusterModule.java#L339-L370)

~~~java
addAllocationDecider(deciders, new DiskThresholdDecider(settings, clusterSettings));
~~~

类中定义：

~~~java
public static final String NAME = "disk_threshold";
~~~

所以 allocation explain 中的标签是：

~~~json
{
  "decider": "disk_threshold",
  "decision": "NO",
  "explanation": "..."
}
~~~

### 2.2 decision 还要经过聚合

allocator 调用的是 [`AllocationDeciders`](../../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/AllocationDeciders.java#L30-L241)，不是单独调用某个具体决策器：

~~~text
allocator
  -> AllocationDeciders.canAllocate/canRemain/force...
  -> 依次询问所有 AllocationDecider
  -> 按 NO > THROTTLE > YES 合并
~~~

因此 `disk_threshold=YES` 只代表磁盘约束通过。awareness、filter、same-shard 等仍可拒绝候选操作。

`DiskThresholdDecider` 自身只产生 `YES` 或 `NO`，不产生 `THROTTLE`。

### 2.3 方法覆盖面

| `AllocationDecider` 方法 | 本类行为 |
| --- | --- |
| node-level `canAllocate()` | 覆盖，执行完整磁盘准入判断 |
| `canRemain()` | 覆盖，执行 high watermark 驱逐判断 |
| `canForceAllocateDuringReplace()` | 覆盖，只保留物理容量底线 |
| shard-level `canAllocate()` | 未覆盖，默认 `YES` |
| `canRebalance()` | 未覆盖，默认 `YES` |
| `canForceAllocatePrimary()` | 未覆盖，普通 `NO` 可被基类 force-primary 语义转换为 `YES` |

force primary 与 replacement 是两个不同入口，不应互相类推。

## 3. watermark 如何变成源码里的 free bytes

### 3.1 默认值

| 设置 | 默认值 |
| --- | ---: |
| `cluster.routing.allocation.disk.threshold_enabled` | `true` |
| `cluster.routing.allocation.disk.watermark.low` | `85%` |
| `cluster.routing.allocation.disk.watermark.low.max_headroom` | `200GB`* |
| `cluster.routing.allocation.disk.watermark.high` | `90%` |
| `cluster.routing.allocation.disk.watermark.high.max_headroom` | `150GB`* |
| `cluster.routing.allocation.disk.watermark.flood_stage` | `95%` |
| `cluster.routing.allocation.disk.watermark.flood_stage.max_headroom` | `100GB`* |
| `cluster.routing.allocation.disk.watermark.flood_stage.frozen` | `95%` |
| `cluster.routing.allocation.disk.watermark.flood_stage.frozen.max_headroom` | `20GB`* |
| `cluster.routing.allocation.disk.reroute_interval` | `60s` |

星号表示：只有对应 watermark 未显式配置时才自动采用该 headroom；显式配置 watermark 后，headroom 默认 `-1`，即不启用 cap。

### 3.2 两种配置表达的是不同方向

~~~text
85%   -> 已使用比例，等价于至少空闲 15%
100GB -> 直接表示至少空闲 100GB
~~~

所以 validator 要求：

~~~text
比例：low <= high <= flood
      85% <= 90% <= 95%

字节：low >= high >= flood
      100GB >= 50GB >= 10GB
~~~

low、high、flood 还必须全部使用比例，或全部使用绝对字节，不能混用。

### 3.3 转换源码

[`DiskThresholdSettings#getFreeBytesThreshold()`](../../server/src/main/java/org/elasticsearch/cluster/routing/allocation/DiskThresholdSettings.java#L433-L455)：

~~~java
private static ByteSizeValue getFreeBytesThreshold(
    ByteSizeValue total,
    RelativeByteSizeValue watermark,
    ByteSizeValue maxHeadroom
) {
    if (watermark.isAbsolute()) {
        return watermark.getAbsolute();
    }
    return ByteSizeValue.subtract(
        total,
        watermark.calculateValue(total, maxHeadroom)
    );
}
~~~

等价公式：

~~~text
绝对字节：
  requiredFree = configuredBytes

比例：
  requiredFree = min(total * (1 - usedRatio), maxHeadroom)
~~~

上式的 `min` 只在 headroom 启用时成立；`maxHeadroom=-1` 时完全按比例计算。

例如 10TB 磁盘的默认 high 为 `90%`：

~~~text
按比例要求空闲 1TB
high max_headroom = 150GB
最终 highRequiredFree = 150GB
~~~

从这里开始，核心判断都是 free-bytes 的 long 比较；used percentage 主要用于日志和 explain。

## 4. `freeBytes` 为什么不是 NodeStats 原始值

### 4.1 `ClusterInfo` 的输入

`InternalClusterInfoService` 组合两类统计：

~~~text
NodeStats.fs
  -> 节点最空闲 path
  -> 节点最拥挤 path

IndicesStats
  -> shardSizes
  -> shardDataSetSizes
  -> shard 实际 dataPath
  -> node/path 的 reservedSpace
~~~

[`InternalClusterInfoService#fillDiskUsagePerNode()`](../../server/src/main/java/org/elasticsearch/cluster/InternalClusterInfoService.java#L483-L498)：

~~~java
DiskUsage leastAvailableUsage =
    DiskUsage.findLeastAvailablePath(nodeStats);
DiskUsage mostAvailableUsage =
    DiskUsage.findMostAvailable(nodeStats);
~~~

most/least 根据 path 的原始 available bytes 选出，此时尚未扣 reserved space。

| 用途 | 选择的 path | 原因 |
| --- | --- | --- |
| 新 shard 进入 | most available，最空闲 | shard 尚无实际 path |
| 已有 shard 是否保留 | least available，最拥挤 | 只搬走能缓解瓶颈的 shard |

### 4.2 `getDiskUsage()` 是统一入口

[`DiskThresholdDecider#getDiskUsage()`](../../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/DiskThresholdDecider.java#L466-L500)：

~~~java
DiskUsage usage = usages.get(node.nodeId());
if (usage == null) {
    usage = averageUsage(node, usages);
}

return new DiskUsageWithRelocations(
    usage,
    sizeOfUnaccountedShards(
        node,
        subtractLeavingShards,
        usage.getPath(),
        allocation.clusterInfo(),
        allocation.metadata(),
        allocation.routingTable(),
        allocation.unaccountedSearchableSnapshotSize(node)
    )
);
~~~

当前节点 usage 缺失、但其他节点有数据时，`averageUsage()` 使用已知节点的平均 total/free，合成 path 为 `_na_` 的 usage。

整个 map 都为空时不会走平均值，`earlyTerminate()` 会直接返回 `YES_USAGES_UNAVAILABLE`。

### 4.3 corrected free 的最终计算

[`DiskUsageWithRelocations#getFreeBytes()`](../../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/DiskThresholdDecider.java#L540-L568)：

~~~java
long getFreeBytes() {
    try {
        return Math.subtractExact(
            diskUsage.getFreeBytes(),
            relocatingShardSize
        );
    } catch (ArithmeticException e) {
        return Long.MAX_VALUE;
    }
}
~~~

字段名 `relocatingShardSize` 比实际含义窄。它保存的是 `sizeOfUnaccountedShards()` 返回的总修正量：

~~~text
correctedFree = observedFree - unaccountedSize
~~~

修正量中若包含负的 leaving size，“减去负数”就会加回未来释放空间。long 溢出时返回 `Long.MAX_VALUE`，避免符号翻转造成虚假负空间。

## 5. `sizeOfUnaccountedShards()` 如何修正时间差

实现位于 [`DiskThresholdDecider.java`](../../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/DiskThresholdDecider.java#L107-L167)。它补偿的是“routing 已变化，但磁盘 stats 尚未刷新”的空间。

### 5.1 先计入 reserved space

~~~java
final ClusterInfo.ReservedSpace reservedSpace =
    clusterInfo.getReservedSpace(node.nodeId(), dataPath);
long totalSize = reservedSpace.getTotal();
~~~

恢复过程可能已经预留空间，但最近的 `NodeStats.fs` 尚未反映相同的 free 下降。

固定案例：

~~~text
totalSize = 20GB
~~~

### 5.2 再补未被 reserved 覆盖的 initializing shard

~~~java
for (ShardRouting routing : node.initializing()) {
    if (routing.relocatingNodeId() == null
        && metadata.getIndexSafe(routing.index())
            .isSearchableSnapshot() == false) {
        continue;
    }
    if (reservedSpace.containsShardId(routing.shardId())) {
        continue;
    }

    final String actualPath = clusterInfo.getDataPath(routing);
    if (actualPath == null || actualPath.equals(dataPath)) {
        totalSize += getExpectedShardSize(...);
    }
}
~~~

逐个分支解释：

1. 非 relocation、非 searchable snapshot 的 initializing shard 跳过。源码说明这类非零 expected size 通常来自 shrink/split/clone，预期使用 hard link，不应再次扣完整 shard size。
2. reserved space 已包含该 `ShardId` 时跳过，防止重复计数。
3. actual path 未知时保守地算入当前检查 path；已知时只算入相同 path。

固定案例：

~~~text
totalSize = reserved 20 + incoming 10 = 30GB
~~~

### 5.3 补 started searchable snapshot 的统计滞后

~~~java
totalSize += sizeOfUnaccountableSearchableSnapshotShards;
~~~

该值由 [`RoutingAllocation.unaccountedSearchableSnapshotSizes()`](../../server/src/main/java/org/elasticsearch/cluster/routing/allocation/RoutingAllocation.java#L158-L179) 计算。只有同时满足以下条件才计入：

- shard 已 started；
- index 是 searchable snapshot；
- expected size 大于 0；
- reserved space 尚未覆盖；
- `ClusterInfo` 还没有实际 shard size。

固定案例：

~~~text
totalSize = 20 + 10 + 5 = 35GB
~~~

### 5.4 最后按调用者要求减去 leaving shard

~~~java
if (subtractShardsMovingAway) {
    for (ShardRouting routing : node.relocating()) {
        if (dataPath.equals(clusterInfo.getDataPath(routing))) {
            totalSize -= getExpectedShardSize(routing, 0L, ...);
        }
    }
}
~~~

只有参数为 `true`，且 leaving shard 位于当前检查 path，才减去 expected size：

~~~text
canAllocate 传 false：
  unaccountedSize = 35GB
  correctedFree = 200 - 35 = 165GB

canRemain 传 true：
  unaccountedSize = 35 - 30 = 5GB
  correctedFree = 200 - 5 = 195GB
~~~

这组不对称是核心设计：

- 接收新 shard 时，不提前消费尚未真正释放的空间；
- 判断是否继续驱逐时，可以认可已经启动的搬离。

## 6. `canAllocate()` 源码跟踪

完整实现：[`DiskThresholdDecider#canAllocate()`](../../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/DiskThresholdDecider.java#L181-L339)。

它不是简单检查一次 low，而是依次经过五道关卡。

### 6.1 选择 most-available path，并处理快速返回

~~~java
Map<String, DiskUsage> usages =
    allocation.clusterInfo().getNodeMostAvailableDiskUsages();

final Decision decision = earlyTerminate(usages);
if (decision != null) {
    return decision;
}

if (allocation.metadata().index(shardRouting.index())
    .ignoreDiskWatermarks()) {
    return YES_DISK_WATERMARKS_IGNORED;
}
~~~

`earlyTerminate()` 的顺序：

~~~text
threshold_enabled == false
  -> YES_DISABLED

usages.isEmpty()
  -> YES_USAGES_UNAVAILABLE

否则继续
~~~

无任何统计时选择 fail-open，避免 stats 暂时不可用导致 allocation 全停。index ignore 是 private index setting，只跳过该 index 的磁盘 decision。

### 6.2 计算 corrected free

~~~java
final DiskUsageWithRelocations usage =
    getDiskUsage(node, allocation, usages, false);

long freeBytes = usage.getFreeBytes();
final ByteSizeValue total =
    ByteSizeValue.ofBytes(usage.getTotalBytes());

if (freeBytes < 0L) {
    return allocation.decision(Decision.NO, NAME, ...);
}
~~~

参数 `false` 表示不加回 leaving shard。固定案例得到：

~~~text
freeBytes = 165GB
~~~

如果 reserved、incoming 等修正量已经超过 observed free，corrected free 会小于 0，此时不再检查 low/high，直接 `NO`。

### 6.3 识别 fresh empty primary

~~~java
boolean skipLowThresholdChecks = shardRouting.primary()
    && shardRouting.active() == false
    && shardRouting.recoverySource().getType()
        == RecoverySource.Type.EMPTY_STORE;
~~~

这个特例只包含“从未分配过的新建空 primary”，不包括 replica、existing-store primary、snapshot restore 或 local-shards resize。

### 6.4 比较当前 free 与 low

去掉日志后的源码主干：

~~~java
if (freeBytes < lowRequiredFree) {
    if (skipLowThresholdChecks == false) {
        return NO;
    } else if (freeBytes > highRequiredFree) {
        return YES_UNALLOCATED_PRIMARY_BETWEEN_WATERMARKS;
    } else {
        return NO;
    }
}
~~~

普通 shard：

~~~text
freeBytes < lowRequiredFree
  -> NO
~~~

fresh empty primary：

~~~text
highRequiredFree < freeBytes < lowRequiredFree
  -> YES，并立即返回

freeBytes <= highRequiredFree
  -> NO
~~~

源码使用严格比较。等于 low 时普通 shard 通过；fresh primary 必须严格大于 high。

固定案例当前为 `165GB >= 150GB`，因此继续下一关。

若只把 observed free 改为 165GB：

~~~text
correctedFree = 165 - 35 = 130GB
high 100 < 130 < low 150

普通 shard                -> NO
fresh EMPTY_STORE primary -> YES
~~~

fresh-primary 的 `YES` 是提前返回，不再做 expected-size 检查。

### 6.5 模拟放入 shard 后再比较 high

~~~java
final long shardSize =
    getExpectedShardSize(shardRouting, 0L, allocation);

long freeBytesAfterShard = freeBytes - shardSize;

if (freeBytesAfterShard < highRequiredFree) {
    return allocation.decision(Decision.NO, NAME, ...);
}

return allocation.decision(Decision.YES, NAME, ...);
~~~

low 判断“节点当前是否还接收普通 shard”，high 判断“这个具体 shard 放入后是否过满”。

固定案例：

~~~text
freeBytesAfterShard = 165 - 80 = 85GB
85 < highRequiredFree 100GB
-> NO
~~~

若候选 shard 只有 50GB：

~~~text
freeBytesAfterShard = 165 - 50 = 115GB
115 >= 100
-> YES
~~~

### 6.6 完整决策树

~~~text
most-available usage
  |
  +-- disabled? ------------------------- YES
  +-- usage map 为空? ------------------ YES
  +-- index ignore? -------------------- YES
  +-- correctedFree < 0? --------------- NO
  |
  +-- correctedFree < low?
  |     +-- 普通 shard ----------------- NO
  |     +-- fresh primary 且 free > high? YES
  |     +-- fresh primary 且 free <= high? NO
  |
  +-- correctedFree - shardSize < high?  NO
  +------------------------------------- YES
~~~

### 6.7 expected shard size 从哪里来

[`ExpectedShardSizeEstimator#getExpectedShardSize()`](../../server/src/main/java/org/elasticsearch/cluster/routing/ExpectedShardSizeEstimator.java#L38-L56) 保留分支结构后可简化为：

~~~java
if (resizeSourceExists
    && shard.active() == false
    && recoveryType == LOCAL_SHARDS) {
    return getExpectedSizeOfResizedShard(...);
} else if (shard.unassigned()
    && recoveryType == SNAPSHOT) {
    return snapshotShardSizeInfo.getShardSize(...);
} else {
    return clusterInfo.getShardSize(...);
}
~~~

选择顺序：

1. local-shards resize：按源 shard 估算；
2. unassigned snapshot：使用 `SnapshotShardSizeInfo`；
3. 其他场景：使用 `ClusterInfo.shardSizes`；
4. 仍未知：返回调用者给的 default。

| 调用位置 | default | 效果 |
| --- | ---: | --- |
| 候选 shard 的 post-high 检查 | `0L` | 未知时不凭空增加大小 |
| initializing incoming | routing 上已有 expected size，最小 0 | 尽量利用已有估算 |
| leaving shard | `0L` | 未知时不提前兑现空间 |

## 7. `canRemain()` 源码跟踪

完整实现：[`DiskThresholdDecider#canRemain()`](../../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/DiskThresholdDecider.java#L377-L464)。

### 7.1 验证调用节点并选择 least-available path

~~~java
if (shardRouting.currentNodeId().equals(node.nodeId()) == false) {
    throw new IllegalArgumentException(...);
}

final Map<String, DiskUsage> usages =
    clusterInfo.getNodeLeastAvailableDiskUsages();
~~~

“能否保留”只能询问 shard 当前所在节点。与新分配不同，这里选择最拥挤 path。

disabled、usage map 为空、index ignore 的快速返回与 `canAllocate()` 相同。

### 7.2 加回正在搬离的空间

~~~java
final DiskUsageWithRelocations usage =
    getDiskUsage(node, allocation, usages, true);
~~~

参数为 `true`，固定案例得到：

~~~text
remainCorrectedFree = 195GB
~~~

判断其他 shard 是否还需搬走时，可以相信已启动的 relocation 最终释放空间，从而避免过度驱逐。

### 7.3 只处理最拥挤 path 上的 shard

~~~java
final String dataPath = clusterInfo.getDataPath(shardRouting);

if (dataPath == null
    || usage.getPath().equals(dataPath) == false) {
    return YES_NOT_MOST_UTILIZED_DISK;
}
~~~

- 当前 shard path 未知：本轮 `YES`；
- shard 不在最拥挤 path：`YES`；
- shard 位于最拥挤 path：继续比较 high。

这样不会搬走对瓶颈 path 没有帮助的 shard。

### 7.4 只比较 high

去掉日志和 explain 参数后，判断等价于：

~~~java
if (freeBytes < 0L) {
    return NO;
}
if (freeBytes < highRequiredFree) {
    return NO;
}
return YES;
~~~

固定案例：

~~~text
remainCorrectedFree = 195GB
highRequiredFree = 100GB
-> YES
~~~

`canRemain()` 不检查 low，由此形成滞回区：

~~~text
free >= low
  允许普通 shard 进入，也允许现有 shard 保留

high <= free < low
  不再接收普通 shard，但不驱逐现有 shard

free < high
  最拥挤 path 上的 shard 不能继续保留
~~~

### 7.5 与 `canAllocate()` 对照

| 维度 | `canAllocate()` | `canRemain()` |
| --- | --- | --- |
| path | most available | least available |
| leaving space | 不使用 | 加回 |
| low | 检查 | 不检查 |
| high | 检查放入后 free | 检查当前 corrected free |
| path 未知 | incoming 保守计入检查 path | 当前 shard 允许保留 |
| 目的 | 防止新压力进入 | 避免过度驱逐 |

## 8. `canForceAllocateDuringReplace()` 源码跟踪

实现位于 [`DiskThresholdDecider.java`](../../server/src/main/java/org/elasticsearch/cluster/routing/allocation/decider/DiskThresholdDecider.java#L341-L369)：

~~~java
final DiskUsageWithRelocations usage =
    getDiskUsage(node, allocation, usages, false);

final long shardSize =
    getExpectedShardSize(shardRouting, 0L, allocation);

final long freeBytesAfterShard =
    usage.getFreeBytes() - shardSize;

if (freeBytesAfterShard < 0) {
    return Decision.single(Decision.Type.NO, NAME, ...);
}
return super.canForceAllocateDuringReplace(...);
~~~

它仍使用 most-available path，不加回 leaving，并扣 reserved、incoming、searchable snapshot 和目标 shard size。但它不比较 low/high：

~~~text
freeAfterShard < 0  -> NO，会超过 100% 物理容量
freeAfterShard >= 0 -> 基类 YES
~~~

固定案例：

~~~text
correctedFree = 165GB
shardSize = 80GB
freeAfterShard = 85GB

普通 canAllocate：
  85 < high 100 -> NO

replacement force：
  85 >= 0 -> YES
~~~

所以 replacement 可以推过 low/high，甚至 flood stage，但不能超过物理容量。其他 decider 仍会检查 replacement 对应关系、same-shard、version、awareness 和 throttling。

调用入口：

- [`BalancedShardsAllocator.Balancer#decideCanForceAllocateForVacate()`](../../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/BalancedShardsAllocator.java#L1042-L1044)
- [`DesiredBalanceReconciler.Reconciliation#decideCanForceAllocateForVacate()`](../../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/DesiredBalanceReconciler.java#L622-L625)

## 9. 把三个入口放到一次计算中

~~~text
原始统计：
  observedFree = 200GB

共同修正：
  reserved + incoming + stale snapshot
  = 20 + 10 + 5
  = 35GB

canAllocate：
  correctedFree = 200 - 35 = 165GB
  当前 165 >= low 150
  放入后 165 - 80 = 85 < high 100
  -> NO

canRemain：
  correctedFree = 200 - 35 + leaving 30 = 195GB
  195 >= high 100
  -> YES

force replacement：
  correctedFree = 165GB
  放入后 85 >= 0
  -> YES
~~~

结果不同并不矛盾：

- 普通 allocation 保护目标节点不越过 high；
- remain 认可已启动的迁移，避免继续多搬 shard；
- replacement 优先完成节点替换，只保留物理容量底线。

### 9.1 多 data path 的读法

假设：

~~~text
/data-a free = 80GB   <- least available
/data-b free = 300GB  <- most available
~~~

- `canAllocate()` 以 `/data-b` 判断；
- `canRemain()` 以 `/data-a` 判断；
- 但只有实际位于 `/data-a` 的 shard 才继续检查 high；
- 位于 `/data-b` 的 shard 返回 `YES_NOT_MOST_UTILIZED_DISK`。

most/least 按原始 fs free 选出，reserved space 在 path 选定后才扣除；源码不会扣完每条 path 的 reserved 后重新排序。

## 10. 外围组件边界

### 10.1 `DiskThresholdMonitor`

`DiskThresholdDecider` 不主动 reroute，也不设置 flood-stage block。这些行为位于 [`DiskThresholdMonitor#onNewInfo()`](../../server/src/main/java/org/elasticsearch/cluster/routing/allocation/DiskThresholdMonitor.java#L127-L419)：

| 阈值 | decider | monitor |
| --- | --- | --- |
| low | 阻止普通新 shard 进入 | 记录状态，恢复后可能 reroute |
| high | post-high 拒绝；`canRemain=NO` | 触发 automatic reroute |
| flood | 不检查 | 添加 `read_only_allow_delete` |
| frozen flood | 无专用分支 | dedicated frozen 节点告警 |

协作链：

~~~text
新 ClusterInfo
  -> monitor 发现 high 超限
  -> 请求 reroute
  -> allocator 调用 canRemain()
  -> 最拥挤 path 上的 shard 得到 NO
  -> allocator 寻找 canAllocate() 通过的目标
  -> 建立 relocation
~~~

monitor 决定“何时再运行 allocation”，decider 决定“本轮哪些操作允许”。

flood 写保护基于原始 least-available usage；reserved space 参与之后的 high/low reroute 判断，但不会单独触发 flood block。

### 10.2 `balanced`

`BalancedShardsAllocator` 的使用位置：

~~~text
allocateUnassigned()
  -> canAllocate(target)

moveShards()
  -> canRemain(current)
  -> canAllocate(target)

balance()
  -> canAllocate(target)
~~~

磁盘 balance factor 是候选节点权重；`DiskThresholdDecider` 是可行性闸门。权重更优不能覆盖磁盘 `NO`。

### 10.3 `desired_balance`

[`DesiredBalanceComputer#compute()`](../../server/src/main/java/org/elasticsearch/cluster/routing/allocation/allocator/DesiredBalanceComputer.java#L76-L365) 在 simulation 中反复调用 delegate `BalancedShardsAllocator`：

~~~text
克隆 RoutingAllocation
  -> ClusterInfoSimulator
  -> 模拟 initializing shard started
  -> 写入 simulated ClusterInfo
  -> delegateAllocator.allocate()
  -> 继续下一轮
~~~

[`ClusterInfoSimulator`](../../server/src/main/java/org/elasticsearch/cluster/ClusterInfoSimulator.java#L19-L113) 对 relocation 执行 `source free += size`、`target free -= size`；对 new shard 执行 `target free -= size`。

`DesiredBalanceReconciler` 推进真实 routing 时仍会用最新状态重新调用三个磁盘入口，因此后台计算通过不等于真实操作一定执行。

## 11. 排障速查

### 11.1 统计缺失

| 缺失内容 | 行为 |
| --- | --- |
| 整个 usage map 为空 | `YES_USAGES_UNAVAILABLE`，fail-open |
| 当前节点 usage 缺失 | 使用其他节点的平均 total/free |
| incoming actual path 未知 | 算入当前检查 path |
| leaving size 未知 | 按 0，不提前兑现空间 |
| `canRemain()` 当前 shard path 未知 | `YES_NOT_MOST_UTILIZED_DISK` |
| 候选 shard size 未知 | post-high 使用 default `0L` |

缺失位置不同，策略可能保守，也可能 fail-open。

### 11.2 等于阈值

源码大多使用严格 `<`：

| 边界 | 结果 |
| --- | --- |
| current free == low | 普通 low 检查通过 |
| free-after-shard == high | post-high 通过 |
| remain free == high | 允许保留 |
| replacement free-after == 0 | 磁盘决策允许 |
| fresh primary free == high | 不通过特例，因为要求 `free > high` |

### 11.3 快速返回和特殊设置

- `threshold_enabled=false`：三个入口返回 `YES_DISABLED`；monitor 还会清理 flood-stage block。
- `index.routing.allocation.disk.watermark.ignore=true`：目标 index 的磁盘 decision 直接 `YES`，但 monitor 仍按节点状态工作。
- `enable_for_single_data_node` 当前只能为 `true`；配置 `false` 抛出 `SettingsException`。
- `canRemain=NO` 表示 shard 应离开，不代表已找到满足所有 decider 的目标节点。

### 11.4 explain 文案到源码分支

| 关键词 | 分支 |
| --- | --- |
| `disk threshold decider is disabled` | disabled 快速返回 |
| `disk usages are unavailable` | usage map 为空 |
| `disk watermarks are ignored` | index ignore |
| `fewer free bytes ... incoming shards` | corrected free < 0 |
| `above the low watermark` | 当前 free 低于 low |
| `primary shard has never been allocated` | fresh empty primary 特例 |
| `allocating the shard ... above the high` | post-high |
| `not allocated on the most utilized disk` | remain path 不匹配 |
| `shard cannot remain ... high watermark` | remain high |
| `exceed 100%` | replacement 物理容量 |

分析 explain 时依次确认：

1. 当前调用是 allocate、remain 还是 replacement；
2. path 是 most available 还是 least available；
3. free 是否已扣 reserved/incoming；
4. 命中 current low、post high 还是 remain high；
5. expected shard size 是否为 0；
6. 最终聚合结果是否被其他 decider 改变。

## 12. 设置校验与测试索引

`DiskThresholdSettings` 通过 `ClusterSettings.addSettingsUpdateConsumer()` 更新 volatile 字段，动态设置无需重建 `AllocationService`。

`MaxHeadroomValidator` 保证：

- headroom 只能配合比例 watermark；
- 显式 headroom 不能是 `-1`；
- 显式 low headroom 时必须同时设置 high headroom；
- 显式 high headroom 时必须同时设置 flood headroom；
- `flood headroom <= high headroom <= low headroom`。

主要测试入口：

- [`DiskThresholdDeciderUnitTests.java`](../../server/src/test/java/org/elasticsearch/cluster/routing/allocation/decider/DiskThresholdDeciderUnitTests.java)：most/least path、relocation、searchable snapshot、shrink、ignore、replacement 100% 限制。
- [`DiskThresholdDeciderTests.java`](../../server/src/test/java/org/elasticsearch/cluster/routing/allocation/decider/DiskThresholdDeciderTests.java)：percentage、absolute bytes、headroom、unknown usage、snapshot size。
- [`DiskThresholdSettingsTests.java`](../../server/src/test/java/org/elasticsearch/cluster/routing/allocation/DiskThresholdSettingsTests.java)：默认值、动态更新和 validator。
- [`DiskThresholdMonitorTests.java`](../../server/src/test/java/org/elasticsearch/cluster/routing/allocation/DiskThresholdMonitorTests.java)：flood block、reroute interval、auto-release。

## 13. 最终记忆模型

~~~text
设置：
  percentage / bytes / max_headroom
  -> required free bytes

统计修正：
  observed free
  - reserved
  - incoming
  - stale searchable snapshot
  + optional leaving

三个问题：
  allocate -> most path + current low + post high
  remain   -> least path + leaving + current high
  replace  -> most path + physical capacity only

外围推进：
  AllocationDeciders 聚合
  allocator 分配或迁移
  DiskThresholdMonitor 触发 reroute / flood block
~~~

最值得记住的是这组源码不对称：

| 入口 | path | leaving | 阈值 |
| --- | --- | --- | --- |
| `canAllocate()` | 最空闲 | 不加回 | 当前 low + 放入后 high |
| `canRemain()` | 最拥挤 | 加回 | 当前 high |
| replacement force | 最空闲 | 不加回 | 只检查是否超过物理容量 |

沿着这三条线阅读源码，就能解释绝大多数 `disk_threshold` allocation explain 结果。
