# 05 API 协议设计：普通响应、流式响应与消息管理

Agent 平台不是只有模型调用，还必须把能力设计成稳定的 API 协议。API 协议决定前端如何调用、如何展示、如何处理错误、如何管理会话历史。

这一篇讲当前项目的聊天 API 设计，包括普通聊天、流式聊天、消息查询、消息清空，以及为什么生产级 Agent 平台需要这些接口。

## 1. 为什么 Agent 需要 API 协议设计

如果只是本地脚本，你可以直接调用一个 Python 函数。但真实产品中，Agent 通常被前端页面、移动端、业务系统或第三方服务调用。

这就要求 Agent 后端提供稳定协议：

- 请求路径稳定。
- 请求体结构稳定。
- 响应结构稳定。
- 错误格式可理解。
- 支持不同交互模式。
- 支持历史消息管理。

API 协议是 Agent 平台和外部世界的契约。

## 2. 当前项目有哪些聊天相关接口

聊天接口集中在：

```text
app/api/v1/chatbot.py
```

主要包括：

- `POST /api/v1/chatbot/chat`：普通聊天。
- `POST /api/v1/chatbot/chat/stream`：流式聊天。
- `GET /api/v1/chatbot/messages`：获取当前 Session 消息历史。
- `DELETE /api/v1/chatbot/messages`：清空当前 Session 消息历史。

这四个接口覆盖了一个聊天产品最基础的交互闭环：发送消息、接收回答、查看历史、清理历史。

## 3. 普通聊天接口

普通聊天接口适合一次性返回完整结果。

请求路径：

```text
POST /api/v1/chatbot/chat
```

请求体示例：

```json
{
  "messages": [
    {"role": "user", "content": "请解释什么是 LangGraph"}
  ]
}
```

响应体是 `ChatResponse`，其中包含消息列表。

普通响应的优点：

- 实现简单。
- 客户端处理方便。
- 适合短回答。
- 适合系统间调用。

普通响应的缺点：

- 用户必须等完整回答生成后才能看到内容。
- 如果模型耗时较长，体验较差。

## 4. 流式聊天接口

流式聊天接口适合边生成边返回。

请求路径：

```text
POST /api/v1/chatbot/chat/stream
```

响应类型：

```text
text/event-stream
```

当前项目使用 Server-Sent Events 格式，每个 chunk 形如：

```text
data: {"content": "一段增量内容", "done": false}
```

结束时发送：

```text
data: {"content": "", "done": true}
```

流式响应的优点：

- 首 token 延迟更低。
- 用户可以看到模型正在生成。
- 长回答体验更好。
- 更接近 ChatGPT 类产品体验。

流式响应的缺点：

- 客户端处理更复杂。
- 错误处理更复杂。
- 不适合所有系统间调用。

## 5. 为什么同时保留普通接口和流式接口

生产级平台通常需要同时提供两种模式。

普通接口适合：

- 后台任务。
- 自动化调用。
- 测试脚本。
- 第三方系统集成。
- 响应较短的场景。

流式接口适合：

- Web 聊天界面。
- 移动端聊天界面。
- 长文生成。
- 需要实时反馈的场景。

当前项目同时支持两者，说明它不是只服务单一 Demo，而是考虑了不同客户端需求。

## 6. 消息查询接口

接口：

```text
GET /api/v1/chatbot/messages
```

它用于获取当前 Session 的历史消息。

为什么需要这个接口？

- 用户刷新页面后需要恢复历史。
- 前端需要展示已有对话。
- 调试时需要确认状态是否保存。
- 产品上需要支持会话列表和历史回看。

当前项目不是从某个普通消息表读取历史，而是通过 LangGraph state 读取当前 thread 的状态，再转换成用户和助手消息。

这说明历史消息和 Agent Checkpoint 是绑定的。

## 7. 消息清空接口

接口：

```text
DELETE /api/v1/chatbot/messages
```

它用于清空当前 Session 的聊天状态。

为什么需要清空？

- 用户希望重新开始。
- 测试时需要重置上下文。
- 长会话可能导致上下文污染。
- 某些业务需要删除敏感会话。

当前项目清空的是 LangGraph Checkpoint 相关表中当前 `thread_id` 的记录。

## 8. API 为什么依赖 Session Token

所有聊天相关接口都依赖：

```text
session: Session = Depends(get_current_session)
```

也就是说，客户端不能直接传任意 `session_id`。

这样设计的好处：

- 防止用户访问别人的会话。
- 防止伪造会话 ID。
- 确保每个请求都能绑定 user_id。
- 让日志、记忆、Checkpoint 都有一致上下文。

生产级 Agent API 必须把安全边界放在协议层，而不是只靠前端约束。

## 9. Pydantic Schema 的作用

API 协议不仅是 URL，还包括请求和响应结构。

当前项目使用 Pydantic Schema 定义：

- `ChatRequest`
- `ChatResponse`
- `StreamResponse`
- `UserResponse`
- `SessionResponse`
- `TokenResponse`

Schema 的生产价值：

- 自动校验请求。
- 自动生成 Swagger 文档。
- 明确字段类型。
- 防止响应泄漏内部字段。
- 为前后端协作提供契约。

## 10. 错误处理设计

聊天接口中捕获异常后，会：

1. 使用 `logger.exception` 记录错误和堆栈。
2. 返回 `HTTPException(status_code=500, detail=str(e))`。

流式接口内部还会把错误封装成一个 `StreamResponse`，通过 SSE 返回给客户端。

这样设计的原因是：流式响应一旦开始，HTTP 状态码可能已经发出，后续错误需要通过流内容表达。

## 11. API 层不应该做什么

API 层不应该直接：

- 调用 OpenAI SDK。
- 拼接复杂 Prompt。
- 操作 LangGraph 节点细节。
- 直接写大量数据库逻辑。
- 处理长期记忆细节。

这些应该交给 Agent 层和 Service 层。

当前项目中的 `chatbot.py` 很好地保持了边界：它负责接入和调用 `agent.get_response`，而不是自己实现 Agent 流程。

## 12. 普通聊天和流式聊天的内部差异

普通聊天调用：

```text
agent.get_response(...)
```

流式聊天调用：

```text
agent.get_stream_response(...)
```

前者等待完整 Graph 执行结果，后者使用 LangGraph `astream` 持续产出 token。

这说明 API 协议差异会影响 Agent 层方法设计。

## 13. 生产级 API 设计原则

从当前项目可以总结出几个原则：

- **协议稳定**：路径和 Schema 明确。
- **认证前置**：业务逻辑前先校验 Session。
- **限流前置**：高成本接口必须受保护。
- **响应可预测**：普通响应和流式响应结构明确。
- **职责分离**：API 层只负责接入，不承担核心 Agent 逻辑。
- **错误可观测**：异常必须记录结构化日志。
- **支持恢复**：提供历史消息接口。
- **支持重置**：提供清空消息接口。

## 14. 来源与依据

本篇依据当前项目以下文件：

- **app/api/v1/chatbot.py**：四个聊天相关接口的实现依据。
- **app/schemas/chat.py**：聊天请求、响应、流式响应的数据结构依据。
- **app/api/v1/auth.py**：Session 依赖认证依据。
- **app/core/config.py**：聊天接口限流配置依据。
- **docs/getting-started.md**：注册、创建 Session、聊天调用流程依据。
- **README.md**：项目支持 stateful conversations、streaming、auth、rate limiting 的依据。

## 15. 依据示例

### 15.1 普通聊天接口

`app/api/v1/chatbot.py` 中 `@router.post("/chat", response_model=ChatResponse)` 说明普通聊天返回结构化 JSON 响应。

### 15.2 流式聊天接口

`StreamingResponse(event_generator(), media_type="text/event-stream")` 说明流式接口采用 SSE。

### 15.3 限流依据

`@limiter.limit(settings.RATE_LIMIT_ENDPOINTS["chat"][0])` 说明聊天接口使用配置化限流。

### 15.4 Session 认证依据

`session: Session = Depends(get_current_session)` 说明聊天接口必须通过 Session Token 认证。

## 16. 本篇总结

API 协议是 Agent 平台对外服务的边界。当前项目提供了普通聊天、流式聊天、消息查询、消息清空四类核心接口，并通过 Schema、认证、限流、错误处理保证协议稳定。

普通接口适合简单调用，流式接口适合聊天体验，消息接口保证会话可恢复和可重置。理解 API 协议设计，才能理解 Agent 后端如何真正服务产品。

## 17. 本篇练习

1. 打开 `app/api/v1/chatbot.py`，列出所有路由路径。
2. 找到普通聊天接口的 response_model。
3. 找到流式接口返回的 media_type。
4. 找到 `StreamResponse` 的字段定义。
5. 解释为什么聊天接口不能直接接受用户传入的 `session_id`。
6. 思考：如果要新增“重新生成上一条回答”接口，应该使用普通响应还是流式响应？为什么？
