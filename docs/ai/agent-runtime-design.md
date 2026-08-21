# Agent Runtime 设计深度解析：从对话循环到可观测的执行内核

> LLM 是「会推理的脑」，工具是「会做事的手」，而 Runtime 才是把脑和手协调起来、让 Agent 真正自主运转的「神经系统」。本文以 [aipack](https://github.com/luoguoxiong/aipack) 的 `packages/agent/runtime` 实现为蓝本，系统拆解一个生产级 Agent Runtime 的关键设计：多会话管理、对话循环、工具执行、上下文压缩与可观测性。

## 一、为什么需要 Runtime

如果你只是「调一次 LLM 拿一段文本」，根本不需要 Runtime——一个 `chat.completions.create` 就够了。但只要任务进入 Agent 形态，就会立刻遇到这些问题：

- **多轮对话**：模型说「我去查一下天气」，你就要执行工具、把结果塞回去、再让模型继续推理——这是一个 `while` 循环
- **多会话并发**：一个进程同时服务几十个用户，每个用户的消息历史相互隔离，不能串台
- **上下文溢出**：聊到第 30 轴，token 量超过模型窗口，模型直接报错或静默截断
- **工具不可信**：模型可能调用危险工具（删文件、转账），需要权限裁决与「人在环」审批
- **失败要可观测**：哪一步慢、哪个工具报错、哪次重试，必须能追查
- **会话要持久化**：进程重启后用户还能继续上一次对话

这些问题没有任何一个能靠 LLM 本身解决，它们全部压在 Runtime 肩上。换句话说，**Runtime 是把「会推理的模型」变成「可靠运行的服务」的那一层**。

aipack 的 Runtime 大约 2000 行 TypeScript，没有引入任何额外依赖，核心职责可以归纳为四件事：

```
┌─────────────────────────────────────────────────────────────┐
│                        AgentRuntime                          │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ 会话管理     │  │ 对话循环     │  │ 工具执行         │   │
│  │ LRU + 串行化 │  │ ReAct loop   │  │ 并行/权限/审批   │   │
│  │ + 存储锁     │  │ + 流式       │  │ + before/after  │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
│         │                  │                    │            │
│         └──────────┬───────┴────────────────────┘            │
│                    ▼                                         │
│         ┌──────────────────────────────┐                     │
│         │ 上下文管理 + 可观测性         │                     │
│         │ 转换器链 / 溢出恢复 / 阈值压缩 │                     │
│         │ traceId / spanId / 遥测钩子   │                     │
│         └──────────────────────────────┘                     │
└─────────────────────────────────────────────────────────────┘
            │                              │
            ▼                              ▼
      SessionStorage                  Model (LLM)
      (持久化/锁)                     (streamFn)
```

下面按这四块逐层拆解。

## 二、多会话管理：LRU + 串行化 + 存储锁

### 2.1 为什么不能「一个全局 messages 数组」

最朴素的实现是一个 Runtime 持有一个 `messages: Message[]`，所有请求都往里 push。这在新手 Demo 里没问题，一到生产环境立刻崩盘：

- 用户 A 的请求还在执行，用户 B 的请求就 push 进来，两条对话历史交错
- 用户 A 中途 abort 了，abortController 被 B 的请求覆盖，A 根本停不下来
- 进程重启，内存里的 messages 全丢

所以 Runtime 必须**按 sessionKey 维护多个独立会话状态**。aipack 用一个 `Map<string, SessionState>` 承载：

```ts
interface SessionState {
  messages: Message[];
  isStreaming: boolean;
  abortController: AbortController | null;
  createdAt: string;
  hydrated: boolean;        // 是否已从存储恢复（防重复加载竞态）
  queue: Promise<void>;     // 串行队列：同会话请求依次执行
  idleResolvers: Array<() => void>;  // 等待空闲的 resolver
  lockHeld: boolean;        // 是否持有执行锁（LRU 淘汰保护）
}
```

模型、工具、扩展、转换器这些「资源」是跨会话共享的，只有 messages / abortController / queue 是按会话隔离的。这是 Agent 服务最常见的一种「无状态服务 + 有状态会话」的折中：服务实例本身可以随便扩缩容，会话状态落在 `SessionStorage` 里，内存里只放活跃会话。

### 2.2 LRU 淘汰：保护运行中的会话

会话表不能无限增长，aipack 设了 `DEFAULT_MAX_SESSIONS = 256` 的上限。超限时淘汰「最久未用」的非活动会话：

```ts
private evictIdleSessions(): void {
  if (this._sessions.size <= this._maxSessions) return;
  for (const [key, session] of this._sessions) {
    if (this._sessions.size <= this._maxSessions) break;
    // 运行中 / 已持锁（已入队待执行）不淘汰
    if (session.isStreaming || session.lockHeld) continue;
    this._sessions.delete(key);
  }
}
```

这里有两个关键细节值得注意：

1. **只清内存态，不删存储**。被淘汰的会话下次被请求时会从 `SessionStorage` 重新 `hydrate`，等价于一次缓存未命中
2. **运行中和已入队的会话不淘汰**。`lockHeld` 标记位保护那些「刚入队还没开始运行」的会话——如果按 LRU 朴素淘汰，一个请求刚排队就被踢出，下一次进来又重新创建，会话历史全丢

### 2.3 串行化：同一会话的请求必须排队

这是 Runtime 最容易被忽视、也最容易踩坑的一点。考虑下面这个时序：

```
T0: 用户连发两个请求 A、B 到同一会话
T1: A 开始 hydrate（异步 await storage.load）
T2: B 也开始 hydrate（同样异步）
T3: A 的 hydrate 完成，开始运行对话循环
T4: B 的 hydrate 完成，也开始运行对话循环
T5: 两个循环并发 push assistant 消息，互相覆盖
```

为了杜绝这种交错，aipack 给每个会话维护一个 `queue: Promise<void>`，每次请求先 `await prev` 再执行，本质是一个**基于 Promise 链的轻量级互斥锁**：

```ts
private async acquire(session: SessionState): Promise<() => void> {
  let release!: () => void;
  const prev = session.queue;
  session.queue = new Promise<void>(resolve => {
    release = () => resolve();
  });
  await prev;
  session.lockHeld = true;
  return () => {
    session.lockHeld = false;
    release();
  };
}
```

这种写法的好处是**零依赖、无 native 锁、天然异步友好**。坏处是它只能保证「同一会话内串行」，跨会话的并发仍是真实的并行——但这恰好是我们想要的。

### 2.4 存储级锁：防多进程 last-write-wins

进程内的串行化解决了单实例并发，但生产环境往往是多实例部署。两个进程同时为同一会话服务时，又会回到「读-改-写」竞态：A 进程读了历史、B 进程也读了历史，A 写回、B 写回，B 覆盖 A 的更新，消息丢失。

aipack 把这层交给 `SessionStorage` 的 `withLock` / `acquireLock` 实现（可以是 Redis 分布式锁、文件锁等），Runtime 层只做适配：

```ts
private async runWithStorageLock<T>(
  request: Request,
  sessionKey: string,
  fn: () => Promise<T>,
): Promise<T> {
  const storage = this._sessionStorage;
  if (request.ephemeral || !storage?.withLock) return fn();
  return storage.withLock(sessionKey, fn);
}
```

注意 `ephemeral` 请求跳过锁——「一次性、不持久化」的请求（如健康检查、临时问答）不需要承担分布式锁的开销，这是性能与正确性的权衡。

| 层级 | 作用域 | 解决的问题 | 实现 |
|------|--------|-----------|------|
| 串行队列 | 进程内 / 单会话 | 消息数组交错、abort 覆盖 | Promise 链 |
| 内存锁 | 进程内 / LRU 淘汰 | 保护刚入队会话不被踢出 | `lockHeld` 标记位 |
| 存储锁 | 跨进程 / 单会话 | last-write-wins 丢消息 | `SessionStorage.withLock` |

这三层锁一层套一层，从「单实例并发」到「多实例并发」全覆盖，是 Runtime 设计中最容易被新手漏掉的部分。

## 三、对话循环：ReAct 的工程化实现

### 3.1 主循环骨架

Runtime 的心脏是 `runLoop`，本质就是 ReAct 的工程化实现：

```ts
while (maxTurns-- > 0) {
  turnCount += 1;
  // 1. 链式转换上下文
  await this.transformMessages(compilation, sessionKey);
  // 2. 阈值触发摘要压缩（防溢出）
  await this.maybeCompactByThreshold(compilation, sessionKey, signal);
  // 3. 调用模型（含溢出自动恢复）
  const assistantMessage = await this.streamModel(compilation, signal, sessionKey);
  compilation.messages.push(assistantMessage);
  // 4. 实时持久化
  await this.persistSessionSafe(request, sessionKey);
  // 5. 检查工具调用
  const toolCalls = extractToolCalls(assistantMessage.content);
  if (toolCalls.length === 0) break;  // 无工具调用，结束循环
  // 6. 执行工具
  const outcome = await this.executeToolCalls(compilation, toolCalls, signal);
  await this.persistSessionSafe(request, sessionKey);
  if (outcome.terminate) {  // 钩子请求终止
    compilation.terminateReason = outcome.terminateReason ?? 'terminated';
    break;
  }
}
```

这个骨架看起来平平无奇，但每一行背后都藏着工程取舍。挑几个关键点说。

### 3.2 maxTurns 与 turnCount 解耦

很多 Agent 框架的 `maxTurns` 既是「循环上限」又是「对外报告的轮数」，这两个语义其实是冲突的：

- 循环上限是**安全阀**——防止模型陷入「调工具-失败-再调工具」的死循环把账单打爆
- 对外轮数是**业务指标**——这次对话实际进行了几轮推理

aipack 把它们分开：`maxTurns` 递减用于控制循环，`turnCount` 递增用于上报。当 `maxTurns < 0`（注意是小于 0，不是等于 0）时，说明是循环条件失效而非 break 退出，标记 `maxTurnsExhausted = true`，最终在 `buildResult` 里把 `stopReason` 设为 `'max_turns'`，让调用方能区分「正常结束」和「被强行截断」：

```ts
if (maxTurns < 0) compilation.maxTurnsExhausted = true;
```

```ts
// buildResult
if (compilation.maxTurnsExhausted) {
  builder.stopReason('max_turns').metadata('maxTurns', true);
}
```

这个细节之所以重要，是因为 UI 层要根据 `stopReason` 决定是否显示「对话被截断，继续吗」的提示——如果用 `turns === maxTurns` 判断，模型恰好用完预算正常结束的对话也会被误判。

### 3.3 流式 vs 同步：两条独立循环

aipack 维护了 `runLoop` 和 `runLoopStream` 两套几乎一致的循环。为什么不复用？因为 Generator 的控制流和 async 函数有本质区别：

- 同步路径：`await streamModel()` 直接拿到 `AssistantMessage`
- 流式路径：必须 `for await (const chunk of turn)` 逐个 yield 出去，最后再拿到 `AssistantMessage`

```ts
const turn = this.modelTurnWithRecovery(compilation, signal, sessionKey, true);
let turnResult = await turn.next();
while (!turnResult.done) {
  yield turnResult.value;        // 把 text_delta / tool_start 透传给前端
  turnResult = await turn.next();
}
const assistantMessage = turnResult.value;
```

两条循环的差异仅在「如何拿到 assistantMessage」和「如何把事件送给消费者」，其余的转换、压缩、工具执行、持久化逻辑完全一致。这是把「流式」当一等公民的设计——而不是把流式当成同步的「后续加工」。

### 3.4 实时持久化：每轮落盘

注意循环里有两处 `persistSessionSafe`：一次在 assistant 回复完成后，一次在工具结果完成后。这意味着即使对话循环跑到第 5 轮时进程崩溃，前 4 轮的完整结果已经在磁盘上了。

```ts
// 3.1 实时持久化：assistant 回复完成即落盘
await this.persistSessionSafe(request, sessionKey);
// ...
// 5.1 实时持久化：工具结果落盘
await this.persistSessionSafe(request, sessionKey);
```

这其实是「快照隔离」的简化版：每次持久化都是当前会话的完整状态，最差情况丢失「正在执行的那一轮」，但已完成的轮次全部安全。代价是 IO 次数翻倍——如果会话特别长、存储特别慢，可以考虑改成「每 N 轮一存」或「仅 assistant 完成时存」，aipack 选择了最保守的策略。

## 四、工具执行：并行、权限、审批、钩子

工具执行是 Agent 区别于 Chatbot 的根本。aipack 把工具执行抽象成一个明确的管线：

```
prepareArguments → PermissionPolicy → beforeToolCall → execute → onToolCall → afterToolCall
       │                  │                │            │           │             │
       │                  │                │            │           │             │
    参数预处理        框架级裁决       扩展级裁决     真正执行    遥测上报     结果改写
```

### 4.1 并行 vs 串行

模型在一轮里可能产出多个 toolCall（比如同时查天气和查日历）。aipack 用 `parallelToolCalls` 开关控制：

```ts
if (this._parallelToolCalls && toolCalls.length > 1) {
  // 并行：全部执行，聚合 terminate
  const outcomes = await Promise.all(toolCalls.map(execute));
  const terminated = outcomes.find(o => o.terminate);
  return {
    results: outcomes.map(o => o.result),
    terminate: !!terminated,
    terminateReason: terminated?.terminateReason,
  };
}
// 串行：遇 terminate 后剩余工具跳过（生成 skipped 保持配对）
```

并行模式快但有一个语义代价：**串行模式下，前一个工具 `terminate` 后，后面的工具不会执行**，而是生成 `[skipped]` 结果填进消息数组。这是为了让 LLM 在下一轮看到「我调了 3 个工具，前一个让我停，后两个被跳过」的完整配对，而不是「我调了 3 个工具，只回来 1 个结果」的错乱状态。

### 4.2 权限裁决：框架级安全底线

工具调用不能裸跑，尤其涉及文件、网络、支付的工具有毁灭性后果。aipack 在执行工具前插入 `PermissionPolicy` 检查：

```ts
const decision = await this._permissionPolicy.check(permissionReq);
let allowed = decision === 'allow';
if (decision === 'confirm') {
  allowed = this._permissionPolicy.confirm
    ? await this._permissionPolicy.confirm(permissionReq)
    : false;
}
if (decision === 'pending') {
  // 挂起等待外部审批（Human-in-the-loop）
  if (this._approvals) {
    const approval = this._approvals.create(permissionReq, {
      timeoutMs: this._approvalTimeoutMs,
      signal,
    });
    await this.emitTelemetry('onApprovalPending', { ... });
    const outcome = await this._approvals.wait(approval);
    allowed = outcome.status === 'approved';
    await this.emitTelemetry('onApprovalResolved', { ... });
  } else {
    allowed = false;  // 未配置审批管理器 → 保守拒绝
  }
}
```

三种决策模式对应三种场景：

- **allow**：直接执行。读操作、纯计算工具默认放行
- **confirm**：同步确认，UI 弹个确认框。适合「删一个文件」这种低风险写操作
- **pending**：异步审批，把请求挂起、推送给审批人，等到批准或超时。适合「转账」「删整个目录」这种高风险操作

注意一个保守默认：**未配置 `ApprovalManager` 时，`pending` 决策一律按拒绝处理**。这是因为「不知道怎么审批」总比「默认放行危险操作」安全。安全系统的默认值永远是「拒绝」。

### 4.3 钩子：beforeToolCall / afterToolCall

权限是「能不能执行」，钩子是「执行前后做点什么」。aipack 在权限裁决通过后插入两个钩子：

- **beforeToolCall**：参数校验后、执行前。可 `block`（阻断）、`terminate`（终止整个 run）、改写 `args`
- **afterToolCall**：执行后、事件发出前。可改写 `result`、`terminate`

```ts
const before: BeforeToolCallDecision = await this._hooks.beforeToolCall.promise(
  { block: false, terminate: false, args },
  beforeCtx,
);
if (before.terminate) {
  return { result: this.makeBlockedResult(reason), terminate: true, terminateReason: reason };
}
if (before.block) {
  return { result: this.makeBlockedResult(reason), terminate: false };
}
args = before.args;  // 允许 beforeToolCall 改写参数
```

权限与钩子的关系是**层级而非替代**：权限是框架级硬规则（任何扩展都不能绕过），钩子是扩展级软规则（业务逻辑）。权限先于钩子执行，权限拒绝则钩子根本看不到这次调用。

### 4.4 工具超时：审批不占预算

一个细节体现了设计者的工程经验：

```ts
// ─── 工具超时信号：权限裁决（含审批挂起）完成后起表，
// 审批等待不占用工具执行超时 ───
const { signal: timedSignal, clear } = withTimeoutSignal(signal, this._toolTimeoutMs);
```

工具超时计时器是在权限裁决（包括 pending 审批的等待）完成之后才启动的。如果用户花 5 分钟审批一个操作，这 5 分钟不该算进「工具执行超时」里——否则审批刚通过、工具刚开始就被超时杀掉，体验极差。

## 五、上下文管理：溢出恢复 + 阈值压缩

上下文管理是 Runtime 最有技术含量的部分，因为 LLM 的 contextWindow 是硬约束——超了就报错或静默截断，没有任何协商余地。

### 5.1 三种溢出模式

aipack 识别三种溢出，处理方式不同：

| 模式 | 表现 | 处理 |
|------|------|------|
| 显式错误 | provider 返回 error 事件，stopReason='error' | 截断历史后同回合重试 |
| 零产出截断 | stopReason='length' 但 output=0 | 截断后重试 |
| 静默溢出 | stopReason='stop' 且有完整产出，但 usage 超窗 | 保留回复，仅压缩后续 |

「静默溢出」是最阴险的——表面看对话正常，但实际 token 已经超窗，下一轮必然失败。aipack 用 `isContextOverflow(final, contextWindow)` 统一识别这三种情况，传入 `model.contextWindow` 做基准判断。

### 5.2 溢出自动恢复闭环

核心是 `modelTurnWithRecovery` 这个生成器：

```ts
private async *modelTurnWithRecovery(
  compilation, signal, sessionKey, stream,
): AsyncGenerator<ResultChunk, AssistantMessage | null> {
  const contextWindow = this._model.contextWindow;
  let recoveries = 0;

  while (true) {
    let assistant: AssistantMessage | null = null;
    let suppressed: AssistantMessage | null = null;  // 被吞掉的可恢复错误

    for await (const event of this.streamModelEvents(compilation, signal, stream)) {
      if (event.type === 'error') {
        if (recoveries < OVERFLOW_RECOVERY_LIMIT && isContextOverflow(msg, contextWindow)) {
          suppressed = msg;  // 吞掉 error chunk，稍后恢复重试
          continue;
        }
        // 不可恢复 → 补发给消费者
        const chunk = this.streamEventToChunk(event);
        if (chunk) yield chunk;
        assistant = msg;
      } else {
        const chunk = this.streamEventToChunk(event);
        if (chunk) yield chunk;
        if (event.type === 'done') assistant = event.message;
      }
    }

    const final = assistant ?? suppressed;
    if (!final) return null;

    if (isContextOverflow(final, contextWindow)) {
      const failed = final.stopReason === 'error' || (final.usage?.output ?? 0) === 0;
      if (failed && recoveries < OVERFLOW_RECOVERY_LIMIT) {
        recoveries += 1;
        if (await this.recoverFromOverflow(compilation, recoveries, sessionKey, signal)) {
          await this.emitTelemetry('onRetry', { ... willRetry: true });
          continue;  // 同回合重试（不消耗回合数）
        }
        // 无可丢弃（单条请求即超窗）：补发被吞的 error chunk
        if (suppressed) {
          yield { type: 'error', content: final.errorMessage || '模型调用出错' };
        }
        return final;
      }
      if (failed) return final;  // 恢复次数耗尽
      // 静默溢出：保留回复，仅压缩旧上下文供后续轮次
      await this.recoverFromOverflow(compilation, 1, sessionKey, signal);
    }
    return final;
  }
}
```

关键设计：

1. **同回合重试，不消耗回合数**。如果计入回合数，恢复机制会和 maxTurns 上限打架——压缩历史后还没等模型说话，回合就用完了
2. **恢复上限 `OVERFLOW_RECOVERY_LIMIT = 2`**。截断后仍溢出最多再试 2 次，防止「截断-溢出-截断」死循环
3. **流式吞 chunk**。可恢复的 error chunk 不送给前端（消费者看到瞬态错误会困惑），只在最终不可恢复时才补发

### 5.3 摘要优先 + 截断兜底

`recoverFromOverflow` 的策略是**摘要优先**：被压缩的旧消息先尝试让 LLM 生成一份摘要，替换成单条 `compactionSummary` 消息；摘要失败或输入本身就超预算（doomed 调用），降级为纯截断。

```ts
const summaryBudget = Math.floor(contextWindow * COMPACTION_SUMMARY_BUDGET_RATIO);
const inputText = compacted.map(m => messageToSummaryLine(m)).filter(l => l.length > 0).join('\n');
// 预算判断：被压缩段超预算时摘要请求自身必然超窗，不发 doomed 请求
if (inputText && estimateTextTokens(inputText) <= summaryBudget) {
  const summary = await this.summarizeMessages(inputText, sessionKey, traceId, signal);
  if (summary && summary.trim()) {
    summaryText = summary.trim();
    mode = 'summary';
  }
}
```

注意 `COMPACTION_SUMMARY_BUDGET_RATIO = 0.6` 这个阈值——如果被压缩段已经占了 contextWindow 的 60% 以上，让 LLM 摘要它**自身就会超窗**，这种 doomed 请求直接跳过，降级为硬截断。这是对「优雅降级」的极致追求：宁可走粗糙但一定生效的截断，也不发起注定失败的摘要调用。

### 5.4 截断点的指数收紧

截断不是简单的「砍掉一半」，而是按恢复次数指数收紧：

```ts
private computeOverflowSplit(messages: Message[], recovery: number): number {
  const target = Math.max(
    Math.floor(contextWindow * this._contextBudgetRatio * 0.5 ** recovery),
    1,
  );
  // ...
  const mustDrop = Math.max(Math.floor(droppable / 2), 1);  // 至少丢一半
}
```

第 1 次恢复砍到 `budget × 0.5`，第 2 次砍到 `budget × 0.25`。这保证每次重试的规模**必然小于**上次溢出的规模——即使 token 估算偏小（实际超窗了但估算没超），也至少砍掉一半可丢弃消息作为兜底。这是把「估算误差」纳入了恢复逻辑的设计。

### 5.5 阈值压缩：溢出前的预防针

光有溢出恢复还不够——模型调用失败后再压缩是「亡羊补牢」，更优的是「未雨绸缪」。`maybeCompactByThreshold` 在每轮模型调用前检查 token 总量，超阈值时主动压缩：

```ts
const triggerTokens = Math.floor(
  contextWindow * (this._compaction?.triggerRatio ?? this._contextBudgetRatio),
);
let total = 0;
for (const m of messages) total += estimateMessageTokens(m);
if (total <= triggerTokens) return;

// 保留最新消息（目标的一半），其余摘要替换
const targetTokens = Math.floor(contextWindow * (this._compaction?.targetRatio ?? 0.5));
const keepTokens = Math.max(Math.floor(targetTokens / 2), 1);
```

注意保留策略是**从尾部累计**——最新的消息一定保留，旧消息优先被压缩。这符合对话的「近因效应」：用户最关心的是刚刚说的几轮，几十轮前的细节可以接受丢失。

### 5.6 工具配对修复：ensureToolPairing

压缩和截断有个隐藏的副作用：可能把「assistant 的 toolCall」和「toolResult」割裂开——比如压缩点恰好落在 toolCall 之后、toolResult 之前。这会让下一轮模型看到「我调了工具但没结果」的错乱状态。

aipack 在每次截断/摘要后都调用 `ensureToolPairing(kept)`，把保留段开头那些「没有对应 toolCall 的孤儿 toolResult」剔除，保证 messages 数组的工具调用配对完整。这是个细节但极其重要——不处理的话，模型会反复「补调」那些其实已经执行过的工具。

## 六、可观测性：traceId、spanId 与遥测钩子

生产级 Runtime 必须可观测。aipack 用一套可选的 `Telemetry` 钩子覆盖全链路：

| 事件 | 触发时机 | 关键字段 |
|------|---------|---------|
| onRunStart | 请求入队前 | traceId, queuedAt |
| onRunEnd | run/stream 结束 | durationMs, activeMs, queuedMs, turnCount, success, errorClass |
| onModelCall | 每次模型调用 | spanId, attempts, inputTokens, outputTokens, ttft |
| onToolCall | 每次工具执行 | spanId, durationMs, status, errorClass |
| onRetry | 重试发生 | attempt, errorClass, delayMs |
| onCompaction | 上下文压缩 | mode, trigger, tokensBefore, tokensAfter |
| onApprovalPending | 审批挂起 | approvalId, expiresAt |
| onApprovalResolved | 审批完成 | outcome, waitedMs |
| onPermissionDenied | 权限拒绝 | toolName, reason |

### 6.1 traceId / spanId：跨进程关联

aipack 自行生成 traceId 和 spanId，不依赖 OpenTelemetry 等 APM 库：

```ts
function newTraceId(): string {
  return `${Date.now().toString(36)}-${randomUUID()}`;
}
function newSpanId(): string {
  return randomUUID();
}
```

traceId 标识一次完整的 run（从入队到结束），spanId 标识一次模型调用或工具执行。一次 run 内可能有多次模型调用和工具执行，它们共享 traceId 但各有独立 spanId，这样在监控面板上既能看到「这次对话总耗时」，也能下钻到「第 3 次模型调用比平时慢 2 倍」。

注释里写了「零新依赖」——这是 aipack 的整体设计哲学，Runtime 故意不引入任何外部包，连 `AbortSignal.any`（Node 18 没有）都用手动桥接代替：

```ts
function withTimeoutSignal(parent: AbortSignal | undefined, ms: number) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(...), ms);
  if (parent) {
    if (parent.aborted) {
      controller.abort(parent.reason);
    } else {
      parent.addEventListener('abort', () => controller.abort(parent.reason), { once: true });
    }
  }
  return { signal: controller.signal, clear: () => clearTimeout(timer) };
}
```

### 6.2 遥测失败不阻断主流程

```ts
private async emitTelemetry<E extends keyof Telemetry>(event, info): Promise<void> {
  const fn = this._telemetry?.[event];
  if (!fn) return;
  try {
    await Promise.resolve((fn as (arg: unknown) => unknown)(info));
  } catch (err) {
    console.warn(`[aipack] telemetry "${String(event)}" 上报失败:`, err);
  }
}
```

遥测本身的失败不能影响对话循环——这是可观测性的铁律。如果上报监控把对话打挂了，那就是「为了治病把人治死」。aipack 用 try/catch 兜底，且 `if (!fn) return` 让未配置的遥测完全无开销。

### 6.3 ttft：首 token 延迟

流式路径有个特别字段 `ttftMs`（Time To First Token）：

```ts
if (stream && event.type === 'text_delta' && ttftAt === undefined) {
  ttftAt = Date.now();
}
// ...
if (stream && ttftAt !== undefined && compilation.ttftMs === undefined) {
  compilation.ttftMs = ttftAt - modelStartedAt;
}
```

这是用户体验的关键指标——用户感知的「响应快不快」主要看第一个字什么时候到，而不是整个回复什么时候结束。aipack 把 ttft 单独记录并上报，监控面板可以直接画 P99 ttft 曲线。

## 七、扩展系统：Hook + Transformer + Extension

Runtime 本身是骨架，业务逻辑通过三种扩展点注入：

### 7.1 Hook：横切关注点

Hook 是事件式的，定义在 Runtime 生命周期的关键节点：

```
beforeInitialize → afterInitialize → beforeRun → [run loop] → beforeEmit → afterEmit → done
                                                          ↓
                                                  failed（异常时）
```

Hook 用 `promise()` 调用，意味着多个扩展的钩子按顺序串行执行，前一个可以修改后一个看到的数据。`beforeRun` 是 waterfall——可以修改 request；`beforeToolCall` 是 decision——可以 block/terminate/改写 args。

### 7.2 Transformer：上下文改写

Transformer 是链式的，每轮模型调用前执行：

```ts
let resources = messagesToResources(compilation.messages);
const context: TransformContext = {
  graph: compilation.graph,
  runtime: { sessionKey, turn, contextWindow, maxTokens, contextBudgetRatio },
};
for (const transformer of this._transformers) {
  try {
    resources = await transformer.transform(resources, context);
  } catch (err) {
    console.warn(`[Runtime] 转换器 "${transformer.name}" 失败，已跳过:`, ...);
  }
}
const messages = resourcesToMessages(resources);
// 原地替换而非重新赋值
compilation.messages.splice(0, compilation.messages.length, ...messages);
```

Transformer 接收 `Resource[]`（比 Message 更结构化的中间表示），返回改写后的 resources。典型用途：

- RAG：注入检索到的知识
- 记忆系统：注入长期记忆摘要
- Prompt 加工：动态拼接系统提示
- 安全过滤：屏蔽敏感信息

注释里特别强调了「原地替换而非重新赋值」：

```ts
// 原地替换而非重新赋值：createCompilation 将 session.messages 按引用共享，
// 重新赋值会导致后续 push 的 assistant/tool 消息脱离会话（持久化丢失）
compilation.messages.splice(0, compilation.messages.length, ...messages);
```

这是个非常容易踩的坑：如果直接 `compilation.messages = messages`，新数组就和 `session.messages` 脱钩了，后续 push 的 assistant 消息只会进 compilation 不会进 session，持久化时丢消息。`splice` 是保持引用的写法。

### 7.3 Extension：插件式打包

Extension 是更上层的打包——一个 Extension 可以同时注册 Hook、注册工具、注册 Transformer。`ExtensionManager.applyAll(ctx)` 在 Runtime 创建时一次性应用所有扩展，传入共享的 `ExtensionContext`：

```ts
const ctx: ExtensionContext = {
  config: runtime._config,
  workspace: options.workspace ?? process.cwd(),
  sessionKey: runtime._sessionKey,
  shared: new Map(),
};
runtime._extensionContext = ctx;
runtime._extensions.applyAll(ctx);
```

`shared: Map` 是一个跨扩展、跨工具调用的共享状态容器，让多个扩展能在同一次 run 内传递数据（比如一个扩展往 shared 里塞用户偏好，另一个扩展的工具调用时读取）。这是避免了全局变量的轻量级依赖注入。

## 八、设计哲学小结

读完整个 Runtime，能看出几个贯穿始终的设计原则：

### 8.1 零依赖

2000 行代码没有一个 `npm install`。AbortSignal.any 用手动桥接，traceId 用 `node:crypto.randomUUID`，锁用 Promise 链。好处是 Runtime 可以被嵌入任何宿主环境（CLI、Edge Function、Electron）而不挑依赖树。

### 8.2 保守默认 + 优雅降级

- 未配置 `PermissionPolicy` → 放行（向后兼容）
- 未配置 `ApprovalManager` + pending 决策 → 拒绝（安全默认）
- 摘要失败 → 截断（功能降级而非报错）
- 遥测失败 → 仅告警（不阻断主流程）
- 溢出恢复失败 → 维持旧行为（错误消息落库）

每一层都有「正常路径」和「降级路径」，正常路径走最优解，降级路径走最稳解。这是生产代码和 Demo 代码最大的区别——Demo 只写正常路径，生产代码 70% 的行数都在处理降级。

### 8.3 引用透明

`getMessages` 返回深拷贝：

```ts
getMessages(sessionKey?: string): Message[] {
  const session = this._sessions.get(sessionKey ?? this._sessionKey);
  if (!session) return [];
  try {
    return structuredClone(messages);
  } catch {
    return JSON.parse(JSON.stringify(messages));
  }
}
```

外部拿到的 messages 数组怎么改都不会污染内部状态。`transformMessages` 用 splice 而非赋值保持引用一致。这些都是「引用透明」的细节——内部状态的修改路径是受控的，不会被外部引用意外污染。

### 8.4 流式是一等公民

很多框架的流式是「先跑完同步、再把结果切成 chunk 假装流式」，aipack 是真正的端到端流式——从 provider 的 `text_delta` 事件到 Runtime 的 `ResultChunk` 到消费者的 yield，整条链路都是 chunk-by-chunk 推送的。`runLoop` 和 `runLoopStream` 是两条平行实现，而不是一条带流式包装。

### 8.5 错误分类驱动运维

每个错误都有 `errorClass` 字段：

- `validation` - 请求校验失败
- `context-overflow` - 上下文溢出
- `tool_error` - 工具执行失败
- `terminated` - 被钩子终止
- `unknown` - 兜底

这让运维可以按错误类型设告警策略——`context-overflow` 频繁触发说明该调大 contextWindow 或开启摘要压缩，`tool_error` 频繁触发说明某个工具实现有 bug，`terminated` 频繁触发说明权限策略太严。错误不只是「失败了」，而是「失败了 + 为什么」，后者才是可运维的。

## 结语

Runtime 是 Agent 系统里最不性感但最吃功夫的一层。它没有模型推理的玄妙，也没有工具调用的酷炫，但所有线上事故、所有用户痛点、所有运维告警，最终都会落到 Runtime 的某一处设计上。

aipack 的 Runtime 实现给了一个相当完整的范本：多会话管理、串行化、存储锁、对话循环、工具执行管线、上下文压缩、可观测性、扩展系统，每一块都经过生产环境打磨。如果你想自己实现一个 Agent 框架，或者评估某个 Agent 框架的生产成熟度，这份设计是个不错的对照参考。

> 「能跑起来的 Demo 和能上线的服务之间，隔着一整个 Runtime。」
