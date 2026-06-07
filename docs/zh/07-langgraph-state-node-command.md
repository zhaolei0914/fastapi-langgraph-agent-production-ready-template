# 07 LangGraph 基础：State、Node、Edge、Command

LangGraph 是当前项目 Agent 编排的核心。要理解这个项目，必须先理解 LangGraph 的几个基础概念：State、Node、Edge、Command、Checkpointer、Thread。

这一篇不会泛泛介绍 LangGraph，而是结合当前项目的代码讲：这些概念为什么存在，在生产级 Agent 中解决什么问题。

## 1. 为什么需要 LangGraph

一个简单聊天程序可以写成：

```text
用户输入 -> 调用模型 -> 返回结果
```

但 Agent 不止如此。真实 Agent 可能需要：

- 维护多轮消息。
- 检索长期记忆。
- 判断是否调用工具。
- 执行工具。
- 将工具结果再交给模型。
- 支持中断和恢复。
- 保存状态到数据库。

这已经不是线性函数，而是状态机或图。

LangGraph 的作用就是把 Agent 流程建模成一个有状态图。

## 2. State 是什么

State 是图执行过程中的共享状态。

当前项目的 State 定义在：

```text
app/schemas/graph.py
```

核心结构是：

```text
GraphState
  messages
  long_term_memory
```

其中：

- `messages`：当前对话消息。
- `long_term_memory`：检索到的长期记忆文本。

这说明当前 Agent 的核心上下文由“短期消息 + 长期记忆”组成。

## 3. 为什么 State 不能随便用 dict

初学者可能会想：直接传 dict 不就行了吗？

生产级项目不建议这样做，原因是：

- 字段容易拼错。
- 节点之间约定不清晰。
- 类型检查困难。
- 以后扩展状态时容易混乱。
- 文档和代码无法形成稳定契约。

当前项目使用 Pydantic `BaseModel` 定义 `GraphState`，让状态结构清晰可维护。

## 4. messages 为什么使用 add_messages

`GraphState.messages` 使用了 LangGraph 的 `add_messages` 注解。

这表示当节点返回新的 messages 时，不是简单覆盖原来的消息，而是按消息语义追加或合并。

Agent 对话状态和普通变量不同。每个节点可能产生新消息：

- 用户消息。
- AI 消息。
- 工具消息。

`add_messages` 帮助 LangGraph 正确管理消息列表。

## 5. Node 是什么

Node 是图中的一个执行单元。

在当前项目中，最核心的两个节点是：

- `chat`
- `tool_call`

它们定义在：

```text
app/core/langgraph/graph.py
```

`chat` 节点负责调用 LLM。

`tool_call` 节点负责执行工具。

Node 的设计让 Agent 流程被拆成可理解、可测试、可扩展的步骤。

## 6. chat 节点做什么

`chat` 节点对应方法：

```text
LangGraphAgent._chat
```

它主要做：

1. 从 config 读取 username 和 thread_id。
2. 加载系统 Prompt。
3. 把长期记忆注入 Prompt。
4. 准备消息。
5. 调用 LLM Service。
6. 处理响应。
7. 判断是否有工具调用。
8. 返回 Command。

这个节点体现了 Agent 的“大脑”部分。

## 7. tool_call 节点做什么

`tool_call` 节点对应方法：

```text
LangGraphAgent._tool_call
```

它主要做：

1. 从最后一条 AI 消息中读取工具调用。
2. 根据工具名找到工具函数。
3. 执行工具。
4. 把结果封装成 `ToolMessage`。
5. 回到 `chat` 节点。

这个节点体现了 Agent 的“行动”部分。

## 8. Edge 是什么

Edge 表示节点之间的连接关系。

传统图会显式定义边，但当前项目更多通过 `Command(goto=...)` 控制流转。

概念上可以理解为：

```text
chat -> tool_call
chat -> END
tool_call -> chat
```

也就是：

- LLM 要工具：去工具节点。
- LLM 不要工具：结束。
- 工具执行完：回到 LLM。

## 9. Command 是什么

`Command` 是 LangGraph 中用于控制图流转的重要对象。

它通常包含：

- `update`：更新 State。
- `goto`：指定下一个节点。
- `resume`：恢复中断。

当前项目中，`chat` 节点返回：

```text
Command(update={"messages": [response_message]}, goto=goto)
```

`tool_call` 节点返回：

```text
Command(update={"messages": outputs}, goto="chat")
```

这意味着节点不仅产出数据，还决定流程走向。

## 10. 为什么使用 Command 而不是 if/else 调函数

你可以用普通 Python 写：

```text
if has_tool_call:
    call_tool()
    chat_again()
else:
    return result
```

但生产级 Agent 会越来越复杂：

- 多节点。
- 中断恢复。
- 状态持久化。
- 并发节点。
- 动态路由。
- 人工审批。
- 多 Agent 协作。

使用 Graph 和 Command 可以让流程可视化、可持久化、可扩展，而不是散落在嵌套 if/else 中。

## 11. Thread 是什么

Thread 是 LangGraph 用来区分不同状态线的标识。

当前项目使用：

```text
session_id
```

作为：

```text
thread_id
```

所以每个聊天 Session 都有独立 Graph 状态。

没有 Thread，所有用户的对话状态可能混在一起。

## 12. Checkpointer 是什么

Checkpointer 负责保存 Graph 状态。

当前项目使用：

```text
AsyncPostgresSaver
```

将状态保存到 PostgreSQL。

这让 Agent 具备：

- 多轮会话持久化。
- 服务重启后恢复。
- 中断续跑。
- 多实例共享状态。

这是生产级 Agent 和本地 Demo 的关键差异。

## 13. RunnableConfig 是什么

`RunnableConfig` 是 LangGraph/LangChain 执行时配置。

当前项目使用它传递：

- `thread_id`
- callbacks
- user_id
- username
- session_id
- environment
- debug

这些信息不是 State 的核心业务内容，但对执行、追踪、隔离、观测很重要。

可以理解为：

- State 是 Agent 正在处理的业务上下文。
- Config 是这次执行的运行上下文。

## 14. State 和 Config 的区别

这是初学者容易混淆的点。

State 中放的是图执行要读写的状态，例如：

- messages
- long_term_memory

Config 中放的是运行时元信息，例如：

- thread_id
- callbacks
- user_id
- username
- session_id

如果某个数据要被节点更新和持久化，通常放 State。
如果某个数据只是执行上下文，通常放 Config。

## 15. 当前项目 Graph 创建流程

Graph 创建发生在：

```text
LangGraphAgent.create_graph
```

主要步骤：

1. 创建 `StateGraph(GraphState)`。
2. 添加 `chat` 节点。
3. 添加 `tool_call` 节点。
4. 设置入口节点为 `chat`。
5. 创建 PostgreSQL connection pool。
6. 创建 `AsyncPostgresSaver`。
7. 编译 graph。

编译后得到 `CompiledStateGraph`，后续请求就可以调用它。

## 16. 为什么启动时预热 Graph

`app/main.py` 的 lifespan 中会调用：

```text
agent.create_graph()
```

这是预热。

好处：

- 提前创建 Graph。
- 提前建立数据库连接池。
- 提前设置 Checkpointer。
- 降低第一次聊天请求延迟。

生产系统中，冷启动延迟会影响第一个用户体验，因此常常在应用启动时预热关键组件。

## 17. 来源与依据

本篇依据当前项目以下文件：

- **app/schemas/graph.py**：`GraphState` 定义，包含 `messages` 和 `long_term_memory`。
- **app/core/langgraph/graph.py**：`LangGraphAgent`、`_chat`、`_tool_call`、`create_graph`、`get_response`。
- **docs/architecture.md**：Agent 图结构 `chat -> tool_call -> chat` 和 Checkpointer 说明。
- **app/main.py**：启动时预热 Graph。
- **README.md**：项目包含 stateful agent、checkpointing、tool calling、人机中断支持等能力。

## 18. 依据示例

### 18.1 GraphState 依据

`app/schemas/graph.py` 中：

```text
messages: Annotated[list, add_messages]
long_term_memory: str
```

说明当前 Agent 状态由消息和长期记忆构成。

### 18.2 Node 依据

`graph.py` 中存在 `_chat` 和 `_tool_call` 两个方法，并在 `create_graph` 中通过 `add_node` 注册。

### 18.3 Command 依据

`_chat` 和 `_tool_call` 都返回 `Command`，说明节点通过 Command 更新状态并控制下一步。

### 18.4 Checkpointer 依据

`create_graph` 中使用 `AsyncPostgresSaver(connection_pool)`，说明状态持久化到 PostgreSQL。

## 19. 本篇总结

LangGraph 的核心思想是用图管理 Agent 流程，用 State 管理上下文，用 Command 控制流转，用 Thread 隔离会话，用 Checkpointer 持久化状态。

当前项目将这些概念落地为：

- `GraphState`：消息和长期记忆。
- `chat` 节点：调用 LLM。
- `tool_call` 节点：执行工具。
- `Command`：控制下一步。
- `session_id`：作为 thread_id。
- `AsyncPostgresSaver`：保存状态。

## 20. 本篇练习

1. 打开 `app/schemas/graph.py`，解释 `GraphState` 两个字段。
2. 打开 `graph.py`，找到 `StateGraph(GraphState)`。
3. 找到 `add_node("chat", ...)` 和 `add_node("tool_call", ...)`。
4. 找到 `_chat` 返回 `Command` 的位置。
5. 找到 `_tool_call` 返回 `Command` 的位置。
6. 用自己的话解释 State 和 Config 的区别。
7. 思考：如果要增加“人工审批”节点，应该增加 State 字段还是 Config 字段？为什么？
