# 08 当前 Agent 图详解：chat 与 tool_call 双节点循环

上一篇讲了 LangGraph 的基础概念。这一篇进入当前项目最核心的 Agent 图：一个由 `chat` 和 `tool_call` 两个节点构成的循环。

这个图看起来很简单，但它已经包含了 Agent 的核心能力：模型思考、工具调用、工具结果回填、再次思考、最终回答。理解这个图，就能理解大多数工具调用型 Agent 的基本运行方式。

## 1. 当前项目的 Agent 图是什么

当前项目的 Agent 图可以表示为：

```text
START -> chat -> tool_call -> chat -> END
```

更准确地说：

```text
START -> chat
chat -- 有 tool_calls --> tool_call
tool_call -> chat
chat -- 没有 tool_calls --> END
```

也就是说，`chat` 是入口，也是判断是否结束的节点。`tool_call` 只是当模型请求工具时才进入。

## 2. 为什么图这么简单

很多人会期待一个生产级 Agent 图非常复杂，有 Planner、Executor、Critic、Memory、Router 等很多节点。

但当前项目选择了一个非常克制的基础图，原因是：

- 简单图更容易理解和维护。
- 工具调用型 Agent 的最小闭环就是 chat + tool_call。
- LangGraph Checkpointer 已经提供了状态持久化。
- Memory Search 在进入图前完成，不必单独做成节点。
- 复杂业务可以在这个基础上逐步扩展。

生产级框架不等于一开始就复杂，而是要有清晰、可扩展的核心。

## 3. chat 节点的定位

`chat` 节点是 Agent 的推理节点。

它对应方法：

```text
LangGraphAgent._chat
```

它承担几个关键职责：

- 构造系统 Prompt。
- 整合长期记忆。
- 准备消息上下文。
- 调用 LLM Service。
- 记录 LLM 调用指标。
- 处理模型响应。
- 判断是否进入工具节点。

可以把 `chat` 节点理解成 Agent 的“大脑”。

## 4. chat 节点为什么要加载系统 Prompt

模型本身不知道当前系统要它扮演什么角色，也不知道用户长期偏好。

所以 `chat` 节点会调用：

```text
load_system_prompt(username=username, long_term_memory=state.long_term_memory)
```

这一步会把以下信息合成系统 Prompt：

- Agent 的行为准则。
- 用户名。
- 当前长期记忆。
- 其他动态上下文。

这样模型才知道应该如何回答。

## 5. chat 节点为什么不直接调用 OpenAI

`chat` 节点调用的是：

```text
self.llm_service.call(...)
```

而不是直接调用 OpenAI SDK。

原因是模型调用有很多生产问题：

- 超时。
- 限流。
- API 错误。
- 模型 fallback。
- 工具绑定。
- 结构化输出。
- 指标记录。

这些不应该散落在 Agent 节点中，而应该封装在 LLM Service。

这体现了职责分离：Graph 负责流程，LLM Service 负责模型可靠性。

## 6. chat 节点如何决定下一步

`chat` 节点会检查 LLM 返回的消息是否包含工具调用。

逻辑是：

```text
如果 response_message 是 AIMessage 且存在 tool_calls：
    goto = "tool_call"
否则：
    goto = END
```

然后返回：

```text
Command(update={"messages": [response_message]}, goto=goto)
```

这就是当前 Agent 图的路由核心。

## 7. tool_calls 是什么

当模型认为需要调用工具时，它不会直接执行工具，而是在响应中声明工具调用。

一个工具调用通常包含：

- 工具名称。
- 工具参数。
- 工具调用 ID。

程序收到后，才能在受控环境中执行对应工具。

这种设计的好处是：

- 模型负责决策。
- 程序负责执行。
- 工具执行在后端受控环境中完成。
- 工具结果可以被记录和追踪。

## 8. tool_call 节点的定位

`tool_call` 节点对应：

```text
LangGraphAgent._tool_call
```

它是 Agent 的行动节点。

主要职责是：

- 读取上一条 AIMessage 中的工具调用。
- 根据工具名找到工具对象。
- 执行工具。
- 把工具结果封装为 `ToolMessage`。
- 回到 `chat` 节点。

它不负责解释工具结果，也不直接回答用户。解释和最终回答仍然交给 `chat` 节点中的 LLM。

## 9. 为什么工具结果要回填给模型

假设用户问：

```text
帮我查询订单 123 的状态
```

模型可能调用 `get_order_status(order_id=123)`。

工具返回：

```text
订单已发货，预计明天送达
```

如果工具结果直接返回给用户，可能缺少自然语言组织和上下文解释。

更好的方式是把工具结果作为 `ToolMessage` 回填给模型，让模型生成最终回答：

```text
你的订单 123 已经发货，预计明天送达。
```

这就是 `tool_call -> chat` 的意义。

## 10. 为什么 tool_call 执行后必须回到 chat

工具调用只是获取信息或执行动作，不等于完成回答。

回到 `chat` 后，模型可以：

- 阅读工具结果。
- 判断是否还需要另一个工具。
- 整理最终回答。
- 用更符合用户语境的方式表达。

这形成了 Agent Loop：

```text
模型思考 -> 调用工具 -> 获得结果 -> 再思考 -> 回答
```

## 11. 多工具并发执行

当前项目支持一次响应中包含多个工具调用。

如果只有一个工具调用，直接执行。
如果有多个工具调用，使用：

```text
asyncio.gather(...)
```

并发执行。

生产价值：

- 多个独立工具可以并行。
- 减少总等待时间。
- 提升复杂任务响应速度。

例如模型同时需要查询用户资料和订单状态，如果两个工具互不依赖，就可以并发执行。

## 12. 工具是如何找到的

Agent 初始化时会构建：

```text
self.tools_by_name = {tool.name: tool for tool in tools}
```

当 LLM 返回工具名称时，`tool_call` 节点通过名称查找对应工具。

这意味着工具名称是模型和程序之间的协议。

如果工具名称不稳定，模型调用就会失败。

## 13. 工具在哪里绑定给模型

Agent 初始化时会调用：

```text
self.llm_service.bind_tools(tools)
```

这一步告诉模型有哪些工具可用、工具参数是什么。

如果不绑定工具，模型就不知道可以调用哪些外部能力。

## 14. 当前图如何支持多轮对话

多轮对话不是靠 `chat` 节点本身记住，而是靠：

- `GraphState.messages`
- `add_messages`
- Checkpointer
- `thread_id=session_id`

每次请求进入同一个 Session 时，LangGraph 可以根据 thread_id 找回之前状态。

这让多轮对话不依赖单个进程内存。

## 15. 当前图如何支持长期记忆

长期记忆没有单独作为图节点，而是在 `get_response` 中进入图前检索。

流程是：

```text
memory_service.search(user_id, latest_user_message)
```

检索结果放入：

```text
GraphState.long_term_memory
```

然后在 `chat` 节点注入 Prompt。

这样设计的好处是：

- 图保持简单。
- 记忆检索可以和 state 读取并发。
- `chat` 节点只关心使用记忆，不关心检索细节。

## 16. 当前图如何支持观测

`chat` 节点调用 LLM 时使用指标：

```text
llm_inference_duration_seconds.labels(model=model_name).time()
```

执行 config 中也包含 Langfuse callback。

这样可以观测：

- 模型调用耗时。
- 使用了哪个模型。
- 哪个 Session 触发调用。
- LLM 输入输出 Trace。

生产级 Agent 必须可观测，否则图越复杂越难排查。

## 17. 当前图如何处理错误

`chat` 节点中如果 LLM 调用失败，会记录日志并抛出异常。

`get_response` 外层也会捕获异常，使用：

```text
logger.exception("get_response_failed", ...)
```

这保证错误有堆栈信息。

不过当前图中的工具执行错误主要依赖工具自身和 LangGraph retry policy。后续如果业务需要，可以增加工具级错误包装、降级回答或人工介入节点。

## 18. 为什么 tool_call 节点有 RetryPolicy

在创建图时，`tool_call` 节点配置了 retry policy。

工具调用常见失败包括：

- 网络波动。
- 下游服务超时。
- 临时数据库错误。
- 第三方接口限流。

对工具节点增加重试，可以提升整体稳定性。

但要注意：不是所有工具都适合重试。比如支付、下单等有副作用操作，需要幂等设计。

## 19. 当前图的优点

### 19.1 简单清晰

只有两个核心节点，学习成本低。

### 19.2 可扩展

后续可以加节点：

- plan
- reflect
- human_approval
- memory_write
- router
- supervisor

### 19.3 状态持久化

通过 Checkpointer 支持多轮和恢复。

### 19.4 工具闭环完整

支持模型选择工具、后端执行工具、模型解释结果。

### 19.5 生产能力集成

结合 LLM Service、Memory Service、Metrics、Langfuse、Session，实现生产级闭环。

## 20. 当前图的局限

这个基础图也有局限：

- 没有显式最大循环次数控制。
- 没有单独规划节点。
- 没有工具调用前审批。
- 没有工具结果压缩节点。
- 没有多 Agent 协作。
- 没有复杂任务分解。

但这不是缺陷，而是模板的设计选择：先提供稳定最小核心，再让业务根据需要扩展。

## 21. 如何扩展这个图

### 21.1 添加人工审批节点

适合高风险工具，例如转账、删除数据、发送邮件。

图可以变成：

```text
chat -> approval -> tool_call -> chat
```

### 21.2 添加规划节点

适合复杂任务：

```text
plan -> chat -> tool_call -> chat
```

### 21.3 添加反思节点

适合高质量回答：

```text
chat -> critique -> chat -> END
```

### 21.4 添加记忆写入节点

当前记忆写入在后台任务中。如果需要强一致，也可以做成图节点。

## 22. 来源与依据

本篇依据当前项目以下文件：

- **docs/architecture.md**：明确说明 Agent 是两节点 `StateGraph`：`chat -> tool_call -> chat`。
- **app/core/langgraph/graph.py**：`_chat`、`_tool_call`、`create_graph` 的具体实现。
- **app/schemas/graph.py**：`GraphState` 的消息和长期记忆字段。
- **app/core/langgraph/tools/**：工具集合来源。
- **app/services/llm/service.py**：工具绑定和 LLM 调用服务。
- **app/services/memory.py**：长期记忆检索和写入。
- **app/core/metrics.py**：LLM 调用耗时指标来源。

## 23. 依据示例

### 23.1 两节点图依据

`create_graph` 中添加了两个节点：

```text
chat
tool_call
```

并设置入口为 `chat`。

### 23.2 chat 路由依据

`_chat` 中检查：

```text
AIMessage.tool_calls
```

有工具调用则 `goto="tool_call"`，否则 `goto=END`。

### 23.3 tool_call 回环依据

`_tool_call` 返回：

```text
Command(update={"messages": outputs}, goto="chat")
```

说明工具执行后一定回到 `chat`。

### 23.4 并发工具依据

`_tool_call` 中多个工具调用使用：

```text
asyncio.gather
```

说明项目支持多工具并发执行。

## 24. 本篇总结

当前项目的 Agent 图虽然只有两个节点，但它已经实现了工具调用型 Agent 的核心闭环：

- `chat` 节点负责模型推理。
- `tool_call` 节点负责工具执行。
- `Command` 负责更新状态和路由。
- `tool_calls` 决定是否进入工具节点。
- 工具结果以 `ToolMessage` 回填给模型。
- Checkpointer 保存多轮状态。
- Memory 和 Prompt 提供个性化上下文。

理解这个双节点循环，是后续学习 Loop 控制、工具系统、记忆系统、多 Agent 扩展的基础。

## 25. 本篇练习

1. 在 `graph.py` 中找到 `_chat` 方法，列出它做的 5 件事。
2. 找到 `_chat` 中判断工具调用的代码。
3. 找到 `_tool_call` 中读取 `state.messages[-1].tool_calls` 的代码。
4. 找到工具执行后返回 `goto="chat"` 的代码。
5. 思考：为什么工具结果不直接返回用户，而要回到模型？
6. 设计一个新节点 `human_approval`，画出它应该插入到当前图的哪个位置。
