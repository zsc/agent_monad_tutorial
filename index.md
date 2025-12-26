# 《用 IO Monad 搭建 LLM Agent：从 Kleisli 到事件建模与 FRP》目录（index.md）

> 本教程面向：想把 LLM agent 抽象成“可组合的、可测试的、可观测的 effectful program”的工程/研究读者。  
> 写作目标：用 **IO Monad / Kleisli Arrow** 统一 agent 的“调用工具、读写记忆、超时重试、日志追踪、A/B 实验、循环检测”等机制，并解释其与 **Functional Reactive Programming (FRP)** 的关系。  
> 代码风格：伪代码 + 可落地的类型签名；示例会覆盖 *Haskell/Scala/F#/TypeScript(函数式风格)* 的对照（每章末尾给“语言映射”小节）。

---

## 仓库结构（建议）

- `index.md`（你正在阅读）
- `chapter1.md`：为什么用 IO Monad 抽象 LLM agent
- `chapter2.md`：IO Monad 速成与工程化
- `chapter3.md`：Kleisli Arrow 是什么、为什么对 agent 特别合适
- `chapter4.md`：Agent DSL：把“工具/记忆/控制流/指标”都变成 effect
- `chapter5.md`：运行时（Runtime）：解释器、调度器、并发与取消
- `chapter6.md`：事件建模：timeout / retry / backoff / circuit breaker / budget
- `chapter7.md`：Coding Agent：loop detection 与“安全可终止”保证
- `chapter8.md`：实验与评估：A/B test、interleaving、离线回放与反事实
- `chapter9.md`：可观测性：trace、span、结构化日志、因果与可复现
- `chapter10.md`：与 FRP 的关系：事件流驱动 agent、流式 token 与 UI
- `chapter11.md`：综合案例：从零实现一个可插拔的 coding agent
- `chapter12.md`：扩展阅读：Free/Final Tagless、Algebraic Effects、Arrowized FRP

> 你后续可以按这个索引逐章生成 `chapterX.md`。

---

## 全书导航（Chapters & Sections）

### Chapter 1 — LLM Agent 的“抽象边界”：为什么从 IO Monad 出发（chapter1.md）
1.1 Agent 到底是什么：函数、进程、还是状态机？  
1.2 典型 agent 代码为何会“难组合、难测、难复现”  
1.3 把 agent 视为：`Input -> IO Output` 的可组合计算  
1.4 “工具调用/记忆/网络/文件/人类反馈”都是 IO effect  
1.5 抽象的收益清单：组合性、可替换性、可测试性、可观测性  
1.6 一个最小 agent：从 prompt-only 到 tool-augmented  
1.7 章节小结：本书会统一哪些问题（loop、A/B、timeout、事件流）

---

### Chapter 2 — IO Monad 速成：从概念到工程实现（chapter2.md）
2.1 Monad 三件套：`pure` / `bind(>>=)` / `map`  
2.2 IO 的直觉：把“世界交互”封装为值（描述）  
2.3 组合 effect：错误（Either/Result）、可选（Option）、状态（State）  
2.4 `Reader` 作为依赖注入：API key、配置、feature flag  
2.5 `Writer`/日志：结构化日志 vs 业务输出  
2.6 `State`/记忆：对话上下文、向量库句柄、缓存  
2.7 IO 与并发：async、取消（cancellation）、资源作用域（bracket）  
2.8 工程落地：在非纯语言里“模拟 IO Monad”（TypeScript/JS）  
2.9 章节练习：把一段脚本式 agent 重构成组合式 IO

---

### Chapter 3 — Kleisli Arrow：把“带 IO 的函数”当作可组合箭头（chapter3.md）
3.1 什么是 Kleisli：`a -> m b` 的组合规则  
3.2 与普通函数组合的区别：为什么 `a->IO b` 不能直接 `(.)`  
3.3 Kleisli 的 `compose` / `andThen`：把 agent pipeline 串起来  
3.4 Kleisli Category：把“effectful step”当作 morphism  
3.5 在 agent 中的意义：Planner、ToolRouter、MemoryUpdater 都是 Kleisli  
3.6 Kleisli 与错误短路：`a -> IO (Either e b)` 的组合语义  
3.7 Kleisli 与观测：在组合点插入 tracing、metrics、debug hooks  
3.8 Kleisli vs Arrow：何时需要更强的结构（并行/分支）  
3.9 章节小结：Kleisli 是 agent 的“管道胶水”

---

### Chapter 4 — Agent DSL：把工具、记忆与控制流做成“可解释的 effect”（chapter4.md）
4.1 需求：为什么仅有 `IO` 还不够（可测试、可回放、可替换）  
4.2 定义 Agent 能力集合：`ToolCall` / `Mem` / `Clock` / `Rand` / `Trace`  
4.3 方案 A：Free Monad（命令式 DSL + Interpreter）  
4.4 方案 B：Final Tagless（类型类/接口 + 多解释器）  
4.5 把 LLM 调用当 effect：`Chat : Req -> m Resp`  
4.6 把工具当 effect：`Tool : ToolReq -> m ToolResp`  
4.7 把记忆当 effect：KV、向量检索、结构化记忆图谱  
4.8 控制流 effect：`sleep`、`timeout`、`race`、`retry`、`budget`  
4.9 可替换解释器：真实运行 / mock / 录制回放 / 故障注入  
4.10 章节练习：写一个“纯测试解释器”验证 agent 行为

---

### Chapter 5 — Runtime 与调度：从解释器到并发、取消与资源管理（chapter5.md）
5.1 Runtime 的责任边界：解释、调度、隔离、限流  
5.2 运行模型：同步 vs 异步；pull vs push  
5.3 取消语义：用户取消、超时取消、上游取消如何传播  
5.4 资源作用域：文件句柄、网络连接、临时目录清理（bracket）  
5.5 并发组合：`parTraverse`、`race`、`timeout` 的类型与语义  
5.6 工具调用隔离：sandbox、权限、配额、速率限制  
5.7 多模型/多供应商：模型路由、fallback、健康检查  
5.8 章节小结：Runtime 是“让 DSL 变成现实”的机器

---

### Chapter 6 — 事件建模：IO timeout、重试、退避、熔断与预算（chapter6.md）
6.1 “事件”是什么：可观测的时间点/区间状态变化  
6.2 Timeout 的建模：  
- 6.2.1 `timeout : Duration -> m a -> m (Maybe a)`  
- 6.2.2 `timeoutOr : Duration -> e -> m a -> m (Either e a)`  
- 6.2.3 超时是异常？是值？是事件？各自 trade-off  
6.3 Retry 的建模：重试策略作为数据（Policy）  
6.4 Backoff/Jitter：避免羊群效应；如何把随机性纳入 effect（Rand）  
6.5 Circuit Breaker：半开/关闭/打开状态机（State + Clock）  
6.6 Budget/配额：token budget、工具调用次数、wall-clock budget  
6.7 Deadline vs Timeout：绝对截止时间的组合性更好  
6.8 Rate limit：令牌桶/漏桶作为 effect（或解释器能力）  
6.9 故障注入：网络抖动、慢响应、部分失败（用于鲁棒性测试）  
6.10 “再举一些事件”：  
- tool 返回结构不合法（schema error）  
- 权限拒绝（authz）  
- 输出被安全策略截断（policy truncation）  
- 上下文溢出（context overflow）  
- 记忆库不可用（vector db down）  
- 关键依赖降级（degraded mode）  
6.11 章节小结：把事件变成可组合的“值/流/状态”

---

### Chapter 7 — Coding Agent 的 loop detection：让 agent 可终止、可解释（chapter7.md）
7.1 Coding agent 的常见“死循环”类型：  
- 修同一个 bug 来回改  
- 测试失败→瞎改→更糟  
- 工具调用重复（同一命令/同一补丁）  
7.2 Loop detection 作为 effect：`DetectLoop : Trace -> m Decision`  
7.3 信号来源（features）：  
- 最近 N 次 action 的 n-gram  
- patch/AST diff 相似度  
- tool 调用序列的重复子串  
- failure signature（同一错误栈/同一测试用例）  
7.4 判定策略：阈、投票、多臂 bandit（对不同策略）  
7.5 处置策略：  
- 自动“升级计划”（replan）  
- 降温（cooldown）与强制探索  
- 请求人类介入（human-in-the-loop）  
- 回滚到 checkpoint  
7.6 终止性设计：max steps、deadline、budget、状态单调性  
7.7 章节练习：实现一个“重复工具调用”检测器 + 单元测试

---

### Chapter 8 — A/B Test 与评估：把实验嵌进 IO/Kleisli 管道（chapter8.md）
8.1 为什么 agent 更需要实验：随机性、非平稳、工具环境变化  
8.2 A/B 的最小抽象：`Assign : UserCtx -> m Variant`  
8.3 实验单位：session、task、step、tool-call 级别的选择  
8.4 指标建模：成功率、耗时、token、工具成本、满意度  
8.5 统计注意：  
- 8.5.1 分桶一致性（sticky assignment）  
- 8.5.2 干预泄漏（prompt contamination）  
- 8.5.3 多重比较与 early stopping  
8.6 更适合生成式的评估：interleaving、pairwise preference  
8.7 反事实与离线回放record/replay interpreter 做“重跑”  
8.8 与 bandit/自动路由：把策略当 Kleisli step  
8.9 章节小结：实验不是外挂，是运行时能力

---

### Chapter 9 — 可观测性：Trace/Span、日志、指标与可复现（chapter9.md）
9.1 观测的三件套：logs、metrics、traces  
9.2 Trace 作为 effect：`withSpan : Name -> m a -> m a`  
9.3 结构化事件：ToolCallStart/End、LLMRequest/Response、Timeout  
9.4 因果链：把 step-id、parent-id 串起来（Kleisli 天然适配）  
9.5 可复现运行：  
- 固定随机种子（Rand effect）  
- 录制外部 IO（HTTP、文件）  
- 版本锁定（prompt、模型、工具）  
9.6 调试与解释：把 reasoning/plan 当数据，不当字符串  
9.7 章节练习：输出一份“可回放的执行记录”并复跑

---

### Chapter 10 — 与 FRP 的关系：当 agent 变成“事件流处理器”（chapter10.md）
10.1 FRP 核心：Behavior（随时间变化的值）与 Event（离散事件）  
10.2 为什么 agent 与 FRP 会相遇：  
- token streaming  
- 工具进度、长任务状态  
- UI 交互（用户随时插话/打断）  
10.3 把 agent runtime 做成 FRP：  
- 输入流：用户消息、系统事件、工具回调  
- 输出流：token、动作、状态变更  
10.4 IO Monad vs FRP：  
- IO 强调“序列化的 effect 组合”  
- FRP 强调“随时间传播的依赖关系”  
10.5 二者结合的常见形态：  
- `IO` 里跑 FRP network（FRP 作为内部引擎）  
- FRP 流里嵌 `IO`（每个 event 触发 effect）  
10.6 超时/取消在 FRP 中的表达：debounce、throttle、switchMap/race  
10.7 章节小结：FRP 解决“持续交互与流式状态”，IO/Kleisli 解决“可组合 effect”

---

### Chapter 11 — 综合案例：实现一个可插拔的 coding agent（chapter11.md）
11.1 需求清单：工具（git/test/lint）、记忆、超时、重试、loop detection  
11.2 定义 DSL：`Chat`、`Tool`、`FS`、`Clock`、`Trace`、`Experiment`  
11.3 写解释器：真实解释器 + mock + record/replay  
11.4 写 agent pipeline（Kleisli）：Plan -> Act -> Observe -> Update  
11.5 事件处理：timeout、budget、失败分类与降级  
11.6 loop detection 接入点：每一步后检查 + 对策执行  
11.7 A/B：对不同 planner/prompt/tool-router 做实验  
11.8 FRP 接口：给 UI 输出 token 与进度条  
11.9 端到端演示：从“一个 bug”到“提交 PR”  
11.10 章节小结：从抽象到可运行系统

---

### Chapter 12 — 扩展阅读与路线图（chapter12.md）
12.1 Free Monad vs Final Tagless vs Algebraic Effects：选择指南  
12.2 Arrow、Profunctor、Optics：更强的组合工具箱  
12.3 Arrowized FRP：把 FRP 网络也写成 Arrow（与 Kleisli 对照）  
12.4 形式化语义：小步语义、可证明的终止性/安全性边界  
12.5 工程路线图：从 toy 到 production（多租户/合规/成本）  
12.6 推荐资料与关键词表（便于自行检索）

---

## 附录（chapter13.md）
A.1 术语表：Monad、Kleisli、Interpreter、Effect、FRP…  
A.2 类型签名速查表：timeout/retry/race/withSpan…  
A.3 常见坑：把日志写进业务输出、把异常当控制流、不可回放随机性  
A.4 练习题与参考答案索引（按章节编号）
