# Chapter 6 — 事件建模：IO timeout、重试、退避、熔断与预算

## 1. 开篇段落

在构建简单的 Demo Agent 时，我们往往假设一切顺利：API 总是毫秒级响应，JSON 格式总是完美无缺，Token 永远够用。但在生产环境中，**LLM Agent 本质上是一个运行在不可靠基础设施上的、概率性的分布式系统**。

LLM 的不确定性（幻觉、格式错误）叠加网络的不确定性（超时、断连），如果不加以控制，会使 Agent 变成一个脆弱的玩具。在面向对象编程中，我们习惯用 `try-catch` 满地打补丁，这导致核心业务逻辑（Agent 的思考过程）被错误处理代码淹没。

本章的目标是利用 **IO Monad** 的组合特性，将这些“故障与约束”建模为**一等公民（First-class values）**。我们将不再“处理”异常，而是通过组合子（Combinators）构建一个**自带弹性（Resilient）**的运行时环境。你将学会如何用纯函数式的方式描述“在预算内、带有指数退避重试、且受熔断器保护的 LLM 调用”。

---

## 2. 文字论述

### 2.1 范式转换：从“异常”到“代数数据类型”

在 IO Monad 的世界里，错误不是代码执行流的“中断”，而是数据流的一个“分支”。

* **传统视角**：调用 `callLLM()`，如果网络断了，抛出 `NetworkException`，栈回溯，程序崩溃或跳转。
* **IO 视角**：`callLLM` 的返回值类型明确告诉了你结果的可能性。
* `IO (Either Error String)`: 明确指出了可能失败。
* `IO (Option String)`: 明确指出了可能无结果（如超时）。



这种显式建模强迫开发者在**编译期**（或编写逻辑时）就处理所有分支，而不是等到运行时 crash。

### 2.2 Timeout：时间的边界与取消语义

超时不仅仅是“时间到了就报错”，它涉及到底层的**资源回收**与**取消语义（Cancellation）**。

#### 2.2.1 竞态模型（Race）

`timeout` 本质上是两个 IO 操作的竞态：

1. **Task A**: 你的业务逻辑（如 LLM 推理）。
2. **Task B**: 一个睡眠  秒的计时器。

```text
       +--- Task A (LLM) ----> Result A
Start -+
       +--- Task B (Timer) --> Result B (Timeout)

```

使用 `race` 组合子，谁先完成，就取谁的结果，并**取消（Cancel）**另一个正在运行的任务。这要求底层的 IO Runtime 支持“中断”操作（例如关闭 HTTP Socket，释放显存）。

#### 2.2.2 绝对截止时间 (Deadline) vs 相对超时 (Timeout)

* **Timeout (`duration`)**: 相对时间。每次重试都会重置计时器。
* *风险*：如果重试 10 次，每次 10 秒，总耗时可能达到 100 秒，拖死上游。


* **Deadline (`instant`)**: 绝对时间点。
* *优势*：更好的组合性。无论内部怎么重试，Deadline 就像一道不可逾越的墙，到了时间点必须全局终止。



### 2.3 Retry 与 Backoff：避免“惊群效应”

当 Agent 遇到故障时，立即重试往往是错误的。

#### 2.3.1 为什么需要 Backoff（退避）？

如果一个后端服务暂时过载，成千上万个 Agent 同时发起重试，会瞬间将刚恢复的服务再次打垮。这被称为**惊群效应（Thundering Herd）**。
解决方案是**指数退避（Exponential Backoff）**：


#### 2.3.2 为什么必须加 Jitter（抖动）？

即使有了指数退避，如果所有 Agent 都在同一时刻（T=0）失败，它们会在 T=2, T=4, T=8 同时醒来重试，形成**波峰共振**。
**Full Jitter** 算法是目前的最佳实践：



在 IO Monad 中，随机数生成也是一种副作用（Effect），需要被封装在 `Rand` 或 `IO` 中。

### 2.4 Circuit Breaker（熔断器）：保护系统

重试解决的是“暂时性故障，但如果 LLM 服务彻底挂了，或者 Agent 的 Prompt 有严重的逻辑漏洞导致 100% 报错，无限重试就是在浪费预算。

熔断器是一个**状态机**，通常通过 `MVar` 或 `Ref`（可变引用）实现：

```text
[Closed] --(Failure > Threshold)--> [Open]
   ^                                   |
   |                               (Sleep Window)
(Success)                              |
   |                                   v
[Half-Open] <--(Allow 1 Request)-------+

```

1. **Closed（闭合）**：正常状态，请求直通。
2. **Open（断开）**：错误率超标，立即短路所有请求（Fail Fast），不消耗网络和 Token。
3. **Half-Open（半开）**：经过一段时间冷却，放行一个请求“探路”。成功则复位，失败则继续断开。

### 2.5 Budgeting（预算）：Token 即金钱

Agent 极易陷入死循环或过度思考。我们需要构建一个 **Budget Algebra（预算代数）**。

#### 2.5.1 多维预算

* **Hard Currency**: 实际 API 开销（美元）。
* **Token Count**: 上下文窗口限制（防止溢出）。
* **Step Count**: 防止死循环（Loop）。

#### 2.5.2 实现方式

我们可以使用 `StateT` Monad Transformer 来隐式传递预算状态：

* `checkBudget :: Cost -> AppM ()`
* 在执行任何 Tool 或 LLM 调用前，先运行 `checkBudget`。
* 如果余额不足，抛出特定错误，中断整个 Kleisli 管道。

### 2.6 The Onion Architecture（洋葱架构）

如何组合上述所有能力？利用高阶函数（Wrapper/Middleware）层层包裹。

```text
Input 
  -> [ Budget Check ]       <-- 没钱直接拒
    -> [ Circuit Breaker ]  <-- 下游挂了直接拒
      -> [ Global Deadline ] <-- 总时间控制
        -> [ Retry Loop ]    <-- 局部容错
          -> [ Rate Limiter ] <-- 避免超频
            -> [ Trace Span ] <-- 记录日志
              -> [ Actual Effect (LLM/Tool) ]

```

**组合的顺序至关重要**。例如，如果 `Timeout` 在 `Retry` 里面，就是“单请求超时重试”；如果 `Timeout` 在 `Retry` 外面，就是“总执行时间超时”。

---

## 3. 本章小结

* **故障即值**：用 `Either Error a` 代替异常，用 `Option a` 代替空指针/超时，让错误处理具备类型安全性。
* **组合子模式**：我们不修改 Agent 的核心逻辑，而是通过 `retry(policy, agent)`, `timeout(duration, agent)` 这样的包装器来增强它。
* **退避三要素**：**Base** (基准时间), **Cap** (最大时间), **Jitter** (随机抖动)。缺一不可。
* **熔断器**：是分布式系统的保险丝，防止错误的级联传播和资源的无效消耗。
* **预算控制**：必须作为一种强制的 Effect 贯穿整个生命周期，不仅仅是为了省钱，也是为了程序的**可终止性（Termination）**。

---

## 4. 练习题

### 基础题（熟悉概念与签名）

**练习 6.1：Retry Policy 的代数定义**
请定义一个数据结构 `RetryPolicy`，它能够描述以下策略：
"初始等待 1秒每次失败等待时间翻倍，最大等待 10秒，最多重试 5次"。
不需要写代码逻辑，只需要定义数据类型（Struct/Interface）。

<details>
<summary>点击查看提示</summary>

思考需要哪些字段来存储这些参数。

</details>
<details>
<summary>参考答案</summary>

```haskell
-- 伪代码定义
data RetryPolicy = RetryPolicy {
    initialDelay :: Duration,  -- 1s
    maxDelay     :: Duration,  -- 10s
    maxRetries   :: Int,       -- 5
    backoffBase  :: Double     -- 2.0 (exponential)
}
-- 注意：Jitter 通常由运行时的解释器决定，或者作为 Policy 的一个 Bool 开关。

```

</details>

**练习 6.2：超时与返回类型**
如果函数 `askLLM` 的类型是 `String -> IO String`。
我们应用一个 5秒 的超时。请问新的函数类型签名应该是什么？
如果在这个基础上，我们再应用一个“捕获错误并返回默认值 'Error'” 的操作，新的类型签名又是什么？

<details>
<summary>点击查看提示</summary>

第一步引入了“可能无结果”的状态；第二步消除了这种可能性。

</details>
<details>
<summary>参考答案</summary>

1. 应用超时后：`String -> IO (Option String)` 或 `String -> IO (Maybe String)`。
2. 应用默认值后：`String -> IO String`。
* 逻辑：`IO (Option String)` -> map (orElse "Error") -> `IO String`。
* 这展示了类型系统如何跟踪错误处理的状态。



</details>

**练习 6.3：计算 Token 预算**
假设 Agent 初始预算为 1000 Tokens。
步骤 1: User Input (50 tokens) -> 剩余 950。
步骤 2: Agent Tool Call (Input 100 + Output 50) -> 剩余 ?
步骤 3: System Warning -> 如果剩余 < 100，停止。
请计算步骤 2 后的剩余预算，并说明为什么 Tool Call 的 Input 和 Output 都要计费。

<details>
<summary>点击查看提示</summary>

LLM 的计费通常是 (Prompt Tokens + Completion Tokens)。Tool Use 的过程，LLM 实际上是生成了 Tool 的调用指令（Output），然后你把结果作为新的 Prompt（Input）喂回去。

</details>
<details>
<summary>参考答案</summary>

步骤 2 后的剩余： Tokens。
*注意*：如果是 Tool Call 场景，通常意味着：LLM 生成调用代码（Output），Runtime 执行代码得到结果，结果被拼接到下一次的 Prompt 中（Input）。所以每一轮对话的累积上下文都在消耗预算。

</details>

### 挑战题（架构设计与深度思考）

**练习 6.4：设计一个“智能”的重试策略**
对于 LLM Agent，普通的指数退避可能不够。如果 LLM 返回 "Context Limit Exceeded"（上下文超长），简单的重试会永远失败。
请设计一个 `SmartRetry` 策略，它不仅看重试次数，还看**错误类型**。
请列出至少三种不同的错误类型，并给出对应的处理策略（Retry, Stop, or Modify）。

<details>
<summary>点击查看提示</summary>

分类错误：暂时性网络错误 vs 确定性逻辑错误 vs 资源配额错误。

</details>
<details>
<summary>参考答案</summary>

1. **Transient Error (500/503/Timeout)**:
* *Action*: 标准指数退避重试 (Exponential Backoff)。


2. **Context Limit Error (400 Bad Request)**:
* *Action*: **Modify & Retry**。执行“摘要（Summarize）”操作压缩历史记录，然后重试。


3. **Authentication Error (401)**:
* *Action*: **Stop**。重试无效，立即报错并通知管理员。


4. **Rate Limit (429)**:
* *Action*: **Wait & Retry**。根据 Header 里的 `Retry-After` 字段精确等待，而不是盲目退避。



</details>

**练习 6.5：实现 Circuit Breaker 的 State Monad**
熔断器需要维护状态（失败计数、上次失败时间、当前状态）。如果我们在一个纯函数式环境（即变量不可变）中，不使用外部数据库，如何在一个长时间运行的 Agent Loop 中实现熔断器？
*提示：思考 `State` Monad 和递归循环的关系。*

<details>
<summary>点击查看提示</summary>

在 FP 中，状态是通过函数参数“传递”下去的。

</details>
<details>
<summary>参考答案</summary>

我们需要将 CircuitBreaker 的状态 `CBState` 作为 Agent 运行时的 `StateT` 的一部分。
每次循环（Step）：

1. 解包当前的 `CBState`。
2. 检查是否 Open。
3. 如果 Open 且未到冷却时间，直接返回错误，状态不变。
4. 如果 Closed，执行 Action。
* 成功：重置 `failureCount = 0`，返回新状态。
* 失败：`failureCount + 1`，如果超阈值则置为 Open，记录 `lastFailureTime`，返回新状态。


5. 将新的 `CBState` 传递给下一次递归调用。

</details>

**练习 6.6：预算耗尽的“优雅降级”**
当 Token Budget 仅剩 5% 时，直接抛出异常终止对话体验很差。
请设计一个 Kleisli Arrow 流程，当 `BudgetCheck` 发现余额低时，不终止，而是**动态切换** Agent 的行为模式。

<details>
<summary>点击查看提示</summary>

这涉及到了控制流的分支。`if low_budget then finalize_strategy else normal_strategy`.

</details>
<details>
<summary>参考答案</summary>

定义两个策略：

* `NormalStrategy`: 允许使用 Tool，允许深思熟虑。
* `WrapUpStrategy`: 禁用 Tool，强制 LLM 生成结语（"由于资源限制，我将总结当前进度..."）。

组合逻辑：

```haskell
runStep = do
  budget <- getBudget
  if budget < threshold
     then WrapUpStrategy
     else NormalStrategy

```

这展示了 **Dynamic Planning**：根据元数据（Budget）动态改变计算图结构。

</details>

**练习 6.7：Jitter 算法对比**
请对比 "Equal Jitter" 和 "Full Jitter" 的区别。

* Equal Jitter: `temp = min(cap, base * 2^n); sleep = temp/2 + random(0, temp/2)`
* Full Jitter: `temp = min(cap, base * 2^n); sleep = random(0, temp)`
在竞争非常激烈的场景下（例如 1000 个 Agent 抢 1 个 API），哪种更好？为什么？

<details>
<summary>点击查看提示</summary>

思考平均等待时间和分布的离散程度。

</details>
<details>
<summary>参考答案</summary>

**Full Jitter 更好**。

* Equal Jitter 保证了至少等待 `temp/2` 的时间，这虽然避免了立即重试，但让所有请求都挤在 `[temp/2, temp]` 这个较窄的区间内。
* Full Jitter 的范围是 `[0, temp]`，分布更加均匀（Uniform Distribution），能最大限度地利用时间窗口分散请求压力。虽然有极小概率随机到 0，但整体吞吐量通常更高。

</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 5.1 幽灵请求 (Zombie Requests)

* **现象**：Agent 已经因为超时向用户报错了，但后台服务器还在傻傻地跑那个耗时 2 分钟的推理任务，消耗昂贵的 GPU。
* **原因**：实现了 `timeout` 逻辑，但没有实现**Cancellation（取消）**逻辑。只是客户端不再等待结果，但服务端没有收到“停止”信号。
* **修复**：确保你的 IO Runtime 支持 `bracket` 或 `resource` 模式，并且底层 HTTP 客户端能传播 `Context.Cancel` 信号断开 TCP 连接。

### 5.2 错误的 Jitter 实现

* **错误代码**`sleep(base * 2^n + random(0, 100ms))`
* **问题**：这里的随机量（100ms）相对于指数增长的基数（例如 10秒、20秒）太小了，几乎起不到分散流量的作用。
* **修复**：随机性必须与当前的退避时间**成比例**（Proportional），而不是一个固定常数。

### 5.3 预算泄露 (Budget Leak)

* **现象**：设置了 `max_tokens=4000`，结果跑出了 8000 tokens 的账单。
* **原因**：
1. 只计算了 Response，忘了计算 Prompt（Prompt 往往比 Output 长得多）。
2. Agent 进行了多次“内部思考”或“工具尝试”，这些中间步骤虽然没有展示给用户，但都实打实消耗了 Token。


* **Rule of Thumb**：Budget Wrapper 必须包裹在**最底层**的 LLM 请求函数上，而不是包裹在 Agent 的顶层逻辑上，确保统计无死角。

### 5.4 熔断器“永不开闸”

* **现象**：服务恢复了，但 Agent 还是拒绝所有请求。
* **原因**：熔断器进入 `Half-Open` 状态后，放行的那个“探针请求”如果因为偶发原因（如超时）失败了，熔断器会再次 `Open` 并重置冷却时间。如果网络不稳定，可能导致系统长期卡在“断开”状态。
* **修复**：在 Half-Open 状态下，可以考虑允许稍微多一点的探针（如 2-3 个），或者对探针请求使用更宽容的超时策略。

### 5.5 重试了不该重试的错误

* **现象**：Prompt 里面写错了变量名，导致 LLM API 返回 `400 Bad Request`。Agent 坚持重试了 5 次。
* **后果**：浪费时间，给开发者造成困扰。
* **修复**：建立严格的 **Error Predicate（错误断言）**。只有被标记为 `Retryable` 的错误（网络、5xx）才走重试流程。`4xx` 错误通常意味着代码 bug，应该立即 Fail。
