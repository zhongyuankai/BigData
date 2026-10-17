# Retention Lease 理解及源码跟踪

## 1. 核心结论

`Retention Lease` 通常译为“保留租约”。在 ES 中，完整名称是 `shard history retention lease`，可译为“分片历史保留租约”或“操作历史保留租约”。

它表达一项有期限、可续约的要求：**某个副本或远端跟随者仍可能需要从指定序列号开始的操作历史，请源分片暂时不要回收。**

- 它保护 Lucene 中的历史内容，不直接负责阻止 Translog 文件删除。
- 它约束 segment merge 对软删除历史的回收，不禁止 merge 本身执行。
- 删除操作通过删除墓碑 `delete tombstone` 留下可重放记录；仅保留被删除文档的旧正文并不足以表达删除操作。
- 必要历史完整时，可以基于操作恢复；历史不足时，普通副本可改走基于文件的恢复。
- 默认 `12h` 是租约期限，不是业务文档 TTL，也不是每条操作固定保留 12 小时。

## 2. 为什么需要保留操作历史

假设主分片继续写入，但副本短暂离线。副本重新加入时，补发缺失操作通常比重新传输大量索引文件更便宜。

问题在于：**当前业务状态与完整操作历史是两种不同的信息。**

- 文档更新后，正常查询只需要最新版本，但恢复可能需要之前发生过的写入。
- 文档删除后，查询中没有该文档，但落后副本必须知道应该执行删除。
- 某个文档先新增再删除，最终状态中没有它，但操作增量恢复仍可能需要这两条历史。

保留全部历史会持续占用磁盘。因此 ES 使用软删除承载必要历史，再使用租约表达哪些历史仍有保留需求。

## 3. 与 Soft Deletes、segment merge、Translog 的关系

| 机制 | 主要职责 | 与租约的关系 |
|---|---|---|
| Lucene 软删除 `Soft Deletes` | 让失效记录对正常查询不可见，同时允许物理数据暂时保留 | 承载可供恢复读取的历史 |
| segment merge | 合并 segment，并在允许时回收失效记录 | 通过保留策略留下仍被需要的软删除记录 |
| Translog | 支持本地崩溃恢复，恢复尚未安全纳入 Lucene 提交的操作 | 有独立的日志代际保留和清理规则 |
| `RetentionLease` | 记录消费者需要的历史下界及续约时间 | 为 Lucene 历史保留策略提供输入 |

### 3.1 租约保护内容，不固定某个 segment 文件

概念模型：

```text
S1、S2 中存在当前文档和历史记录
    ↓ 正常执行 merge，同时应用历史保留策略
S3 中保留当前文档以及仍需要的历史
    ↓ 旧文件不再被引用后
S1、S2 可以被清理
```

因此，“merge 已经执行过”不能推出“需要的删除历史已经丢失”。需要看合并后的历史保留边界。

force merge 也不是绕过历史保留策略、强制清空全部软删除历史的手段。

### 3.2 旧 Translog 被清理，不代表不能进行操作增量恢复

一个操作可以同时存在于 Translog 和 Lucene 中。旧日志满足安全清理条件后，即使 Translog 文件被删除，Lucene 中保留的历史仍可用于副本恢复。

本仓库的 `InternalEngine.newChangesSnapshot()` 创建的是 `LuceneChangesSnapshot`，历史数据来自 Lucene。

虽然返回类型是 `Translog.Snapshot`，操作对象也叫 `Translog.Index`、`Translog.Delete`，但这些是接口与操作表示，**类型名称不能用来判断数据一定从 Translog 文件读取。**

## 4. 一份租约记录什么

`RetentionLease` 主要包含四项信息：

| Java 字段或访问方法 | 含义 |
|---|---|
| `id` / `id()` | 租约标识，用于区分不同保留需求 |
| `retainingSequenceNumber` / `retainingSequenceNumber()` | 保留起点，包含这个序列号 |
| `timestamp` / `timestamp()` | 创建或最近一次续约时间 |
| `source` / `source()` | 来源，例如 `peer recovery`、`ccr` |

假设起点为 `101`，租约要求保留 `seq_no >= 101` 的历史。序列号属于单个分片的操作序列，不是全集群统一编号。

保留起点是一条安全下界，可能比较保守，不应直接当成消费者实时、精确的处理进度。

**字段名区分：**`RetentionLease` 自身的内容序列化使用 `retaining_sequence_number`；分片 `_stats` 中的 `RetentionLeaseStats` 使用 `retaining_seq_no`。排查 `_stats` 时应读取后者。

### 4.1 多份租约共同决定需求

| 消费者 | 保留起点 |
|---|---:|
| 副本 A | `101` |
| 副本 B | `151` |
| 远端跟随者 C | `81` |

仅从租约看，需要满足最早的需求：`min(101, 151, 81) = 81`。

当 C 将起点推进到 `181`，租约要求的下界可推进到 `101`。但实际清理边界还要满足安全提交点等条件，不能直接等同于这个最小值。

## 5. 创建、续约、同步与过期

### 5.1 创建及续约

`ReplicationTracker` 在主分片模式下管理租约集合。新增租约后触发同步；普通续约更新集合，后续可通过后台同步传播到副本。

续约包含两个维度：

1. 更新时间：表示仍然需要保留历史。
2. 推进起点：表示更早的历史已经不再需要。

两者不必同时变化。因此，时间持续更新但起点长期不动，可能说明消费者仍在维持租约，但复制进度没有推进。

同一份租约续约时，起点不能后退。已从 `101` 推进到 `201`，不能再通过普通续约改回 `51`；之前的历史可能已经被回收。

普通 peer recovery 租约的 ID 形如 `peer_recovery/<node-id>`，来源为 `peer recovery`。实现会依据所跟踪的检查点信息保守地推进起点，并不是每收到一条复制操作就立即更新一次租约。

### 5.2 同步与持久化

新增或移除租约使用主动同步路径；续约信息由后台同步路径传播。`RetentionLeaseSyncAction` 与 `RetentionLeaseBackgroundSyncAction` 对同步失败的处理语义不同：前者使用写操作同步语义，后者容忍漏掉个别后台续约同步。

租约会持久化到分片状态路径中的独立状态文件。集合包含 `primaryTerm` 和 `version`，用于比较状态新旧；它不是仅存在于原主分片内存中的临时对象。

后台调用链：

```text
IndexService.AsyncRetentionLeaseSyncTask.runInternal()
  → IndexService.syncRetentionLeases()
  → IndexShard.syncRetentionLeases()
      → ReplicationTracker.renewPeerRecoveryRetentionLeases()
      → getRetentionLeases(true)：检查过期
      → RetentionLeaseSyncer.backgroundSync()
```

本仓库后台同步任务的默认间隔为 30 秒。源码中的 `index.soft_deletes.retention_lease.sync_interval` 明确未作为常规设置注册，仅用于测试，不应把它当成生产环境可直接配置的公开参数。

### 5.3 默认 12 小时的准确含义

公开索引设置 `index.soft_deletes.retention_lease.period` 默认 `12h`，支持动态更新。

对于通常按时间判断的租约，条件是：

```text
当前时间 - 最近创建或续约时间 > 租约期限
```

这不表示每条历史固定保留 12 小时。持续续约且起点不推进的租约，可能保护比 12 小时更老的历史；起点不断推进时，较新的历史也可能较早解除保护。

普通 peer recovery 租约还有额外规则：

- 与当前仍分配的、可提升为主分片的副本对应的租约，不会仅因时间到期而删除。
- 所有分片启动后，不再使用的旧 peer recovery 租约可以被清理。
- 在特定条件下，如果旧租约要求保留的历史过多，超出实现判断的合理范围，也可以提前放弃，转而允许文件恢复。
- 对仍分配的副本，续约实现会考虑半个租约期限等条件，并不保证掉线瞬间恰好刚完成续约。

所以不能把 `12h` 理解成“每次掉线后都保证拥有完整的 12 小时增量恢复窗口”。

租约过期是解除相应保留要求，历史是否实际消失还取决于其他约束及后续 merge。重新创建租约也不能恢复已经被清理的数据。

## 6. 删除操作如何保留：旧文档与删除墓碑

假设 B 原来的写入序列号是 `91`，随后在 `102` 删除 B。

此时需要区分：

| 记录 | 表达的操作 | 为什么不同 |
|---|---|---|
| B 的旧文档，序列号 `91` | 当时写入 B | 不能单凭它表达后来发生的删除 |
| B 的删除墓碑，序列号 `102` | 在 `102` 删除 B | 可以重建删除操作，通知落后副本删除 B |

`InternalEngine.deleteInLucene()` 的关键步骤：

```java
final ParsedDocument tombstone = ParsedDocument.deleteTombstone(delete.id());
tombstone.updateSeqID(delete.seqNo(), delete.primaryTerm());
tombstone.version().setLongValue(plan.versionOfDeletion);
final LuceneDocument doc = tombstone.docs().get(0);
doc.add(softDeletesField);
```

随后根据删除策略，选择添加墓碑，或通过 `IndexWriter.softUpdateDocument()` 将原记录软删除并写入墓碑。

概念上的删除墓碑包含文档 ID、操作序列号、主分片任期、版本及墓碑标记。它不需要保留被删除文档的完整业务正文，本身也被标记为软删除，因此不作为业务文档出现在正常查询中。

读取操作历史时，内部读取器能够访问仍然保留的软删除记录，并将删除墓碑转换为：

```java
new Translog.Delete(id, seqNo, primaryTerm, version);
```

这也是为什么普通业务查询看不到 B，并不意味着系统无法重放删除 B。

## 7. 完整例子：副本离线期间发生更新与删除

### 7.1 初始条件

主分片 P 与副本 R 已同步到 `seq_no=100`。为便于说明，假设恢复检查允许 R 从 `101` 开始追赶，源端已有有效租约覆盖 `101` 及之后的操作。真实起点由恢复协议和检查点决定。

两边初始业务数据：

```text
A = apple-v1    原写入序列号 90
B = banana      原写入序列号 91
C = cherry      原写入序列号 92
```

### 7.2 R 离线，P 继续写入

| 序列号 | 操作 | P 的当前可见业务数据 |
|---:|---|---|
| `101` | 更新 A 为 `apple-v2` | A=v2、B、C |
| `102` | 删除 B | A=v2、C |
| `103` | 新增 D | A=v2、C、D |
| `104` | 删除 D | A=v2、C |

### 7.3 merge 时哪些内容需要保留

| 序列号 | Lucene 记录 | 正常查询是否可见 | 在此例中的保留需求 |
|---:|---|---|---|
| `90` | A=v1 的旧写入 | 否 | 租约不要求，其他条件允许时可清理 |
| `91` | B 的旧写入 | 否 | 租约不要求，其他条件允许时可清理 |
| `92` | C 的当前写入 | 是 | 作为当前有效文档保留，不因低于租约起点而删除 |
| `101` | A=v2 的写入 | 是 | 保留 |
| `102` | 删除 B 的墓碑 | 否 | 保留，用于重放删除 |
| `103` | D 的写入 | 否 | 保留，用于补齐操作历史 |
| `104` | 删除 D 的墓碑 | 否 | 保留，用于重放删除 |

租约下界约束的是可回收的历史，不会将所有低于下界的当前有效文档一并删除。

B 的旧正文可以被回收，但 B 的删除墓碑仍然保留。副本原本已有 B，只需要收到删除操作，不需要从源端重新读取 B 的旧正文。

D 当前虽然不存在，但本次按操作恢复仍需要覆盖它的新增和删除，不能因为最终没有 D 就任意跳过历史序列。

### 7.4 R 回来，基于操作恢复

源端从 Lucene 历史重建操作，副本按序列号及任期等恢复语义应用这些操作。下面按逻辑顺序展示效果：

```text
R 初始：A=v1、B、C
  应用 101：INDEX A=v2 → A=v2、B、C
  应用 102：DELETE B   → A=v2、C
  应用 103：INDEX D    → A=v2、C、D
  应用 104：DELETE D   → A=v2、C
```

最终 P、R 的业务状态一致。副本本地处理收到的操作时，也会维护自己的引擎状态和 Translog；“源端历史来自 Lucene”不等于“目标端不使用 Translog”。

### 7.5 租约失效且历史已清理，改走文件恢复

假设源端已无法提供从 `101` 开始的完整历史。此时不能只把当前 A、C 追加到副本原索引，否则副本的旧 B 可能残留。

普通 peer recovery 会在前置决策中选择文件恢复路径：

1. 获取源端安全提交，作为索引状态基线。
2. 传输或复用基线所需的 Lucene 文件，使目标获得该提交对应的索引状态。
3. 打开目标引擎，重放基线之后需要补齐的操作，并完成后续同步。

删除不会因此丢失：

- 如果基线已经包含删除 B 的结果，目标从基线恢复出来时 B 就不可见。即使旧 B 的物理数据还在文件里，删除可见性状态也会随提交恢复。
- 如果基线早于删除 B，删除操作属于之后必须补齐的历史，随后重放删除。

文件恢复不是简单地往目标原有业务文档集合追加数据，也不一定每次都传输整个分片：协议可以复用匹配文件。

历史不足时应选择合适恢复路径；运行中出现意外缺失仍可能导致本次恢复失败并重试，不能假定任何中途异常都能在原请求内无缝切换。

## 8. 源码跟踪：建议阅读顺序

### 8.1 租约是什么、由谁维护

1. `RetentionLease`：四个字段及类注释，明确要求保留的序列号范围。
2. `RetentionLeases`：集合的 `primaryTerm`、`version` 及状态序列化。
3. `ReplicationTracker.addRetentionLease()`：新增并触发同步。
4. `ReplicationTracker.renewRetentionLease()`：检查起点不能倒退，更新时间及版本。
5. `ReplicationTracker.renewPeerRecoveryRetentionLeases()`：peer recovery 租约的推进和续约时机。
6. `ReplicationTracker.getRetentionLeases(true)`：主分片执行过期判断，留意 peer recovery 特殊分支。

### 8.2 租约如何影响 merge

```text
租约集合
  → SoftDeletesPolicy.getMinRetainedSeqNo()
  → SoftDeletesPolicy.getRetentionQuery()
  → InternalEngine 创建 IndexWriter 时安装 SoftDeletesRetentionMergePolicy
  → merge 保留命中查询的软删除历史
```

未持有历史保留锁时，`getMinRetainedSeqNo()` 的计算可概括为：

```text
L = 所有租约保留起点的最小值；没有租约时取 Long.MAX_VALUE
G = 全局检查点
O = retentionOperations
C = 安全提交的 local checkpoint

候选下界 = min(G + 1 - O, L, C + 1)
新的实际下界 = max(旧的实际下界, 候选下界)
```

含义：同时满足租约、检查点和额外保留操作数等约束，并防止下界后退。持有历史保留锁时，不推进这个下界。

最后生成 `_seq_no` 的范围查询，从实际下界到 `Long.MAX_VALUE`。**新增一个要求更老历史的租约，不会使已经前进的实际下界倒退，更不能找回已经丢失的历史。**

### 8.3 删除如何写入、如何重建

```text
InternalEngine.deleteInLucene()
  → ParsedDocument.deleteTombstone(id)
  → 设置 seq_no、primary_term、version
  → 墓碑标记软删除
  → 写入 Lucene；必要时将旧记录软删除

LuceneChangesSnapshot
  → 使用内部 reader 访问保留的历史
  → 按序列号范围检索并排序
  → readDocAsOp()
      → 普通记录：Translog.Index
      → 有 ID 的墓碑：Translog.Delete
      → 无 ID 的 noop 墓碑：Translog.NoOp
```

`LuceneChangesSnapshot` 使用 `Lucene.wrapAllDocsLive()` 构造历史读取视图，不等同于正常业务搜索的可见文档视图。相关数据若已在 merge 中物理移除，内部读取器也无法恢复它。

### 8.4 恢复如何选择操作路径或文件路径

`RecoverySourceHandler.recoverToTarget()` 先获取历史保留锁，再检查关键条件：

```text
startingSeqNo 有效
  AND 源、目标属于相同历史
  AND hasCompleteHistoryOperations(startingSeqNo)
  AND 租约覆盖起点（或符合不使用租约的兼容路径）
```

条件成立时，可以选择基于操作的恢复；否则进入文件恢复路径。

有效租约并非历史完整性的唯一证明，所以仍需检查 `hasCompleteHistoryOperations()`。本仓库 `InternalEngine` 的该判断依据实际的最小保留序列号。

文件路径取得安全提交后，将后续重放起点设置为该提交的 `local checkpoint + 1`。两条路径随后进入操作发送阶段：

```text
RecoverySourceHandler
  → IndexShard.newChangesSnapshot()
  → InternalEngine.newChangesSnapshot()
  → LuceneChangesSnapshot
  → RecoverySourceHandler.phase2()
  → recoveryTarget.indexTranslogOperations()
  → RecoveryTarget.indexTranslogOperations()
  → IndexShard.applyTranslogOperation(..., PEER_RECOVERY)
```

这里的 `phase2`、`indexTranslogOperations` 和 `Translog.Snapshot` 仍沿用日志操作相关命名，不应据此误判源端数据来源。

恢复期间先获取历史保留锁；有覆盖所需历史的租约后，部分路径可以释放该锁，由租约继续保护历史。租约与保留锁是不同机制。

### 8.5 对照 Translog 的清理入口

从 `Translog.trimUnreferencedReaders()` 跟到 `getMinReferencedGen()`，可以看到最小保留日志代际主要取决于：

```text
min(
    仍被 Translog 快照等引用锁需要的最早代际,
    安全提交 local checkpoint + 1 所需的日志代际
)
```

这与 `SoftDeletesPolicy` 使用租约计算 Lucene 历史下界是两条不同路径。不要将 Translog 的引用锁、历史保留锁和历史保留租约混为一谈。

## 9. 源码导航与断点建议

以下链接定位本次核对的本地源码；行号会随后续修改变化，长期跟踪时以类名、方法名为准。

| 关注点 | 源码入口 | 建议观察 |
|---|---|---|
| 租约数据结构 | [RetentionLease.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/main/java/org/elasticsearch/index/seqno/RetentionLease.java:24) | 起点、来源、时间 |
| 创建租约 | [ReplicationTracker.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/main/java/org/elasticsearch/index/seqno/ReplicationTracker.java:308) | 新集合、同步回调 |
| 续约 | [ReplicationTracker.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/main/java/org/elasticsearch/index/seqno/ReplicationTracker.java:396) | 新旧起点、时间、版本 |
| peer recovery 续约 | [ReplicationTracker.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/main/java/org/elasticsearch/index/seqno/ReplicationTracker.java:578) | 分配状态、检查点、续约阈值 |
| 过期判断 | [ReplicationTracker.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/main/java/org/elasticsearch/index/seqno/ReplicationTracker.java:240) | 当前路由、是否全部启动、时间差 |
| 后台同步 | [IndexShard.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/main/java/org/elasticsearch/index/shard/IndexShard.java:2986) | 续约、过期、后台同步的顺序 |
| 实际保留边界 | [SoftDeletesPolicy.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/main/java/org/elasticsearch/index/engine/SoftDeletesPolicy.java:100) | 最小租约起点、检查点、旧下界、锁计数 |
| merge policy 接入 | [InternalEngine.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/main/java/org/elasticsearch/index/engine/InternalEngine.java:2830) | `SoftDeletesRetentionMergePolicy` 的查询供应者 |
| 构造删除墓碑 | [ParsedDocument.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/main/java/org/elasticsearch/index/mapper/ParsedDocument.java:71) | ID、墓碑元数据 |
| 执行删除 | [InternalEngine.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/main/java/org/elasticsearch/index/engine/InternalEngine.java:1925) | 墓碑、软删除字段、写入分支 |
| 创建历史快照 | [InternalEngine.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/main/java/org/elasticsearch/index/engine/InternalEngine.java:3238) | 内部搜索器、读取范围 |
| 重建操作 | [LuceneChangesSnapshot.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/main/java/org/elasticsearch/index/engine/LuceneChangesSnapshot.java:309) | 墓碑类型、ID、重建的操作 |
| 恢复路径选择 | [RecoverySourceHandler.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/main/java/org/elasticsearch/indices/recovery/RecoverySourceHandler.java:210) | 起点、历史一致性、租约覆盖 |
| 接收并应用恢复操作 | [RecoveryTarget.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/main/java/org/elasticsearch/indices/recovery/RecoveryTarget.java:423) | `PEER_RECOVERY` 操作及结果 |
| Translog 清理 | [Translog.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/main/java/org/elasticsearch/index/translog/Translog.java:1665) | 最早被引用代际、安全提交检查点 |
| 统计输出字段 | [RetentionLeaseStats.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/main/java/org/elasticsearch/index/seqno/RetentionLeaseStats.java:77) | `retaining_seq_no` |

优先断点组合：`deleteInLucene()` → `getMinRetainedSeqNo()` → `recoverToTarget()` → `readDocAsOp()` → `RecoveryTarget.indexTranslogOperations()`。这样可以把一次删除从源端写入一直跟到目标应用。

## 10. CCR 与 DiDi DCDR 的适用边界

CCR 的 follower 拉取 leader 的操作，租约用于保留它仍需要的历史。如果租约失效且必要历史已被清理，后续跟随可能因历史不完整而失败，需要重新引导或重建 follower。不能照搬普通副本的文件恢复结论，声称 CCR 会自动无缝全量恢复。

本仓库 [DcdrRecoverySourceHandler.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/modules/didi-cross-datacenter-replication/src/main/java/org/elasticsearch/dcdr/recovery/DcdrRecoverySourceHandler.java:202) 也检查恢复起点、历史一致性、完整历史及租约覆盖情况，并通过 `newChangesSnapshot()` 读取历史。

这说明 DCDR 的这段恢复路径复用了类似的保留与恢复基础设施，不代表 DCDR 的全部跨集群复制行为都等同于上游 CCR。远端角色、复制关系及重试重建行为需要沿 DCDR 自身调用链继续确认。

## 11. 运行时查看与排查

### 11.1 查看租约及设置

```http
GET /your-index/_stats?level=shards

GET /your-index/_settings?include_defaults=true&flat_settings=true
```

在分片统计中查看 `retention_leases`。下面仅展示相关局部结构，示例数值不代表真实集群：

```json
{
  "retention_leases": {
    "primary_term": 3,
    "version": 20,
    "leases": [
      {
        "id": "peer_recovery/example-node-id",
        "retaining_seq_no": 101,
        "timestamp": 1730000000000,
        "source": "peer recovery"
      }
    ]
  }
}
```

`timestamp` 为毫秒时间戳。主、副本的瞬时统计可能因同步时机不同而有差异，分析过期决策时优先结合主分片状态。

### 11.2 常见现象如何理解

| 现象 | 可得出的线索 | 需要继续核对 |
|---|---|---|
| 时间更新，起点长期不动 | 租约仍在维持，保留需求未前移 | 复制任务进度、远端错误、检查点 |
| 租约起点很老，磁盘持续增加 | 旧历史可能受保护 | 更新删除量、其他租约、实际保留边界、merge 状态 |
| 有租约却走文件恢复 | 租约本身不足以证明能增量恢复 | 起点、历史 UUID、实际历史覆盖范围 |
| 租约过期但磁盘没立刻下降 | 解除保护不等于立即回收 | 其他保留条件、merge、文件引用 |
| Translog 变小，但仍能增量恢复 | 历史可能由 Lucene 提供 | `newChangesSnapshot()` 及实际恢复路径 |
| 普通查询没有文档，但仍能重放删除 | 墓碑对业务查询不可见，内部历史仍可读取 | 墓碑及对应序列号是否保留 |

更新或删除频繁的分片上，落后消费者会让更多软删除历史无法回收，增加磁盘占用及合并成本。延长租约期限只是扩大失联期间保留历史的机会，不能修复已经缺失的历史，也不能解决消费者长期无法追赶的问题。

## 12. 相关测试与继续阅读

可先阅读已有测试理解边界：

- [SoftDeletesPolicyTests.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/test/java/org/elasticsearch/index/engine/SoftDeletesPolicyTests.java)：重点看 `testWhenRetentionLeasesDictateThePolicy()`、`testWhenGlobalCheckpointDictatesThePolicy()`、`testWhenLocalCheckpointOfSafeCommitDictatesThePolicy()` 和 `testSoftDeletesRetentionLock()`。
- [ReplicationTrackerTests.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/test/java/org/elasticsearch/index/seqno/ReplicationTrackerTests.java:1044)：从 `testPeerRecoveryRetentionLeaseCreationAndRenewal()` 跟踪创建和续约。
- [LuceneChangesSnapshotTests.java](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/server/src/test/java/org/elasticsearch/index/engine/LuceneChangesSnapshotTests.java)：结合历史读取、操作转换和范围检查阅读。

如需验证，可按仓库要求配置 JDK，并只执行最小相关范围；本文整理未运行这些测试：

```bash
./gradlew :server:test --tests org.elasticsearch.index.engine.SoftDeletesPolicyTests
./gradlew :server:test --tests org.elasticsearch.index.engine.LuceneChangesSnapshotTests
./gradlew :server:test --tests 'org.elasticsearch.index.seqno.ReplicationTrackerTests.testPeerRecoveryRetentionLeaseCreationAndRenewal'
```

参考资料：

- [Elasticsearch 8.12：History retention](https://www.elastic.co/guide/en/elasticsearch/reference/8.12/index-modules-history-retention.html)
- [Elasticsearch 8.12：Translog](https://www.elastic.co/guide/en/elasticsearch/reference/8.12/index-modules-translog.html)
- [Elasticsearch 8.12：Cross-cluster replication](https://www.elastic.co/guide/en/elasticsearch/reference/8.12/xpack-ccr.html)
- [仓库内历史保留说明](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/docs/reference/index-modules/history-retention.asciidoc)
- [仓库内 Translog 说明](/Users/didi/zyk/code/IDEAProjects/elasticsearch-didi/docs/reference/index-modules/translog.asciidoc)
