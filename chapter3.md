# Chapter 3 — Kleisli Arrow：把“带 IO 的函数”当作可组合箭头

## 1. 开篇段落：Agent 开发中的“胶水危机”

在构建 LLM Agent 时，我们本质上是在编排一系列**不确定**且**有副作用**的操作：

* 用户输入  检索向量库（可能失败、网络延迟）
* 构建 Prompt  调用 LLM（耗时、可能超时、返回乱码）
* 解析结果  执行工具（文件读写、API 调用）

在传统的命令式编程（Imperative Programming）中，这种流程通常长这样：

```python
# 典型的 "Spaghetti Code" Agent
async def run_agent(user_input):
    # 1. 检索
    try:
        context = await vector_db.search(user_input)
    except DbError:
        context = "" # 容错逻辑混在主流程里
    
    # 2. 构造 Prompt (纯逻辑)
    prompt = f"Context: {context}\nQuestion: {user_input}"
    
    # 3. LLM 调用
    response = await llm_client.chat(prompt)
    if not response.content: 
        return "Error: Empty response" # 错误检查
    
    # 4. 解析与执行
    if "<tool>" in response.content:
        cmd = parse_tool(response.content)
        result = await exec_tool(cmd) # 又是 IO
        return result
    else:
        return response.content

```

这段代码的问题在于：**业务逻辑（做什么）与控制流（错误处理、等待、重试）紧密耦合**。如果你想给每一步都加上“Tracing 追踪”或“Retry 重试”，代码体积会爆炸。

**Kleisli Arrow（克莱斯利箭头）** 是函数式编程提供的一把手术刀。它将形式为  的函数（即“输入 ，产生带  上下文的 ”）视为基本的**组合单元**。通过 Kleisli 组合，我们可以像搭积木一样， Agent 的思考、行动、观察串联成一条**线性的、类型安全的、自带错误处理的**管道。

**本章学习目标：**

1. 彻底理解 Kleisli Arrow 的定义  与普通函数的区别。
2. 掌握核心操作符 **Fish Operator (`>=>`)** 及其在不同语言中的实现。
3. 利用 Kleisli Category 重构 Agent：将 Planner、Executor、Memory 变成可组合模块。
4. **中间件模式**：如何利用 Kleisli 组合在不修改业务代码的前提下，透明地注入 Log、Trace 和 Retry。
5. 辨析 Kleisli 与 Arrow 的区别：何时我们需要并行能力。

---

## 2. 文字论述

### 2.1 什么是 Kleisli Arrow？

在数学范畴论中，“箭头”（Morphism）通常指纯函数 。我们可以直接组合它们：。
但在 Agent 工程中，绝大多数函数是**不纯的**（Effectful）。我们称这类函数为 **Kleisli Arrow**。

定义：一个 **Kleisli Arrow** 是一个具有如下类型签名的函数：

其中：

* : **输入类型**（Input。
* : **Monad 上下文**（Context/Effect）。在 Agent 中，这通常是 `IO`、`Promise`、`Future`，或者是包含错误处理的 `IO[Either[Error, ?]]`。
* : **输出值类型**（Result）。

**Agent 组件作为 Kleisli Arrow 的映射：**

| Agent 组件 | 输入 () | Effect () | 输出 () | 函数签名示例 |
| --- | --- | --- | --- | --- |
| **Retriever** | `Query` | `IO` (网络/DB) | `List[Doc]` | `search: Query -> IO [Doc]` |
| **LLM Model** | `Prompt` | `IO` (网络/延迟) | `String` | `chat: Prompt -> IO String` |
| **Parser** | `String` | `Either Error` (解析可能失败) | `Action` | `parse: String -> Either Err Action` |
| **Tool** | `Action` | `IO` (副作用) | `Observation` | `run: Action -> IO Observation` |

### 2.2 组合的几何学：为什么不能直接调用？

如果我们有两个 Kleisli Arrow：

1.   (例如：`getUserInput`)
2.   (例如：`searchGoogle`)

我们**不能**写 。
因为  返回的是一个“盒子” `IO B`，而  需要的是盒子里的裸值” `B`。即类型不匹配：


在命令式语言里，我们用 `await` 或 `.then()` 强行拆包。但在抽象层面，我们需要一种操作符，能够自动完成“**拆包 -> 传递 -> 封包**”的过程。

### 2.3 The Fish Operator (`>=>`)

这种组合操作符被称为 **Kleisli Composition**，符号通常写作 `>=>`（形状像一条鱼），也称为“鱼操作符”。

定义如下（以 Haskell 风格为例）：

它的执行逻辑是：

1. 拿到输入 ，传给 。
2.  运行，产生 （比如一个 Promise）。
3. 利用 Monad 的 `bind` () 能力，等待  完成并取出 。
4. 将  传给 。
5.  运行，产生 。
6. 返回这个 。

#### ASCII 流程图：Kleisli Pipeline

```text
Pipeline: agent = step1 >=> step2 >=> step3

      [ Input A ]
          |
          v
    +-----------+
    |  Step 1   | f: A -> m B
    +-----------+
          | outputs (m B)
          v
    [ Bind/FlatMap Magic ] <--- 自动拆包，如果是 Error 则跳过后续
          | passes (B)
          v
    +-----------+
    |  Step 2   | g: B -> m C
    +-----------+
          | outputs (m C)
          v
    [ Bind/FlatMap Magic ]
          | passes (C)
          v
    +-----------+
    |  Step 3   | h: C -> m D
    +-----------+
          |
          v
      [ Output m D ]

```

### 2.4 工程实战：重构 Agent Pipeline

假设我们有以下三个原子能力（Atomic Capabilities）：

1. `promptTemplate`: 将用户问题转换为 Prompt。这是一个纯函数，但为了组合，我们可以用 `pure` 提升它，或者使用 `map`。
* 类型：`String -> Prompt`


2. `llmInference`: 调用模型。
* 类型：`Prompt -> IO String`


3. `extractTool`: 解析 JSON。
* 类型：`String -> IO ToolCall` (假设解析失败抛出 IO 错误)



**使用 Kleisli 组合的代码（伪代码）：**

```haskell
-- 定义 pipeline
thinkingProcess :: String -> IO ToolCall
thinkingProcess = 
    (pure . promptTemplate)  -- 1. 纯函数提升为 Kleisli
    >=> llmInference         -- 2. IO 操作
    >=> extractTool          -- 3. IO 操作 (含解析逻辑)

-- 运行
result = thinkingProcess("帮我查一下天气") 
-- result 是 IO ToolCall

```

**这一行的价值在于：** 我们定义了**控制流的形状**，而没有执行它。这让我们可以对 `thinkingProcess` 进行整体的测试、超时控制或修饰，而不需要深入到函数内部。

### 2.5 强大的中间件能力 (Middleware)

Kleisli Arrow 最迷人的地方在于它非常适合做 **AOP (面向切面编程)**。我们可以编写高阶函数来“修饰”任何一个 Kleisli Arrow。

#### 场景 1：自动重试 (Retry)

定义一个修饰器 `withRetry`：


它接收一个箭头，返回一个新的箭头：如果原箭头失败，新箭头会重试 N 次。

#### 场景 2：链路追踪 (Tracing)

定义 `withTrace(name)`：
它会在执行前后记录开始和结束时间，并向分布式追踪系统发送 Span。

#### 组合后的 Agent：

```text
robustAgent = 
   (pure . template)
   >=> withTrace("LLM_Call", withRetry(3, llmInference))  // 带追踪和重试的 LLM
   >=> withTrace("Parser", extractTool)                   // 带追踪的解析

```

这种**声明式**的写法，让“功能需求”（业务逻辑）和“非功能需求”（稳定性、观测性）完全解耦。

### 2.6 Kleisli 的局限性：串行 vs 并行

Kleisli Arrow 本质上是**串行**的（Sequential）。
`f >=> g` 意味着  必须等待  完成才能获得输入。

如果你需要**并行**（例如：同时调用 3 个不同的 LLM 模型投票），Kleisli Arrow 就不够用了。这时我们需要引入 **Arrow** (更通用的接口) 或 **Applicative Functor** (`parMap`, `parZip`)。

* **Kleisli**:  (链式依赖)
* **Applicative**:  (无依赖并行)

在第 5 章（Runtime）中，我们会详细讨论如何在 Kleisli 管道的某个节点内部嵌入并行计算。

---

## 3. 本章小结

1. **Kleisli Arrow** 是类型为  的函数，它是 Agent 系统中“带副作用步骤”的标准抽象。
2. **Fish Operator (`>=>`)** 是 Kleisli 的组合工具，它自动处理了 Monad 上下文的解包与传递，消除了回调地狱。
3. **短路特性**：基于 `IO` 或 `Either` Monad 的 Kleisli 组合天生具有短路能力。上一步报错，下一步自动跳过。
4. **中间件架构**：Kleisli Arrow 极易被装饰。我们可以通过高阶函数无侵入地为 Agent 的任意步骤添加 Retry、Timeout、Tracing 和 Logging。
5. **设计原则**：将 Agent 拆解为细粒度的 Kleisli Arrow，然后通过组合子（Combinators）将它们拼装成完整的运行时。

---

## 4. 练习题

> **提示**：除了思考类型签名，尝试用你熟悉的语言（JS/TS/Python）构思其实现逻辑。

### 基础题 (50%)

**Q1. 类型体操：拼接管道**
已知：

* `fetchUser : UserID -> IO UserProfile`
* `vectorSearch : UserProfile -> IO [Document]`
* `summarize : [Document] -> IO String`

请写出将 `UserID` 转换为 `String` (Summary) 的 Kleisli 组合表达式。
*(Hint: 只需要关注输入输出类型的首尾相接)*

<details>
<summary>参考答案 (折叠)</summary>

表达式：
`fetchUser >=> vectorSearch >=> summarize`

类型推导过程：

1. `UserID -> IO UserProfile`
2. `UserProfile -> IO [Document]` (输入匹配上一步的输出)
3. `[Document] -> IO String` (输入匹配上一步的输出)
最终得到：`UserID -> IO String`

</details>

---

**Q2. 纯函数的介入**
假设我们有一个纯函数 `filterDocs : [Document] -> [Document]`，它不过滤网络，只在内存里操作。
如何将它插入到 Q1 的管道中 `vectorSearch` 和 `summarize` 之间？
*(Hint: 纯函数不能直接用 `>=>`，需要提升)*

<details>
<summary>参考答案 (折叠)</summary>

纯函数 `A -> B` 需要提升为 `A -> IO B` 才能参与 Kleisli 组合。
方法是使用 `pure` (或 `return` / `async (x) => x`)。

表达式：
`fetchUser >=> vectorSearch >=> (pure . filterDocs) >=> summarize`

或者利用 Functor 的 `map` 语义，在组合外部操作：
`fetchUser >=> (vectorSearch.map(filterDocs)) >=> summarize` (这种写法取决于具体语言库的实现，第一种更通用)

</details>

---

**Q3. TypeScript 实现**
在 TypeScript 中实现 `composeK` (即 `>=>`)。
类型定义参考：`type KFunc<A, B> = (a: A) => Promise<B>`。

<details>
<summary>参考答案 (折叠)</summary>

```typescript
// 这是一个泛型的高阶函数
const composeK = <A, B, C>(
    f: (a: A) => Promise<B>, 
    g: (b: B) => Promise<C>
) => {
    // 返回一个新的函数
    return async (a: A): Promise<C> => {
        const b = await f(a); // 1. 执行 f 并等待解包
        return g(b);          // 2. 将结果传给 g 并返回
    };
};

// 使用示例
// const pipeline = composeK(step1, step2);

```

</details>

---

### 挑战题 (50%)

**Q4. "Circuit Breaker" (熔断器) 中间件**
设计一个高阶函数 `circuitBreaker`，它包装一个 Kleisli Arrow。
逻辑：如果最近 5 次调用中有 3 次失败，则在接下来的 1 分钟内直返回 Error，不再实际执行被包装的函数。
问题：实现这个逻辑需要什么额外的“副作用”？这是否破坏了 Kleisli 的纯粹性？
*(Hint: `IO` Monad 内部可以包含 `State` 或 `Ref`)*

<details>
<summary>参考答案 (折叠)</summary>

**所需能力**：需要一个可变的、跨请求持久化的状态（State/Ref），用来记录失败计数和上次失败时间。

**类型签名设计**：
`circuitBreaker : StateRef -> (A -> IO B) -> (A -> IO B)`

**纯粹性分析**：
这**没有**破坏纯粹性，前提是“读取/写入状态”这个动作本身被封装在了 `IO` Effect 中。
当我们组合出这个 Agent 时，我们只是构建了一个“描述”。只有当 Agent 运行时，这个状态才会被修改。
在 Haskell/Scala Cats 中，通常使用 `Ref[IO, BreakerState]` 来实现这种并发安全的内部状态。

</details>

---

**Q5. 分支逻辑 (Switching)**
有时候我们需要根据上一步的结果决定下一步走哪条路（例如：如果意图是 "Search" 走搜索管道，如果是 "Chat" 走闲聊管道）。
请定义一个函数 `branch`：
输入：

1. `check : A -> Boolean` (判断条件)
2. `ifTrue : A -> IO B` (Kleisli Arrow 1)
3. `ifFalse : A -> IO B` (Kleisli Arrow 2)
输出：
`A -> IO B`

请问这个 `branch` 函数本身是 Kleisli Arrow 吗？它能被组合进管道吗？

<details>
<summary>参考答案 (折叠)</summary>

**实现逻辑**：

```typescript
const branch = (check, ifTrue, ifFalse) => (input) => {
    return check(input) ? ifTrue(input) : ifFalse(input);
}

```

**回答**：
是的，`branch` 返回的结果依然是一个 `A -> IO B` 类型的函数。
因此，它**完美符合** Kleisli Arrow 的定义。
你可以把它像普通步骤一样组合进管道：
`preprocess >=> branch(isSearch, searchPipe, chatPipe) >=> postprocess`
这就是 Kleisli 的强大之处：**控制流也是管道的一部分**。

</details>

---

**Q6. 调试 Kleisli**
在一个长长的 `f >=> g >=> h >=> ...` 链条中，如果中间某一步数据不对，如何调试？
能不能写一个 `tap` 函数，允许我们在管道中间 `console.log` 数据，但不改变数据流向？

<details>
<summary>参考答案 (折叠)</summary>

可以。这个模式通常叫 `tap` 或 `inspect`。

**定义**：
`tap : (A -> IO Unit) -> (A -> IO A)`
或者更简单版本（只打印）：
`logResult : String -> (A -> IO A)`

**实现 (TS)**:

```typescript
const inspect = <A>(tag: string) => async (x: A): Promise<A> => {
    console.log(`[${tag}]`, x);
    return x; // 原样返回，保持管道流动
};

```

**使用**:
`step1 >=> inspect("After Step1") >=> step2`
这让调试变得像在管道上打孔一样简单。

</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 1. 隐式状态丢失

* **现象**：在管道的一开始获取了 `UserID`，但在执行了 3 步之后，第 4 步又需要 `UserID`。
* **错误**：因为 Kleisli 是 `A -> m B`, `B -> m C`，中间的信息如果没有显式传递，就会丢失。
* **解决**：
* **Payload 传递**：让每一步返回 `(Result, Context)` 元组。
* **Reader Monad**：将 Monad 栈升级为 `ReaderT Context IO`，这样任何步骤都可以随时 `ask` 获取环境信息。



### 2. 这里的 `Error` 到底是谁的 `Error`？

* **现象**：LLM 返回了 `"I don't know"`（业务层面的失败），但 IO Monad 认为这是成功的（网络请求成功了）。
* **陷阱**：直接用 `IO` 的错误机制处理业务逻辑错误。
* **最佳实践**：
* **IO Error** (Exception)：保留给网络断连、超时、API 500 等基础设施错误。
* **Domain Error** (Either)：LLM 拒绝回答、工具参数解析错误等，应该作为**返回值**的一部分（`IO (Either Refusal Answer)`），而不是抛出异常。



### 3. "Promise Hell" 的变体

* **现象**：虽然用了 Kleisli 概念，但实现时还是手动写 `step1().then(res => step2(res))`。
* **建议**：一定要封装通用的 `compose` 或 `pipe` 函数。如果语言支持（如 F# `>=>`, Haskell `>=>`, Scala Cats `andThen`），请直接使用库函数。在 TS/JS 中，使用 `fp-ts` 或简单的 utility function 来保持代码的平铺。

### 4. 忽略了 `pure` 的开销

* **现象**：把大量的纯计算逻辑（如复杂的字符串正则匹配）强行拆成微小的 Kleisli Arrow。
* **权衡**：虽然组合性好了，但每次 `pure` 提升并在 Monad 中 bind 都会带来微小的运行时开销（Event Loop tick）。对于极度密集的 CPU 计算，直接写成纯函数块，只在最后提升一次即可。
