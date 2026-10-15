# RAG 知识库文档变更设计：新增、修改、删除

> RAG 系统上线只是开始。文档每天都在变化：新增一篇、修订一版、下线一批。如果索引更新策略设计不好，就会出现「答案引用已删除的文档」「同一内容新旧版本同时被检索」「改一个字全库重建」等问题。本文系统梳理 RAG 知识库增量变更（增、删、改）的设计方案。

## 一、为什么文档变更是个难题

RAG 的索引链路是：`文档 → 切分 → 向量化 → 入库`。链路一长，变更就麻烦：

1. **粒度不一致**：变更以「文档」为单位，向量库存储以「chunk」为单位。改一个文档，可能影响几十个 chunk
2. **双存储一致性**：源文档库（对象存储/数据库）和向量库是两套系统，没有跨系统事务
3. **成本敏感**：Embedding 是要花钱、花时间的，全量重建既慢又贵
4. **错误会被放大**：删除失败留下「孤儿 chunk」，检索时依然召回已不存在的知识，且用户无从察觉

一个反面案例：文档 v1 说「退款周期 7 天」，修订为 v2「退款周期 15 天」。如果只插入新 chunk 不删旧 chunk，检索时新旧两个说法都会被召回，模型可能随机采用其一 —— 这是比召回率低更严重的事故。

## 二、核心设计：身份标识与元数据

一切的基石是给每个 chunk 一个**可追溯的身份**，并让向量库记住足够多的元数据。

### 2.1 关键字段设计

```
┌─────────────────────────────────────────────────┐
│ chunk 元数据                                     │
├─────────────────────────────────────────────────┤
│ doc_id        文档唯一 ID（不随版本变化）          │
│ doc_version   文档版本号 / 内容 hash              │
│ chunk_id      切片唯一 ID = doc_id + 序号/hash    │
│ chunk_hash    切片内容 hash（判断 chunk 是否变化） │
│ status        状态：indexing / active / deleting │
│ source_path   溯源信息：文件路径、URL、起始行号     │
│ updated_at    入库时间                            │
└─────────────────────────────────────────────────┘
```

设计要点：

- `doc_id` 是变更操作的锚点，所有增删改都先定位 doc，再操作它名下的 chunk
- `content_hash`（对整个文档内容做 hash）用于**快速判断文档是否真的变了**，mtime 变化但内容没变（常见于同步工具 touch 文件）可以跳过
- `chunk_hash` 用于**细粒度 diff**：文档变了，但只有部分 chunk 变了，只重建变化的 chunk

### 2.2 状态机

chunk 不能只有「存在/不存在」两种状态，否则删除过程中断会留下半成品：

```
            写入向量+索引
  (创建) ────────────────► [indexing] ──► [active] ──► (物理删除)
                             │                ▲
                             │ 失败            │
                             ▼                │ 标记删除成功
                          [failed] ──重试──► [deleting]
                                             │
                                             ▼ 清理失败
                                          定时对账任务兜底
```

- **indexing**：向量已写入但还未对外可见（或标记待确认），防止半成品参与检索
- **active**：正常可检索
- **deleting**：已标记删除，等待物理清理。标记与物理删除分离，保证幂等

## 三、变更检测：怎么知道文档变了

三种主流方案，通常组合使用：

| 方案         | 原理                     | 优点             | 缺点               | 适用场景           |
| ------------ | ------------------------ | ---------------- | ------------------ | ------------------ |
| 定时全量扫描 | 周期性遍历源，比对 hash  | 实现简单、不漏   | 有延迟、扫描有成本 | 小规模、兜底对账   |
| 事件驱动     | Webhook / 消息队列推送   | 实时、成本低     | 依赖源头改造、可能丢消息 | 文档系统可控的场景 |
| CDC 监听     | 监听数据库 binlog        | 实时、可靠       | 只覆盖 DB 类来源   | 知识存在数据库时   |

生产上推荐：**事件驱动为主 + 定时对账兜底**。事件丢失是常态（发消息失败、消费失败），定时任务（如每日低峰全量比对 `doc_id + content_hash`）保证最终一致。

### 3.1 全量比对的实现

```
每日对账任务：
1. 拉取源文档列表 { doc_id, content_hash }
2. 拉取向量库元数据 { doc_id, doc_version(即hash), status }
3. diff 出三类：
   - 源有、库无（或 hash 不同）→ upsert
   - 源无、库有              → 删除
   - 两边都有且 hash 相同     → 跳过（幂等）
```

比对只操作元数据，不碰向量，成本可控。

## 四、新增流程

新增是最简单的路径，重点在**幂等**：

```
收到文档新增事件
   │
   ▼
按 doc_id 查向量库，已存在？
   ├── 是且 hash 相同 → 跳过（重复事件）
   ├── 是但 hash 不同 → 走「修改」流程
   └── 否
         │
         ▼
   清洗 → 切分 → 逐 chunk 生成 chunk_hash
         │
         ▼
   以 status=indexing 写入向量库（向量 + 元数据）
         │
         ▼
   全部写入成功 → 批量置为 active
   任一失败     → 重试 / 标记 failed，等对账兜底
```

切分和 Embedding 可以做成**批量异步**：新增事件先入队列，消费端攒批调用 Embedding 接口，降低成本（多数 Embedding API 按请求计费而非按 token 分档时，攒批更划算）。

## 五、修改流程

修改是三种变更里最值得设计的，核心思想：**chunk 级 diff，只重建变化的部分**。

### 5.1 先删后插（简单版）

```
1. 删除 doc_id 下所有旧 chunk
2. 重新切分、向量化、写入新 chunk
```

- 优点：逻辑简单，不容易漏
- 缺点：全量重算，贵；删除和插入之间有窗口期，检索会「短暂查不到该文档」

适合小文档、低频修改。

### 5.2 chunk 级 diff（推荐）

```
文档 v1 切分结果: [c1, c2, c3, c4, c5]   (各带 chunk_hash)
文档 v2 切分结果: [c1', c2', c4', c5']   (重新计算 hash)

diff 结果:
  c2' 与 c3 hash 相同  → 复用，不动向量库
  c1' hash 变化        → 删旧 c1，写新 c1'
  c3 在 v2 中消失      → 删除 c3
  c4', c5' hash 相同   → 复用
```

典型场景：文档只在中间加了一段，只有受影响的 chunk 需要重算，其余直接复用。

注意事项：

- **切分器版本要一致**：修改文档时如果同时升级了切分策略（chunk 大小、重叠窗口），hash 会全部变化，diff 退化为全量重建。切分参数变了，应触发该文档（甚至全库）的显式重建任务，而不是混在增量 diff 里
- **元数据要更新**：`doc_version`、`updated_at`、行号偏移等溯源信息即使 chunk 内容没变也可能要改

### 5.3 版本保留（可选进阶）

对合规要求高的场景（如法务、医疗文档），可以保留版本链：

- chunk 带上 `valid_from` / `valid_to` 时间区间
- 检索时默认只查 `valid_to is null`（当前版本），需要时可按时间点回溯
- 定期归档过期版本

大多数业务场景不需要，用 `doc_version` 能溯源到源文档即可。

## 六、删除流程

删除最大的风险是**误删**和**删不干净**。

### 6.1 两阶段删除（推荐）

```
阶段一：软删除
  按 doc_id 将 chunk 置为 deleting
  → 立即从检索索引中排除（filter status=active）
  → 效果：用户马上搜不到，但数据还在

阶段二：物理删除（延迟执行，如 T+1 天）
  定时任务扫描 deleting 超过 N 天的 chunk
  → 物理删除向量和元数据
```

两阶段的好处：

1. **可反悔**：误删文档，恢复源文件后重新同步即可，甚至不用恢复
2. **绕开删除 API 的时序问题**：有些向量库删除是异步的，先标记后清理可以避免「删了但检索还能搜到」的诡异现象

### 6.2 按 doc_id 删除，绝不按内容删

删除必须通过 `doc_id`（或显式的 chunk_id 列表）定位，**永远不要**用「内容相似检索 → 删除」的方式删数据 —— 相似检索会误伤其他文档的 chunk。

批量删除时注意向量库 API 限制（很多库按 filter 删除有单次条数上限），需要分页循环删除直到删空。

### 6.3 孤儿 chunk 对账

无论流程多严谨，都会因为各种故障产生孤儿 chunk（源文档已不存在，向量库还有 active 状态的 chunk）。每日对账任务负责兜底：

```
对账逻辑（双向）：
  库中 active 的 doc_id ∉ 源文档集合  → 触发删除流程
  源文档 doc_id ∉ 库中 active 集合    → 触发新增/修复
```

## 七、一致性保障：没有分布式事务怎么办

源文档库和向量库之间没有事务，只能靠「**状态先行 + 幂等 + 对账**」三件套逼近一致：

```
           ┌──────────────┐
           │  变更事件队列  │  （可重试、可回放）
           └──────┬───────┘
                  ▼
        ┌───────────────────┐      写失败可重试（幂等：
        │  同步器 Sync Worker │      同一事件执行多次结果不变）
        └──────┬────────────┘
               ▼
        ┌───────────┐   ┌────────────┐
        │  向量库     │   │ 每日对账任务 │──► 差异报告 ──► 自动修复/告警
        └───────────┘   └────────────┘
```

三个关键原则：

1. **幂等**：所有同步操作以 `(doc_id, doc_version)` 为天然幂等键。重复消费同一条消息，第二次执行会因为 hash 相同而跳过
2. **顺序性**：同一 doc_id 的变更要串行处理（消息队列按 doc_id 哈希分区），否则「删除」和「修改」乱序到达会产生脏数据
3. **可观测**：每次同步记录审计日志（doc_id、操作类型、chunk 数量、耗时、结果），失败进死信队列 + 告警。检索结果中展示 chunk 的 `updated_at`，方便排查「答案为什么是旧信息」

## 八、整体架构

```
┌──────────┐  webhook   ┌──────────────┐
│ 文档源系统 │ ────────► │  变更事件队列  │
│ (CMS/Git/ │           └──────┬───────┘
│  对象存储) │                  │
└────┬─────┘                  ▼
     │ 定时扫描        ┌─────────────────┐
     └──────────────► │   同步服务        │
                      │  diff / 切分 /    │
                      │  Embedding / 写入 │
                      └───┬─────────┬───┘
                          ▼         ▼
                    ┌──────────┐ ┌─────────┐
                    │  向量库    │ │ 对账任务  │
                    │ chunk+元数据│ │ (每日)   │
                    └──────────┘ └─────────┘
```

## 九、同步器伪代码

以 TypeScript 描述核心逻辑：

```typescript
async function syncDocument(event: DocChangeEvent) {
  const { docId } = event;
  const newDoc = await fetchSourceDoc(docId); // 拉取源文档
  const newHash = sha256(newDoc.content);

  // 1. 幂等检查：内容没变直接跳过
  const existing = await vectorStore.getDocMeta(docId);
  if (existing?.status === 'active' && existing.docVersion === newHash) {
    return; // 重复事件
  }

  // 2. 文档已从源删除 → 走删除流程
  if (event.type === 'delete' || !newDoc) {
    await softDelete(docId);
    return;
  }

  // 3. 切分 + chunk 级 diff
  const newChunks = split(newDoc.content).map(c => ({
    ...c,
    chunkHash: sha256(c.text),
  }));
  const oldChunks = existing
    ? await vectorStore.listChunks(docId)
    : [];

  const oldByHash = new Map(oldChunks.map(c => [c.chunkHash, c]));
  const toInsert = [];
  const toDeleteIds = new Set(oldChunks.map(c => c.chunkId));

  for (const chunk of newChunks) {
    const reused = oldByHash.get(chunk.chunkHash);
    if (reused) {
      toDeleteIds.delete(reused.chunkId); // 复用，不重算
    } else {
      toInsert.push(chunk);               // 变化，需重建
    }
  }

  // 4. 先写后删，缩小检索空窗
  const embedded = await embedBatch(toInsert); // 攒批 Embedding
  await vectorStore.upsert(docId, newHash, embedded, { status: 'indexing' });
  if (toDeleteIds.size > 0) {
    await vectorStore.deleteChunks([...toDeleteIds]);
  }
  await vectorStore.activateDoc(docId);
}

// 软删除：先标记，物理清理交给定时任务
async function softDelete(docId: string) {
  await vectorStore.markDocStatus(docId, 'deleting');
}
```

## 十、总结

| 变更类型 | 核心策略 | 关键点 |
| -------- | -------- | ------ |
| 新增     | 幂等写入，indexing → active | hash 去重，攒批 Embedding |
| 修改     | chunk 级 diff，只重建变化部分 | 切分器版本一致，先写后删 |
| 删除     | 两阶段：软删除 → 延迟物理删除 | 按 doc_id 删，可反悔 |
| 兜底     | 每日对账 + 死信队列 | 双向 diff，自动修复或告警 |

一句话概括：**用 `doc_id + content_hash + chunk_hash` 建立身份体系，用状态机管理生命周期，用事件驱动 + 定时对账保障最终一致**。做到这三点，知识库就能在文档持续变更的情况下，始终给 RAG 提供可信的检索结果。
