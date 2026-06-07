# 04 一次聊天请求的完整链路：从 FastAPI 到 LangGraph

这一篇从一次真实的 `POST /api/v1/chatbot/chat` 请求出发，拆解它在当前项目中经过了哪些层、每一层为什么存在、它们解决什么生产问题。

学习 Agent 框架时，不要只看 LangGraph。生产级 Agent 请求不是直接进入模型，而是要经过 HTTP 接入、认证、限流、Schema 校验、会话绑定、记忆检索、Graph 执行、模型调用、工具循环、状态保存、响应封装等步骤。

## 1. 本篇目标

读完后你应该能回答：

- 一个聊天请求从哪里进入项目？
- 为什么聊天接口需要 Session Token？
- 请求如何进入 `LangGraphAgent`？
- 长期记忆在什么时候检索？
- LangGraph 如何决定是否调用工具？
- 响应如何返回给客户端？
- 当前项目中哪些文件可以作为依据？

## 2. 请求入口在哪里

普通聊天接口定义在：

```text
app/api/v1/chatbot.py
```

核心接口是：

```text
POST /chat
```

它通过 API Router 注册到 `/api/v1/chatbot` 前缀下，所以完整路径是：

```text
POST /api/v1/chatbot/chat
```

这个接口接收 `ChatRequest`，返回 `ChatResponse`。

## 3. 请求进入前发生了什么

请求到达路由函数前，已经经过了 FastAPI 应用层和中间件层。

入口文件：

```text
app/main.py
```

它负责：

- 创建 FastAPI 应用。
- 注册 API Router。
- 注册中间件。
- 初始化 metrics。
- 初始化 Langfuse。
- 在 lifespan 中预热 Graph、Memory、Cache。

生产级框架需要这些入口装配能力，因为真实请求进入业务函数之前，通常必须先处理日志、请求 ID、限流、指标、异常等横切逻辑。

## 4. 第一步：路由层接收请求

在 `app/api/v1/chatbot.py` 中，普通聊天路由大致做了这些事：

1. 通过 `@router.post("/chat", response_model=ChatResponse)` 声明接口。
2. 通过 `@limiter.limit(...)` 添加限流。
3. 通过 `ChatRequest` 校验请求体。
4. 通过 `Depends(get_current_session)` 获取当前 Session。
5. 调用 `agent.get_response(...)`。
6. 返回 `ChatResponse`。

这体现了 API 层的职责：接请求、做协议校验、做认证依赖、调用核心能力、封装响应。

## 5. 第二步：限流保护

聊天接口有成本，因为它可能触发：

- LLM 调用。
- 长期记忆检索。
- 工具调用。
- 数据库 Checkpoint。

所以项目使用 slowapi 做限流。配置来源在：

```text
app/core/config.py
```

默认配置中聊天接口类似：

```text
RATE_LIMIT_CHAT = 30 per minute
```

限流的生产价值是：

- 防止恶意刷接口。
- 控制 LLM 成本。
- 防止数据库和模型服务被打满。
- 给系统留出降级和恢复空间。

## 6. 第三步：Session 认证

聊天函数中有一个重要参数：

```text
session: Session = Depends(get_current_session)
```

这意味着客户端必须携带 Session Token，而不是随便传一个 `session_id`。

`get_current_session` 定义在：

```text
app/api/v1/auth.py
```

它会做几件事：

1. 从 Authorization Header 读取 Bearer Token。
2. 清洗 token 字符串。
3. 调用 `verify_token` 解析 JWT。
4. 得到 `session_id`。
5. 查询数据库确认 Session 存在。
6. 将 `user_id` 绑定到日志上下文。
7. 返回 Session 对象。

这样设计的好处是：聊天请求不相信客户端直接传入的会话 ID，而是从签名 Token 中解析可信身份。

## 7. 第四步：请求体 Schema 校验

聊天接口使用 `ChatRequest` 接收请求体。Schema 的作用是确保请求格式明确。

典型请求如下：

```json
{
  "messages": [
    {"role": "user", "content": "你好"}
  ]
}
```

Schema 校验的价值：

- 防止字段缺失。
- 防止类型错误。
- 生成 OpenAPI 文档。
- 让后续 Agent 逻辑收到稳定结构。

生产级项目不能让任意字典直接进入核心逻辑，否则错误会在很深的地方才爆发。

## 8. 第五步：触发会话自动命名

在聊天接口中，如果配置打开：

```text
SESSION_NAMING_ENABLED=true
```

项目会调用：

```text
maybe_name_session(...)
```

这一步的作用是根据用户第一条消息自动生成会话标题。

设计重点是：它不阻塞主聊天链路，而是后台处理。这样用户不会因为标题生成额外等待。

这是生产级体验优化：非关键任务异步化，主链路只保留必须步骤。

## 9. 第六步：调用 LangGraphAgent

路由层最终调用：

```text
agent.get_response(
    chat_request.messages,
    session.id,
    user_id=str(session.user_id),
    username=session.username,
)
```

这里传入了四类关键信息：

- `messages`：用户本轮消息。
- `session.id`：作为 LangGraph 的 thread_id。
- `user_id`：用于长期记忆隔离。
- `username`：用于 Prompt 个性化。

这一步是从 HTTP 世界进入 Agent 世界的边界。

## 10. 第七步：构造 LangGraph config

在 `app/core/langgraph/graph.py` 的 `get_response` 中，会构造 `RunnableConfig`。

其中最关键的是：

```text
configurable.thread_id = session_id
metadata.user_id = user_id
metadata.username = username
metadata.session_id = session_id
```

`thread_id` 的意义非常大：它决定 LangGraph 从哪里读取和保存当前会话状态。

生产级 Agent 必须把会话状态和用户身份绑定起来，否则多用户、多会话时会混乱。

## 11. 第八步：并发读取状态和长期记忆

`get_response` 中有一个重要设计：

```text
graph.aget_state(config)
memory_service.search(user_id, messages[-1].content)
```

这两个操作通过 `asyncio.gather` 并发执行。

它们分别做什么？

- **`aget_state`**：检查当前 thread 是否已有状态，是否处于中断等待。
- **`memory.search`**：根据用户最新问题检索相关长期记忆。

为什么要并发？因为两个操作互不依赖。并发可以减少 200～500ms 延迟，这在聊天体验中很重要。

## 12. 第九步：判断是否恢复中断状态

LangGraph 支持中断和恢复。如果当前状态中存在 `state.next`，说明 Graph 正在等待用户补充输入。

此时项目会使用：

```text
Command(resume=messages[-1].content)
```

恢复执行。

如果没有中断状态，则正常构造新的 graph input：

```text
{
  "messages": dump_messages(messages),
  "long_term_memory": relevant_memory
}
```

这就是 Agent 平台比普通聊天接口更复杂的地方：它不只是一次请求一次响应，还要支持状态化工作流。

## 13. 第十步：进入 LangGraph 的 chat 节点

Graph 的入口节点是 `chat`。

`chat` 节点会：

1. 从 config 中读取 username 和 thread_id。
2. 加载系统 Prompt。
3. 将长期记忆注入 Prompt。
4. 准备消息列表。
5. 调用 LLM Service。
6. 判断 LLM 是否返回工具调用。
7. 返回 `Command` 控制下一步。

如果 LLM 返回 `tool_calls`，下一步是：

```text
goto = "tool_call"
```

否则：

```text
goto = END
```

## 14. 第十一步：可能进入 tool_call 节点

如果模型要求调用工具，Graph 会进入 `tool_call` 节点。

该节点会：

1. 读取上一条 AIMessage 中的工具调用。
2. 根据工具名称找到对应工具。
3. 调用工具的 `ainvoke`。
4. 把结果包装成 `ToolMessage`。
5. 返回 `Command(update=..., goto="chat")`。

也就是说，工具结果不会直接返回给用户，而是再次交给 LLM，让模型结合工具结果生成自然语言回答。

## 15. 第十二步：最终响应处理

Graph 执行完成后，项目会把 LangChain 消息转换成 OpenAI 风格消息，再筛选出用户和助手消息。

处理逻辑在：

```text
LangGraphAgent.__process_messages
```

然后路由层返回：

```text
ChatResponse(messages=result)
```

响应给客户端。

## 16. 第十三步：后台写入长期记忆

在得到最终响应后，项目会启动后台任务：

```text
memory_service.add(user_id, openai_msgs, metadata)
```

这一步不会阻塞用户响应。

生产价值：

- 用户快速拿到回答。
- 系统仍然能在后台更新长期记忆。
- 后续请求可以使用新记忆。

## 17. 完整链路图

```text
Client
  -> FastAPI app
  -> Middleware / Metrics / Logging
  -> /api/v1/chatbot/chat
  -> Rate Limit
  -> get_current_session
  -> ChatRequest validation
  -> maybe_name_session
  -> LangGraphAgent.get_response
  -> graph.aget_state + memory.search
  -> chat node
  -> LLMService.call
  -> tool_call node? -> chat node?
  -> Checkpointer saves state
  -> memory.add background task
  -> ChatResponse
  -> Client
```

## 18. 来源与依据

本篇内容主要依据当前项目以下文件：

- **README.md**：说明项目包含 LangGraph、长期记忆、LLM 服务、认证、观测等能力。
- **docs/architecture.md**：提供系统概览和请求生命周期图。
- **app/main.py**：FastAPI 应用入口、lifespan、预热逻辑、中间件注册。
- **app/api/v1/chatbot.py**：普通聊天、流式聊天、消息管理接口实现。
- **app/api/v1/auth.py**：`get_current_session` 认证依赖。
- **app/core/langgraph/graph.py**：`get_response`、`_chat`、`_tool_call`、Graph 执行逻辑。
- **app/services/memory.py**：长期记忆检索和后台写入。
- **app/core/config.py**：限流、LLM、数据库、记忆等配置来源。

## 19. 依据示例

### 19.1 聊天接口依据

`app/api/v1/chatbot.py` 中的 `chat` 函数证明：请求会通过 Session 认证后调用 `agent.get_response`。

示例逻辑：

```text
session = Depends(get_current_session)
agent.get_response(messages, session.id, user_id, username)
```

### 19.2 Graph 执行依据

`app/core/langgraph/graph.py` 中的 `get_response` 证明：项目会并发读取 Graph 状态和长期记忆。

示例逻辑：

```text
asyncio.gather(graph.aget_state(config), memory_service.search(...))
```

### 19.3 工具循环依据

`_chat` 节点中根据 `AIMessage.tool_calls` 判断是否进入 `tool_call`，`_tool_call` 执行工具后回到 `chat`。

这就是当前项目 Agent Loop 的直接代码依据。

## 20. 本篇总结

一次聊天请求不是简单调用模型。它是一条完整生产链路：

- FastAPI 接入。
- 限流保护。
- Session 认证。
- Schema 校验。
- Agent 调用。
- 状态读取。
- 长期记忆检索。
- LLM 推理。
- 工具循环。
- 状态持久化。
- 后台记忆更新。
- 响应返回。

理解这条链路，是学习当前项目的核心基础。

## 21. 本篇练习

1. 打开 `app/api/v1/chatbot.py`，找出普通聊天接口的完整函数。
2. 找出它依赖的 `get_current_session` 在哪个文件。
3. 打开 `graph.py`，找到 `get_response` 中的 `asyncio.gather`。
4. 找到 `_chat` 中判断 `tool_calls` 的逻辑。
5. 找到 `_tool_call` 中执行工具的逻辑。
6. 用自己的话画出一次聊天请求的完整链路。
