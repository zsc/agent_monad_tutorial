# Chapter 7 — Coding Agent 的 Loop Detection：让 Agent 可终止、可解释

## 7.1 开篇：西西弗斯的编程助手

在构建 Coding Agent 时，最令人沮丧的时刻莫过于此：你满怀期待地看着 Agent 开始修复 Bug，它修改了代码，运行测试，失败了；它再次修改代码（甚至和上次一模一样），再次运行测试，错误依旧……十分钟后，你发现它在反复修改同一个文件的同一行，消耗了 $10 的 API 额度，而代码库的状态没有任何实质进展。

这种“死循环”在图灵完备的编程任务中是不可避免的（停机问题）。但在工程实践中，我们不能放任不管。

传统的做法是在代码里到处插 `if (count > 5) break`。但在本书推崇的 **IO Monad / Kleisli** 架构中，我们将采取一种更优雅、更系统的方法：**将 Loop Detection 视为一个 Effectful 的中间件**。

**本章学习目标**：

1. 理解 Coding Agent 陷入循环的三种典型模式（句法、语义、状态）。
2. 掌握如何用 Kleisli Arrow 将检测逻辑“切面化”，插入到 Agent 的思考-行动循环中。
3. 学会构建多维度的特征检测器（N-gram, AST Diff, Error Fingerprint）。
4. 设计分级的“熔断与逃生”策略，实现优雅降级。

---

## 7.2 核心概念：作为 Effect 的“守门人”

在 IO Monad 的世界里，Agent 的每一步都可以抽象为一个 Kleisli Arrow：

Loop Detector 不是业务逻辑的一部分，它是一个**守门人（Gatekeeper）**。它拦截即将发生的 Action，查阅历史 Trace，然后决定是“放行”、“修改”还是“终止”。

### 7.2.1 架构图解

我们不修改 Agent 的核心 `Planner`，而是通过组合子（Combinator）将检测能力挂载”上去。

```text
               +----------------------+
               |      Trace Store     |
               | (History of Actions) |
               +----------+-----------+
                          | (Read)
                          v
[Planner] --(Action)--> <Loop Detector> --(Decision)--> [Executor]
                          ^      |
                          |      +--(Abort/Retry)--> [ErrorHandler]
                          |
                   (Update Trace)

```

### 7.2.2 类型建模

我们需要定义什么是“循环检测”的输入和输出。

**Trace（痕迹）**：不仅是文本日志，更是结构化的事件流。

**Decision（决策）**：这不仅仅是 Boolean，而是一个包含策略的 Sum Type。

**Detector（检测器）**：

*(注：这里使用 State Monad 来隐式传递 Trace，或者从外部数据库读取)*

---

## 7.3 循环的分类学与特征工程

Coding Agent 的循环比简单的“复读机”要复杂得多。我们需要针对不同类的循环设计不同的特征提取器。

### 7.3.1 句法重复 (Syntactic Repetition)

这是最低级的循环，Agent 像是卡住了。

* **表现**：连续发出 `cat file.py`, `cat file.py`。
* **检测算法**：**N-gram 匹配**。
* 将动作序列视为符号流。
* 检测是否存在后缀与之前的子序列完全匹配。


* **Rule-of-Thumb**：对于只读操作（Read-only），允许较高的重复容忍度（例如 3 次）；对于写操作（Write），容忍度极低（通常 1 次即由警告）。

### 7.3.2 语义振荡 (Semantic Oscillation / Ping-Pong)

这是最隐蔽的杀手。Agent 在两个状态间反复横跳。

* **表现**：
1. T=1: 修改 A 文件，引入了 Bug X。
2. T=2: 发现 Bug X，回滚 A 文件。
3. T=3: 觉得 A 文件还是得改，再次修改（回到 T=1 的状态）。


* **检测算法**：**状态哈希（State Hashing）**。
* 每次修改文件系统后，计算受影响文件的 Hash。
* 维护一个 `Set<FileHash>`。如果当前 Hash 曾经出现过，且 `CurrentStep - LastSeenStep > 1`（排除立即撤销），则判定为振荡。



### 7.3.3 顽固性错误 (The Stubborn Failure)

Agent 面对相同的报错，不做任何策略调整，反复尝试。

* **表现**：
* Run Test -> Fail (Error: variable 'x' not found)
* Edit -> Run Test -> Fail (Error: variable 'x' not found)


* **检测算法**：**错误指纹 (Error Fingerprinting)**。
* 原始的报错信息包含时间戳、随机地址，每次都不一样。
* **归一化 (Normalization)**：去除数字、GUID、时间戳、文件路径前缀。
* 比较归一化后的错误签名。如果连续  次 Action 导致的观察结果（Observation）具有相同的错误签名，触发熔断。



---

## 7.4 处置策略：从“劝阻”到“强制接管”

当 `detect` 返回非 `Pass` 结果时，我们通过 IO Monad 执行干预。这体现了 Effect 系统的优势：我们可以将“错误处理”变成“控制流”。

### 策略 1：Prompt 注入 (Soft Intervention)

最轻量的干预。不中断执行，只是修改上下文。

* **动作**：在 Message History 末尾追加一条 System Message。
* **内容**：*"Warning: You have tried this action before. The result was X. Please try a different approach."*
* **IO 实现**：`State.modify (appendSystemMsg warning)`。

### 策略 2：动态降温/升温 (Dynamic Temperature)

* **原理**：循环往往意味着模型陷入了概率分布的局部极值。我们需要引入熵（Entropy）。
* **动作**：
* **Cooldown**: 暂停一会（解决 API Rate Limit 导致的伪失败）。
* **Heatup**: 临时提高 `temperature` (e.g., 0.1 -> 0.7)，强迫模型输出不一样的 Token。


* **IO 实现**：利用 `Reader` Monad 的 `local` 机制，只改变当前 Step 的配置。

### 策略 3：强制回退 (Backtracking)

* **原理**：当前路径已死，回到上一个已知的“好状态”。
* **动作**：`git reset --hard HEAD~1` 并从记忆中抹去导致错误的思考链。
* **IO 实现**：调用文件系统 Effect 还原快照，修改 Memory State。

### 策略 4：人工介入 (Human-in-the-Loop / Escalation)

* **原理**：AI 搞不定了，抛出异常给人类。
* **动作**：暂停 Agent，发送通知，等待回调。
* **IO 实现**：这是一个异步 Effect。程序挂起（Suspend），序列化当前 Contimuation，等待外部信号唤醒。

---

## 7.5 形式化保证：Total Functional Programming 的启示

为什么我们如此在意 Loop Detection？因为我们想把 Agent 变成一个**全函数（Total Function）**，即对所有输入都在有限时间内停机。

虽然通用图灵机不可判定，但我们可以构建一个 **Budgeted Monad**。

**定理（非正式）**：
如果一个 Agent 系统满足以下条件，则必定终止：

1. 拥有有限的离散状态空间（通过 Hash 近似）。
2. 或者拥有一个单调递减的计数器（Budget）。

在工程中，我们通常结合两者：

* **Hard Limit**: `MaxSteps = 50`。
* **Soft Limit**: `MaxRetries = 3` (针对同一错误)。

通过将 Budget 显式建模为 Effect (`StateT Budget m a`)，我们在类型层面保证了 Agent 不会无限运行。

---

## 7.6 练习题

### 基础题

<details>
<summary><strong>习题 7.1：实现 N-Gram 重复检测器</strong></summary>

**场景**：你需要检测 Agent 是否在重复说话或重复调用工具。
**输入**：`history: List[String]`, `n: Int` (N-gram 长度)。
**要求**：编写一个纯函数，如果 `history` 的最后 `n` 个元素与紧邻的前 `n` 个元素完全相同，返回 `True`。
**提示**：使用列表切片。注意处理列表长度不足 `2*n` 的情况。

<details>
<summary>参考答案思路</summary>

```text
Function detectRepetition(history, n):
    If length(history) < 2 * n:
        Return False
    
    // 取最后 n 个
    suffix = history.slice(-n)
    // 取倒数第 2n 到 倒数第 n 个
    previous = history.slice(-2*n, -n)
    
    Return suffix == previous

```

在实际 Agent 中，通常会遍历 `n` 从 1 到 `max_n`，以检测不同长度的循环。

</details>
</details>

<details>
<summary><strong>习题 7.2：预算单调性测试</strong></summary>

**场景**：我们使用 `BudgetT` Monad Transformer 来管理 Token 和步数预算。
**要求**：设计一个类型签名和测试用例，证明无论 Agent 内部逻辑如何分支，Budget 总是单调递减的。
**提示**：这不需要具体代码，而是思维实验。如果 `step` 函数的类型是 `Budget -> (Result, Budget)`，如何断言 `Budget_out < Budget_in`？

<details>
<summary>参考答案思路</summary>

**核心断言**：
对于任何 Kleisli Arrow `k :: A -> BudgetT m B`：
执行 `(b, remaining) <- runBudgetT (k input) initialBudget`
必须满足 `remaining < initialBudget`。

这通常通过在 `bind (>>=)` 的定义中强制 `decrement` 来实现。如果这个性质成立，程序就具有“强终止性保证”。

</details>
</details>

### 挑战题

<details>
<summary><strong>习题 7.3：错误指纹归一化 (Error Normalizer)</strong></summary>

**场景**：Python 的 Traceback 包含行号和文件路径，这些在代码修改后会变，导致字符串匹配失效。
**输入**：两个 Python 异常堆栈字符串。
**要求**：设计一个算法或正则策略，判断这两个错误是否“本质上是同一个错误”。
**提示**：

1. 忽略内存地址 `0x...`。
2. 忽略具体的行号 `line 123`。
3. 关注 Exception 类型和最后一行 Error Message。

<details>
<summary>参考答案思路</summary>

**算法步骤**：

1. **提取核心**：只保留 Traceback 的最后一行（如 `ValueError: invalid literal for int()`）。
2. **提取路径**：保留文件名的最后一部分（`basename`），丢弃绝对路径。
3. **正则清洗**：
* 将 `at 0x[0-9a-f]+` 替换为 `<ADDR>`。
* 将 `line \d+` 替换为 `line <N>`。
* 将引号内的动态内容（如果过长）替换为 `<STR>`。


4. **比较**：比较清洗后的指纹。

**进阶**：使 Levenshtein 距离计算相似度，设定阈值（如 90% 相似即视为同一错误）。

</details>
</details>

<details>
<summary><strong>习题 7.4：设计“逃生解释器” (The Escape Interpreter)</strong></summary>

**场景**：当 `detectLoop` 决定终止时，它返回一个 `Abort` 信号。但我们希望 Agent 在临死前能生成一份“遗言”（Post-mortem analysis）。
**要求**：利用 `MonadError` 或 `EitherT`，设计一个控制流：捕获 `LoopDetected` 错误，利用 LLM 总结当前 Trace，输出分析报告，然后才真正退出。

<details>
<summary>参考答案思路</summary>

**伪代码流程**：

```text
program = do
    result <- tryCatch (runAgentLoop) 
              handleError: \err -> case err of
                  LoopDetected trace -> do
                       report <- llm.analyze(trace, "Why did you get stuck?")
                       log(report)
                       return (Failed report)
                  OtherError e -> throw e

```

**键点**：
这里展示了 Effect 系统如何让“错误”变成数据。Loop 不是程序崩溃，而是程序进入了一个特定的处理分支。这要求 Loop Detector 抛出的错误必须携带 `Trace` 上下文。

</details>
</details>

---

## 7.7 常见陷阱与错误 (Gotchas)

### 1. 误杀长耗时任务 (The Impatient Killer)

* **现象**：Agent 启动了一个长时间运行的构建过程，每隔 5 秒轮询一次状态。Loop Detector 看到连续 10 次 `check_status` 调用，判定为死循环并杀掉。
* **原因**：混淆了“轮询（Polling）”和“死循环（Looping）”。
* **对策**：
* **Observation 变化检测**：即使 Action 一样，如果 Observation（如日志输出的新行、进度百分比）在变化，则重置循环计数器。
* **显式 Wait 工具**：提供一个 `wait_for_condition` 的工具，而不是让 LLM 自己在那 `while True`。



### 2. 读操作的幂等性 (Idempotency Trap)

* **现象**：Agent 连续两次读取同一个文件，被判定为冗余操作。
* **原因**：有时 Agent 第一次读没看清（Context 溢出或注意力丢失），想读第二次。这是合理的。
* **对策**：区分 `Side-Effect Action` (Write/Run) 和 `Pure Action` (Read)。对 Pure Action 给予极高的容忍度（甚至不纳入 Loop 检测）。

### 3. Hash 碰撞与噪声

* **现象**：基于 Hash 的 Ping-Pong 检测失效，因为文件里包含了一个自动更新的时间戳注释。
* **对策**：在计算 Hash 前进行 **Content Filtering**。过滤掉注释、空行、或者特定的 metadata 区域，只对代码逻辑部分做 Hash。

### 4. 上下文截断导致的失忆

* **现象**：Agent 陷入循环，因为 Loop Detector 注入的警告信息（"你已经试过这个了"）因为 Context Window 满了被挤出去了。Agent 真的“忘”了。
* **对策**：Loop Detector 的状态必须存储在 **System Prompt** 或 **高优先级 Memory 区块** 中，确保永远不会被截断。如果 Context 满了，优先压缩旧的历史，而不是丢弃 Loop Warning。

---

## 7.8 本章总结

Loop Detection 是 Coding Agent 从“玩具”走向“生产力工具”的关键分水岭。

* **不要硬编码**：使用 IO/Kleisli 中间件模式，保持业务逻辑纯净。
* **多维视角**：结合句法（N-gram）、语义（Ping-Pong）和结果（Error Fingerprint）进行综合判断。
* **拥抱失败**：承认 Agent 会死循环，设计好“逃生舱”和“验尸报告”机制，让每一次失败都成为优化的养料。
