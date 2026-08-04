# 让 Agent 拥有长期记忆：agentpack-memory 设计与实现

> 一个零依赖、零 API Key 开箱即用的 Agent 记忆插件，实现「捕获 -> 检索 -> 注入 -> 合并」的跨会话长期记忆闭环。

## 一、为什么 Agent 需要长期记忆

大语言模型（LLM）本身是无状态的--每次请求都是一次「从零开始」的推理。现有的会话级消息持久化（如 agentpack 的 `FileSessionStorage`）只能保存当前对话的线性消息流，存在两个致命缺陷：

1. **跨会话失忆**：开一个新会话，Agent 就忘了你上次说过的偏好、约束和决策。每次都要重新解释架构、重新发现同样的约束。
2. **上下文膨胀**：长会话的完整消息历史塞进上下文窗口，token 成本线性增长，还会触发窗口溢出截断，丢失早期关键信息。

人类的记忆系统不是这样工作的。我们有**短期记忆**（当前对话上下文）和**长期记忆**（跨会话的知识、偏好、事实）。Agent 也需要类似的能力：从对话中自动提取要点存为可检索的记忆，在新会话中自动检索相关记忆注入上下文，并定期合并去重、修剪过期内容。

这正是 `agentpack-memory` 要解决的问题。

## 二、核心架构：一个完整的记忆闭环

`agentpack-memory` 的核心是一个五阶段闭环，参考了 [rohitg00/agentmemory](https://github.com/rohitg00/agentmemory) 的设计理念：

```
用户消息 ──▶ [Injection]  ──▶ 检索相关记忆 ──▶ 注入到 user 消息
                                                    │
                                                    ▼
                                             [Runtime 运行]
                                                    │
助手回复 ──▶ [Capture] ──▶ 提取要点 ──▶ 存储为记忆 ──▶ 定期合并
```

| 阶段 | 角色 | 触发时机 | 做什么 |
|------|------|---------|--------|
| **Capture（捕获）** | Extension | 每轮对话结束 | 从对话中提取要点，存为可检索记忆 |
| **Index（索引）** | Store | 写入时 | 建立倒排索引 + 可选向量索引 |
| **Recall（检索）** | Retriever | 每轮对话开始 | 根据当前 query 检索 top-K 相关记忆 |
| **Inject（注入）** | Transformer | 每轮对话开始 | 将检索到的记忆注入 user 消息 |
| **Consolidate（合并）** | Consolidator | 周期性触发 | 去重、合并相似记忆、修剪过期内容 |

在 agentpack 框架中，这五个阶段分别对应 **Extension + Transformer + Tool** 三种插件形态的组合：

```
Extension（生命周期钩子）  ->  Capture
Transformer（消息变换管道） ->  Inject
Tool（Agent 可调用工具）   ->  save / search / list / delete
```

## 三、快速上手

### 安装

```bash
pnpm add agentpack-memory
# 或
npm install agentpack-memory
```

`agentpack` 为 peer 依赖，需同时安装。

### 零配置接入

在 `agentpack.config.js` 中三行代码即可启用：

```js
import { createMemoryPlugin } from 'agentpack-memory';

const mem = createMemoryPlugin({
  baseDir: '~/.agentpack/memory',  // 记忆存储目录
  maxMemories: 5,                  // 每轮注入 top-5
});

const r = mem.install();

export default {
  provider: 'deepseek',
  model: 'deepseek-v4-flash',
  systemPrompt: '你是一个有用的助手',
  sessions: { enabled: true, baseDir: './sessions', maxAge: 30 },
  extensions: r.extensions,
  transformers: r.transformers,
  tools: r.tools,
};
```

接入后，Agent 立刻获得四种能力：

- **自动记忆**：每轮对话结束自动提取要点存为记忆
- **自动召回**：每轮对话开始自动检索相关记忆注入上下文
- **手动工具**：4 个 Agent 可调用的记忆工具
- **零依赖**：默认纯 BM25 检索，无需 API Key，无需数据库

### 编程式 API

也可以直接操作 store：

```typescript
import { createMemoryPlugin, InMemoryStore } from 'agentpack-memory';

const store = new InMemoryStore();
const mem = createMemoryPlugin({ store });

// 直接保存一条记忆
await mem.store.save({
  content: '用户偏好深色主题',
  concepts: ['ui', 'dark-mode'],
  confidence: 0.8,
  source: 'tool',
});

// 检索相关记忆
const results = await mem.store.search('主题偏好', 5);
console.log(results[0]?.entry.content);  // '用户偏好深色主题'

// 手动触发合并去重
await mem.store.consolidate({ similarityThreshold: 0.85 });
```

## 四、核心设计决策

### 决策一：用 sentinel 哨兵包裹的记忆块注入 user 消息

这是整个插件最精巧的设计。问题是：检索到的记忆放在哪里，模型才能看到？

**方案对比：**

| 方案 | 否定原因 |
|------|---------|
| 注入 system 消息 | agentpack 的 `buildContext` 会过滤 `role==='system'` 的消息，根本到不了模型 |
| 新增独立 user 资源 | 产生「连续两条 user 消息」，部分 provider 解析异常 |
| 用 resource.meta 标记 | meta 在 message ↔ resource 往返中会丢失，无法跨轮识别 |

**最终方案**：把记忆用 sentinel 包裹块合并进**最新 user 消息的 content**中：

```
<<<AGENTPACK_MEMORY>>>
[Relevant memories]
- 用户偏好 React + TypeScript (score=0.82, id=mem_xxx)
<<</AGENTPACK_MEMORY>>

<原始用户消息>
```

sentinel 是 content 文本的一部分，随消息持久化进 session 存储。下一轮对话开始时，注入器会**先剥除所有 user 消息中的旧 sentinel 块**，再注入新的--这就是「先剥后注」机制。

对应源码 [sentinels.ts](file:///Users/kye/Documents/ai/agentpack/packages/agentpack-memory/src/injection/sentinels.ts)：

```typescript
export const MEMORY_BLOCK_START = '<<<AGENTPACK_MEMORY>>>';
export const MEMORY_BLOCK_END = '<<</AGENTPACK_MEMORY>>>';

/** 剥离 sentinel 包裹块，返回干净原文 */
export function stripMemoryBlock(text: string): string {
  const re = new RegExp(
    `${escapeRegex(MEMORY_BLOCK_START)}[\\s\\S]*?${escapeRegex(MEMORY_BLOCK_END)}\\s*`,
    'g',
  );
  return text.replace(re, '').trim();
}

/** 将检索结果构造为可读的记忆块文本 */
export function buildMemoryBlock(results: MemorySearchResult[]): string {
  if (results.length === 0) return '';
  const lines = ['[Relevant memories]'];
  for (const r of results) {
    const score = r.score.toFixed(2);
    const content = r.entry.content.replace(/\s+/g, ' ').trim().slice(0, 500);
    lines.push(`- ${content} (score=${score}, id=${r.entry.id})`);
  }
  return wrapMemoryBlock(lines);
}
```

注入器 [injection-transformer.ts](file:///Users/kye/Documents/ai/agentpack/packages/agentpack-memory/src/injection/injection-transformer.ts) 在 Pipeline 中以 `priority=5` 最先执行，每轮严格遵循四步：

1. 遍历所有 user 消息资源，剥除其中的旧 sentinel 块
2. 取最新 user 消息纯文本作为检索 query
3. HybridRetriever 取 top-K，过滤 minScore
4. 非空则把记忆块前插进最新 user 消息内容

```typescript
protected async run(resources: ContextResource[]): Promise<ContextResource[]> {
  // 1. 先剥：清除所有 user 消息中的旧 sentinel 块
  let cleaned = resources.map((r) => this.stripResource(r));

  // 2. 找最新 user 消息作为检索 query
  const latest = this.findLatestUserMessage(cleaned);
  if (!latest) return cleaned;
  const query = this.extractUserText(latest);

  // 3. 检索 top-K
  const results = await this.retriever.search(query, this.maxMemories);
  if (results.length === 0) return cleaned;

  // 4. 后注：把记忆块前插进最新 user 消息
  const block = buildMemoryBlock(results);
  return this.injectIntoLatest(cleaned, latest, block);
}
```

**为什么幂等？** 因为「先剥后注」保证无论执行多少次，结果一致：旧块被剥除，只注入当前轮的新块。即使 transformer 每次 runLoop iteration 都跑，也不会累积。

### 决策二：零依赖 BM25 + 可插拔 Embedder

检索是记忆系统的核心。`agentpack-memory` 的检索策略是**渐进增强**的：

| 模式 | 触发条件 | 原理 |
|------|---------|------|
| 纯 BM25 | 未配置 `embedder`（默认） | 关键词倒排索引，min-max 归一化 |
| 混合检索 | 配置了 `embedder` | BM25 候选 + 向量 cosine 加权融合 |

#### BM25 倒排索引

BM25 是经典的关键词检索算法，公式为：

```
score(q, d) = Σ_t idf(t) × (tf(t,d) × (k1+1)) / (tf(t,d) + k1×(1 - b + b×|d|/avgdl))
idf(t) = ln((N - df(t) + 0.5) / (df(t) + 0.5) + 1)
```

其中 `k1=1.5`（词频饱和参数）、`b=0.75`（文档长度归一化参数）。

实现 [bm25.ts](file:///Users/kye/Documents/ai/agentpack/packages/agentpack-memory/src/retrieval/bm25.ts) 采用倒排表结构，检索时只扫描 query token 命中的文档，避免全量计算：

```typescript
export class BM25Index {
  private docs = new Map<string, DocEntry>();
  private inverted = new Map<string, Set<string>>(); // token -> 文档 id 集合

  search(queryTokens: string[], limit = 10): Array<{ id: string; score: number }> {
    const scores = new Map<string, number>();
    for (const t of queryTokens) {
      const set = this.inverted.get(t);  // 只扫描命中文档
      if (!set) continue;
      const idf = Math.log((N - set.size + 0.5) / (set.size + 0.5) + 1);
      for (const docId of set) {
        const tf = docs.get(docId).tf.get(t);
        // BM25 公式计算贡献
        scores.set(docId, (scores.get(docId) ?? 0) + contrib);
      }
    }
    return [...scores.entries()].sort((a, b) => b[1] - a[1]).slice(0, limit);
  }
}
```

#### CJK 分词

分词器 [tokenizer.ts](file:///Users/kye/Documents/ai/agentpack/packages/agentpack-memory/src/retrieval/tokenizer.ts) 同时支持中英文：

- **Latin**：小写化 + 按非字母数字分割
- **CJK**：`\u4e00-\u9fff` 范围逐字符成 token（适合中文短文本语义检索）

```typescript
export function tokenize(text: string): string[] {
  const tokens: string[] = [];
  let latinBuf = '';
  for (const ch of text) {
    if (isCJK(ch)) {
      flushLatin();
      tokens.push(ch);         // CJK 逐字成 token
    } else if (isCJKSymbol(ch)) {
      flushLatin();
      // CJK 标点视为分隔符
    } else {
      latinBuf += ch;          // 累积 latin
    }
  }
  flushLatin();  // 小写化 + 正则分割
  return tokens;
}
```

#### 混合检索

配置 `embedder` 后，自动升级为 BM25 + 向量混合检索 [hybrid-retriever.ts](file:///Users/kye/Documents/ai/agentpack/packages/agentpack-memory/src/retrieval/hybrid-retriever.ts)：

```
1. BM25 取 limit×3 候选（扩大召回池）
2. query 与每个候选用 embedding 算 cosine 相似度
3. 两路分数分别 min-max 归一化
4. 加权融合：final = w_bm25 × bm25_norm + w_embed × cos_norm
5. 过滤 < minScore，截取 top-K
```

```typescript
async search(query: string, limit = 5): Promise<MemorySearchResult[]> {
  const bm25Results = await this.bm25.search(query, limit * 3);
  if (!this.embedder) return this.normalizeAndFilter(bm25Results);  // 纯 BM25

  const queryVec = await this.embedder.embed(query);
  const cosScores = bm25Results.map(r => cosine(queryVec, r.entry.embedding));
  const bm25Normed = minMaxNormalize(bm25Results.map(r => r.score));
  const cosNormed = minMaxNormalize(cosScores);

  return bm25Results.map((r, i) => ({
    entry: r.entry,
    score: (this.bm25Weight * bm25Normed[i] + this.embedWeight * cosNormed[i]) / wSum,
    matchedBy: cosNormed[i] >= bm25Normed[i] ? 'embedding' : 'hybrid',
  })).filter(r => r.score >= this.minScore);
}
```

接入 ollama embedding 示例：

```typescript
const ollamaEmbedder: Embedder = {
  async embed(text: string): Promise<number[]> {
    const res = await fetch('http://localhost:11434/api/embeddings', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ model: 'nomic-embed-text', prompt: text }),
    });
    return (await res.json()).embedding;
  },
  dimension: 768,
};

const mem = createMemoryPlugin({ embedder: ollamaEmbedder });
```

### 决策三：beforeRun + done 的 FIFO 配对捕获

Capture 需要在每轮对话结束时提取要点存盘。但 agentpack 框架的 `done` 钩子只接收 `Result`，不含 `sessionKey` 和原始 user 消息--这是框架的限制。

解决方案 [capture-extension.ts](file:///Users/kye/Documents/ai/agentpack/packages/agentpack-memory/src/capture/capture-extension.ts)：在 `beforeRun` 钩子中先把请求信息 stash 起来，`done` 时通过 FIFO 队列配对取出：

```typescript
export class MemoryCaptureExtension extends BaseExtension {
  private pending = new Map<string, PendingCapture>();
  private queue: string[] = [];

  protected setup(hooks: RuntimeHooks): void {
    // beforeRun：stash 请求信息（不改请求）
    hooks.beforeRun.tapPromise('memory-capture', async (request: Request) => {
      this.pending.set(request.sessionKey, {
        message: request.message,
        timestamp: Date.now(),
      });
      this.queue.push(request.sessionKey);
      return request;
    });

    // done：从队列取出 sessionKey，组装并保存记忆
    hooks.done.tapPromise('memory-capture', async (result: Result) => {
      const sessionKey = this.queue.shift();  // FIFO 配对
      const stashed = this.pending.get(sessionKey);
      if (!stashed || !result.success) return;

      const extracted = await runCaptureExtractor({
        userMessage: stashed.message,
        assistantContent: result.content,
        toolsUsed: result.toolsUsed,
        summarizeFn: this.summarizeFn,  // 可选 LLM 摘要
      });

      await this.store.save({
        content: extracted.content,
        concepts: extracted.concepts,
        confidence: extracted.summarized ? 0.8 : 0.6,
        source: 'capture',
        sessionKey,
      });
    });
  }
}
```

**配对正确性**：在典型的顺序 `await runtime.run()` 调用下，`beforeRun -> ... -> done` 在同一 async frame 中顺序执行，FIFO 队列精确配对。并发多会话场景下为 best-effort（框架限制），建议用 `save_memory` 工具或顺序运行。

### 决策四：记忆合并与生命周期管理

随着记忆不断积累，重复、过时、低质量的内容会稀释检索效果。Consolidator [consolidator.ts](file:///Users/kye/Documents/ai/agentpack/packages/agentpack-memory/src/consolidation/consolidator.ts) 负责三件事：

1. **去重合并**：用每条记忆的 content 作为 query 检索相似记忆，相似度 >= 阈值（默认 0.85）则合并
2. **修剪过期**：删除超过 `maxAgeMs` 的记忆
3. **修剪低质**：删除置信度低于 `minConfidence` 的记忆

合并两条记忆时的策略：

```typescript
function mergeTwo(a: MemoryEntry, b: MemoryEntry): MemoryEntry {
  const newer = a.updatedAt >= b.updatedAt ? a : b;
  return {
    id: newer.id,
    content: newer.content.length >= older.content.length ? newer.content : older.content,
    concepts: [...new Set([...a.concepts, ...b.concepts])],  // 并集
    confidence: Math.min(1, a.confidence + b.confidence + 0.1),  // 累加截断
    recallCount: a.recallCount + b.recallCount,
    source: 'consolidation',
    // ...
  };
}
```

## 五、记忆的数据结构

每条记忆是一个 `MemoryEntry`，包含正文、置信度、生命周期和检索统计：

```typescript
interface MemoryEntry {
  id: string;              // 唯一标识
  content: string;         // 记忆正文
  concepts: string[];      // 关键词标签（用于 BM25 索引）
  confidence: number;      // 置信度 0..1
  source: 'capture' | 'tool' | 'consolidation';  // 来源
  sessionKey?: string;     // 来源会话
  createdAt: number;       // 创建时间
  updatedAt: number;       // 更新时间
  lastRecalledAt?: number; // 最后被检索注入的时间
  recallCount: number;     // 被检索注入次数
  embedding?: number[];     // 向量（配置 embedder 时填充）
  meta?: Record<string, unknown>;
}
```

- **confidence**：捕获默认 0.6，LLM 摘要 0.8，合并后累加并截断到 1，可衰减
- **recallCount / lastRecalledAt**：记录记忆被使用的频率，可用于未来优化召回策略
- **source**：区分自动捕获、Agent 工具主动保存、合并产生

## 六、Agent 可调用的记忆工具

插件自动注册 4 个 Agent 工具 [memory-tools.ts](file:///Users/kye/Documents/ai/agentpack/packages/agentpack-memory/src/tools/memory-tools.ts)，让 Agent 可以主动管理记忆：

| 工具 | 参数 | 说明 |
|------|------|------|
| `save_memory` | `content`, `concepts?` | 保存一条长期记忆 |
| `search_memory` | `query`, `limit?` | 检索相关记忆 |
| `list_memories` | `limit?` | 列出最近记忆 |
| `delete_memory` | `id` | 删除一条记忆 |

工具用纯 JSON Schema 定义参数（不依赖 TypeBox），返回 `ToolResult`。Agent 在对话中可以像调用其他工具一样调用它们：

```
用户：记住我偏好用 React + TypeScript 做项目
Agent：[调用 save_memory] -> 已保存记忆（id=mem_abc123）
```

## 七、文件存储与原子写入

默认的 `FileMemoryStore` [file-memory-store.ts](file:///Users/kye/Documents/ai/agentpack/packages/agentpack-memory/src/store/file-memory-store.ts) 采用每条记忆一个 JSON 文件的方式持久化：

```
~/.agentpack/memory/
├── mem_001.json
├── mem_002.json
└── ...
```

写入采用 **temp + rename 原子替换**（镜像 agentpack 的 `FileSessionStorage`），避免写入中断导致文件损坏：

```typescript
private async writeEntry(entry: MemoryEntry): Promise<void> {
  const target = this.entryPath(entry.id);
  const tmp = `${target}.${process.pid}.tmp`;
  await fs.writeFile(tmp, JSON.stringify(entry, null, 2), 'utf-8');
  await fs.rename(tmp, target);  // 原子替换
}
```

内存中维护 `MemoryIndex` + `BM25Index` 增量索引，save/delete 时同步更新，首次访问时懒加载并按 `maxAge` 惰性清理过期条目。

## 八、配置选项速查

```typescript
createMemoryPlugin({
  baseDir: '~/.agentpack/memory',  // 存储目录
  maxMemories: 5,                    // 每轮注入 top-K
  minScore: 0.1,                     // 最低相关度阈值
  capture: true,                     // 捕获开关
  inject: true,                      // 注入开关
  tools: true,                       // 记忆工具开关
  embedder: ollamaEmbedder,          // 向量化器（启用混合检索）
  summarizeFn: mySummarize,          // LLM 摘要函数
  consolidateOn: 10,                 // 每 10 次捕获自动合并
});
```

| 选项 | 默认 | 说明 |
|------|------|------|
| `baseDir` | `<cwd>/.agentpack/memory` | 存储目录（支持 `~`） |
| `maxMemories` | `5` | 每轮注入 top-K |
| `minScore` | `0.1` | 最低相关度阈值 |
| `embedder` | - | 向量化器（启用混合检索） |
| `summarizeFn` | - | LLM 摘要函数 |
| `consolidateOn` | `0` | 每 N 次捕获自动合并（0=不自动） |

## 九、设计哲学总结

`agentpack-memory` 的设计遵循三个原则：

1. **零依赖开箱即用**--默认纯 BM25 检索，不引入任何运行时依赖，不需要 API Key，不需要数据库。`pnpm install` 后三行配置即可用。

2. **渐进增强**--提供 `embedder` 接口后自动升级为混合检索，提供 `summarizeFn` 后 capture 走 LLM 摘要。能力随配置增强，但默认行为已经足够好用。

3. **框架原生集成**--不是独立的记忆系统，而是 agentpack 的 Extension + Transformer + Tool 组合插件。记忆的捕获和注入完全融入 Runtime 生命周期，Agent 无感知。

```
agentpack Runtime 生命周期
├── beforeRun ──▶ [Capture Extension] stash 请求
├── Pipeline
│   ├── priority 5  ──▶ [Injection Transformer] 先剥后注
│   ├── priority 10 ──▶ ToolPairing
│   ├── priority 20 ──▶ SystemMessageCleaner
│   └── priority 90 ──▶ Truncation
├── streamFn (LLM 调用)
└── done ──▶ [Capture Extension] 提取要点存盘
```

记忆是 Agent 从「金鱼记忆」走向「长期助手」的关键一步。`agentpack-memory` 用不到 1000 行零依赖代码，实现了完整的记忆闭环--这正是它设计的精妙之处。
