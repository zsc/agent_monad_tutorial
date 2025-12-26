# Chapter 5 — Runtime 与调度：从解释器到并发、取消与资源管理 (chapter5.md)

## 1. 开篇段落

在上一章中，我们定义了 Agent 的灵魂（DSL）：它**想**做什么。本章我们将构建 Agent 的躯体（Runtime）：它**如何**在物理世界中行动。

对于大多数 LLM 应用开发者来说，"Runtime" 往往是隐形的——它就是 Python 解释器或者 Node.js 的 Event Loop。但在构建**高可靠、长时间运行的 Autonomous Agent** 时，这种默认的运行时是远远不够的。你是否遇到过以下情况？

* 用户点击了“停止生成”，但后台的 Tool 还在疯狂调用 API 扣费。
* Agent 在写入文件时报错崩溃留下了一个写了一半的损坏文件（Corrupted File）。
* 同时处理 5 个网页总结任务，结果因为串行执行让用户等了 3 分钟。
* Agent 进入死循环，无法从外部强制杀掉，只能重启服务。

IO Monad 架构的核心优势在于它强制我们显式地设计 Runtime。Runtime 不仅仅是“执行代码的地方”，它是一个**调度器（Scheduler）**、一个**资源管理器（Resource Manager）和一个安全沙箱（Sandbox）**。本章将带你深入 Agent 的“引擎室”，学习如何将纯粹的代数指令转化为安全、高效、可控的物理副作用。

**学习目标：**

* **深度理解解释器模式**：如何通过“堆叠解释器”实现层级化的功能（如在执行前自动加 Log、自动鉴权）。
* **掌握结构化并发 (Structured Concurrency)**：理解 Fiber（纤程）树模型，以及它是如何实现“父任务取消，子任务自动清理”的。
* **精通资源安全 (Resource Safety)**：深入 `bracket` 模式的原理，解决异步环境下的资源泄漏问题。
* **Runtime 隔离与防护**：如何在运行时层面实施速率限制（Rate Limiting）和沙箱隔离。

---

## 2. 文字论述

### 2.1 解释器架构：洋葱模型

在 Chapter 4 中，我们通过 Free Monad 或 Tagless Final 定义了 `Program`。Runtime 的首要职责就是提供一个解释器（Interpreter）来运行它。

但生产级的 Runtime 很少只有一个单一的解释器。相反，我们通常使用**“洋葱模型”**或**“解释器堆叠”**。

```text
User Request -> [ Trace Interpreter ]  <-- 负责生成 Span ID, 记录开始/结束时间
                      |
                [ Auth/Policy Interpreter ] <-- 负责检查工具调用权限, 预算扣除
                      |
                [ Retry/CircuitBreaker Interpreter ] <-- 负责遇到网络错误自动重试
                      |
                [ Real Implementation Interpreter ] <-- 真正发起 HTTP 请求, 读写 DB

```

这种设计的巨大优势在于**关注点分离**。写业务逻辑（Prompt Engineering）的人不需要关心重试策略，也不需要关心分布式追踪（Tracing）；这些都由 Runtime 的外层解释器自动注入。

**代码直觉 (伪代码):**

```haskell
-- 组合解释器
runAgent :: Program a -> IO a
runAgent program = 
    program 
      |> interpretRectry Policy.default  -- 注入重试能力
      |> interpretRateLimit (PerMinute 10) -- 注入限流能力
      |> interpretLog "agent-run-1"      -- 注入日志能力
      |> interpretRealWorld              -- 真正执行

```

### 2.2 结构化并发：驾驭并行的野兽

LLM Agent 是天生的 IO 密集型应用。模型推理慢、搜索慢、数据库慢。为了性能，并发是必须的。

#### 2.2.1 为什么不用 Thread？

操作系统线程（OS Thread）太重了（MB 级栈空间，上下文切换开销大）。IO Monad Runtime 通常构建在 **Fiber（纤程 / Green Thread）** 之上。Fiber 是用户态的轻级线程（KB 级），由 Runtime 自己调度。这允许一个 Agent 进程轻松启动成千上万个并发任务。

#### 2.2.2 结构化并发 (Structured Concurrency)

这是 Runtime 调度的核心原则：**并发任务的生命周期必须受限于其父任务的作用域。**

想象一棵树：

* **Root**: Agent 主流程
* **Child 1**: 思考 (LLM 推理)
* **Child 2**: 并行工具调用
* **Grandchild A**: 搜索 Google
* **Grandchild B**: 爬取网页





**规则**：如果 Child 2 失败或被取消，Grandchild A 和 B **必须**被强制终止。绝不允许出现“孤儿任务”（Orphan Tasks）在后台默默运行。

在 IO Monad 中，我们使用 `parTraverse`（并行遍历）和 `race`（竞态）等原语来隐式地构建这棵树，而不是手动 spawn 线程。

```text
[ parTraverse 语义 ]
Input: [Task A, Task B, Task C]
Behavior:
1. Fork 3 Fibers for A, B, C.
2. Join all of them.
3. If ANY fails -> Cancel others immediately -> Return Error.
4. If ALL succeed -> Return [Result A, Result B, Result C].

```

### 2.3 取消（Cancellation）：不仅仅是停止

“取消”是分布式系统中最难实现的功能之一。在 Agent 语境下，取消通常由两个来源触发：

1. **用户干预**：用户点击 "Stop Generation"。
2. **超时 (Timeout)**：`race(Task, sleep(5s))`，5秒到了，Task 必须死。

#### 取消的传播机制

Runtime 维护着 Fiber 树。当取消信号到达某个 Fiber 时：

1. **状态变更**：该 Fiber 状态标记为 `Canceling`。
2. **向下传播**：Runtime 遍历该 Fiber 的所有活跃子 Fiber，递归发送取消信号。
3. **屏蔽区 (Masking)**：如果某个子 Fiber 正在执行关键的清理操作（如 `closeFile`），取消信号会被**暂时屏蔽**，直到清理完成。这叫 `uncancellable` 区域。
4. **物理中断**：如果是阻塞 IO（如 `socket.read`），Runtime 会尝试关闭文件描述符或注入异常来唤醒线程。

**Gotcha**: 很多简单的 Python 脚本用 `KeyboardInterrupt` 处理取消，但这只能中断主线程。IO Monad Runtime 能确保取消信号准确传递到深层嵌套的每一个异步任务中。

### 2.4 资源安全：Bracket 模式详解

这是本章最重要的工程概念。
在异步并发且可取消的环境中，`try-catch-finally` 是**不安全**的。

**经典漏洞：**

```python
# 伪代码：不安全的资源管理
f = open_file("data.txt")  # 1. 分配
# <--- 如果在这里被取消了怎么办？ f 已经打开，但 try 还没进，finally 永远不会跑
try:
    await do_stuff(f)      # 2. 使用
finally:
    close_file(f)          # 3. 释放

```

IO Monad 引入了 **`bracket`** (也叫 `acquireRelease`) 原语，它将这三步原子化绑定：

Runtime 保证：

1. **Acquire 不可取消**：一旦开始申请资源，就必须等到申请完（得到句柄）。
2. **Release 必执行**：只要 Acquire 成功了，无论 Use 是正常结束、抛出异常、还是被**取消**，Release 都会运行。
3. **Use 可取消**：在 Use 执行期间收到取消信号，Runtime 会打断 Use，跳转到 Release。

**应用场景**：

* **数据库事务**：Acquire=开启事务, Release=回滚/提交, Use=执行SQL。
* **分布式锁**：Acquire=抢锁, Release=释放锁, Use=执行任务。
* **临时文件**：Acquire=创建文件, Release=删除文件, Use=写入数据。

### 2.5 运行时防护：沙箱与限流

Runtime 是 Agent 与世界的边界，它也是实施安全策略的最佳位置。

#### 2.5.1 工具沙箱 (Sandbox)

解释器层可以拦截所有具有副作用的工具调用。

* **代码执行**：拦截 `RunCode` Effect。Runtime 启动一个临时的 Docker 容器或 Firecracker MicroVM，将代码通过 Socket 发送进去执行，只取回 stdout/stderr。宿主机完全隔离。
* **网络访问**：拦截 `HttpRequest` Effect。Runtime 检查 URL 是否在白名单（Allowlist）内，防止 SSRF 攻击（例如 Agent 试图访问 `http://localhost:8080/admin`）。

#### 2.5.2 速率限制 (Rate Limiting)

不要让 Agent 代码自己 `sleep`。Runtime 应该持有**令牌桶 (Token Bucket)** 状态。
当解释器遇到 `LLMRequest` 时，先去令牌桶拿令牌。如果拿不到，Runtime 自动挂起该 Fiber（排队），而不是报错。这样可以平滑地处理 API 限流，极大提升稳定性。

---

## 3. 本章小结

Runtime 是让 DSL 梦想成真的机器。一个优秀的 IO Monad Runtime 提供了传统脚本无法比拟的鲁棒性。

* **关注点分离**：通过解释器堆叠，将日志、鉴权、重试逻辑从业务代码中剥离。
* **性能**：利用 `parTraverse` 和 Fiber 实现高并发 I/O，无需手动管理线程。
* **安全性**：`bracket` 模式消除了“取消”场景下的资源泄漏风险。
* **控制力**：结构化并发确保了没有任何一个任务可以逃脱 Runtime 的生命周期管理（没有僵尸任务）。

---

## 4. 练习题

### 基础题

**练习 5.1：设计解释器层级**
设计一个 Agent Runtime，需要包含下功能：1. 计时（统计耗时），2. 错误重试（最多3次），3. 鉴权（检查 API Key），4. 真实执行。请给出这些解释器的**组合顺序**，并解释为什么顺序很重要。

<details>
<summary>参考答案</summary>
<strong>推荐顺序（从外到内）：</strong>

1. <strong>鉴权 (Auth)</strong>：最外层。如果没权限，直接拒绝，不要浪费后续的重试或计时资源。
2. <strong>重试 (Retry)</strong>：第二层。重试是针对“单次执行失败”的策略。
3. <strong>计时 (Metrics/Timing)</strong>：第三层。通常我们要统计的是“包含重试在内的总耗时”还是“单次 HTTP 请求耗时”？
* 如果是“总耗时”（用户感知的延迟），放 Retry 外面。
* 如果是“单次请求延迟”（监控网络质量），放 Retry 里面。通常建议放 Retry 里面或两处都放。


4. <strong>真实执行 (Real Execution)</strong>：最内层。

<strong>关键点</strong>：如果 Retry 在 Auth 外，鉴权失败会导致疯狂重试，这是错误的。

</details>

**练习 5.2：并发的 List 处理**
Agent 需要分析 10 份 PDF 文档。
方案 A：`pdfs.map(analyze).sequence` (相当于 `Promise.all`)
方案 B：`pdfs.map(analyze).sequence` (但在 Runtime 配置里限制并发度 concurrency=3)
请分析方案 A 在处理 1000 份文档时可能引发的 Runtime 问题。

<details>
<summary>参考答案</summary>
方案 A 会瞬间启动 1000 个 Fiber，发起 1000 个并发 HTTP 请求或打开 1000 个文件句柄。
<strong>后果：</strong>

1. <strong>文件描述符耗尽 (EMFILE)</strong>：操作系统限制单一进程打开的文件数。
2. <strong>内存溢出 (OOM)</strong>：1000 个 PDF 同时加载进内存。
3. <strong>API 限流 (429)</strong>：瞬间击穿 LLM 服务商的 Rate Limit。
<strong>Runtime 解决方案：</strong> 使用 `parTraverseN(limit=3)` 或类似信号量机制，控制同时运行的 Fiber 数量。
</details>

### 挑战题

**练习 5.3：实现“优雅的超时与回滚”**
场景：Agent 正在执行一个复杂的“写代码 -> 跑测试 -> 提交代码”流程。要求整个流程最长只能运行 30 秒。如果 30 秒到了：

1. 立即停止当前的 LLM 生成或测试运行。
2. **但是**，如果代码已经 commit 到了本地 git，必须执行 `git reset --hard HEAD^` 来回滚，保持环境干净。
请用 `race`, `bracket` (或 `onCancel`) 描述这个逻辑。

<details>
<summary>参考答案</summary>
这是一个经典的 `bracket` + `timeout` 组合场景。

<pre>
action = bracket(
acquire = pure(CURRENT_GIT_HASH), -- 记录当前状态
release = \start_hash ->
-- Release 逻辑：检查当前 hash 是否变了，如果变了且是异常/取消状态，回滚
if (current_hash != start_hash) git reset --hard start_hash
else pure (),
use = _ ->
step1_write_code()
step2_run_test()
step3_git_commit()
)

-- 加上超时控制
timeout(30.seconds, action)
</pre>

<strong>解析</strong>：
`timeout` 本质上是一个 `race`。如果超时发生，`use` 块内的逻辑（写代码、跑测试）会被收到取消信号而中断。Runtime 保证跳转到 `release` 块。在 `release` 块中，我们执行清理逻辑（回滚 Git）。注意：如果 `step3` 刚执行完正好超时，Release 依然会运行，这取决于你对“成功但超时”的定义，通常 Release 需要检查 ExitCase (Success/Error/Canceled) 来决定是否回滚。

</details>

**练习 5.4：可中断的 Human-in-the-loop**
设计一个 Runtime 原语 `askHumanOrTimeout(question, timeout)`。
逻辑：发送问题给用户 -> 等待用户回答。
同时：如果用户 1 分钟不理，或者用户点了“取消”，或者 Agent 的整体任务被取消，这个等待必须立即结束。
这就要求 Runtime 能整合：Websocket 事件流（用户输入）、定时器（超时）、父 Fiber 信号（取消）。请描述如何建模。

<details>
<summary>参考答案</summary>
这需要将 `IO` 与 `FRP` (Event Stream) 结合，或者使低级的 `Async/Callback` 桥接。

类型签名：`askHuman : Question -> IO Answer`

内部实现：
使用 `Async.async { callback -> ... }` 创建一个异步边界。

1. 注册 Websocket 监听器：收到消息 -> 调用 `callback(Right(answer))`。
2. 启动定时器：时间到 -> 调用 `callback(Left(Timeout))`。
3. 注册取消回调（onCancel）：如果 Fiber 被取消 -> 移除 Websocket 监听器，停止定时器。

这展示了 Runtime 如何将外部世界（用户行为）桥接到内部的 IO Monad 系统中。

</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 5.1 吞没异常 (Swallowed Exceptions)

**现象**：使用 `fork` 或 `async` 启动后台任务，但没有调用 `join` 或 `await`。任务失败了，日志里什么都没有，Agent 看起来“卡死”了或者行为怪异。
**原因**：在结构化并发中，父任务应该负责处理子任务的错误。如果“发后即忘 (Fire and Forget)”，异常无处抛出。
**调试技巧**：配置 Runtime 的 `UncaughtExceptionHandler`。更好的做法是**禁止** Fire and Forget，总是使用 `Supervisor` 或 `map` 将后台任务的结果/错误汇聚到主流程。

### 5.2 阻塞主线程 (Thread Starvation)

**现象**：Agent 变得反应迟钝，心跳包超时。
**原因**：在异步 Runtime 中直接调用了阻塞操作，如 `requests.get` (同步版) 或 `shutil.rmtree` (大文件夹删除)，甚至是一个巨大的 `for` 循环计算。这会卡住处理 Event Loop 的那个单线程。
**Rule of Thumb**：

* 任何网络/磁盘 IO：必须用异步 API。
* 任何 CPU 密集计算：必须 wrap 在 `shift` / `yield` 中，或扔进专门的 ThreadPool。

### 5.3 错误的 Bracket 嵌套

**现象**：

```haskell
res1 <- openFile "a"
res2 <- openFile "b"
-- do something
close res2
close res1

```

**问题**：如果在 open "b" 的时候报错了，res1 永远不会被释放。
**修正**：必须嵌套 `bracket`：

```haskell
bracket (open "a") close $ \res1 ->
  bracket (open "b") close $ \res2 ->
    -- do something

```

或者使用 `Resource` Monad (Scala/Haskell) 来自动处理这种线性组合。

---

## 6. 语言映射 (Language Mapping)

| 概念 | Haskell (polysemy / mtl) | Scala (Cats Effect / ZIO) | TypeScript (Effect-TS) | Python (Asyncio 裸写) |
| --- | --- | --- | --- | --- |
| **Interpreter** | `Semr` / `Interpreters` | `Interpreter` / `Layer` | `Effect.provide` | 无内置，需手动依赖注入 |
| **Fiber/Task** | `ThreadId` (Green) | `Fiber` | `Fiber` | `asyncio.Task` |
| **Parallel List** | `traverse` (Applicative) | `parTraverse` / `ZIO.foreachPar` | `Effect.all(..., {concurrency: n})` | `asyncio.gather` |
| **Cancellation** | `throwTo` (Async Exception) | `Fiber.cancel` (Interruption) | `Fiber.interrupt` | `task.cancel()` (Cooperative) |
| **Resource Safety** | `bracket` / `Resource` | `Resource` / `ZIO.acquireRelease` | `Effect.acquireRelease` | `try...finally` (弱保证) |
| **Scope** | `runResourceT` | `Scope` | `Scope` | `async with` / `TaskGroup` |
