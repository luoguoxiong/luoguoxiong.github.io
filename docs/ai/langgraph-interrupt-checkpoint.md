# LangGraph 的 interrupt 与 checkpoint 深度解析：从挂起恢复到多实例部署

> 「让 Agent 停下来问人一句话」，听起来只是一个 `await`，做起来却是一整套分布式状态机工程。LangGraph 用 **checkpoint（检查点）** 记录图的每一步状态，用 **interrupt（中断）** 把「等待外部输入」建模成图的一等公民。本文从 Pregel 超步模型讲起，拆解 checkpoint 的数据结构、interrupt 的真实执行时序、以及把这套东西搬上多实例部署时必须处理的九个问题。

## 一、为什么 interrupt 不是一个 `await`

先看一个人机协同（Human-in-the-loop）场景：Agent 帮用户起草了一封邮件，发出去之前需要老板点「同意」。

朴素写法大概是这样：

```ts
async function handle(input: string) {
  const draft = await llm.invoke(input);          // 耗时 3s
  const decision = await waitForHumanApproval(draft); // 耗时……3 分钟到 3 天
  if (decision === "approve") await sendMail(draft);
}
```

这段代码在 Demo 里能跑，一到生产就四处漏风：

| 问题 | 后果 |
|------|------|
| 进程重启 / 发布滚动 | 等待中的任务连同内存里的 `draft` 一起蒸发 |
| 用户关掉浏览器 | 请求连接断开，Promise 永远 resolve 不了，协程泄漏 |
| 网关超时 | 一个等 3 天的 HTTP 请求，任何一层中间件都会先把它掐断 |
| 水平扩容 | 「问人」发生在实例 A，用户的「回答」打到实例 B，谁也找不到谁 |
| 多轮审批 | 驳回后要带着反馈重跑，草稿、轮次、反馈历史都得自己管 |

问题的本质是：**等待人类的这段时间里，进程不可能、也不应该保持运行状态。** 唯一可靠的办法是把「执行到哪儿了、当时的变量是什么」落盘，让进程彻底死掉也不丢；等人类答复到达时，再由任意一个进程把状态捞回来继续。

这就是 checkpoint 与 interrupt 要解决的问题：

- **checkpoint**：把图的状态（每个 channel 的值、每个 task 的写入）**序列化到外部存储**，形成一条可回溯的时间线
- **interrupt**：一种**受控的「任务未完成」信号**，它告诉图「这个节点还没跑完，先记下来，等有人带值回来」

一句话概括：**checkpoint 解决「状态放哪」，interrupt 解决「什么时候停下来」。**

## 二、checkpoint：图的每一次呼吸都被记下来

### 2.1 先理解 Pregel 超步模型

LangGraph 的执行引擎源自 Google 的 Pregel / BSP 模型。它不像普通程序那样「函数调用到哪，PC 指针就在哪」，而是按 **superstep（超步）** 推进：

```
┌─ superstep N ─────────────────────────────────────────────┐
│                                                            │
│  1. Plan  依据各 channel 的版本号，算出本轮该跑哪些 task    │
│  2. Exec  并发执行这些 task（一个节点可以产生多个 task）    │
│          每个 task 的执行结果 = 一组 write( channel, value) │
│  3. Apply 把这一轮所有 write 按 reducer 合并进各 channel    │
│  4. Save  调用 checkpointer.put()，落一个 checkpoint        │
│                                                            │
└──────────────────────────┬─────────────────────────────────┘
                           ▼  检查是否有 channel 变化 → 进入 superstep N+1
```

三个关键概念：

- **channel**：图的「状态变量」。你用 `Annotation.Root({...})` 声明的每个字段都是一个 channel，各自带一个 reducer（合并函数）
- **task**：一个节点的一次具体执行。一个节点在一轮里可能对应多个 task（比如用于 map-reduce 的 `Send`）
- **write**：task 的输出不是「返回值」，而是若干个 `(channel, value)` 写入项，由 reducer 决定怎么合并

理解「图推进靠的是 channel 版本号，而不是程序计数器」非常重要——这是后面所有「重放」行为的根源。

### 2.2 checkpoint 长什么样

每轮超步结束、引擎 `save` 的那份数据就是 checkpoint。它大概长这样（各语言 SDK 字段名基本一致，都是 snake_case）：

```ts
interface Checkpoint {
  v: number;                              // schema 版本
  id: string;                             // 本 checkpoin 的 id（可读的时间线游标）
  ts: string;                             // 时间戳
  channel_values: Record<string, any>;    // 各 channel 当前值
  channel_versions: Record<string, number>;   // 各 channel 的版本号（自增）
  versions_seen: Record<string, Record<string, number>>; // 每个 task 见过的 channel 版本
  updated_channels: string[] | null;      // 本轮被更新的 channel
  pending_sends: Send[];                  // 待派发的 Send（动态 fan-out）
}

interface CheckpointTuple {
  config: RunnableConfig;                 // 含 thread_id / checkpoint_ns / checkpoint_id
  checkpoint: Checkpoint;
  metadata: CheckpointMetadata;           // source / step / parents 等
  parentConfig?: RunnableConfig;          // ← 指向上一个 checkpoint
  pendingWrites?: PendingWrite[];         // [taskId, channel, value][]
}
```

有三处值得停下来看：

**① `parentConfig` 让 checkpoint 构成链表。** 顺着它能一路走回起点，这就是 **时间旅行（time travel）** 与**人为修改历史状态（`graph.updateState`）**的基础：

```
cp_0 ──parent──▶ cp_1 ──parent──▶ cp_2 ──parent──▶ cp_3 (最新)
                   ▲                                │
                   └──── 用 updateState 在 cp_1 上分叉出新时间线 cp_3'
```

**② `versions_seen` 决定下一轮跑谁。** 调度器会比较「channel 当前版本」和「task 上次记录的版本」，只要有一个被 task 读的 channel 变新了，这个 task 就要再跑一次。**这也是为什么后面 resume 时会「整个节点重跑」而不是「从断点继续」。**

**③ `pendingWrites` 是中断 payload 的家。** 被中断的那个 task，它的产出不会进 channel，而是以 `__interrupt__` 这条特殊 channel 的 write 形式挂在这里，等待 resume 时被消费。

### 2.3 存储：三张表

以 Postgres 实现为例（`PostgresSaver` / `ShallowPostgresSaver`），checkpoint 被拆成三张表：

```sql
-- 主线：checkpoint 的元数据与「大件」载荷
CREATE TABLE checkpoints (
  thread_id            TEXT NOT NULL,
  checkpoint_ns        TEXT NOT NULL DEFAULT '',
  checkpoint_id        TEXT NOT NULL,
  parent_checkpoint_id TEXT,
  type                 TEXT,
  checkpoint           BYTEA,
  metadata             JSONB NOT NULL DEFAULT '{}',
  PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id)
);

-- 每个 channel 的值单独存 blob（懒加载 / 大对象分离）
CREATE TABLE checkpoint_blobs (
  thread_id     TEXT NOT NULL,
  checkpoint_ns TEXT NOT NULL DEFAULT '',
  channel       TEXT NOT NULL,
  version       TEXT NOT NULL,
  type          TEXT NOT NULL,
  blob          BYTEA NOT NULL,
  PRIMARY KEY (thread_id, checkpoint_ns, channel, version)
);

-- 每轮超步里每个 task 的每一次写入，逐条留存
CREATE TABLE checkpoint_writes (
  thread_id     TEXT NOT NULL,
  checkpoint_ns TEXT NOT NULL DEFAULT '',
  checkpoint_id TEXT NOT NULL,
  task_id       TEXT NOT NULL,
  idx           INTEGER NOT NULL,
  channel       TEXT NOT NULL,
  type          TEXT,
  blob          BYTEA NOT NULL,
  PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id, task_id, idx)
);
```

`checkpoint_writes` 的主键 `(thread_id, checkpoint_ns, checkpoint_id, task_id, idx)` 是一把天然的保护锁：**同一个 task 试图写入同一条记录两次，会因主键冲突而失败**。这防止了重放时的重复写入，也是并发场景下最后一道防线。

两个命名要注意：

- **`thread_id`**：一次「图的执行生命周期」的主键。你可以把它理解为「会话 id」，但它比会话更精确——同一用户可以有多个 thread。
- **`checkpoint_ns`**：命名空间。**子图会拥有自己的 ns**（形如 `parent|child`），这样主图和子图可以在同一张表里共存而不会互相覆盖。写自定义 SQL 查询时千万别忘带上它。

### 2.4 落盘时机与幂等

```
 task 执行中 ────▶ putWrites() ──▶ checkpoint_writes（逐条即时写入）
                        │
 superstep 全部 task 结束 ──▶ put() ──▶ checkpoints + checkpoint_blobs（写快照）
                        │
 下一 superstep 开始 ──▶ getTuple() ──▶ 读取上一个 checkpoint + pendingWrites
```

- `put(config, checkpoint, metadata)`：每个超步结束后写主线 checkpoint
- `putWrites(config, writes, taskId)`：task 产生的写入即时落 `checkpoint_writes`

这意味着 **checkpoint 是增量式的，而不是「每次全量覆盖」**：主线记录一份快照，具体写入逐条累积。好处是 task 执行到一半进程崩了，已经产生的部分写入仍在 `checkpoint_writes` 里，重放时可以精确续上；坏处是它依赖「重放必须是确定性的」——下一节就讲这一点。

## 三、interrupt 到底是怎么工作的

### 3.1 它不是线程睡眠

最重要的一句话：**`interrupt()` 不是让线程阻塞，而是抛出一个特殊的异常（`GraphInterrupt`），让当前 task 以「未完成」的姿态终止，并把「我为什么停」记录成一条 write。**

```ts
async function review(state: GraphState) {
  // 1. 这里的代码在「第一个超步」执行
  const summary = summarize(state.draft);

  // 2. 这一行抛出 GraphInterrupt，task 结束，本次 invoke 返回
  const decision = interrupt<Decision>({
    question: "确认发送这封邮件？",
    draft: state.draft,
    summary,
  });

  // 3. 下面这些代码在「resume 的那个超步」才会第一次跑到
  return { feedback: [`审批结果：${decision}`] };
}
```

于是 `interrupt()` 的返回值不是「立即拿到的人的回答」，而是**下一次重跑这个 task 时被注入的值**。这就是整篇文章的核心：

> **interrupt 的语义是「保存现场并退出」；resume 的语义是「按同一份现场把 task 再跑一遍，这次给 interrupt 一个返回值」。**

### 3.2 完整时序

```
 客户端                实例 A                    检查点存储              实例 B
   │                    │                          │                      │
   │  POST /run         │                          │                      │
   ├───────────────────▶│  invoke(input)           │                      │
   │                    │  superstep1: draft ──────┤ put(cp1)             │
   │                    │  superstep2: review      │                      │
   │                    │    ├ summarize()         │                      │
   │                    │    └ interrupt() ✖ throw │                      │
   │                    │  putWrites(__interrupt__)┤                      │
   │  200 {status:"     │◀─────────────────────────┤                      │
   │   interrupted"}    │                          │                      │
   │                    │                          │                      │
   │   … 3 分钟后，用户点了「驳回」…                  │                      │
   │                    │                          │                      │
   │  POST /resume ─────┼──────────────────────────┼─────────────────────▶│
   │                    │                          │  getTuple(thread) ──▶│
   │                    │                          │◀─────────────────────┤
   │                    │                          │  重放 review task     │
   │                    │                          │  interrupt() 返回     │
   │                    │                          │   "reject"           │
   │                    │                          │  put(cp2) ──────────▶│
   │◀───────────────────┼──────────────────────────┼──────────────────────┤
   │  200 {result}      │                          │                      │
```

注意恢复请求打到了**实例 B**上，而它照样能续跑——因为它读的是共享的检查点存储，而不是实例 A 的内存。这正是多实例部署能成立的底层前提。

### 3.3 resume 是怎么注入的

`Command({ resume })` 是携带人类输入的信封：

```ts
// 第一次：发起
await graph.invoke(
  { input: "帮我给供应商写封延期说明" },
  { configurable: { thread_id: "mail-2026-1026-01" } }
);

// 第二次：带着答复回来（可以是别的进程、别的机器、几天之后）
await graph.invoke(
  new Command({ resume: "reject" }),
  { configurable: { thread_id: "mail-2026-1026-01" } }
);
```

执行过程等价于：

1. 用同一份 `thread_id` 从存储尾部取出最后一个 checkpoint
2. 调度器发现 `review` 这个 task 处于未完成状态 → 重新执行 `review`
3. `interrupt()` 这次不再抛出，而是 **`return` 出 `resume` 携带的值**
4. `review` 继续往下走，`return { feedback: [...] }` 这才第一次真正写入 channel
5. 进 `put()` 落 cp2，图继续往下走

**所以第 1 步之前的那些代码（`summarize(state.draft)`）会被再执行一次。** 如果它是一次 LLM 调用，你会多花一次钱；如果它是一次 HTTP 副作用，你会多做一次操作。这一点会在「踩坑清单」里反复提到。

### 3.4 三种挂起方式，适用场景不同

| 方式 | 用法 | 特点 | 适用 |
|------|------|------|------|
| **节点内动态中断** | 节点函数中调用 `interrupt(payload)` | 中断点可以在任意分支、任意条件触发；payload 可携带运行时数据 | 缺参追问、工具前的审批、兜底询问 |
| **静态前置中断** | `compile({ checkpointer, interruptBefore: ["multi_tools"] })` | 图结构固定，进入指定节点前必停 | 高风险工具统一审批、演示/debug |
| **静态后置中断** | `compile({ checkpointer, interruptAfter: ["draft"] })` | 节点跑完、写回 channel 之后停 | 想先看产出再决定是否继续 |

静态中断的恢复很简单——不需要 `Command`，传 `null` 输入即可推进：

```ts
await graph.invoke(null, { configurable: { thread_id: tid } });
```

一个常见误解是「静态中断 = 节点没跑」。`interruptBefore` 确实没跑，**但 `interruptAfter` 的节点已经跑完并且写回了 channel**，resume 时不会重跑它。这两者的重放代价完全不同。

### 3.5 Command 的三件套：resume / update / goto

`Command` 不止能回传中断值：

```ts
// ① resume：把值交还给 interrupt() 的返回值
new Command({ resume: "approve" })

// ② update：直接修改图状态（相当于一次外部写库），asNode 决定记在谁头上
new Command({
  update: { draft: "（管理员已手动改写）" },
  asNode: "draft",
})

// ③ goto：强行把路由导到某个节点，可配合 update 形成「人工纠偏」
new Command({
  goto: "review",
  update: { draft: newDraft },
})
```

`goto` + `update` 的组合非常实用：人类不满意 Agent 的产出，可以直接改 state 再把它丢回指定节点重跑，而不用改图代码。注意 `goto` 后目标节点**一定会被执行**，即使它的 channel 版本没变——这是调度器的显式优先项。

在子图里，还有一个 `Command.PARENT`（Python 里写作 `Command(graph=Command.PARENT, ...)`）用于从子图内部跳回父图的节点，写多 Agent 编排时会用到。

### 3.6 怎么读到「当前挂在哪个中断上」

理想情况下：

```ts
const state = await graph.getState({ configurable: { thread_id: tid } });
console.log(state.tasks[0].interrupts);  // 中断 payload
console.log(state.next);                 // 下一个待执行的节点
```

**但这是一个有版本坑的 API。** 某些版本（例如 LangGraph.js 1.5.x）对处于中断态的线程，`getState()` 返回的 `values` 和 `tasks` 是空的——因为它默认按「最后一个已完成超步」的视角去切。遇到这种情况，直接读 checkpointer 更可靠：

```ts
async function readPendingInterrupts(saver, threadId: string) {
  const tuple = await saver.getTuple({
    configurable: { thread_id: threadId, checkpoint_ns: "" },
  });
  if (!tuple) return null;

  // pendingWrites: [taskId, channel, value][]
  return (tuple.pendingWrites ?? [])
    .filter(([, channel]) => channel === "__interrupt__")
    .flatMap(([, , value]) => (Array.isArray(value) ? value : [value]))
    .map((it: any) => (it && "value" in it ? it.value : it));
}
```

这段代码同时兼容了两种内部形态（单个 Interrupt 对象 / Interrupt 数组），比直接依赖 `state.tasks` 稳得多。**把「读中断」封装成一个带兼容处理的函数，是很有必要的。**

## 四、写对 interrupt 的十二条纪律

这些都是「重放语义」带来的必然要求，不是最佳实践，是**正确性**要求。

### 1）被中断的节点必须是幂等的

中断前的每一行代码在 resume 时都会**再跑一遍**。所以：

```ts
// ❌ 中断前做了非幂等副作用
async function pay(state) {
  await api.createOrder(state.order);        // resume 时会创建第二单！
  const ok = interrupt({ question: "付款？" });
  return { paid: ok };
}

// ✅ 把副作用挪到中断之后
async function pay(state) {
  const ok = interrupt({ question: "付款？", amount: state.amount });
  if (!ok) return { paid: false };
  const orderId = await api.createOrder(state.order);  // 只执行一次
  return { paid: true, orderId };
}
```

如果 LLM 调用本身很贵且结果稳定，可以用 LLM 缓存（同 prompt 命中缓存几乎零成本）代替手工去重。

### 2）用「是否已完成」标记来幂等，而不是靠节点只跑一次

对无法挪走的副作用，加 Idempotency Key：

```ts
async function charge(state, config) {
  const tid = config.configurable?.thread_id;
  const key = `${tid}:charge:${state.step}`;      // 幂等键由 thread + 步骤派生
  if (!await payments.exists(key)) {
    await payments.charge({ key, amount: state.amount });
  }
  const ok = interrupt({ question: "确认支付？" });
  return { paid: ok };
}
```

### 3）一个 task 里多次 interrupt，顺序非常重要

同一个 task 内连续 `interrupt()` 两次时，resume 传来的值**按出现顺序被依次消费**：

```ts
async function collect(state) {
  const city = interrupt({ ask: "城市？" });   // 第 1 个：消耗 resume 的第一个值
  const date = interrupt({ ask: "日期？" });   // 第 2 个
  return { city, date };
}
```

如果你的 resume 是一次一个值的循环式交互，这没问题；如果你期望「一次性给两个」，`resume` 需要传数组且处理逻辑要匹配。**更稳的做法是一次性把要问的字段打包成一个 payload 问完**，把多轮追问拆成多次「进入同一个节点」而不是一个节点里多次 interrupt。

### 4）永远不要在 try/catch 里吞掉它

```ts
// ❌ 致命：GraphInterrupt 被当成普通异常吞了，图会带着「半截状态」继续跑
try {
  const r = interrupt({ ask: "是否继续？" });
} catch (e) {
  console.error(e);
}

// ✅ 要么不包 try/catch，要么显式重抛 SDK 的中断类型
try {
  const r = interrupt({ ask: "是否继续？" });
} catch (e) {
  if (e?.name === "GraphInterrupt") throw e;   // 或 e instanceof GraphInterrupt
  // 其余才是真正的业务异常
}
```

### 5）不要在非受控的异步分支里调用

`interrupt()` 依赖当前执行上下文来确定「我是哪个 task」。如果你在 `Promise.all` 的某个分支、`setTimeout` 回调、或未 await 的 detached promise 里调用它，它要么找不到归属的 task，要么污染到别的 task。

### 6）payload 和 resume 值都必须可序列化

checkpoint 是要过 serde 写进数据库的。传函数、类实例、`Symbol`、`WeakMap` 进去，第一次可能「看起来没事」（因为写文件系统/内存 saver 宽容），换到 Postgres + JsonPlusSerializer 就直接报错。

### 7）State 里带默认值的字段，必须配 reducer

一个非常隐蔽的版本差异：某些版本的 `Annotation<string>({ default: () => "" })` **只有在同时提供了 `reducer` 时才生效**，否则得到的 channel 读出来是 `undefined`。表现就是「第一次跑得好好的，resume 之后某个字段突然变成 NaN / undefined」。稳妥写法：

```ts
const State = Annotation.Root({
  turns: Annotation<number>({
    reducer: (_a, b) => b ?? 0,
    default: () => 0,
  }),
});
```

### 8）中断 payload 就是你的前端协议

`interrupt(payload)` 的 payload 会被完整传到调用方，通常直接渲染成一个表单或确认卡。把它定义为**版本号化的结构**：

```ts
interrupt({
  v: 1,
  kind: "slot_filling",        // 缺参补全 / 危险操作审批 / 二选一
  fields: [{ key: "city", label: "出发城市", type: "enum", options: [...] }],
  context: { originalQuery: state.query },
})
```

前端按 `kind` 渲染，后端按 `fields` 校验。不要在 payload 里塞 HTML 或自由文本。

### 9）resume 的值必须校验

人类带来的值是外部输入。审批分支要按白名单取值，槽位补全要按 schema 校验类型与范围：

```ts
const decision = interrupt<unknown>({ question: "是否发送？" });
if (decision !== "approve" && decision !== "reject") {
  // 重新中断，而不是让脏值流进业务
  throw new Error("非法审批值");
}
```

### 10）给中断一个「人没来」的出路

用户问了一半关掉浏览器，这个 thread 就永远挂在数据库里了。**必须有 TTL 回收机制**（见 5.4）。

### 11）注意依赖升级带来的中断协议变化

不同大版本对「中断 payload 存哪、叫什么」的实现有差异（例如某些版本把 payload 塞进 `__interrupt__` channel 的 write，某些版本放在 task 元数据里；`getState()` 是否返回挂起态 tasks 也不同）。**升级后第一件事是跑一遍挂起-恢复的回归用例**，而不是只跑通 happy path。

### 12）不要在持有 HTTP 连接时干等

中断返回后 HTTP 请求应该**立即响应**「已挂起，等待输入」，然后由另一个请求来 resume。用长轮询或让连接挂 3 分钟，只会同时收获网关超时和连接耗尽。

## 五、多实例部署：checkpoint 共享只是迈出的第一步

### 5.1 根本矛盾

```
传统无状态服务：  任意请求 → 任意实例 → 结果一致        ✅ 天然支持水平扩容
带中断的 Agent：   进程内存里有「哪儿停了、转到哪儿去」  ❌ 换个实例就失忆
```

LangGraph 已经帮我们解决了最大的一块：**检查点落在共享存储里，所以「续跑」这件事本身天然支持跨实例**。这是好消息。

剩下的是那些**被留在进程内存里的东西**。多实例部署的全部工作，就是把这些残留逐个揪出来：

```
┌────────────────────────────────────────────────────────────┐
│  实例内残留清单（都必须外移或加跨实例协议）                  │
│                                                             │
│  1. 等待人类输入的「槽位」/ Promise                          │
│  2. 正在流式输出的 SSE 连接句柄                              │
│  3. 「这个 thread 正跑到一半」的并发互斥                     │
│  4. 热数据在内存里的会话缓存                                 │
│  5. 「谁在线」「几点接入」这类 presence 状态                 │
│  6. 限流计数                                                │
│  7. 挂起线程的登记表与清理器                                 │
└────────────────────────────────────────────────────────────┘
```

下面逐个说。

### 5.2 检查点必须外置，且要注意连接开销

**`MemorySaver` / `SqliteSaver` 只能用于单进程。** 一旦起了第二个实例，两边各写各的本地文件/内存，同一 thread 的两份时间线立刻分叉。

```ts
// ❌ 多实例下绝对不可用
const graph = flow.compile({ checkpointer: new MemorySaver() });

// ✅ 共享存储
const saver = new PostgresSaver(pool);
await saver.setup();   // 幂等建表，可在每个实例启动时调用
const graph = flow.compile({ checkpointer: saver });
```

几个工程细节：

- **`setup()` 要幂等**，让每个实例启动都调用；多个实例同时建表时依赖 `CREATE TABLE IF NOT EXISTS` 语义
- **和业务库共用连接池时要算清楚连接数**。每个超步会有 `put` + 若干 `putWrites`，一个长对话对连接的占用远高于普通 CRUD
- **给 `thread_id` 建好索引**（上面的建表语句里主键即索引），否则 `getTuple` 只会越来越慢
- **大 blob 注意行膨胀**：`checkpoint_blobs` 里的 `BYTEA` 随对话历史线性增长，需要有保留策略（见 5.4 的清理）

### 5.3 「人类的回答」怎么传递 —— 别在内存里攒答案

这是最常见的错误设计：

```ts
// ❌ 实例 A 持有一个 Promise 在等人类输入
const waiters = new Map<string, (v: unknown) => void>();
await new Promise(res => waiters.set(threadId, res));
```

如果 resume 请求落到实例 B，`waiters` 里没有这个 key，请求直接悬空；实例 A 重启，这个 Promise 永远无人兑现。

**方案 A（推荐）：回答直接由新请求携带，不经过任何中间态。**

用户的回答本来就是一个 HTTP 请求体，把它做成 `Command({ resume })` 交给**当前处理这个请求的任意实例**即可，不需要先落到 A 手里：

```ts
app.post("/threads/:tid/resume", async (req, res) => {
  const value = schema.parse(req.body.value);            // 先校验
  const cfg = { configurable: { thread_id: tid(req.params.tid) } };

  // 谁接到这个请求谁就续跑，A / B 都无所谓
  await graph.invoke(new Command({ resume: value }), cfg);
  res.json({ ok: true });
});
```

简单、无状态、天然支持多实例。**优先选这个。**

**方案 B：需要「先落地再异步消费」时用 Redis 槽位。**

某些场景下答复是从别处（消息队列、另一个后台任务）推来的，需要暂存：

```ts
// 登记（带 TTL，到期自动清理，防止答案永久占着内存）
await redis.set(`resume:${threadId}`, JSON.stringify(payload), "PX", ttlMs);

// 任意实例原子取出（GETDEL 需要 Redis >= 6.2；低版本用 Lua GET+DEL）
const raw = await redis.getdel(`resume:${threadId}`);
if (raw) await graph.invoke(new Command({ resume: JSON.parse(raw) }), cfg);
```

要点：

- **`GETDEL` 保证只有一个实例拿到**（原子性），避免双投
- **必须有 TTL**，否则一旦消费方没起来，答案永久占着内存
- **Redis 不可用时要有进程内存降级**，否则 Redis 一抖整个链路瘫痪；但降级态必须带**容量上限 ledger**，否则大量挂起会 OOM。超过上限时快速失败并提示用户重试，比拖垮实例好
- 注意 Redis 客户端要**新建一条连接做订阅**（`duplicate()`），订阅连接不能复用普通连接

### 5.4 悬挂线程的回收：最容易被忘的一件事

用户问到一半走了。**如果 Resume 请求永远不来，这个 thread 的 checkpoint 会永远躺在数据库里**，而且：

- 它可能持有一个不可序列化的资源引用（数据库连接、文件句柄）
- 它在 UI 上仍是「等待你确认」的僵尸会话

做法：**维护一张挂起注册表 + 定时清扫。**

```sql
CREATE TABLE pending_threads (
  thread_id   TEXT PRIMARY KEY,
  created_ts  TIMESTAMPTZ NOT NULL DEFAULT now(),
  ttl_ms      INTEGER NOT NULL DEFAULT 86400000,
  meta        JSONB NOT NULL DEFAULT '{}'
);
```

```ts
// 清扫器：每个实例都跑，靠 RETURNING 保证只有一个人拿到清理权
const expired = await pool.query(
  `DELETE FROM pending_threads
   WHERE created_ts < now() - ($1 || ' milliseconds')::interval
   RETURNING thread_id`,
  [ttlMs]
);
for (const r of expired.rows) {
  await saver.deleteThread?.({ configurable: { thread_id: r.thread_id } });
  bus.publish(`thread:${r.thread_id}`, { type: "interrupted", reason: "timeout" });
}
```

三个关键设计：

1. **`DELETE ... RETURNING` 天然是互斥的**：多个实例同时清扫，只有实际删掉行的那个实例会拿到结果，不需要额外加锁
2. **登记要早于任何可能失败的后续步骤**。哪怕后面出了问题，注册表里也有记录可以被清扫回收
3. **显式 `deleteThread`** 释放 checkpoint 行与关联 blob，别指望上层 TTL

有些实现还需要区分「用户主动取消」与「超时回收」：主动取消时在注册表里标记 `released`，清扫器跳过它。这样主动中断不会被误统计成超时，指标才是准的。

### 5.5 同一个 thread 绝对不能并发执行

两个请求同时用同一 `thread_id` 调 `invoke`，会发生什么？它们会从同一个父 checkpoint 出发，各自算出一条新时间线，最后互相覆盖写入；更糟的是触发 §2.3 那个主键冲突，出现莫名其妙的写失败。

处理思路按复杂度递增：

**① 路由级串行（最简，推荐先做）**：同一个用户的请求尽可能落到同一实例（但注意别用 LB sticky session，见 5.9），进程内用队列串行：

```ts
const running = new Map<string, Promise<unknown>>();
function enqueue(tid: string, fn: () => Promise<any>) {
  const prev = running.get(tid) ?? Promise.resolve();
  const cur = prev.then(fn, fn).finally(() => {
    if (running.get(tid) === cur) running.delete(tid);
  });
  running.set(tid, cur);
  return cur;
}
```

**② 分布式互斥**：单实例队列跨实例无效，需要一把跨进程锁：

```sql
INSERT INTO thread_locks (thread_id, owner, expire_at)
VALUES ($1, $2, now() + interval '30 seconds')
ON CONFLICT (thread_id) DO NOTHING
RETURNING thread_id;   -- 拿到行才算抢到锁
```

配 TTL 防死锁，释放时比对 `owner` 再删。

**③ 乐观并发**：允许并发进入，靠 checkpoint 的父版本做 CAS，冲突时重取再试。适合读多极少写冲突的场景，一般不必做到这一步。

另外，**业务层的状态变更也要跟着用 CAS**，别用「读-改-写」：

```sql
-- ❌ 两个实例可能同时判定自己是第一个接入者
UPDATE sessions SET status='active' WHERE id=$1;

-- ✅ 带条件的原子更新
UPDATE sessions SET status='active', joined_at=now()
WHERE id=$1 AND status='queued';   -- 只有真正抢到的那一个会 rows=1
```

### 5.6 流式输出（SSE）要有一条事件总线

这是最隐蔽的一个坑：用户在实例 A 上建立了 SSE 连接看流式输出，**但 resume 后的续跑发生在实例 B**。B 产生的 token 根本不会经过 A 的内存，用户盯着一个再也不吐字的连接。

```
用户 SSE ──▶ 实例 A           （持有连接）
                 ▲
                 │ 需要 token 事件跨实例送达
                 │
resume 请求 ──▶ 实例 B ──▶ 图执行 ──▶ 逐 token 产出
```

解法是**进程内直写 + Redis Pub/Sub 桥接**的事件总线：

```ts
const CH = (tid: string) => `lg:bus:thread:${tid}`;

// 发布（任意实例，按需发布）
async function emit(tid: string, ev: BusEvent) {
  writeLocalConnections(tid, ev);                  // 本机有的话直接写，零延迟
  await publisher.publish(CH(tid), JSON.stringify(ev)); // 同时广播给其他实例
}

// 订阅：只订阅本机持有连接的那些 thread
subscriber.on("message", (ch, raw) => {
  const tid = ch.split(":").pop()!;
  const ev = JSON.parse(raw);
  if (env.instanceId === ev.origin) return;        // 是自己发的本地消息，去重
  writeLocalConnections(tid, ev);
});
await subscriber.subscribe(CH(tid));   // 有本地连接时才 subscribe
```

工程要点：

- **消息里带 `origin`（实例 id）和 `seq`**：本地已经直写过了，收到自己的广播要丢弃
- **只订阅本机有连接的 thread**，别全局 `PSUBSCRIBE *`，否则每个实例都要处理全量流量
- **订阅必须用独立连接**（`duplicate()`），且要在初始化普通连接之后再做
- 需要断线续传时，事件按 `seq` 落一份短 TTL 缓冲，重连用 `Last-Event-ID` 回放
- Redis 不可用时降级为「本地直写」：跨实例的用户会丢流式输出，但不影响最终结果落到 DB

### 5.7 进程内的会话缓存会脏读

多实例下，「本实例缓存的对象」随时可能被别的实例改掉。

```ts
// ❌ 缓存不设上限 TTL，实例 B 改了，实例 A 一直给出旧值
const cache = new Map<string, Session>();

// ✅ 短 TTL 回源：允许短暂不一致，但要保证窗口足够小（如 2s）
function getCached(id: string) {
  const hit = cache.get(id);
  if (hit && Date.now() - hit.at < 2000) return hit.value;
  return loadFromDb(id).then(v => (cache.set(id, { value: v, at: Date.now() }), v));
}
```

对一致性要求高的操作（比如「接单」这种只能成功一次的动作），**不要读缓存，直接走 CAS 的 SQL**。

同样地，「客服在线状态」这类 presence 数据不能放进程内存，应该放 Redis 心跳：

```ts
await redis.set(`presence:agent:${agentId}`, instanceId, "EX", 20);
// 每 10s 续一次；连接断开后由最后一次断开的实例 DEL，或者直接靠 EX 过期
```

### 5.8 图版本升级：滚动发布时的隐形炸弹

多实例最难处理的一环：**新版本的图结构变了，数据库里还有上一个版本留下的 checkpoint。**

```
实例 v1 运行中 ────┐
                   ├──▶ 共享 checkpoint 库（混存 v1/v2 写入的时间线）
实例 v2 新上线 ────┘
```

可能发生的灾难：

- 新增了一个 channel，老 checkpoint 没有这个字段 → 读到 `undefined`，后续 reducer 崩
- 某个 channel 的 reducer 改了 → 合并结果与已存的时间线不一致
- 删除/重命名了一个节点 → 已挂起的 task（`checkpoint_writes` 里的 `task_id`）找不到对应节点，重放直接失败

可行策略：

1. **灰度 + 双跑**：新版本只承接新 thread，老 thread 继续由保留的旧实例消化，直到自然结束
2. **thread_id 带版本前缀**（如 `v2-${uuid}`），天然隔离两代时间线，也方便按前缀统计与清理
3. **破坏性变更必须换 thread 空间**：凡是改了 channel 名 / reducer 语义 / 节点拓扑，都视为数据格式升级，不要指望原地兼容
4. **保留 N 个版本的可反序列化能力**，至少能读到老 checkpoint 做一次性迁移或优雅报错
5. **发布顺序**：先扩 v2 → 观察 → 摘 v1 流量 → v1 挂起线程全部 resume 完或超时回收 → 下线 v1

### 5.9 关于负载均衡的两个反直觉结论

- **不要用 sticky session 来解决「结果显示不出来」**。sticky 会带来热点，而且一旦目标实例下线，会话照样断。正确做法是 §5.6 的事件总线：让「谁生成结果」和「谁持有连接」彻底解耦
- **但要可以让同一用户的普通请求尽量落到同一实例**（比如按 `userId` 哈希），这只是**提高本地缓存命中率**的优化手段，不是正确性依赖。业务正确性必须建立在「落到哪个实例都一样」的前提上

### 5.10 限流、并发闸与 thread_id 鉴权

- **限流计数必须外置**（Redis `INCR` + `PEXPIRE` 固定窗口 / 令牌桶，内存降级）。每个实例各自限 N，实际总配额会放大到实例数倍
- **每进程并发闸**：单个实例同时执行的图任务数上限要设（如 `MAX_CONCURRENT_TURNS`），超了返回 429 快速失败，比排队到 OOM 好
- **`thread_id` 必须服务端签发并鉴权**。如果直接用 URL 里的 `thread_id` 去 resume，用户改个 id 就能续跑别人挂起的图、拿到别人的 payload。做法：`thread_id` 里带用户维度，或从签名 token 派生，并在入口校验归属

### 5.11 可观测性：让「多次请求」串成一个 thread

一个 thread 的生命周期横跨多个 HTTP 请求、多个实例。没有统一标识，出问题就是睁眼瞎：

```
请求1  POST /run        instance=B  traceId=t1  thread=T-992
请求2  POST /resume     instance=A  traceId=t2  thread=T-992   ← 必须能关联到同一个 thread
请求3  SSE  /events     instance=C  traceId=t3  thread=T-992
```

建议：日志、指标、Trace 全部带上 `thread_id` + `instance_id` + `checkpoint_id` 三个维度。特别关注这几个观测点：

- 每个 superstep 的耗时（区分 LLM 等待 / 工具等待 / 检查点写入耗时）
- `put` / `putWrites` 的延迟与失败率——**这是最难察觉的性能瓶颈**，检查点慢会让每个超步都变慢
- 挂起 thread 的数量与年龄分布（及时发现「悬挂线程泄漏」）
- 跨实例 resume 的比例（衡量事件总线的负载）

## 六、落地清单

把上面这些内容压缩成一张自查表：

| # | 检查项 | 单实例 Demo | 多实例生产 |
|---|--------|:-----------:|:----------:|
| 1 | 检查点存储 | MemorySaver | Postgres / Redis（外置、共享） |
| 2 | 挂起-恢复 | Promise 等待 | 新请求携带 `Command({resume})`，或 Redis `SET PX` + `GETDEL` 槽位 |
| 3 | Redis 不可用降级 | - | 进程内存降级 + 容量 ledger + 快速失败 |
| 4 | 悬挂线程回收 | 不需要 | `pending_threads` 注册表 + `DELETE...RETURNING` 清扫 + `deleteThread` |
| 5 | 同 thread 并发 | 顺其自然 | 路由串行 / Redis 锁；业务状态一律 CAS |
| 6 | 流式输出 | 直接写 | 本地直写 + Redis Pub/Sub 总线（带 origin/seq 去重） |
| 7 | 会话缓存 | 随便 | 短 TTL 回源 + 关键动作 CAS |
| 8 | 在线状态 | 不关心 | Redis 心跳 TTL |
| 9 | 限流/并发 | 不需要 | Redis 固定窗口 + 每进程并发闸 429 |
| 10 | 图版本升级 | 重启即可 | 灰度双跑 + thread 版本前缀 + 老实例排空后下线 |
| 11 | thread_id 鉴权 | 无所谓 | 服务端签发 + 归属校验 |
| 12 | 可观测性 | console | `thread_id` + `instance_id` 贯穿日志/Trace；监控 `put` 延迟 |
| 13 | 节点幂等 | 建议 | **必须**（resume 会重跑整个 task） |

最小可行的多实例改造顺序建议：

```
① 换掉 MemorySaver → 共享 Postgres（这一步做完，跨实例续跑就成立了）
② 去掉进程内的等待 Promise → 回答由 resume 请求直接携带
③ 加 pending_threads 注册表 + 清扫器（防泄漏）
④ 加 SSE 事件总线（解决流式输出跨实例）
⑤ 同 thread 串行（进程队列 → Redis 锁）
⑥ 限流与并发闸、鉴权、可观测三项补齐
⑦ 最后才处理版本灰度策略
```

## 七、小结

回到开头那句「中断只是一个 `await`」：

- **checkpoint** 把图的每一步状态落成一条可回溯的时间线，让「进程死了」不再等于「任务死了」。它的核心是 Pregel 的超步模型、channel 版本号、以及 `parentConfig` 串起来的链表
- **interrupt** 不是阻塞，而是「保存现场并退出」；**resume** 不是接着跑，而是「按同一现场把 task 再跑一遍，这次给 interrupt 一个返回值」。**重放语义**是所有踩坑的根源：被中断节点之前的每一行代码都会再执行一次
- **多实例**的真正难点从来不是「checkpoint 放共享存储」（LangGraph 已经解决了），而是那些**仍旧留在进程内存里的东西**：等待句柄、SSE 连接、并发互斥、会话缓存、限流计数、挂起登记表。把它们逐个外移，服务才真正无状态

一个判断改造是否到位的简单标准：**把任意一个实例随时杀掉，用户的会话能不能在另一个实例上继续推进、且不丢任何一个挂起请求。** 能做到，就达标了。
