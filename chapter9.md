# Chapter 9 — 可观测性：Trace/Span、日志、指标与可复现

## 1. 开篇段落：打破“随机性”的诅咒

在传统软件工程中，如果一个函数输入 `x` 输出 `y`，它永远都输出 `y`。但在 LLM Agent 开发中，我们面临着**双重不确定性**：

1. **模型的不确定性**：同样的 Prompt，LLM 可能会给出不同的推理路径。
2. **环境的不确定性**：工具调用的结果（如搜索结果、数据库状态）随时间变化。

当 Agent 在生产环境中“发疯”时（例如陷入死循环、输出乱码、乱调用工具），开发者往往束手无策。传统的 `print` 调试法在异步、并发、多步骤的 Agent 面前毫无招架之力。

**本章的核心观点**：可观测性（Observability）不是系统写完后“外挂”上去的功能，而是**Effect 的一部分**。通过 IO Monad 和 Kleisli Arrow，我们可以：

* 把 **Tracing** 做成一个包裹在所有计算外层的“洋葱皮”。
* 把 **Logging** 做成结构化的事件流。
* 把 **Randomness** 和 **IO** 做成可替换的代数结构，从而实现**100% 的确定性回放（Deterministic Replay）**。

---

## 2. 深度论述

### 2.1 可观测性的三支柱（在 Agent 语境下）

| 支柱 | 传统后端定义 | **LLM Agent 特有定义** | 关键数据示例 |
| --- | --- | --- | --- |
| **Logs** | 离散的系统事件 | **思维快照**：记录 Agent 的每一次“内心独白”和决策瞬间。 | User Input, Prompt, Raw LLM Output, Parsed Thought |
| **Traces** | 请求的调用链路 | **推理因果链**：将 Prompt 组装、LLM 请求、工具执行串联起来，展示耗时与依赖。 | Latency, Parent-Span-ID, Token Usage per Step |
| **Metrics** | 聚合的值指标 | **健康度与成本**：监控 Token 消耗速率、工具错误率、意图识别准确率。 | Cost($), Tool_Error_Rate, Tokens/sec |

### 2.2 Kleisli Arrow 与隐式上下文传播

在构建 Agent Pipeline 时，我们通常将多个步骤组合：
`Plan >=> Execute >=> Summarize`

如果手动传递 `TraceId`，函数签名会变得非常丑陋：
`Plan: (Input, TraceId) -> IO (Plan, TraceId)`

**Kleisli 的魔法**在于，我们可以将 `TraceId` 隐藏在 Monad `m` 中。如果 `m` 是 `ReaderT TraceContext IO`，那么所有的步骤**自动**拥有读取和携带上下文的能力，而无需修改函数签名。

#### 2.2.1 `withSpan` 的高阶抽象

我们定义一个组合子（Combinator） `withSpan`，它接受一个名字和一个 Kleisli Arrow，并返回一个新的 Kleisli Arrow。

```haskell
-- 伪代码类型签名
withSpan :: String -> (a -> m b) -> (a -> m b)

```

**它的内部逻辑是：**

1. 从上下文中获取当前的 `parent_span_id`。
2. 生成一个新的 `span_id`。
3. 记录 `SpanStart` 事件（包含时间戳）。
4. 执行原本的计算 `(a -> m b)`，并将新的 `span_id` 设为下游的 parent。
5. 计算结束后（无论成功失败），记录 `SpanEnd` 事件。

#### ASCII 图解：基于 Kleisli 的 Trace 树

```text
[Root Span: Process User Request] ID: 100
  |
  +--- [Span: Retrieve Memory] ID: 101, Parent: 100
  |      |
  |      +--- (Event: Vector DB Query embedding=[0.1, ...])
  |
  +--- [Span: LLM Reason] ID: 102, Parent: 100
  |      |
  |      +--- (Attribute: model="gpt-4")
  |      +--- (Attribute: temperature=0.7)
  |      +--- (Event: Token Stream Start)
  |      +--- (Event: Token Stream End)
  |
  +--- [Span: Tool Execution "calc"] ID: 103, Parent: 100
         |
         +--- (Attribute: args="1+1")
         +--- (Error: Timeout) <--- 自动捕获异常

```

### 2.3 结构化日志：Canonical Log Lines

不要打印 "Start processing..." 这种废话。在 Agent 系统中，每一行日志都应该是一个**结构化对象（JSON）**。

**关键设计原则**：

1. **关联性**：所有日志必须包含 `trace_id` 和 `span_id`。
2. **分层**：
* **Level 1 (Business)**: `UserQuery`, `FinalAnswer`
* **Level 2 (Reasoning)**: `Thought`, `Plan`, `ToolSelection`
* **Level 3 (Debug)**: `RawPrompt`, `FullLLMResponse`, `HttpPayload`


3. **脱敏**：在写入底层 IO 之前，必须经过一个 Redactor（脱敏器）。

### 2.4 圣杯：可复现性（Reproducibility）与“VCR 模式”

这是本章的重头戏。Agent 难以调试是因为它不仅有逻辑，还有**副作用（Side Effects）**。IO Monad 允许我们将副作用**解耦**为描述（Description）和解释（Interpretation）。

#### 2.4.1 控制熵源 (Entropy)

LLM 的 `temperature` 依赖随机数生成器。如果我们在 Effect 系统中抽象了 `Rand`：

```typescript
interface Rand {
  nextFloat(): IO<number>;
  nextInt(max: number): IO<number>;
}

```

* **生产模式**：使用 `System.Random`。
* **回放模式**：使用 `PseudoRandom(seed)`。只要 Seed 固定，且程序逻辑未变，随机数序列就固定，LLM 在相同参数下的行为（理论上）更可控，或者至少我们可以控制像“重试抖动时间”、“负载均衡路由”这些非 LLM 的随机性。

#### 2.4.2 录制与回放 (Record & Replay)

我们实现两个特殊的 Interpreter：

1. **Recorder (录制器)**：
* 作为“中间人”代理。
* 当 Agent 请求 `Http.get("google.com")` 时，Recorder 执行真实请求。
* 将 `{"req": "google.com", "res": "...", "timestamp": 12345}` 写入磁盘上的 `tape.json` 文件。


2. **Replayer (回放器)**：
* **断网运行**。
* 当 Agent 请求 `Http.get("google.com")` 时，Replayer 拦截请求。
* 它计算请求的**指纹 (Hash)**，在 `tape.json` 中查找匹配项。
* 如果找到，直接返回录制的结果（哪怕那是上周的数据）。
* 如果没找到，报错 `NonDeterminismError`：说明你的代码逻辑变了，发起了新的请求。



**这一机制价值**：

* **CI/CD 集成**：你可以把一次复杂的 Agent 失败案例录制下来，放入单元测试库。以后每次代码提交，都会瞬间跑完这个测试，**无需消耗 Token，也无需联网**。
* **调试**：可以在本地单步调试线上发生的错误，复现那一刻的所有变量状态。

---

## 3. 本章小结

* **Trace 是骨架**：利用 Kleisli Arrow 的组合性，使用 `withSpan` 自动管理 Trace Context，避免手动传参的“代码污染”。
* **结构化是血肉**：日志必须是机器可读的 JSON，包含 `trace_id` 以便在可视化工具（如 Jaeger/Grafana）中串联。
* **IO 是边界**：通过将所有外部交互（网络、时间、随机数）抽象为 Effect，我们获得了“上帝视角”。
* **回放是时间机器**：通过 Interpreter 替换，我们可以将不可预测的 Agent 运行转化为确定性的测试用例。

---

## 4. 练习题

### 基础题 (50%)

**练习 9.1：Trace Context 的设计**
设计一个不可变的数据结构 `TraceContext`。它需要包含哪些字段才能支持 OpenTelemetry 标准？

> **Hint**: 至少需要 Trace ID, Span ID, Trace Flags (sampled?), Baggage (跨服务传递的 KV)。

<details>
<summary>参考答案</summary>

```typescript
// TypeScript 示例
interface TraceContext {
  // 整个调用链的唯一标识 (128-bit hex)
  traceId: string;
  // 当前步骤的唯一标识 (64-bit hex)
  spanId: string;
  // 父步骤的标识 (根节点为 null)
  parentSpanId?: string;
  // 采样标志 (00=不采样, 01=采样)
  traceFlags: number;
  // Baggage: 随上下文传播的键值对，如 userId, environment
  baggage: Record<string, string>;
}

```

</details>

**练习 9.2：日志等级分类**
以下信息分别属于什么日志等级（DEBUG, INFO, WARN, ERROR）？

1. LLM 返回 429 Too Many Requests。
2. Agent 决定使用 Calculator 工具。
3. 原始的 JSON Prompt 字符串（20KB）。
4. 工具调用返回了非法的 UTF-8 字符，Agent 自动重试。

<details>
<summary>参考答案</summary>

1. **WARN** (如果是偶尔发生且能重试) 或 **ERROR** (如果重试耗尽)。
2. **INFO** (这是业务流程的关键节点)。
3. **DEBUG** (数据量大，仅调试用)。
4. **WARN** (系统出现了异常但自动恢复了)。

</details>

**练习 9.3：Span 的颗粒度**
在 `Agent -> LLM -> Agent` 的循环中，Tokenizer 的编码（Encode）和解码（Decode）操作通常非常快（微秒级）。你应该为每一次 Encode/Decode 都创建一个 Span 吗？为什么？

> **Hint**: 考虑 Trace 存储成本和可视化时的信噪比。

<details>
<summary>参考答案</summary>

**不应该**。

1. **性能开销**：创建 Span 本身有开销，如果操作比 Span 创建还快，得不偿失。
2. **信噪比**：在 Trace 视图中，成千上万个微小的 Encode Span 会淹没真正的 IO 调用（如网络请求）。
3. **建议**：可以将整个 "Token Processing" 聚合为一个 Span，或者只在 Metric 中记录耗时。

</details>

### 挑战题 (50%)

**练习 9.4：实现简单的 Replayer 匹配逻辑**
编写一个伪代码函数 `findMatch(tape, request)`。`tape` 是录制的列表，`request` 是当前的请求。
挑战：如果 Agent 并发发起了两个相同的请求（例如都查了 "weather: Beijing"），你的匹配逻辑如何保证顺序正确？

> **Hint**: Tape 可以是有序队列，或者是基于 (Hash + Counter) 的 Map。

<details>
<summary>参考答案</summary>

```typescript
type Interaction = { req: any; res: any; id: number };

class Replayer {
  // 必须用 Iterator 或 Cursor 保持状态，因为顺序很重要
  private tapeIterator: Iterator<Interaction>;

  replay(currentReq: any): any {
    const nextRec = this.tapeIterator.next();
    
    if (nextRec.done) {
      throw new Error("Tape exhausted! Code executed more steps than recorded.");
    }

    const recorded = nextRec.value;

    // 关键：校验请求是否匹配
    if (!deepEqual(recorded.req, currentReq)) {
      throw new Error(`
        Non-deterministic behavior detected!
        Expected: ${JSON.stringify(recorded.req)}
        Actual:   ${JSON.stringify(currentReq)}
      `);
    }

    return recorded.res;
  }
}

```

*对于并发场景，简单的线性 Iterator 不够。通常需要计算 `hash(req)`，然后维护一个 map: `Map<RequestHash, Queue<Response>>`。每次取出队列头部的响应。*

</details>

**练习 9.5：分布式 Tracing 的传播**
假设你的 Agent 需要调用一个外部的 Python 服务（Web Search Service）。你需要通过 HTTP Headers 传递 Trace Context。
请写出在 HTTP Client Effect 中，如何注入 `traceparent` header。

> **Hint**: W3C Trace Context 标准格式。

<details>
<summary>参考答案</summary>

```typescript
// 在 Http Effect 的实现中
function callExternalService(url: string, ctx: TraceContext) {
  // W3C Trace Context 格式: version-traceId-spanId-flags
  const traceParent = `00-${ctx.traceId}-${ctx.spanId}-01`;
  
  return fetch(url, {
    headers: {
      "traceparent": traceParent,
      // 可选: 传递 baggage
      "tracestate": serializeBaggage(ctx.baggage)
    }
  });
}

```

</details>

**练习 9.6：高阶思维——"观测者效应"**
在强类型语言中，为了收集 Metrics（例如统计所有工具调用的成功率），你引入了一个 `MetricWriter` Effect。
但是，如果在高并发下，`MetricWriter` 的锁或 I/O 变慢了，会不会拖慢 Agent 的主流程？
如何利用各种手段（如 IO Monad 的异步特性、采样、批处理）来最小化观测者效应？

<details>
<summary>参考答案</summary>

1. **Fire-and-Forget (异步)**：在 `IO` 链中，使用 `fork` 或 `spawn` 将 Metric 写入操作放入后台线程/纤程，主流程不等待其完成。
2. **Buffer & Batch (缓冲与批处理)**：不要每产生一个指标就发一次网络请求。在内存中积累（Buffer），每 10 秒或满 100 条批量发送。
3. **Sampling (采样)**：对于高频 Trace（如 Token 流），只记录 1% 的请求。
4. **Local Aggregation (本地聚合)**：对于 Counter 类指标，在本地原子变量累加，定时上报快照，而不是每次都发 Event。

</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 5.1 陷阱：Prompt 注入攻击日志系统

**现象**：用户输入包含了恶意的控制字符或伪造的 JSON 格式，导致日志解析器崩溃，或者在日志查看器中伪造了假日志。
**调试技巧**：永远不要相信用户输入。在记录日志前，对所有非结构化文本进行 `JSON.stringify` 转义，或者使用专门的 Log Sanitizer。不要直接把字符串拼接进 JSON 模版。

### 5.2 陷阱：记录了过大的 Context

**现象**：Agent 的 Context Window 很大（如 128k tokens）。开发者在每一步都记录了完整的 `History`。
**后果**：日志体积爆炸，磁盘瞬间写满，网络带宽被占满，Trace 系统因为 Payload 过大而丢弃数据。
**调试技巧**：

* **截断**：对于 Prompt，只记录 500 和后 500 字符。
* **摘要**：计算 Prompt 的 Hash 值记录下来，如果需要查看内容，去专门的 Blob 存储（S3）里找（如果开启了 Full Capture）。
* **引用**：只记录 `message_id`，不记录 `content`。

### 5.3 陷阱：Span 未正确关闭

**现象**：程序抛出异常，导致 `span.end()` 代码没执行。Trace 界面上出现大量“永不结束”的长条，导致统计数据严重失真。
**调试技巧**：这就体现了 IO Monad `bracket` (resource safe acquisition/release) 的价值。
**Rule of Thumb**：永远不要手动调用 `startSpan` 和 `endSpan`。**必须**使用 `withSpan` 这种接收回调函数（Callback/Lambda）的 API，利用语言层面的 `finally` 或 Monad 的 `bracket` 机制保证关闭。

### 5.4 陷阱：混淆了 "Tracing" 和 "Eval"

**现象**：试图在 Trace 系统里做复杂的 Prompt 效果分析。
**区别**：Trace 是**实时**的、面向**过程**的（它挂了吗？慢吗？）。Eval 是**离线**的、面向**质量**的（回答得好吗？）。
**建议**：Trace 系统只负责把原始数据 dump 下来。复杂的质量分析（如“这个 Prompt 修改是否提高了准确率”）应该在专门的数据仓库或 Eval 平台（如 LangSmith, LangFuse）中离线进行。
