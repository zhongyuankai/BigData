# LRUQueryCache 实现跟踪

## 1. 分析范围与核心作用

本文跟踪本地 `lucene-didi` 仓库提交 `79d0f05ced9b679a0e7514b88fb290848bfdc61a` 的实现，不将其他 Lucene 版本的行为混入本文。源码根目录为 `/Users/didi/zyk/code/IDEAProjects/lucene-didi`。下文源码位置均相对于该目录；为便于阅读，Java 源文件统一使用 `lucene/core/src/java/org/apache/lucene/` 前缀，测试使用 `lucene/core/src/test/org/apache/lucene/` 前缀。

`LRUQueryCache` 解决的是重复执行无评分查询时，反复计算相同匹配集合的开销。它保存的是 **Query 在某个 Segment core 上匹配的局部 doc ID 集合与数量**，不保存最终 TopN、文档内容或相关性分数。

整体过程是：

```text
IndexSearcher.createWeight
  → 对不需要评分的 Weight 调用 LRUQueryCache.doCache
  → CachingWrapperWeight
  → 按 Segment 检查资格、查找缓存
      → 命中：从 DocIdSet 创建无评分的 Scorer
      → 未命中且值得缓存：完整收集匹配集合，写入缓存
          → 插入后检查容量，按 Query 淘汰所有 Segment 上的对应集合
      → 不适合缓存：委托原 Weight 执行
  → 外层搜索用当前 liveDocs 排除删除文档
```

核心入口证据：`search/IndexSearcher.java:898`、`search/LRUQueryCache.java:474`、`search/LRUQueryCache.java:724`、`search/IndexSearcher.java:763`。

## 2. 贯穿案例：重复使用价格范围过滤

设搜索请求包含一个需要评分的关键词条件，以及过滤条件 Q：

```text
MUST   关键词查询（每次请求可以不同）
FILTER Q = LongPoint.newRangeQuery("price", 100, 200)
```

固定以下条件，避免把示意过程误当作所有查询都必然发生的行为：

- 索引有两个 Segment：S1 的 `maxDoc = 20000`，S2 的 `maxDoc = 30000`；没有删除文档。
- 使用 `new LRUQueryCache(2, 1024 * 1024)` 和默认 `UsageTrackingQueryCachingPolicy`。
- 请求连续执行，每次重新创建 Weight；历史窗口中没有其他查询干扰，也不发生锁竞争。
- Q 的 Weight 在两个 Segment 都可缓存，并提供 core cache helper；执行没有触发成本跳过。
- 为解释集合表示，假设 Q 在 S1 上的 scorer 估算 cost 为 400，并实际匹配 400 个文档；S2 上估算 cost 为 100，实际匹配 100 个文档。

选择 Point 范围查询有一个实际原因：默认策略把它视为成本较高的查询，近期使用两次便允许缓存；普通 `TermQuery` 则被默认策略明确排除。前一轮用“状态 = 上架”说明过滤复用的原理，如果它实际是 `TermQuery`，还需要自定义缓存策略才会单独缓存。

证据：`search/UsageTrackingQueryCachingPolicy.java:33`、`search/UsageTrackingQueryCachingPolicy.java:56`、`search/UsageTrackingQueryCachingPolicy.java:124`。

## 3. 初始化：两个维度的缓存与一个全局 LRU

### 3.1 配置的含义

| 配置 | 当前实现的含义 |
| --- | --- |
| `maxSize` | 最多保存多少个不同 Query，不是 `(Query, Segment)` 条目数 |
| `maxRamBytesUsed` | 缓存记账的内存上限，插入后检查并淘汰 |
| `leavesToCache` | 判断当前 Segment 是否值得缓存 |
| `skipCacheFactor` | 在 `ScorerSupplier.get(leadCost)` 路径控制是否值得完整构建集合 |

两参数构造器实际使用 `MinSegmentSizePredicate(10000)` 和 `skipCacheFactor = 10`。Segment 必须同时满足：

```text
segment.maxDoc >= 10000
segment.maxDoc * 2 > 顶层 reader.maxDoc / Segment 数量
```

除法是整数除法，使用的都是 `maxDoc`，包括尚未物理移除的删除文档。案例中平均值为 25000，S1、S2 均通过。

另外，`shouldCache(context)` 会估算整个顶层 reader 的最坏 bitset 大小，并要求：

```text
(topLevelReader.maxDoc / 8) * 5 < maxRamBytesUsed
```

案例中为 `50000 / 8 * 5 = 31250 < 1048576`。这是防止单次构建挤爆缓存的启发式门槛，不是精确对象内存测量。

证据：`search/LRUQueryCache.java:118`、`search/LRUQueryCache.java:147`、`search/LRUQueryCache.java:160`、`search/LRUQueryCache.java:695`、`search/LRUQueryCache.java:717`。

`IndexSearcher` 自带的默认实例允许 1000 个 Query，内存取 32 MiB 与最大堆的 5% 中较小者；这与案例的自定义参数不同。证据：`search/IndexSearcher.java:92`。

### 3.2 为什么有三张 Map

```text
uniqueQueries: LinkedHashMap<Query, Query>，按访问顺序排列
  Q 的等价对象 → Q 的规范实例

cache: IdentityHashMap<CoreCacheKey, LeafCache>
  S1.coreKey → LeafCache(S1)
  S2.coreKey → LeafCache(S2)

LeafCache.cache: IdentityHashMap<Query, CacheAndCount>
  Q 的规范实例 → { DocIdSet, count }
```

`uniqueQueries` 使用 Query 的 `equals/hashCode` 识别等价查询，并返回统一的实例。LeafCache 随后按对象身份查找，避免在每个 Segment 中持有等价 Query 的多个副本。`mostRecentlyUsedQueries` 是第一张 Map 的 `keySet` 视图，并非另一个独立维护的集合。

这里的不变式是：任意 LeafCache 中的 Query 都必须属于全局 Query 集合。因此插入、淘汰、清理必须用同一把锁同步。

证据：`search/LRUQueryCache.java:88`、`search/LRUQueryCache.java:132`、`search/LRUQueryCache.java:292`、`search/LRUQueryCache.java:623`。

## 4. 入口：为什么关键词仍评分，过滤条件可以缓存

`IndexSearcher.createWeight` 先调用 Query 自身的 `createWeight`，随后只有在 `scoreMode.needsScores() == false` 且缓存非空时，才调用 `queryCache.doCache`。

外层 BooleanQuery 即使需要评分，`BooleanWeight` 创建子 Weight 时也会将非评分子句切换为 `COMPLETE_NO_SCORES`。因此案例的关键词沿评分路径执行，Q 可以进入缓存包装；`MUST_NOT` 同样属于非评分子句。

`ConstantScoreQuery` 则通过无评分模式创建内部 Query 的 Weight，再按外层需要提供常量分数。缓存所服务的始终是内部匹配集合。

证据：`search/IndexSearcher.java:898`、`search/BooleanWeight.java:52`、`search/ConstantScoreQuery.java:125`。

`doCache` 本身不遍历文档。它先剥离已有的 `CachingWrapperWeight`，再创建新的包装 Weight，实际动作延迟到具体 Segment 执行时。`matches` 仍委托原 Weight，并不从 bitset 还原匹配位置信息。

证据：`search/LRUQueryCache.java:474`、`search/LRUQueryCache.java:683`、`search/LRUQueryCache.java:691`。

## 5. 第一次请求：先统计使用，再决定是否建立缓存

以 `CachingWrapperWeight.scorerSupplier(context)` 为主线，其执行顺序如下。

### 5.1 每个包装 Weight 只报告一次使用

`used.compareAndSet(false, true)` 成功后调用 `policy.onUse(getQuery())`。同一包装 Weight 被两个 Segment、多个线程或不同入口访问，不会重复报告使用。

这不是“每个 Segment 访问一次就增加一次查询频次”。案例第一次请求处理 S1 时 Q 频次变成 1，随后处理 S2 不再增加；第二次请求创建新的包装 Weight 后才增加为 2。

默认策略只在大小为 256 的近期使用窗口里记录 Query 的 hash code，不持有 Query 引用。它使用同步块保护频率环形缓冲区。Hash 碰撞最多使某个查询提前达到缓存门槛，不会让缓存返回另一个查询的结果，因为真正缓存仍依赖 Query 等价判断。

默认门槛：成本较高的查询为 2，BooleanQuery / DisjunctionMaxQuery 为 4，其他可缓存查询为 5。TermQuery、FieldExistsQuery、MatchAllDocsQuery、MatchNoDocsQuery 以及空的上述复合查询被排除。

证据：`search/LRUQueryCache.java:675`、`search/LRUQueryCache.java:724`、`search/UsageTrackingQueryCachingPolicy.java:95`、`search/UsageTrackingQueryCachingPolicy.java:124`、`search/UsageTrackingQueryCachingPolicy.java:143`。

### 5.2 查缓存前的快速退出

| 检查顺序 | 条件 | 行为 |
| --- | --- | --- |
| 1 | 原 Weight 的 `isCacheable(context)` 为 false | 使用原 supplier |
| 2 | 内存估算或 Segment 筛选未通过 | 使用原 supplier |
| 3 | 无 `getCoreCacheHelper()` | 使用原 supplier |
| 4 | `lock.tryLock()` 失败 | 不等待查询锁，使用原 supplier |

只有通过这些条件才调用 `get`。因此被这些分支跳过的访问不会计入缓存的 hit / miss。

证据：`search/LRUQueryCache.java:729`、`search/LRUQueryCache.java:736`、`search/LRUQueryCache.java:740`、`search/LRUQueryCache.java:747`。

### 5.3 get 的两层查找与 LRU 更新

`get` 先按 core key 找 LeafCache。没有则立即 miss。存在时再调用 `uniqueQueries.get(key)`，找到规范 Query 并将其移到 LRU 队尾，然后在 LeafCache 中按身份查匹配集合。

有一个细节：LeafCache 存在、全局 Query 存在，但该 Query 尚未在此 Segment 建立集合时，也会更新全局 LRU，随后报告 miss；若 LeafCache 根本不存在，则不会走到 LRU 更新。

案例第一次执行时两个 Segment 都 miss。Q 的使用频次仅为 1，`policy.shouldCache(Q)` 返回 false，于是调用原 supplier 执行，不插入任何缓存。

证据：`search/LRUQueryCache.java:267`、`search/LRUQueryCache.java:758`。

## 6. 第二次请求：构建完整集合并写入

### 6.1 成本门槛：不能为缓存把一个便宜查询拖慢

第二次执行 Q 时频次达到 2。未命中后策略允许缓存，代码先取得原 supplier，然后返回一个新的 supplier；此时集合还没有生成。

真正调用这个 supplier 的 `get(leadCost)` 时才判断：

```java
if (cost / skipCacheFactor > leadCost) {
  return supplier.get(leadCost);
}
```

假设 Q 的 cost 为 400，默认 factor 为 10，而其他条件使调用方传入的 `leadCost` 仅为 10，那么 `40 > 10`，本次跳过缓存。原因是组合查询可能只需检查少量候选文档，而建立缓存需要完整遍历 Q 的匹配。

通过后调用 `supplier.get(Long.MAX_VALUE)`，以 `DefaultBulkScorer` 完整收集集合。案例假设通过该门槛。直接使用包装 Weight 的 `scorer(context)` 会以 `Long.MAX_VALUE` 请求 supplier；成本跳过主要发生在调用方传入有限 `leadCost` 的执行中。

如果原 supplier 为 null，则直接存入 `CacheAndCount.EMPTY` 并返回 null，后续也可以命中“没有结果”。

证据：`search/LRUQueryCache.java:759`、`search/LRUQueryCache.java:769`、`search/LRUQueryCache.java:824`。

### 6.2 按估算 cost 选择集合格式

`cacheImpl` 以 `scorer.cost() * 100 >= maxDoc` 选择表示，并非先算实际命中率再选择。

| Segment | 估算 cost | maxDoc | 选择 |
| --- | --- | --- | --- |
| S1 | 400 | 20000 | `40000 >= 20000`，`FixedBitSet`，外包 `BitDocIdSet` |
| S2 | 100 | 30000 | `10000 < 30000`，`RoaringDocIdSet` |

bitset 适合较密集集合并支持快速随机访问；Roaring 适合较稀疏集合。两条构建路径都只收集 doc ID，不使用 score，并同时产生数量 `count`。构建时传入 `acceptDocs = null`，即集合不主动过滤当前删除标记。

证据：`search/LRUQueryCache.java:501`、`search/LRUQueryCache.java:511`、`search/LRUQueryCache.java:530`、`search/LRUQueryCache.java:949`。

### 6.3 putIfAbsent 的状态变化

构建完成后，`putIfAbsent` 在锁内依次执行：

1. 向 `uniqueQueries` 登记 Q；如果已有等价 Query，改用已有规范实例。
2. 按 core key 找到或创建 LeafCache。
3. 新建 LeafCache 时向 core cache helper 注册 `clearCoreCacheKey` 关闭监听器。
4. 在 LeafCache 中按规范 Q 插入集合；只有真正插入才增加条目数和内存。
5. 调用 `evictIfNecessary()`，把超限缓存降下来。

案例先在 S1 插入 Q，再在 S2 插入 Q。最终全局 Query 数是 1，而 `cacheSize` 是 2。两者不能混淆。

证据：`search/LRUQueryCache.java:292`、`search/LRUQueryCache.java:651`。

缓存建立后，当前请求也直接使用刚刚生成的集合，返回 `ConstantScoreScorer(..., 0f, COMPLETE_NO_SCORES, iterator)`。

证据：`search/LRUQueryCache.java:775`。

## 7. 第三次请求与其他执行入口

### 7.1 命中时只生成迭代器

第三次请求在每个 Segment 找到 `CacheAndCount`，增加 hit，从集合取迭代器并创建无评分 Scorer；不再调用原 Weight 重新计算 Q 的完整集合。缓存是 `EMPTY` 或 iterator 为 null 时返回 null，表示该 Segment 无匹配。

`policy.shouldCache` 仅在 miss 后调用，因此策略后来不再允许新建缓存，也不会禁止复用已存在的条目。

证据：`search/LRUQueryCache.java:758`、`search/LRUQueryCache.java:800`；测试 `search/TestLRUQueryCache.java:707` 的第三轮将策略切为 `NEVER_CACHE`，仍断言旧条目命中。

### 7.2 bulkScorer 路径并不经过成本倍数判断

普通 Segment 搜索从 `IndexSearcher` 调用 `weight.bulkScorer(ctx)`。包装类覆盖了这个入口，也有自己的使用登记、资格检查、tryLock 查找流程。

miss 且策略允许时，它直接调用 `cache(context)`，后者调用原 `in.bulkScorer(context)` 并生成集合，然后写入缓存。最后使用缓存迭代器包装 `DefaultBulkScorer`。

**这条路径没有 `cost / skipCacheFactor > leadCost` 的判断。** 不能把 supplier 路径的跳过规则说成所有入口统一执行的规则。

证据：`search/IndexSearcher.java:763`、`search/LRUQueryCache.java:707`、`search/LRUQueryCache.java:891`。

### 7.3 count 路径只查已有缓存

`count(context)` 按下面的顺序处理：

1. 先调用原 `in.count(context)`；如果结果不是 -1，直接采用其快速计数。
2. 原 Weight 无法快速计数时，有删除文档则返回 -1。
3. 否则进行使用登记、资格检查与 tryLock 查缓存。
4. 命中则返回 `cached.count()`，未命中返回 -1，不主动构建。

这里 -1 表示“无法快速计数”，不是匹配数为零。外层可继续走普通收集流程。因此并不是调用 `IndexSearcher.count` 就一定能建立或命中缓存；底层快速计数可能直接结束。

证据：`search/LRUQueryCache.java:832`。

### 7.4 案例状态汇总

仅统计 Q 的两段访问，假定没有其他缓存条目与统计干扰：

| 阶段 | Q 在近期窗口中的频次 | 全局 Query 数 | cacheSize | 累计 miss | 累计 hit | 累计 cacheCount |
| --- | --- | --- | --- | --- | --- | --- |
| 初始 | 0 | 0 | 0 | 0 | 0 | 0 |
| 第一次执行后 | 1 | 0 | 0 | 2 | 0 | 0 |
| 第二次执行后 | 2 | 1 | 2 | 4 | 0 | 2 |
| 第三次执行后 | 3 | 1 | 2 | 4 | 2 | 2 |

## 8. 淘汰：以 Query 为单位跨 Segment 删除

`requiresEviction()` 检查全局 Query 数超过 `maxSize`，或内存记账超过 `maxRamBytesUsed`。`evictIfNecessary()` 从访问顺序 Map 的最旧端取 Query，将它从全局集合移除，再通过 `onEviction` 遍历所有 LeafCache 删除相应集合。

案例中随后建立另一个 Query R 的缓存，LRU 顺序为 `[Q, R]`；再次命中 Q 后变为 `[R, Q]`；插入第三个 Query T，顺序暂为 `[R, Q, T]`。因 `maxSize = 2`，淘汰 R 在所有 Segment 中的集合，留下 `[Q, T]`。

因此淘汰既不是选某个 Segment，也不是只删除一个 `(Query, Segment)` 条目。单次 Query 淘汰要遍历所有已登记 LeafCache，代价随这类 Segment 数量增长。

淘汰后空 LeafCache 仍可能留在外层 Map，直至 core 关闭或整体 clear。容量限制主要约束稳定存储状态；完整集合在锁外创建，并发构建和插入前的临时对象也可能额外占用内存，不能把配置当作整个进程此时的绝对内存上限。

证据：`search/LRUQueryCache.java:257`、`search/LRUQueryCache.java:321`、`search/LRUQueryCache.java:381`、`search/LRUQueryCache.java:660`。

## 9. 生命周期：删除、刷新与 Segment 关闭

### 9.1 删除文档为什么不会让旧集合返回错误结果

缓存按 core key 绑定，构建时也不应用当前 liveDocs。正常搜索收集时，`IndexSearcher` 才将 `ctx.reader().getLiveDocs()` 传给 BulkScorer；`Weight.DefaultBulkScorer` 据此跳过删除文档。

若 S1 上缓存包含局部 doc ID 7，随后该文档被删除，但新 reader 仍共享同一 core，则旧集合可以继续使用，收集时 doc 7 被过滤。这里的前提是 reader 共享 core 且原 Weight 仍声明可缓存；仅凭“发生刷新”不能判断缓存必然可复用或必然失效。

数量缓存不能直接应用这个掩码，因此 `count` 在原 Weight 不能快速计数且 reader 有删除时拒绝返回缓存 count。

证据：`search/LRUQueryCache.java:514`、`search/IndexSearcher.java:769`、`search/Weight.java:313`、`search/LRUQueryCache.java:840`。

### 9.2 三类清理各自删什么

| 操作 | 清理范围 | 全局 Query 是否随之移除 |
| --- | --- | --- |
| `clearCoreCacheKey(coreKey)` | 一个 core 的整个 LeafCache 与其集合记账 | 否 |
| `clearQuery(query)` | 一个 Query 及其在所有 LeafCache 中的集合 | 是 |
| `clear()` | 所有 LeafCache、所有全局 Query，当前内存与条目归零 | 是 |

core 的关闭监听器在第一次创建 LeafCache 时注册。Segment 合并得到新 core，要按新 core 建立缓存；旧 core 真正关闭后清理旧条目，不是“合并一开始就清空”。一个 core 被移除后，全局 Query 可能暂时没有任何叶子条目，仍留在 LRU 中直到后续清理或淘汰。

DocValues 更新影响查询结果时，正确性还依赖原 Weight 的 `isCacheable`。不能认为同一 core 下所有数据变化都天然安全；现有测试展示了更新后的相关查询绕过缓存。

证据：`search/LRUQueryCache.java:312`、`search/LRUQueryCache.java:349`、`search/LRUQueryCache.java:369`、`search/LRUQueryCache.java:390`、`search/TestLRUQueryCache.java:1734`。

## 10. 并发、异常与统计边界

查询查找用 `tryLock`，争用时退回原计算；插入、淘汰、清理则使用 `lock.lock()`。因此“整个缓存绝不等待锁”是不准确的。完整集合构建在查找锁释放后执行，避免长时间占锁，但也允许多个线程同时发现 miss 并重复构建同一集合；最终插入靠两层 `putIfAbsent` 去重。实现没有按键合并构建任务或异步加载任务。

统计字段为 volatile；更新依靠锁，读取无需锁。一次读取多个 getter 不是全局原子快照。

| 指标 | 含义 |
| --- | --- |
| `hitCount` / `missCount` | 真正进入 get 的逐 Segment 命中 / 未命中次数 |
| `totalCount` | hit + miss，不是用户搜索请求数 |
| `cacheSize` | 当前 `(Query, core)` 集合条目数，空结果条目也计数 |
| `cacheCount` | 历史真正插入的集合条目数 |
| `evictionCount` | `cacheCount - cacheSize`，含手工清理与 core 关闭等移除 |
| `ramBytesUsed` | Query、Map 条目及集合的记账内存；普通 Query 使用默认估算，可 Accountable 的 Query 使用自身报告值 |

`clear()` 只归零当前内存、当前条目；历史 hit、miss、cacheCount 不清零。因此不能把 `getEvictionCount()` 当作仅由 LRU 容量不足触发的淘汰数。

异常边界：

- `skipCacheFactor < 1` 或 NaN 在构造时抛出 `IllegalArgumentException`。
- Query 入缓存后若修改了参与 hash 的状态，淘汰检测到异常时可能抛出 `ConcurrentModificationException`；必须保持键的等价与 hash 稳定。
- 原 Weight、策略、集合迭代或构建产生的 `IOException` 按调用链传播，没有统一吞掉并自动重试。构建失败时尚未插入该次生成的完整集合；这不是整个搜索的事务回滚保证。
- 回调在锁内同步执行。扩展统计回调应避免耗时操作；覆盖基础回调时若要保留原始记账，还需调用父类实现。

证据：`search/LRUQueryCache.java:126`、`search/LRUQueryCache.java:179`、`search/LRUQueryCache.java:250`、`search/LRUQueryCache.java:296`、`search/LRUQueryCache.java:331`、`search/LRUQueryCache.java:403`、`search/LRUQueryCache.java:549`、`search/LRUQueryCache.java:747`。

## 11. 相关测试：用于对照哪些行为

本次阅读了相关测试源码，未运行构建或测试；下表是已有断言的覆盖范围，不代表本次执行结果。

| 测试与位置 | 对应验证点 |
| --- | --- |
| `testLRUEviction`，`search/TestLRUQueryCache.java:237` | Query 访问顺序与容量淘汰 |
| `testOnUse`，`search/TestLRUQueryCache.java:633` | onUse 的调用语义 |
| `testStats`，`search/TestLRUQueryCache.java:707` | 分 Segment 的统计、策略禁止新建时继续命中、关闭时移除 |
| `testBooleanQueryCachesSubClauses`，`search/TestLRUQueryCache.java:1009` | 评分查询中的 FILTER / MUST_NOT 可被包装缓存；使用自定义 ALWAYS_CACHE |
| `testDetectMutatedQueries`，`search/TestLRUQueryCache.java:1211` | 修改缓存键的异常检测 |
| `testRefuseToCacheTooLargeEntries`，`search/TestLRUQueryCache.java:1246` | 内存容量资格门槛 |
| `testMinSegmentSizePredicate`，`search/TestLRUQueryCache.java:1428` | 最小 Segment 筛选 |
| `testReaderNotSuitedForCaching`，`search/TestLRUQueryCache.java:1509` | 不支持缓存的 reader 路径 |
| `testQueryNotSuitedForCaching`，`search/TestLRUQueryCache.java:1572` | 原 Weight 不可缓存的路径 |
| `testDocValuesUpdatesDontBreakCache`，`search/TestLRUQueryCache.java:1734` | DocValues 更新后相关查询不再使用旧缓存 |
| `testQueryCacheSoftUpdate`，`search/TestLRUQueryCache.java:1802` | 全部文档被软删除后旧 Segment 被丢弃，条目随生命周期清理 |
| `testSkipCachingForRangeQuery`，`search/TestLRUQueryCache.java:1923` | 不同 factor 对 supplier 构建的影响 |
| `testCacheHasFastCount`，`search/TestLRUQueryCache.java:2057` | 填充后快速计数，有删除时不能返回缓存数量 |
| `testNeverCacheTermFilter`，`search/TestUsageTrackingFilterCachingPolicy.java:50` | 默认策略不缓存 TermQuery |
| `testBooleanQueries`，`search/TestUsageTrackingFilterCachingPolicy.java:68` | 复合查询的频率门槛 |

## 12. 源码审阅发现

### 12.1 构造器注释与实现有差异

**源码事实**：两参数构造器注释称成本至多为顶层查询的 100 倍，而实际传入 `skipCacheFactor = 10`。注释还写文档数“more than 10k”，实际判断 `maxDoc < 10000` 才拒绝，因此正好 10000 也可能通过。

证据：`search/LRUQueryCache.java:139`、`search/LRUQueryCache.java:148`、`search/LRUQueryCache.java:162`。

**结论**：解释当前版本时应以实现为准。这是注释不一致，本文不据此认定缓存结果有正确性缺陷。

### 12.2 LRU 的更新与完整集合构建有易误读的边界

**源码事实**：get 访问全局 Query 时可以更新 LRU，即便当前 Segment 最终 miss；bulkScorer 构建路径没有 supplier 的成本倍数判断；稀疏格式选择依赖估算 cost。

**推断**：仅根据“最近命中次数”“实际命中率”或 factor 参数推测全部缓存行为，容易与真实状态不符。

**待验证项**：具体工作负载下重复构建的内存峰值、锁竞争占比、全量构建耗时需要运行时测量。本文没有运行性能基准，也不将这些设计取舍认定为已复现缺陷。

## 13. 心智模型与推荐阅读顺序

把它理解成三个协作部件：**策略决定是否值得新建，core key 决定结果属于哪个 Segment，全局 Query LRU 决定容量不足时一起淘汰谁。** 查找成功后只是遍历集合；删除过滤与最终评分仍由各自的外层执行流程负责。

建议按以下顺序读代码：

1. `search/IndexSearcher.java:898` → `search/BooleanWeight.java:52`：哪些 Weight 可以进入缓存。
2. `search/LRUQueryCache.java:474` → `search/LRUQueryCache.java:724`：包装、资格检查、命中与 miss。
3. `search/UsageTrackingQueryCachingPolicy.java:124`：是否值得为 miss 建缓存。
4. `search/LRUQueryCache.java:501` → `search/LRUQueryCache.java:292`：集合构建和状态插入。
5. `search/LRUQueryCache.java:321` → `search/LRUQueryCache.java:349`：LRU 淘汰与 core 关闭清理。
6. `search/LRUQueryCache.java:832` → `search/LRUQueryCache.java:891`：补齐 count 与 bulkScorer 两条入口。
7. 对照测试表，检查自己对使用频次、Query 数、集合条目数和删除语义的理解。
