# Chapter 11 — 综合案例：实现一个可插拔的 Coding Agent (chapter11.md)

## 1. 开篇段落：从积木到摩天大楼

在前面的十章中，我们拆解了构建 LLM Agent 的所有数学基础与工程原语：IO Monad 负责副作用管理，Kleisli Arrows 负责逻辑组合，State Monad 负责记忆管理，FRP 负责流式交互。现在，是时候将它们组装在一起了。

本章我们将构建 **`PolyGlotCoder`** —— 一个生产级、架构清晰的 Coding Agent。之所以称为“可插拔（Pluggable）”，是因为它遵循**依赖倒置原则**（Dependency Inversion Principle）的函数式版本：

1. **能力与实现分离**：Agent 只知道“我要读文件”，不知道是读本地磁盘还是 GitHub API。
2. **策略与执行分离**：Agent 的 Loop Detection、Retry 策略是作为“中间件”挂载的，而不是硬编码在业务流中。
3. **IO 与 UI 分离**：Agent 核心在 IO Monad 中同步运行，通过 FRP 管道与异步的 UI 世界通信。

**学习目标**：

* **架构设计**：掌握 Final Tagless / Free Monad 风格的 Agent 分层架构。
* **流程编排**：实战使用 Kleisli Arrow 串联 Plan-Act-Observe 循环。
* **安全拦截**：实现基于指纹的 Loop Detection 中间件。
* **实验框架**：如何在不修改核心代码的前提下，通过解释器注入 A/B Test。

---

## 2. 核心架构：洋葱模型

我们将采用**洋葱架构（Onion Architecture）**，由内向外依次是：

1. **Domain Algebra (DSL)**: 定义 Agent 能做什么（Effect 接口）。
2. **Business Logic (Program)**: 使用 DSL 编写的纯逻辑（Kleisli Arrows）。
3. **Middleware (Interceptors)**: 日志、熔断、循环检测。
4. **Interpreters (Runtime)**: 将 DSL 翻译为实际操作（API 调用、Docker 执行）。
5. **External Interface (FRP)**: 处理用户输入流和 UI 输出流。

```text
       +-------------------------------------------------------+
       |                  FRP Interface (UI/Stream)            |
       +-------------------------------------------------------+
              | (Events)                         ^ (Tokens/Logs)
+-------------v----------------------------------|-------------+
|      Runtime / Interpreters (The "Dirty" World)              |
|  [Docker] [OpenAI API] [Vector DB] [Prometheus] [AB Platform]|
+--------------------------------------------------------------+
              | (Inject Implementation)
+-------------v------------------------------------------------+
|      Middleware Layer (Safety & Observability)               |
|  [Loop Detector] [Budget Breaker] [Tracer] [Retry Policy]    |
+--------------------------------------------------------------+
              | (Wrap)
+-------------v------------------------------------------------+
|      Business Logic (Pure Kleisli Arrows)                    |
|  Plan -> Code -> Lint -> Test -> Fix -> Submit               |
+--------------------------------------------------------------+
              | (Depends on)
+-------------v------------------------------------------------+
|      Domain Algebra (DSL Interface)                          |
|  type CodingApp = Chat & FS & Shell & Git & Clock            |
+--------------------------------------------------------------+

```

---

## 3. 第一层：定义能力代数 (Domain Algebra)

我们不直接写代码，而是定义一组 **Effect 接口**。在 TypeScript 中通常表现为 Interface，在 Scala/Haskell 中表现为 Type Class。

### 3.1 核心 Effect 定义

我们需要以下五组核心能力：

1. **`LLM`**: 负责大模型交互。
2. **`WorkSpace`**: 负责文件与 Shell 操作（通常在沙箱中）。
3. **`Git`**: 负责版本控制。
4. **`Observability`**: 负责 Metrics 和 Tracing。
5. **`Experiment`**: 负责获取 A/B 测试配置

```haskell
-- 伪代码形式的代数定义

-- 1. LLM 能力
interface LLM m where
  chat :: [Message] -> ModelConfig -> m Stream<String>
  embed :: String -> m Vector

-- 2. 沙箱环境能力
interface WorkSpace m where
  readFile  :: Path -> m String
  writeFile :: Path -> Content -> m ()
  exec      :: Command -> Timeout -> m (ExitCode, Stdout, Stderr)
  ls        :: Path -> m [Path]

-- 3. 实验能力 (用于 A/B Test)
interface Experiment m where
  getVariant :: ExperimentID -> m Variant -- 返回 'A' 或 'B'
  trackGoal  :: ExperimentID -> MetricValue -> m ()

```

### 3.2 组合 Monad：`App`

我们的 Agent 程序将运行在一个名为 `App` 的 Monad 中。它是所有这些能力的**交集**。

> **Rule of Thumb**: 尽可能保持 Effect 粒度细小。不要定义一个巨大的 `AgentHelper` 接口，而是拆分为 `Reader`, `Writer`, `Shell` 等，这样方便测试时只 Mock 其中一部分。

---

## 4. 第二层：Kleisli 箭头与逻辑编排

这是 Agent 的“大脑”。我们使用 Kleisli Arrow 将各个步骤串联起来。
回想一下，Kleisli Arrow 的形状是 `a -> m b`。

### 4.1 定义 Agent 的状态 (The State)

Agent 在步骤之间流转时，需要携带状态。

```typescript
type AgentState = {
  issue: string;          // 原始需求
  plan: PlanStep[];       // 当前计划
  history: ChatHistory;   // 对话历史
  filesChanged: string[]; // 修改过的文件列表
  attemptCount: number;   // 当前尝试次数
};

```

### 4.2 核心流程箭头

我们将复杂的 Coding 任务拆解为四个原子箭头：

1. **`planner`**: `AgentState -> App AgentState`
* 读取 `issue`，分析代码库，生成 `plan`。


2. **`coder`**: `AgentState -> App AgentState`
* 读取 `plan` 中的当前步骤，调用 `LLM` 生成代码，调用 `WorkSpace` 写入文件。


3. **`verifier`**: `AgentState -> App AgentState`
* 调用 `WorkSpace` 运行 linter 和 test。如果失败，更新 `history` 包含错误信息。


4. **`updater`**: `AgentState -> App AgentState`
* 根据 verify 的结果，决定是标记当前步骤完成，还是增加 `attemptCount` 重试。



### 4.3 组合管道 (The Pipeline)

利用 Kleisli 组合符 `>=>` (andThen)，我们将它们串成一个循环体。

```haskell
-- 单次迭代逻辑
step :: AgentState -> App AgentState
step = planner >=> coder >=> verifier >=> updater

-- 递归执行直到完成或耗尽预算
runLoop :: AgentState -> App AgentState
runLoop state = do
  newState <- step state
  if isComplete(newState) 
     then return newState
     else runLoop newState -- 递归调用

```

这里展示了 IO Monad 的威力：我们像写普通命令式代码一样组合逻辑，但 `step`、`planner` 等函数本身是纯函数，它们只返回“描述计算的值”。

---

## 5. 第三层：中间件 (Middleware) 与安全性

这是区分 Demo 和 Production 的关键层。我们不在 `runLoop` 里写 `if (loop_detected) break`，而是将 `runLoop` 包裹在中间件中。

### 5.1 Loop Detection 中间件

**原理**：
Coding Agent 容易陷入“盲目重试”——修改代码 -> 测试失败 -> 再次修改(相同代码) -> 测试失败。

**实现**：
我们需要一个 `Stateful Middleware`。它维护一个滑动窗口的 **Action-Result Fingerprint**。

```typescript
// 伪代码：中间件高阶函数
function withLoopDetection<T>(
  action: Kleisli<State, T>, 
  windowSize: number = 5
): Kleisli<State, T> {
  
  return async (inputState) => {
    // 1. 获取当前环境的 Trace ID 或 Context
    const history = await getExecutionHistory(); 
    
    // 2. 执行动作
    const result = await action(inputState);
    
    // 3. 计算指纹
    const fingerprint = hash({
      action: inputState.currentAction,
      output: result.testOutput || result.error
    });
    
    // 4. 检测重复
    if (countOccurrences(history, fingerprint) > 3) {
       // 触发熔断
       throw new LoopDetectedError("Detected repetitive futile actions.");
    }
    
    return result;
  };
}

```

### 5.2 A/B Test 注入

A/B Test 可以在两个层面进行：

1. **Prompt 层面**：在 `coder` 箭头内部，调用 `Experiment.getVariant('prompt_strategy')`，根据返回是 'A' (COT) 还是 'B' (Direct) 选择不同的 Prompt 模板。
2. **流程层面**：在 `step` 组合时动态路由。

```haskell
-- 动态管道构建
buildPipeline :: App (AgentState -> App AgentState)
buildPipeline = do
  variant <- getVariant "review_strategy"
  case variant of
    "strict" -> return (planner >=> coder >=> strictVerifier)
    "lax"    -> return (planner >=> coder >=> basicVerifier)

```

---

## 6. 第四层：解释器 (Interpreters)

解释器是 Effect 真正落地的地方。

### 6.1 生产环境解释器 (The Production Runtime)

* **FileSystem**: 映射到 Docker 容器内的 `docker exec fs_container ...`。
* **LLM**: 调用 OpenAI/Anthropic API，并处理 Rate Limit。
* **Observability**: 将日志发送到 Datadog/Jaeger。

### 6.2 录制与回放解释器 (The Replay Runtime)

为了调试极其贵的 Agent 运行，我们需要“时光机”。

* **Record Mode**: 在执行真实 IO 时，将 `(Function, Args) -> Result` 序列化存入 JSON 文件。
* **Replay Mode**: 拦截所有 Effect。当请求 `chat(msg)` 时，不联网，而是查阅 JSON 文件返回历史数据。

> **关键点**：`Clock` 和 `Random` 也必须被 Mock/Replay，否则无法实现确定性复现。

---

## 7. 第五层：FRP 桥接 (Bridging to UI)

Agent 在服务端跑得欢，但用户看浏览器不能一直 loading。我们需要将 IO 执行过程“广播”出去。

### 7.1 事件总线设计

我们可以定义一个 `Event` 代数类型：

```typescript
type AgentEvent 
  = { type: 'Thinking', plan: string }
  | { type: 'ToolCall', tool: string, input: any }
  | { type: 'ToolOutput', output: string }
  | { type: 'TokenStream', chunk: string } // 流式文字
  | { type: 'Error', error: string }

```

### 7.2 IO 到 Stream 的转换

我们在解释器层引入一个 `Subject` (来自 RxJS 或类似库)。

```typescript
class StreamingInterpreter implements LLM, WorkSpace {
  constructor(private eventBus: Subject<AgentEvent>) {}

  async chat(msgs: Message[]) {
    this.eventBus.next({ type: 'Thinking', plan: 'Contacting LLM...' });
    // ... 调用 LLM ...
    // ... 收到 chunk ...
    this.eventBus.next({ type: 'TokenStream', chunk: chunk });
  }

  async exec(cmd: string) {
    this.eventBus.next({ type: 'ToolCall', tool: 'shell', input: cmd });
    const res = await realExec(cmd);
    this.eventBus.next({ type: 'ToolOutput', output: res.stdout });
    return res;
  }
}

```

前端 UI 只需要订阅这个 `eventBus`，即可实时渲染“正在思考...”、“正在修改文件 A...”等状态。

---

## 8. 本章小结

通过本章的 `PolyGlotCoder` 案例，我们验证了：

1. **复杂性管理**：通过 IO Monad 和 DSL，我们将复杂的 Coding Agent 拆解为可管理、可测试的小块。
2. **鲁棒性**：Loop Detection 和 Timeout 不再是业务逻辑的负担，而是过 Middleware 统一治理。
3. **可观测性**：通过解释器模式，我们可以轻松实现 A/B 测试插桩和全量录制回放。
4. **交互性**：利用 FRP，我们打通了同步的 Agent 逻辑与异步的 UI 交互。

**Rule of Thumb**:

> 如果你的 Agent 代码中充满了 `try-catch`、`if (config.isTest)` 或者手写的重试循环，说明你的抽象层级不够。**把控制流变成 Effect，把环境变成 Interpreter。**

---

## 9. 练习题

### 基础题

<details>
<summary><strong>练习 11.1：实现简单的沙箱文件系统解释器 (Mock Interpreter)</strong></summary>

**题目**：
请用内存中的 `Map<Path, Content>` 实现 `WorkSpace` 接口。
要求：

1. `readFile` 如果路径不存在应抛出明确的 Domain Error。
2. `exec` 对于 `ls` 命令应返回 Map 中的 keys。

**Hint**: 这展示了如何在不依赖真实 OS 的情况下测试 Agent 逻辑。

<details>
<summary><strong>参考答案</strong></summary>

```typescript
// 简单的内存文件系统解释器
class InMemoryWorkSpace {
  private files: Map<string, string> = new Map();

  async readFile(path: string): Promise<string> {
    if (!this.files.has(path)) {
      throw new Error(`FileNotFound: ${path}`);
    }
    return this.files.get(path)!;
  }

  async writeFile(path: string, content: string): Promise<void> {
    this.files.set(path, content);
  }

  async exec(cmd: string): Promise<{ stdout: string }> {
    if (cmd.startsWith("ls")) {
      return { stdout: Array.from(this.files.keys()).join("\n") };
    }
    return { stdout: "mock exec result" };
  }
}

```

</details>
</details>

<details>
<summary><strong>练习 11.2：Kleisli 错误处理 (Error Handling)</strong></summary>

**题目**：
假设 `coder` 步骤可能会因为 Token 超限而失败。请修改 Kleisli 管道，使得：
如果 `coder` 失败，自动回退到一个更简单的 `summarizer` 步骤（压缩上下文），然后再重试 `coder`。

**Hint**: 使用 `catchError` 或 `orElse` 组合子。结构类似于 `coder.orElse(summarizer >=> coder)`。

<details>
<summary><strong>参考答案</strong></summary>

```haskell
-- 伪代码逻辑
safeCoder :: AgentState -> App AgentState
safeCoder = catchError coder handler
  where
    handler error = do
      log ("Coder failed, summarizing context: " ++ show error)
      -- 组合：先压缩，再重试编码
      summarizer >=> coder 

```

</details>
</details>

### 挑战题

<details>
<summary><strong>练习 11.3：实现 Time-Travel Debugger 的核心数据结构</strong></summary>

**题目**：
设计一个 JSON Schema，用于存储 Agent 的一次完整运行记录（Tape）。
要求：

1. 必须能支持并发请求的录制（如果 Agent 并行调用了 3 个工具）。
2. 必须包含随机种子的状态。
3. 思考如何处理非确定性的时间戳（Clock Effect）。

**Hint**: 记录不是简单的 List，可能是一个以 `(StepID, EffectType, InputHash)` 为键的 Map。

<details>
<summary><strong>参考答案</strong></summary>

```json
{
  "meta": {
    "agentVersion": "1.0.0",
    "timestamp": "2023-10-27T10:00:00Z",
    "randomSeed": 12345
  },
  "interactions": [
    {
      "stepId": 1,
      "effect": "LLM.chat",
      "inputHash": "sha256-of-messages",
      "output": { "content": "Sure, I can help..." },
      "durationMs": 1500
    },
    {
      "stepId": 2,
      "effect": "WorkSpace.exec",
      "params": { "cmd": "ls -la" },
      "output": { "exitCode": 0, "stdout": "main.py\nREADME.md" },
      "mockedTime": "2023-10-27T10:00:05Z" 
    }
  ]
}

```

**关键设计**：在 Replay 时，解释器不仅要匹配函数名，最好还要匹配 `inputHash`，以防止代码逻辑变更导致请求参数变化，从而错误地匹配了历史记录。

</details>
</details>

<details>
<summary><strong>练习 11.4：Loop Detection 的“语义”判重</strong></summary>

**题目**：
Coding Agent 有时会修改代码，但改动仅仅是加了一个空格或换行。对于编译器来说这是不同的文件，但对于决 Bug 来说这是死循环。
请设计一个 `SemanticLoopDetector`，它不比较文件 Hash，而是比较 AST（抽象语法树）或 Token 序列的相似度。

**Hint**: 你需要在 `WorkSpace` Effect 中增加一个 `diff(fileA, fileB)` 的能力，或者在中间件中解析代码。

<details>
<summary><strong>参考答案</strong></summary>

**实现思路**：

1. **标准化（Normalization）**：
在计算 Hash 之前，对代码内容进行预处理：
* 去除所有注释。
* 去除所有空行和多余空格。
* (进阶) 重命名局部变量为 v1, v2... (Alpha-renaming)。


2. **中间件逻辑**：
```typescript
function normalizeCode(code: string): string {
  // 简单正则去空行去空格
  return code.replace(/\s+/g, ' ').trim();
}

// 在 Loop Detector 中
const currentCodeHash = sha256(normalizeCode(fileContent));
if (historyHashes.includes(currentCodeHash)) {
   throw new Error("Semantic Loop Detected: Code effectively unchanged.");
}

```



</details>
</details>

---

## 10. 常见陷阱与错误 (Gotchas)

### 1. 泄露的抽象 (Leaky Abstractions)

* **现象**：你的 DSL 定义了 `exec(cmd)`，但在业务逻辑里你写了 `exec('ls -la | grep "foo"')`。
* **问题**：这在 Linux 上能跑，在 Windows 开发者的本地环境或者某些精简版 Docker 容器里会报错。
* **对策**：**DSL 应该更具语义化**。定义 `listFiles(pattern)` 而不是依赖 Shell 命令。如果必须用 Shell，请在解释器层做 OS 判断和适配。

### 2. A/B Test 的“幽灵变量”

* **现象**：你在解释器里根据 A/B 标记修改了 LLM 的 Temperature。但是这个 Temperature 的值没有被记录在 Trace 日志里。
* **后果**：当你发现某组实验效果极差时，你无法复现，因为你只记录了 Prompt，没记录解释器偷偷注入的参数。
* **对策**：**Explicit Context**。所有的配置变更（即使是解释器注入的）都应该显式地写入 `Metadata` 或 `Span Attributes` 中。

### 3. 被吞掉的 FRP 错误

* **现象**：Agent 运行出错，UI 上却一直转圈圈。
* **原因**：FRP Stream 的 `catchError` 处理不当，导致流意外终止（Completed），而不是发送 Error 事件给 UI。
* **对策**：在 `flatMap` 内部捕获错误，并将其转换为 `type: 'Error'` 的值事件发送出去，而不是让整个 Stream 崩溃。
