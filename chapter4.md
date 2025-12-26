# Chapter 4 — Agent DSL：把工具、记忆与控制流做成“可解释的 effect”

## 1. 开篇段落：从“写脚本”到“写剧本”

在传统的 Python 脚本式开发中，我们习惯于直接调用 `openai.ChatCompletion.create` 或 `pinecone.upsert`。这种做法在 Demo 阶段效率极高，但在生产环境和复杂 Agent 系统中，它会导致代码变成一团难以维护的“意大利面条”：

* **不可测试**：单元测试需要 mock 几十个不同的库函数。
* **不可复现**：LLM 的随机性、网络抖动、时间戳的变化，使得 Bug 难以重现。
* **不可观测**：业务逻辑与副作用（IO）紧密耦合，很难看清 Agent 到底“想”做什么，只能看到它“做”了什么。

本章的核心思想：**将 Agent 的业务逻辑（剧本）与执行逻辑（演出）彻底分离**。

我们将定义一套 **Domain Specific Language (DSL)** —— 或者称为“代数（Algebra）” —— 来描述 Agent 的所有能力。Agent 不再直接“打电话给 OpenAI”，而是生成一个“我想打电话”的指令。这个指令由 **解释器（Interpreter）** 接收并执行。这种**依赖倒置**将赋予我们控制时间的上帝视角：我们可以让时间倒流、让 API 必定失败、或者让 Agent 在沙盒中空转。

**本章学习目标**：

1. **能力建模**：如何将 LLM、工具、记忆、时钟、随机数抽象为标准的 Effect。
2. **实现范式**：深入对比 **Final Tagless**（基于接口）与 **Free Monad**（基于数据结构）的工程权衡。
3. **解释器模式**：构建生产（Prod）、测试（Test）、录制回放（VCR）、以及“人类介入（Human-in-the-loop）”解释器。
4. **纯洁性原则**：识别并隔离代码中的式副作用。

---

## 2. 文字论述

### 2.1 架构总览：洋葱模型

在引入 DSL 后，Agent 的架构将呈现出清晰的分层结构。

```ascii
+-------------------------------------------------------+
|  Layer 1: Business Logic (The Script)                 |
|  - "Pure" code                                        |
|  - Describes WHAT to do using DSL interfaces          |
|  - No dependencies on OpenAI/LangChain/AWS SDKs       |
+-------------------------------------------------------+
           |                   |                   |
    (uses Chat Algebra) (uses Mem Algebra)  (uses Tool Algebra)
           |                   |                   |
+-------------------------------------------------------+
|  Layer 2: The DSL Definition (The Vocabulary)         |
|  - Interfaces / Typeclasses / Data Structures         |
|  - Defines types: ChatReq, ChatResp, KVGet, KVPut     |
+-------------------------------------------------------+
           |                   |                   |
    (interpreted by)    (interpreted by)    (interpreted by)
           |                   |                   |
+-------------------------------------------------------+
|  Layer 3: Interpreters (The Actors)                   |
|  [Prod Interpreter]  [Test Interpreter]  [Replay Int] |
|  - Calls OpenAI      - Returns Strings   - Reads File |
|  - Connects Redis    - Uses HashMap      - Deterministic|
+-------------------------------------------------------+

```

### 2.2 核心代数定义 (Defining the Algebras)

我们需要将 Agent 交互世界的触角一一斩断，用抽象接口接管。以下是标准的 Agent 能力集：

#### 2.2.1 认知代数：`LLM` / `Chat`

这是最复杂的 Effect。不要仅仅传递 String，要传递语义结构。

* **操作定义**：
* `chat(messages: List[Message], config: Config) -> m Response`
* `stream(messages: List[Message]) -> m (Stream Response)` (流式)


* **关键抽象**：
* **Model Agnostic**：输入输出结构应屏蔽 `gpt-4` 和 `claude-3` 差异（例如，统一将 `function_call` 和 `tool_use` 映射为标准的 `ToolInvocation` 对象）。
* **Cost Awareness**：返回值应包含 `Usage(prompt_tokens, completion_tokens)`，以便中间件计算成本。



#### 2.2.2 行为代数：`Tools`

Agent 通过工具改变世界。

* **操作定义**：
* `call(tool_name: String, args: Json) -> m Json`
* `list_tools() -> m List[ToolSchema]`


* **为什么需要 list?**：Agent 有时需要根据上下文动态“看到”有哪些工具可用（例如，根据用户权限过滤工具）。

#### 2.2.3 状态代数：`Memory`

分为短期（会话）和长期（知识库）。

* **KV Store (Short-term)**: `get(key)`, `put(key, value, ttl)`
* **Vector Store (Long-term)**: `search(vector, top_k, threshold) -> m List[Doc]`
* **注意**：`Embedding` 计算通常也是一个独立的 Effect（`Embed : String -> m Vector`），因为它涉及 API 调用和费用。

#### 2.2.4 环境代数：`System` (隐形的关键)

这是大多数工程师略的地方。为了实现 100% 的可复现性（Replayability），**所有的非确定性来源必须被托管**。

* **Clock**: `now() -> m Timestamp`。
* *用途*：在测试中，你可以冻结时间，或者让时间快进，测试 `TTL` 过期逻辑。


* **Random**: `uuid() -> m String`, `nextInt(min, max) -> m Int`, `shuffle(list) -> m List`。
* *用途*：Agent 的“探索/利用”策略、ID 生成、重试抖动（Jitter）都需要随机数。如果直接调 `Math.random()`，你就无法回放一次特定的失败。



#### 2.2.5 观测代数：`Telemetry`

日志和追踪不应是代码里的 `print` 语句，而是结构化的 Effect。

* **Trace**: `span(name, attributes) { inner_logic } -> m Result`
* **Metric**: `counter(name, value)`, `gauge(name, value)`

### 2.3 两种流派：Free Monad vs. Final Tagless

如何实现上述 DSL？工程界主要有两种选择。

| 特性 | Free Monad (Reified AST) | Final Tagless (Abstract Interface) |
| --- | --- | --- |
| **核心隐** | **购物清单**。你把要买的东西写在纸上（构建数据结构），然后交给跑腿的人去买（解释器）。 | **依赖注入**。你告诉函数你需要“会买东西的人”（接口约束），编译器注入具体的实现对象。 |
| **代码表现** | 递归的数据类型（Tree）。程序是一个值。 | 带泛型约束的函数：`def run[F[_]: Chat: Tool](...)` |
| **自省能力** | **极强**。你可以在运行前遍历整棵树，统计将会调用几次 API，或者优化执行顺序。 | **弱**。代码编译成了函数调用，运行时很难“预知”后续步骤。 |
| **性能** | 较慢。每一步都需要构建对象和遍历。 | 极快。通常内联为直接的方法调用。 |
| **适用语言** | Haskell, Scala (Cats/Zio), TypeScript (fp-ts) | Scala, Java, C#, TypeScript (Interface) |
| **推荐场景** | 需要对程序进行深度静态分析、优化或序列化传输时。 | **绝大多数生产环境 Agent**。简单、高效、易懂。 |

> **Rule of Thumb**：除非你在写一个需要“序列化 Agent 执行计划并发给另一台机器运行”的分布式系统，否则优先选择 **Final Tagless (Interface pattern)**。它更符合主流工程直觉。

### 2.4 解释器的魔力 (The Power of Interpreters)

一旦我们付出了“定义 DSL”的代价，我们将收获巨大的灵活性。我们可以编写多种解释器来应对不同场景：

1. **Production Interpreter**：
* 连接真实的 `OpenAI` API。
* 连接真实的 `Redis` / `Pinecone`。
* 使用系统时钟。


2. **Mock Interpreter (Unit Test)**：
* `Chat`: 永远返回 "Hello World" 或根据 Prompt 关键词匹配返回。
* `Memory`: 使用内存 `Map` 模拟数据库。
* `Clock`: 时间永远停在 `2024-01-01`。
* **价值**：毫秒级运行整个对话流程测试，零成本。


3. **Record/Replay Interpreter (Integration Test / Debug)**：
* **Record 模式**：作为 Production 的代理。运行时，计算 `Hash(Input)`，将 `(Input, Output)` 序列化存入 JSON 文件（Tape）。
* **Replay 模式**：通过计算 `Hash(Input)` 从 JSON 文件中查找 Response。如果找不到，报错。
* **价值**：捕获一次线上出现的复杂 Bug（例如 10 轮对话后的逻辑错误），将其固化为测试用例，在本地反复调试直到修复。


4. **Human-in-the-Loop Interpreter**：
* 对于 `Tool.call("delete_database")` 这样的高危操作，解释器可以拦截执行，通过 Slack/Email 发送审批请求，等待人类批准后再恢复执行（或抛出拒绝）。
* **注意**：在业务逻辑代码中，这看起来只是普通的 `tool.call`，审批流程完全被解释器封装了。



---

## 3. 本章小结

* **DSL 是契约**：Agent 代码描述意图，Interpreter 负责落实。
* **全量抽象**：不仅仅是 LLM，凡是涉及 IO、状态、时间、随机性的操作，都必须进入 DSL。
* **Final Tagless**：是实现 DSL 的一种轻量级、高性能的工程模式，本质上是“高阶依赖入”。
* **可测性红利**：通过更换解释器，我们可以把“不可测”的 AI 应用变成“完全确定性”的软件系统。
* **回放机制**：Record/Replay 解释器是调试复杂 Agent 行为（幻觉、死循环）的终极武器。

---

## 4. 练习题

### 基础题

**Q1. 定义 Time Algebra**
在许多 Agent 任务中，Agent 需要知道“今天是星期几”来决定是否发送工作邮件。
请定义一个简单的 `Clock` 能力接口（可以用伪代码或 TypeScript 接口），并给出其 Production 实现和 Test 实现。

<details>
<summary>点击展开答案</summary>

**Hints**:
Production 使用 `Date.now()`，Test 使用一个可变的变量。

**Answer (TypeScript 风格)**:

```typescript
// 1. Algebra (Interface)
interface Clock<F> {
  now(): F<Date>;
}

// 2. Production Implementation (IO based)
const prodClock: Clock<IO> = {
  now: () => IO(() => new Date())
};

// 3. Test Implementation (State based)
class MockClock implements Clock<Identity> {
  private _currentTime: Date;
  
  constructor(start: Date) { this._currentTime = start; }
  
  // 允许测试代码推进时间
  advance(ms: number) { 
    this._currentTime = new Date(this._currentTime.getTime() + ms); 
  }
  
  now() { return this._currentTime; }
}

```

</details>

**Q2. 识别隐式依赖**
审查以下代码，找出所有破坏“纯洁性”且需要被抽象进 DSL 的点：

```python
def decide_action(user_input):
    print(f"User said: {user_input}")           # 1
    if "urgent" in user_input:
        return "call_human"
    
    import uuid
    request_id = str(uuid.uuid4())              # 2
    
    try:
        # assume db is a global object
        last_seen = db.get(user_input.user_id)  # 3
    except Exception:
        return "error"
        
    return "process"

```

<details>
<summary>点击展开答案</summary>

**Answer**:

1. `print(...)`: 标准输出副作用。应抽象为 `Logger.info(...)`。
2. `uuid.uuid4()`: 随机性来源。不可复现。应抽象为 `UUIDGen.generate()`。
3. `db.get(...)`: 外部 IO 且依赖全局变量。应抽象为 `KVStore.get(...)`。

</details>

**Q3. 解释器路由**
假设你的 Agent 需要同时使用 GPT-4 处理复杂逻辑，使用 Haiku 处理简单总结。你应该如何在 DSL 中表达这种需求？

<details>
<summary>点击展开答案</summary>

**Answer**:
不要在 DSL 中硬编码模型名。
**方法 A (多代数)**：定义两个能力 `SmartChat` 和 `FastChat`。
**方法 B (参数化)**：在 `chat` 方法中接受一个抽象的 `ModelTier` 枚举（High/Low/Economy），由解释器决定 `High` 映射到 GPT-4，`Low` 映射到 Haiku。推荐方法 B，因为更易于配置切换。

</details>

### 挑战题

**Q4. 设计“录制/回放”的数据结构**
设计一个 JSON 结构（Tape），用于存储一次 Agent 执行的所有 Effect。
考虑：如果 Agent 并发执行了两个 HTTP 请求，如何确保回放时匹配到正确的响应？

<details>
<summary>点击展开答案</summary>

**Hints**:
仅靠顺序匹配在并发场景下是不可靠的。需要指纹（Fingerprint/Hash）。

**Answer Sketch**:

```json
{
  "trace_id": "abc-123",
  "interactions": [
    {
      "effect_type": "Chat",
      "input_hash": "sha256(messages + config)",
      "input_raw": { ... }, 
      "output": { "text": "Hello", "usage": ... },
      "timestamp": 1715000000
    },
    {
      "effect_type": "Tool",
      "tool_name": "search",
      "args_hash": "sha256(query='test')",
      "output": "...",
      "error": null
    }
  ]
}

```

*并发匹配策略*：
在回放时，通常不依赖顺序，而是构建一个 `Map<Hash, Queue<Output>>`。
当 DSL 请求发生时，计算请求的 Hash，从对应的队列中 `pop` 出一个预录制的响应。

</details>

**Q5. 纯函数式“重试”中间件**
在不修改 `Chat` 接口定义，也不修改 `OpenAIInterpreter` 代码的前提下，如何利用 DSL 的组合性，为 `Chat` 能力增加“失败自动重试 3 次”的功能？

<details>
<summary>点击展开答案</summary>

**Answer**:
使用**装饰器模式（Decorator）或高阶解释器**。
编写一个新的解释器 `RetryingChatInterpreter`，它接收一个基础的 `ChatInterpreter`。

```typescript
const retryingChat = (base: Chat, policy: RetryPolicy): Chat => ({
  chat: (req) => {
    // 伪代码：利用 Monad 的递归或 retry 组合子
    return retry(policy, () => base.chat(req));
  }
})

```

这证明了 Effect 抽象的强大之处：横切关注点（如重试、限流、缓存）可以作为独立的层叠加在核心逻辑之上。

</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 5.1 抽象泄漏 (Leaky Abstractions)

* **现象**：你的 DSL 接口中出现了 `import { ChatCompletionChunk } from 'openai'`。
* **问题**：这使得 DSL 绑定到了特定的 SDK。如果 OpenAI 更新了 SDK 版本或者你想换 SDK，接口就得变。
* **对策**：定义自己的 DTO (Data Transfer Objects)。虽然写转换代码很繁琐，但为了架构的整洁是值得的。

### 5.2 只有副作用，没有返回值

* **现象**：`log(msg: String) -> m Unit`。
* **问题**：看起来没问题，但在并发场景下，你可能需要等待日志写入完成再终止程序，或者需要捕获日志系统的磁盘满错误。
* **对策**：即使是 `Unit` 返回值，也要确保它是包裹在 `m` (Effect) 中的，并且要考虑是否需要返回一个 `Handle` 或 `Result`。

### 5.3 混淆“配置”与“上下文”

* **现象**：把 `UserID` 或 `SessionID` 作为 `Chat` 解释器的初始化参数，而不是每次调用的参数。
* **问题**：这意味着你的解释器是有状态的（Stateful），难以在多用户并发环境下复用同一个解释器实例。
* **对策**：解释器应该是无状态的单例。`UserID` 应该通过 `Reader` Monad 传递，或者作为 DSL 方法的一个显式参数（Context Object）。

### 5.4 所有的东西都是 Effect？

* **现象**：把 `JSON.parse` 或 `String.split` 也做成 Effect。
* **问题**：过度工程。
* **Rule of Thumb**：
1. **纯计算**（相同的输入永远得到相同的输出，且无副作用） -> **不需要 Effect**，直接写函数。
2. **不确定性/副作用** -> **必须是 Effect**。

