# Chapter 8 — A/B Test 与评估：把实验嵌进 IO/Kleisli 管道

## 1. 开篇段落：从“运气”到“科学”

在构建 LLM Agent 时，工程师往往面临一种无力感：Agent 的表现似乎高度依赖“运气”。修改了 Prompt 中的一个标点符号，在这个 Case 上变好了，却在另外十个 Case 上崩溃了。这种非确定性（Non-determinism）使得传统的单元测试（Unit Test）捉襟见肘。

**为什么 Agent 评估这么难？**

1. **概率本质**：LLM 是概率模型，同样的输入可能导致不同的输出。
2. **副作用依赖**：Agent 依赖外部工具（搜索、数据库），环境本身在变。
3. **高昂成本**：无法像传统软件那样每秒运行上万次测试。

本章的核心理念是：**不要把实验（Experimentation）当作代码写完后“外挂”的监控系统，而要将其视为 Agent 运行时不可或缺的一个 Effect（副作用）。**

我们将利用 **IO Monad** 的“纯描述”特性，构建可回放的、时间旅行般的评估系统；利用 **Kleisli Arrow** 的组合特性，在不侵入业务逻辑的前提下，像搭积木一样插入 A/B 分流器。无论是在线分流（Online Routing），还是离线反事实评估（Offline Counterfactual Evaluation），都将统一在同一个类型签名之下。

---

## 2. 文字论述

### 2.1 实验即 Effect (Experimentation as an Effect)

在传统的微服务架构中，A/B 测试通常通过一个全局的 `ExperimentClient` 单例来实现。但在函数式编程（FP）和 Agent 架构中，我们追求显式的依赖管理。

我们将“获取实验配置”定义为一种代数能力（Algebraic Ability）：

这意味着，任何需要做实验的代码段，都在类型签名上显式声明了它依赖 `Experiment` 能力。

* **Production Interpreter**：计算哈希或调用 Redis/LaunchDarkly 获取分流结果。
* **Test Interpreter**：总是返回 `Control` 组，或根据测试用例强制返回 `Variant`。
* **Replay Interpreter**：从历史日志中读取当时分配的分组，确保回放的一致性。

### 2.2 Kleisli 管道中的“铁路道岔”

在 Chapter 3 中，我们将 Agent 建模为 Kleisli Arrow：`Input -> M Output`。
A/B 测试本质上是一个**动态路由组合子（Combinator）**。

#### 2.2.1 路由结构图解

想象 Agent 的处理流程是一条铁路，A/B 测试就是道岔。

```ascii
[ User Input ]
      |
      v
(Pre-processing)
      |
      +------------------------+ (Decision Point: assign effect)
      |                        |
[ Variant A: "Reasoning" ] [ Variant B: "ReAct" ]
( Chain-of-Thought )       ( Direct Tool Use )
      |                        |
      +-----------+------------+
                  |
                  v
         ( Action Execution )
                  |
                  v
             [ Output ]

```

#### 2.2.2 代码语义（伪代码）

我们可以定义一个高阶函数 `experiment`，它接受两个 Kleisli Arrow，并返回一个新的 Kleisli Arrow：

```haskell
-- 这是一个组合子，它把实验逻辑“编织”进管道中
experiment :: (Monad m) 
           => ExperimentID 
           -> Kleisli m a b  -- Control (A组策略)
           -> Kleisli m a b  -- Treatment (B组策略)
           -> Kleisli m a b  -- 返回一个具备分流能力的箭头
experiment expId control treatment = Kleisli $ \input -> do
    -- 1. 获取分流上下文 (如 UserID)
    ctx <- askContext 
    -- 2. 执行 assign Effect
    variant <- Effect.assign expId ctx
    -- 3. 根据结果选择路径
    case variant of
        Control   -> runKleisli control input
        Treatment -> runKleisli treatment input

```

这种写法的巨大优势在于**局部性（Locality）**：你不需要复制整个 Agent 代码，只需要在 Prompt 构建、工具选择或特定的决策步骤上应用 `experiment` 组合子。

### 2.3 离线回放与反事实评估 (Counterfactual Evaluation)

这是 IO Monad 架构的杀手级应用。

**问题**：线上跑了 10,000 条数据（使用 V1 策略）。现在我想知道，如果当时用 V2 策略，效果会怎样？
**困难**：V2 策略可能会产生不同的工具调用参数。如果 V1 搜索了 "Apple price"，V2 搜索了 "Apple stock"，我们无法知道 "Apple stock" 的结果，因为那件事在历史上没发生。

**基于 IO Monad 的解决方案**：

我们需要构建一个 **Cached/Mock Interpreter**。

1. **Trace Recording (录制)**：
在线上运行时，记录所有的 `(StepInput, StepOutput)` 以及所有的 `(ToolRequest, ToolResponse)`。这构成了我们的“世界快照”。
2. **Simulation (回放)**：
在离线环境加载 V2 版本的代码，注入录制好的 Log。
* **情况 A：路径重合 (Convergence)**
V2 想要调用 `Search("Apple price")`。解释器查表发现 V1 做过完全一样的调用，于是直接返回历史记录中的 `"150 USD"`。
*结论*：在这个分支上，我们可以安全地评估 V2。
* **情况 B：路径偏离 (Divergence/Drift)**
V2 想要调用 `Search("Apple stock")`。解释器查表发现 V1 没做过这个。
*处置*：
* **保守策略**：抛出 `DriftDetected` 异常，放弃该样本的评估。
* **激进策略**：如果有模拟器（Simulator），尝试模拟结果；否则必须终止。





**Rule of Thumb**：

> 只要 Agent 的决策逻辑（Policy）变化没有导致外部副作用（External Effect）的参数变化，离线回放就是 100% 准确的。
> *适用场景*：Prompt 微调、格式修正、提取逻辑优化。

### 2.4 Interleaving：生成式模型的特有竞技场

对于生成内容（如写邮件、写代码），人类很难给出一个绝对分数。但是，人类非常擅长**比较**。
**Interleaving（错）** 是一种特殊的 A/B 测试形态。

**机制**：

1. 用户发来请求。
2. 系统**并发（Concurrency）** 运行 Agent V1 和 Agent V2（利用 `parZip` 或 `race` combinator）。
3. 系统将两个结果 A 和 B 展示给用户（位置随机打乱）。
4. 用户选择了一个。

**IO 建模**：

```haskell
-- 并发运行两个 Kleisli，并在 IO 层收集结果
interleave :: Kleisli IO In Out -> Kleisli IO In Out -> Kleisli IO In (Out, Out)
interleave k1 k2 = Kleisli $ \input -> do
    -- 并行执行，利用 IO 的异步能力
    (res1, res2) <- parZip (runKleisli k1 input) (runKleisli k2 input)
    return (res1, res2)

```

这种方法能极快地收敛出“哪个模型更好”，因为它消除了用户间的方差（User Variance）。

---

## 3. 本章小结

1. **Experiment as Effect**：将 A/B 分流视为抽象的副作用接口，解耦业务逻辑与实验配置。
2. **Kleisli Routing**：实验是计算管道中的分支节点，利用组合子保持代码整洁。
3. **Causal Validity**：离线评估的核心挑战是处理“反事实路径”的缺失。利用 IO Monad 的录制/回放机制，我们可以精确捕捉路径偏离（Drift）。
4. **Metric Hierarchy**：建立从“代码不崩”（Crash Rate）到“用户爱用”（Retention）再到“模型智能”（LLM-as-a-Judge）的多层指标体系。

---

## 4. 练习题

### 基础题

**Q1. 实验 ID 与上下文设计的陷阱**
我们需要实现 `assign` 函数。为了保证实验结果在统计上有效，我们需要对 UserID 进行 Hash。
请问：为什么简单的 `hash(UserID) % 100` 是不够的？如果在同一个 User 身上同时运行两个独立的实验（比如 UI 颜色实验和 Agent Prompt 实验），会发生什么问题？

<details>
<summary>点击查看提示与参考答案</summary>

* **Hint**: 思考“相关性”（Correlation）。如果一个 ID 的哈希值总是落在低区间，他是否总是会被分到所有实验的 Control 组？
* **Answer**:
* **问题**：如果所有实验共享同一个哈希算法且没有盐（Salt），那么 UserID 哈希值较小的用户将永远被分入所有实验的“前 50%”桶中。这导致实验之间产生强耦合（Orthogonality Violation），无法区分是 UI 变了导致效果好，还是 Prompt 变了导致效果好。
* **解决**：必须引入 **Salt**。公式应为 `hash(UserID + ExperimentID) % 100`。这样同一个用户在实验 A 中是 Control，在实验 B 中可能是 Variant，保证了正交性。



</details>

**Q2. 指标收集的组合性**
假设你有一个 metric 叫 `token_usage`。请用自然语言描述，如何利用 Writer Monad（或类似的累加器结构）在不修改 LLM 调用函数内部代码的前提下，统计一次完整 Agent 交互的总 Token 数？

<details>
<summary>点击查看提示与参考答案</summary>

* **Hint**: 装饰器模式。`FlatMap` 的结合律。
* **Answer**:
* 定义一个 Wrapper 函数，它接受一个 `LLMCall` 动作。
* 在这个 Wrapper 执行完 `LLMCall` 后，解析返回结果中的 `usage` 字段。
* 调用 `tell(Usage(tokens))` 将其写入 Writer Monad 的日志流。
* 由于 Writer Monad 具备 Monoid 性质（可结合），在外层运行整个 Agent Pipeline 时，所有步骤产生的 `Usage` 会自动相加，最终得到总和。



</details>

**Q3. 简单的哈希分流实现**
编写一个伪代码函数 `get_bucket(user_id, experiment_salt, total_buckets=100)`，要求输出确定性且均匀分布。

<details>
<summary>点击查看提示与参考答案</summary>

* **Hint**: MD5/SHA256, Hex string to Int.
* **Answer**:
```python
import hashlib
def get_bucket(user_id, salt, total_buckets=100):
    raw = f"{user_id}:{salt}".encode('utf-8')
    # 使用 SHA256 获取均匀分布的哈希
    hex_hash = hashlib.sha256(raw).hexdigest()
    # 取前 8 位（32-bit int）即可
    int_val = int(hex_hash[:8], 16)
    return int_val % total_buckets

```



</details>

---

### 挑战题

**Q4. 离线回放中的“确定性消除”**
在离线回放 V2 Prompt 时，V2 请求生成一个随机数（比如“从 1 到 10 选一个数作为重试等待时间”）。历史日志里的 V1 也生成过随机数，但是是 7。V2 现在的代码跑起来生成了 3。这会导致后续的行为不一致。
如何利用 IO Monad 的思想，在**不修改业务代码**的情况下，强行让 V2 在回放时也得到 7？

<details>
<summary>点击查看提示与参考答案</summary>

* **Hint**: 随机数生成器（RNG）本身应该是一个 Effect 吗？
* **Answer**:
* **核心思想**：必须将 `Random` 建模为 Effect，而不是直接调用 `Math.random()`。
* **抽象**：定义 `trait Random { def nextInt(n: Int): IO[Int] }`。
* **录制**：在线上 `RealRandomInterpreter` 中，每次调用 `nextInt`，不仅返回随机数，还将其记录到 Trace Log 中（顺序敏感）。
* **回放**：在离线 `ReplayRandomInterpreter` 中，维护一个指向 Log 的游标。每次业务代码请求随机数，直接弹出 Log 中的下一个值返回。这样就消除了随机性带来的偏差。



</details>

**Q5. 实验的“污染”与状态隔离**
在一个长 Session 中，我们在第 3 轮对话时开启了实验 `Memory_V2`（一种新的记忆压缩算法）。用户聊了 10 轮。
第 11 轮时，用户重置了 Session，但后端为了省钱，复用了同一个 Vector DB 的 Namespace。
请问：之前的实验数据会对新的 Session 造成什么影响？在架构层面如何规避？

<details>
<summary>点击查看提示与参考答案</summary>

* **Hint**: 副作用的范围控制。Resource management (`bracket`).
* **Answer**:
* **影响**：这叫 **Carryover Effect（残留效应）**。`Memory_V2` 产生的压缩记忆格式可能与默认版本不兼容，或者其包含的信息偏置会影响后续标准版本的表现。
* **架构规避**：
1. **Ephemeral Sandbox**：对于实验流量，使用临时的、带 TTL（Time-To-Live）的存储空间。
2. **Logic Separation**：在 Vector DB 的 Metadata 中打上 `algo_version: v2` 标签。读取时，标准版本代码必须显式过滤掉 `algo_version != v1` 的记忆。
3. **IO Bracket**：利用 `bracket(setup, teardown, use)` 模式。在实验 Session 结束时（`teardown`），自动触发清理逻辑，物理删除该实验产生的所有脏状态。





</details>

**Q6. 多臂老虎机 (Multi-Armed Bandit) 的 Effect 建模**
如果我们不仅想要 A/B Test，还想要自动化的 Thompson Sampling（谁效果好就多给谁流量）。
这要求我们的 `Assign` Effect 不再是纯函数式的哈希，而是依赖全局可变状态。请设计这个 Effect 的接口，以及一个用于反馈奖励（Reward）的接口。

<details>
<summary>点击查看提示与参考答案</summary>

* **Hint**: 这是一个闭环系统：Route -> Act -> Reward -> Update Model。
* **Answer**:
* **接口设计**：
```scala
trait Bandit[M[_]] {
   // 获取臂（Arm），可能涉及读取共享存储或调用外部 Bandit Service
   def selectArm(feature: Context): M[Arm] 
   // 反馈奖励，用于更新模型权重
   def reportReward(arm: Arm, reward: Double): M[Unit] 
}

```


* **集成点**：
* `selectArm` 发生在 Agent 启动前。
* `reportReward` 发生在 Agent 结束后的评估阶段（可能通过人工点赞，或自动化判题机）。


* **注意**：在 IO Monad 中，这通常意味着 `Experiment` 解释器需要持有 Redis 连接或数据库句柄来同步权重状态。



</details>

**Q7. 嵌套实验 (Nested Experiments) 的正交性验证**
你同时跑了“Prompt 变体”和“RAG 检索参数”两个实验。
离线分析时，你发现 Prompt A 组的平均响应时间比 B 组慢了 500ms。但是，仔细检查发现，Prompt A 组里恰好有 80% 的请求命中了“RAG 慢速检索”组（由于哈希碰撞或配置错误）。
作为架构师，你如何通过 Log/Trace 数据结构设计，来一眼识破这种“混杂偏差（Confounding Bias）”？

<details>
<summary>点击查看提示与参考答案</summary>

* **Hint**: 结构化日志。Canonical Log Line。
* **Answer**:
* **设计**：必须在每一个 Request 的最终日志（Canonical Log）中，包含一个 **Experiments Map** 字段。
* **结构**：`{"prompt_exp": "variant_A", "rag_exp": "slow_mode", ...}`
* **验证**：在分析阶段，进行 **卡方检验 (Chi-Square Test)**。构建一个列联表（Contingency Table），检查 `Prompt Variant` 和 `RAG Variant` 的分布是否独立。如果 ，说明两个实验分配不独立，存在 Bug，此时不能直接比较单因素指标。



</details>

---

## 5. 常见陷阱与错误 (Gotchas)

1. **Peeking (偷看) 导致的假阳性**
* *现象*：每跑完 100 个 Case 就跑一次 T 检验，一旦显著就宣布胜利。
* *原理*：多次检验会累积 Type I Error。
* *Rule of Thumb*：预先确定样本量（Sample Size），或者使用 **序贯概率比检验 (SPRT)** 方法。


2. **缓存击穿实验 (Caching Bias)**
* *场景*：Agent 框架为了快，在底层加了 Semantic Cache。
* *Bug*：User A (Group Control) 问了“天气”，缓存了结果。User B (Group Variant) 问了“天气”，直接拿到了 Control 组生成的答案。
* *修复*：Cache Key 必须包含 `Variant Hash`。
* *公式*：`Key = Hash(Prompt + UserInput + ActiveExperimentVariants)`。


3. **把“错误”当成“负样本” (Survivorship Bias)**
* *现象*：新模型 V2 代码有 Bug，处理复杂问题时直接 Crash（不返回结果）。简单的能处理。
* *数据*：统计“所有成功返回的对话”的质量，发现 V2 均分很高（因为它只处理了简单题）。
* *修复*：任何 Crash 或 Timeout 都必须被视为 **最低分（0分）** 纳入平均分计算。分母必须是 `Assigned Count` 而不是 `Completed Count`。


4. **Prompt 只有微小改动时的“盲目自信”**
* *误区*：“我只加了一句 'Think step by step'，肯定变强，不用测了直接上。”
* *现实*：LLM 对 Prompt 极其敏。这句话可能导致 JSON 输出格式错误率增加 10%，或者导致回复长度倍增从而触发 Token Limit 截断。
* *建议*：所有 Prompt 变更都是代码变更，必须过回放测试。
