# LRUQueryCache 实现跟踪

## 1. 先看它到底省了哪一步

假设商品搜索有两个条件：

```text
关键词包含“手机” AND 价格在 100～200 元之间
```

关键词决定商品与搜索词有多相关，需要评分。价格只决定商品能不能留下，是过滤条件。

下一次用户搜索“耳机”，价格范围仍是 100～200 元。关键词部分需要重新计算，但价格过滤能不能直接复用？`LRUQueryCache` 就用来做这件事：**记住一个查询条件匹配了哪些文档，下次遇到相同条件，直接遍历已保存的文档集合。**

例如，在某个 Segment 中，价格过滤匹配的文档编号是：

```text
Q：价格在 100～200 元之间
计算结果：{3, 7, 12, ...}

缓存前：执行 Q → 查找满足价格范围的文档 → 得到这些编号
缓存后：查缓存 → 直接得到这些编号
```

这里保存的是文档编号集合，不是商品内容，也不是最终搜索结果。关键词评分、与其他条件求交集、排序，仍要继续进行。实现中的结果对象叫 `CacheAndCount`，其中保存 `DocIdSet` 和匹配数量。源码：`LRUQueryCache.java:949`。

读主流程时先记住三个名字即可：

| 名字 | 在这条流程里负责什么 |
| --- | --- |
| `Query` | 描述查什么，例如价格范围 Q |
| `Weight` | 为当前搜索准备查询执行逻辑，能够在每个 Segment 上创建执行器 |
| `Scorer` / `BulkScorer` | 实际找出匹配文档；前者提供逐文档推进能力，后者负责批量收集 |

缓存介入的位置是：**包装 Weight，让它创建执行器前先查缓存。**

```text
创建查询执行计划
  → 用 CachingWrapperWeight 包装可缓存的 Weight
  → 执行到某个 Segment
  → 查缓存
      命中：用缓存集合创建执行器
      未命中：按原逻辑计算，必要时保存集合
  → 继续过滤、评分与收集
```

入口见 `IndexSearcher.java:898`，包装逻辑见 `LRUQueryCache.java:474`。

## 2. 从一次真实搜索走到缓存命中

以下始终使用同一个过滤条件：

```java
Query q = LongPoint.newRangeQuery("price", 100, 200);
```

先只看一个 Segment S1，假设它有 20000 个文档，Q 匹配其中 400 个。它满足缓存的大小和内存要求，执行过程没有锁竞争。每次搜索都新建查询执行计划，缓存实例与默认缓存策略则持续复用。

默认策略认为 Point 范围查询值得较早缓存：在近期使用记录中出现两次后，便允许保存结果。因此下面会看到“第一次计算、第二次保存、第三次复用”。这不是所有 Query 统一的次数规则。源码：`UsageTrackingQueryCachingPolicy.java:45`、`UsageTrackingQueryCachingPolicy.java:124`。

### 2.1 进入缓存：外层搜索需要评分，价格过滤不需要

`IndexSearcher.createWeight` 的判断很直接：

```java
Weight weight = query.createWeight(this, scoreMode, boost);
if (scoreMode.needsScores() == false && queryCache != null) {
  weight = queryCache.doCache(weight, queryCachingPolicy);
}
```

关键是 `scoreMode.needsScores() == false`。

外层 BooleanQuery 虽然需要关键词评分，但 `BooleanWeight` 为 `FILTER` 子句创建 Weight 时，会传入 `COMPLETE_NO_SCORES`。因此价格 Q 可以进入缓存，关键词的正常评分流程继续保留。`MUST_NOT` 子句也是非评分子句。源码：`BooleanWeight.java:52`、`IndexSearcher.java:898`。

`doCache` 返回一个 `CachingWrapperWeight`，内部仍保留原 Weight，下面称它为 `in`：

```text
CachingWrapperWeight
  ├─ in：原来的查询执行逻辑，缓存用不了时继续调用它
  └─ policy：判断一个查询值不值得新建缓存
```

这一步只加包装，还没有计算 400 个文档。源码：`LRUQueryCache.java:474`、`LRUQueryCache.java:683`。

### 2.2 第一次搜索：查不到，也暂时不保存

先沿批量执行入口 `CachingWrapperWeight.bulkScorer(S1)` 阅读。`IndexSearcher` 遍历 Segment 搜索时会调用这个入口。源码：`IndexSearcher.java:763`。

它先调用 `policy.onUse(Q)`，让策略记下“Q 用过一次”。包装 Weight 用 `AtomicBoolean used` 保证同一执行计划只登记一次，不会因为访问多个 Segment 就重复增加查询使用次数。源码：`LRUQueryCache.java:891`。

随后检查这个 Segment 能不能使用缓存，检查通过才查找 Q 的结果。此时缓存为空，得到 miss。

代码接着询问 `policy.shouldCache(Q)`。默认策略中 Q 的使用次数只有 1，还没达到门槛，于是调用：

```java
return in.bulkScorer(context);
```

也就是交回原 Weight，正常计算价格范围的匹配文档。

**第一次搜索结束：结果正常返回，Q 没有被存入缓存。** 查找与分支见 `LRUQueryCache.java:918`。

### 2.3 第二次搜索：计算一次完整结果，并存下来

第二次搜索创建新的包装 Weight，执行到 S1 时再次登记 Q。策略中 Q 的使用次数达到 2。

缓存依然 miss，但这次 `policy.shouldCache(Q)` 返回 true，进入：

```java
cached = cache(context);
putIfAbsent(in.getQuery(), cached, cacheHelper);
```

这两句分别完成“生成集合”和“保存集合”。源码：`LRUQueryCache.java:925`。

`cache(context)` 先取得原 Weight 的 `BulkScorer`，再交给 `cacheImpl`。`cacheImpl` 遍历 Q 的全部匹配文档，将编号收集到集合中：

```text
原执行器依次找出：3、7、12、……，共 400 个
                  ↓
CacheAndCount {
  cache = 保存这 400 个编号的 DocIdSet
  count = 400
}
```

若原执行器为 null，说明没有匹配，也会保存一个空结果，避免之后重复计算。源码：`LRUQueryCache.java:707`、`LRUQueryCache.java:501`。

`putIfAbsent` 再把结果挂到 S1 的缓存下，并检查是否需要淘汰旧 Query。集合构建发生在锁外，缓存结构的修改发生在锁内。源码：`LRUQueryCache.java:292`。

**第二次搜索仍然付出了计算代价，但这次留下了以后可以复用的集合。** 当前搜索也直接使用刚生成的集合继续执行。

### 2.4 第三次搜索：直接遍历缓存集合

第三次查找时，S1 中已经有 Q 的 `CacheAndCount`，于是跳过原 Weight 的集合计算，取出缓存迭代器，创建：

```java
new ConstantScoreScorer(this, 0f, ScoreMode.COMPLETE_NO_SCORES, disi)
```

`disi` 就是遍历缓存文档编号的迭代器。这里的 0 分不会破坏关键词评分，因为这个执行器服务的是无评分的价格条件。源码：`LRUQueryCache.java:934`。

到这里，三次执行的差别可以放在一张表里：

| 搜索次数 | 策略记录的 Q 使用次数 | 缓存查找 | 是否计算价格匹配集合 | 是否保存集合 |
| --- | --- | --- | --- | --- |
| 第一次 | 1 | miss | 是 | 否 |
| 第二次 | 2 | miss | 是 | 是 |
| 第三次 | 3 | hit | 否，遍历已有集合 | 已有集合继续保留 |

缓存带来的收益发生在第三次及后续复用中。实际执行如果不满足缓存资格、发生锁竞争或走了其他快速路径，过程可能不同。

## 3. 扩展到多个 Segment：为什么需要三张 Map

Lucene 把索引划分成多个 Segment，每个 Segment 内部都有自己的文档编号。S1 的文档 7 与 S2 的文档 7 是不同文档，因此 Q 的结果必须分开存。

假设再增加 S2，Q 在其中匹配 100 个文档：

```text
S1：Q → 400 个局部文档编号
S2：Q → 100 个局部文档编号
```

为了管理这些结果，源码分成两层：

```text
全局：管理哪些 Query 已经缓存，以及谁最近被访问
  uniqueQueries：Q、R、……，按最近访问顺序排列

每个 Segment：保存该 Segment 内的查询结果
  cache
    S1 的 core key → LeafCache
                      Q → CacheAndCount(400 个文档)
    S2 的 core key → LeafCache
                      Q → CacheAndCount(100 个文档)
```

`core key` 是 Segment 底层核心的身份标识；`LeafCache` 就是一个 Segment 的缓存容器。**一个 Q 对应两个结果条目，所以 Query 数是 1，`cacheSize` 却是 2。** 源码：`LRUQueryCache.java:88`、`LRUQueryCache.java:623`。

### 3.1 全局 Map 还有一个任务：统一等价 Query 的对象

两次搜索往往各自创建一个价格 Query 对象。只要它们的 `equals/hashCode` 相同，全局 `uniqueQueries` 就把它们归到同一个代表对象上。各个 LeafCache 都使用这个代表对象作 key，因此 LeafCache 可以用 `IdentityHashMap` 按对象身份查找。

可以将查找过程理解为：

```text
本次新建的价格 Q
  → uniqueQueries：找到缓存中的等价 Q 对象
  → S1 的 LeafCache：用这个对象找到 400 个文档的集合
```

具体实现会先确认当前 Segment 的 LeafCache 存在，再查全局 Query，最后查集合。源码：`LRUQueryCache.java:267`。

### 3.2 putIfAbsent 具体改了什么

第二次搜索保存 Q 时，按下面的顺序修改状态：

1. 在全局 Map 中登记 Q，或复用已有的等价 Q 对象。
2. 找到 S1 的 LeafCache；没有就创建，并注册 core 关闭时的清理回调。
3. 向 LeafCache 插入 Q 的集合；已经有结果时不重复保存。
4. 增加实际插入的条目数与内存记账。
5. 检查容量，必要时执行 LRU 淘汰。

源码：`LRUQueryCache.java:292`、`LRUQueryCache.java:651`。

## 4. LRU 怎样淘汰：删一个 Query 的所有结果

LRU 的意思是最近最少使用。实现依靠按访问顺序排列的 `LinkedHashMap`：

```java
uniqueQueries = new LinkedHashMap<>(16, 0.75f, true);
```

最后的 `true` 表示按访问顺序维护。`mostRecentlyUsedQueries` 是这张 Map 的 keySet 视图，并不是额外的一份 Query 列表。源码：`LRUQueryCache.java:132`。

假设最多缓存两个 Query，左侧是最旧的，右侧是最近访问的：

```text
已有 Q、R：       [Q, R]
再次访问 Q：     [R, Q]
插入新 Query T： [R, Q, T]
超过上限，淘汰 R：[Q, T]
```

淘汰 R 时，代码先移除全局 R，再遍历所有 LeafCache，删除 R 在每个 Segment 上的结果。Q 和 T 的结果不受影响。因此 `maxSize = 2` 限制的是不同 Query 数，不是结果集合数。源码：`LRUQueryCache.java:321`、`LRUQueryCache.java:381`。

内存记账超过 `maxRamBytesUsed` 时也会触发同样的淘汰，直到满足约束。单次淘汰要遍历所有叶子缓存，因此把同一个缓存共享给很多 Segment 会增加清理成本。源码：`LRUQueryCache.java:257`。

有个细节：某个 Segment 的 LeafCache 已存在时，`uniqueQueries.get(Q)` 会更新 Q 的最近访问位置，即使 Q 在该 Segment 上还没有结果、最终仍然 miss。因此这里的“使用”并不完全等同于“成功命中”。源码：`LRUQueryCache.java:267`。

## 5. 哪些情况下会跳过缓存

理解了主线，再看为什么实际搜索未必按“三次执行”的例子发展。

### 5.1 不是所有查询都值得保存

默认 `UsageTrackingQueryCachingPolicy` 维护最近 256 次使用记录，根据近期出现频次决定是否新建缓存：

| Query 类型 | 默认策略 |
| --- | --- |
| Point 范围、MultiTermQuery 等成本较高的查询 | 出现至少 2 次 |
| 非空 BooleanQuery、DisjunctionMaxQuery | 出现至少 4 次 |
| 其他未被排除的查询 | 出现至少 5 次 |
| TermQuery、FieldExistsQuery、MatchAllDocsQuery、MatchNoDocsQuery，以及空的上述复合查询 | 不新建缓存 |

例如，单个词项过滤本来就能高效读取倒排表，再复制一份结果集合未必划算。这就是例子选择价格 Point 查询的原因。源码：`UsageTrackingQueryCachingPolicy.java:45`、`UsageTrackingQueryCachingPolicy.java:56`、`UsageTrackingQueryCachingPolicy.java:124`。

这些频次是近期窗口里的次数，不是从进程启动以来永久累加的次数。策略只保存 hash code，允许极少数碰撞导致提前缓存；真正的结果查找仍使用 Query 等价关系。源码：`UsageTrackingQueryCachingPolicy.java:143`。

策略控制的是 miss 后是否建立新条目，已经存在的结果仍可以命中。源码：`LRUQueryCache.java:925`。

### 5.2 Segment 太小、内存不足或结果不稳定

查缓存之前，还要检查：

| 条件 | 为什么需要 |
| --- | --- |
| 原 Weight 的 `isCacheable` 为 true | 由具体查询确认结果适合按 core 缓存 |
| Segment 通过大小筛选 | 避免为很小的 Segment 花费缓存管理成本 |
| 内存估算通过 | 避免一个巨大集合挤掉大量旧缓存 |
| reader 提供 core cache helper | 需要稳定的身份标识以及关闭清理回调 |
| `tryLock` 成功 | 查找时不因锁争用而一直等待，失败则正常计算 |

两参数构造器默认要求 Segment 的 `maxDoc >= 10000`，并且 `maxDoc * 2 > 顶层 maxDoc / Segment 数`。内存检查要求 `(顶层 maxDoc / 8) * 5 < maxRamBytesUsed`；这是为多个过滤集合预留空间的粗略估算。源码：`LRUQueryCache.java:147`、`LRUQueryCache.java:160`、`LRUQueryCache.java:695`。

批量入口的检查见 `LRUQueryCache.java:896`。跳过这些检查时，流程交回原 Weight，查询仍正常执行；这类访问不会计入缓存 hit / miss。

### 5.3 有时只检查少量候选文档，完整缓存反而更慢

除了前面跟踪的 `bulkScorer`，包装 Weight 还提供 `scorerSupplier`。supplier 可以理解为“按调用方需要，稍后再创建 Scorer”的工厂。

在组合查询里，其他条件可能已经把候选缩到 10 个。此时价格过滤只需验证少数候选，没必要为了缓存去完整找出全部匹配。

因此 `ScorerSupplier.get(leadCost)` 在 miss 且策略允许后，还会检查：

```java
if (cost / skipCacheFactor > leadCost) {
  return supplier.get(leadCost);
}
```

假设价格查询的估算 cost 为 400，默认 factor 为 10，调用方给的 leadCost 为 10，那么 `400 / 10 > 10`，本次交回原 supplier，跳过构建。通过检查才完整生成集合并插入。源码：`LRUQueryCache.java:759`。

三个入口的区别归纳如下：

| 入口 | miss 时的行为 |
| --- | --- |
| `bulkScorer` | 策略允许就完整构建，不做上面的 leadCost 比较 |
| `scorerSupplier` | 策略允许后，创建 Scorer 时还可能因成本跳过 |
| `count` | 只查询已有集合，不主动构建；返回 -1 表示不能快速计数 |

`count` 会先尝试原 Weight 自带的快速计数，成功就直接返回；失败后才考虑缓存，并且有删除文档时不使用缓存数量。源码：`LRUQueryCache.java:724`、`LRUQueryCache.java:832`、`LRUQueryCache.java:891`。

## 6. 集合怎样存，删除后为什么还能用

### 6.1 集合格式由估算 cost 决定

缓存不会简单保存一长串 Java 文档对象，而是选择适合文档编号集合的数据结构：

```text
scorer.cost() * 100 >= Segment.maxDoc
  是 → FixedBitSet，以位标记文档是否匹配
  否 → RoaringDocIdSet，压缩保存较稀疏的编号集合
```

例如 S1 有 20000 个文档，估算 cost 为 400，则走 bitset。这里使用估算 cost，而不是先统计实际命中率再选择格式。源码：`LRUQueryCache.java:501`。

### 6.2 删除文档后，搜索阶段再排除它

假设 Q 的缓存中包含文档 7，后来文档 7 被删除。若新 reader 仍共享同一个 core，且查询仍可缓存，旧集合可以保留：

```text
缓存集合：{3, 7, 12, ...}
当前 liveDocs：文档 7 已删除
实际收集：{3, 12, ...}
```

原因是构建集合时不传删除过滤条件；实际搜索时，`IndexSearcher` 将当前 `liveDocs` 传给执行器，收集时排除被删除的编号。源码：`LRUQueryCache.java:511`、`IndexSearcher.java:769`、`Weight.java:313`。

缓存中的 count 仍可能包含已删除文档，因此不能直接返回它作为当前数量。这正是 `count` 路径有删除时拒绝使用缓存数量的原因。源码：`LRUQueryCache.java:840`。

### 6.3 core 关闭才清理对应结果

创建 LeafCache 时，缓存会注册 core 关闭回调。旧 core 真正关闭时，`clearCoreCacheKey` 移除它的整个 LeafCache。Segment 合并会产生新的 core，新 core 需要建立自己的结果。

手工清理则有两个入口：`clearQuery(Q)` 删除 Q 在所有 Segment 中的结果；`clear()` 清空整个缓存。只清理一个 core 时，全局 Q 可以继续保留。源码：`LRUQueryCache.java:312`、`LRUQueryCache.java:349`、`LRUQueryCache.java:369`、`LRUQueryCache.java:390`。

DocValues 更新属于另一种情况：它可能改变查询结果，不能只靠删除过滤保证正确。相关查询还要通过原 Weight 的 `isCacheable` 判断，更新后不适合缓存时绕过旧集合。已有回归测试见 `TestLRUQueryCache.java:1734`。

## 7. 并发、统计与异常：读源码时的补充

查找使用 `tryLock`，失败就正常计算；插入与清理使用阻塞锁。集合在锁外生成，所以两个线程可能同时 miss、同时计算；最终通过 `putIfAbsent` 只保留一份条目。这种设计缩短了持锁时间，但不保证同一个结果只计算一次。源码：`LRUQueryCache.java:292`、`LRUQueryCache.java:747`。

内存限制约束的是缓存记账，并不覆盖锁外并发构建时的全部临时内存。普通 Query 的大小使用默认估算，实现了 `Accountable` 的 Query 使用自身报告值。源码：`LRUQueryCache.java:403`。

最容易误读的几个统计值是：

| 指标 | 应怎样理解 |
| --- | --- |
| `hitCount` / `missCount` | 按 Segment 的真实查找次数，不是用户请求数 |
| `cacheSize` | 当前结果集合数；一个 Query 在两个 Segment 缓存，计为 2 |
| `cacheCount` | 历史真正插入过的集合数 |
| `evictionCount` | `cacheCount - cacheSize`，也包括关闭与手工清理造成的移除 |

`clear()` 归零当前条目与内存，不重置历史 hit、miss 和 cacheCount。源码：`LRUQueryCache.java:250`、`LRUQueryCache.java:549`。

异常方面，factor 小于 1 或为 NaN 会在构造时被拒绝；Query 入缓存后必须保持 hash 与等价关系稳定，修改 key 可能在淘汰时触发 `ConcurrentModificationException`。查询、策略或集合构建产生的 `IOException` 会向上传播。源码：`LRUQueryCache.java:126`、`LRUQueryCache.java:331`、`LRUQueryCache.java:707`。

## 8. 相关测试

以下是已有测试的阅读入口，不表示本次运行了测试：

| 想验证的问题 | 测试入口 |
| --- | --- |
| 最近访问顺序怎样改变，谁被淘汰 | `TestLRUQueryCache.java:237`，`testLRUEviction` |
| 多个 Segment 会不会重复登记使用次数 | `TestLRUQueryCache.java:633`，`testOnUse` |
| 评分查询里的过滤子句能否缓存 | `TestLRUQueryCache.java:1009`，`testBooleanQueryCachesSubClauses` |
| 成本门槛会不会跳过缓存 | `TestLRUQueryCache.java:1923`，`testSkipCachingForRangeQuery` |
| 有删除后为什么不能直接返回缓存数量 | `TestLRUQueryCache.java:2057`，`testCacheHasFastCount` |
| TermQuery 默认是否缓存 | `TestUsageTrackingFilterCachingPolicy.java:50`，`testNeverCacheTermFilter` |

## 9. 源码审阅发现

**构造器注释与实际参数不一致。** 注释写成本倍数为 100，实际默认 factor 为 10；注释写文档数超过 10000，实际正好 10000 也可能通过。源码事实见 `LRUQueryCache.java:139`、`LRUQueryCache.java:148`、`LRUQueryCache.java:162`。解释行为时应使用实际判断，这些注释差异不等于查询结果存在错误。

锁外重复构建可能增加临时内存和计算开销，这是从实现推导出的性能影响；具体影响大小需要运行时测量，不能仅凭源码认定已经发生性能故障。

## 10. 最后串起来读一次

**先由策略决定是否值得存，再按 Segment 存匹配集合，命中后直接遍历集合，容量不足时按 Query 淘汰所有相关集合。**

重新打开源码时，建议沿本文主线阅读：

```text
IndexSearcher.createWeight        IndexSearcher.java:898
  → 加入缓存包装                  LRUQueryCache.java:474
  → 执行到具体 Segment            LRUQueryCache.java:891
  → 查找已有集合                  LRUQueryCache.java:267
  → miss 后判断是否值得缓存       UsageTrackingQueryCachingPolicy.java:177
  → 完整生成集合                  LRUQueryCache.java:707、501
  → 插入并检查淘汰                LRUQueryCache.java:292、321
```

读通这条线后，再补 `scorerSupplier` 的延迟构建与成本判断，以及 `count` 的快速计数路径，就能把三个执行入口对应起来。
