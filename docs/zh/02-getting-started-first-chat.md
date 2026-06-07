# 02 从 0 启动项目并完成第一次对话

学习一个生产级框架，最容易犯的错误是直接钻进复杂源码。更好的方式是先让系统跑起来，完成一次真实请求，然后再回头理解每个模块为什么存在。

这一篇会带你从 0 启动当前项目，并完成第一次完整对话。重点不是死记命令，而是理解每一步背后的意义：为什么要配置环境变量，为什么需要数据库，为什么要先注册用户，为什么聊天前要创建 Session，为什么生产级 Agent 后端不能绕过这些步骤。

## 1. 本篇你会学到什么

读完这一篇，你应该能够：

- 理解启动这个项目需要哪些基础依赖。
- 理解 `.env.development` 的作用。
- 理解 Docker 启动和本地 Python 启动的区别。
- 理解用户、Session、聊天请求之间的关系。
- 完成一次从注册到聊天的完整 API 调用。
- 知道启动失败时应该优先检查哪些地方。

## 2. 启动项目前先理解整体依赖

当前项目不是一个单文件 Demo，它依赖多个外部能力：

- **Python 3.13+**：运行 FastAPI 应用和 Agent 逻辑。
- **uv**：管理 Python 依赖。
- **Docker / Docker Compose**：快速启动 PostgreSQL 等基础设施。
- **PostgreSQL**：保存用户、Session、Checkpoint、长期记忆。
- **pgvector**：支持向量检索，用于长期记忆。
- **OpenAI API Key**：调用 LLM 和 Embedding。
- **Langfuse**：可选，用于 LLM Trace。

你可能会问：为什么学习 Agent 还要数据库和 Docker？

原因是这个项目面向生产级框架，而生产级 Agent 必须处理状态、用户、记忆和部署。如果只调用一次模型，可以不需要这些；但如果要给真实用户使用，它们就是基础设施。

## 3. 环境变量为什么重要

生产项目不能把配置写死在代码里。例如：

- 数据库地址在本地和线上不同。
- API Key 不能提交到 Git。
- 开发环境可以打开 Debug，生产环境不能随便打开。
- 不同环境可能使用不同模型。

所以当前项目使用环境变量管理配置。

配置入口在：

```text
app/core/config.py
```

文档在：

```text
docs/configuration.md
```

你需要从示例文件复制一份开发环境配置：

```bash
cp .env.example .env.development
```

然后至少填写：

```text
OPENAI_API_KEY=你的 OpenAI Key
JWT_SECRET_KEY=一个足够长的随机字符串
```

如果你暂时不想配置 Langfuse，可以设置：

```text
LANGFUSE_TRACING_ENABLED=false
```

## 4. 推荐启动方式：Docker

对于零基础学习者，推荐优先使用 Docker。因为 Docker 可以帮你少处理很多数据库安装和 pgvector 配置问题。

标准流程是：

```bash
make install
make docker-up
make migrate
```

这三步分别做什么？

### 4.1 `make install`

它安装 Python 依赖，并准备开发工具。当前项目使用 `uv` 管理依赖，所以你不需要手动逐个安装包。

### 4.2 `make docker-up`

它启动 API 和 PostgreSQL。PostgreSQL 对这个项目非常关键，因为它不仅保存业务数据，还保存 LangGraph Checkpoint 和长期记忆向量。

### 4.3 `make migrate`

它执行 Alembic 数据库迁移。迁移的作用是创建和更新数据库表结构。

如果没有迁移，应用即使启动了，也可能因为缺少表而无法注册用户或保存 Session。

## 5. 另一种方式：本地 Python 启动

如果你已经有本地 PostgreSQL，也可以不用 Docker 启动应用。

流程是：

```bash
make install
make migrate
make dev
```

但是你需要确保 `.env.development` 里的数据库配置正确：

```text
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=food_order_db
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
```

对于初学者，如果你不确定数据库是否配置正确，优先用 Docker。

## 6. 打开 API 文档

项目启动后，访问：

```text
http://localhost:8000/docs
```

这是 FastAPI 自动生成的 Swagger UI。你可以在这里看到所有接口，也可以直接调试。

生产级后端应该有清晰的 API 文档。FastAPI 的优势之一就是自动从路由和 Pydantic Schema 生成接口文档。

## 7. 第一次完整调用为什么要分三步

很多人会疑惑：为什么不能直接调用 `/chat`？为什么还要注册用户和创建 Session？

因为当前项目是多用户、多会话的 Agent 平台，不是单用户脚本。

一次聊天请求至少需要回答两个问题：

- 这个用户是谁？
- 这条消息属于哪个会话？

所以完整流程是：

1. 注册或登录，得到 User Token。
2. 创建聊天 Session，得到 Session Token。
3. 使用 Session Token 调用聊天接口。

这就是生产级 Agent 平台和玩具 Demo 的关键区别。

## 8. 第一步：注册用户

接口：

```text
POST /api/v1/auth/register
```

示例请求：

```bash
curl -X POST http://localhost:8000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "username": "you"}'
```

返回结果中会包含用户信息和 token。

这里的 `username` 不只是展示字段。当前项目会把用户名传入 Agent 的系统 Prompt，让模型知道当前用户的名字。

## 9. 第二步：创建 Session

接口：

```text
POST /api/v1/auth/session
```

示例请求：

```bash
curl -X POST http://localhost:8000/api/v1/auth/session \
  -H "Authorization: Bearer <user token>"
```

返回结果包含：

- `session_id`
- session 级别的 `token`

注意：后续聊天接口要使用这个 Session Token，而不是 User Token。

## 10. 为什么要区分 User Token 和 Session Token

User Token 表示“你是谁”。Session Token 表示“你正在访问哪个会话”。

这样设计有几个好处：

- 一个用户可以有多个会话。
- 每个会话都有独立上下文。
- Chat 接口不需要每次传 `session_id` 参数。
- LangGraph 可以直接使用 `session.id` 作为 `thread_id`。
- 删除或清空某个会话时，不影响其他会话。

在当前项目中，聊天接口通过依赖注入获得当前 Session：

```text
app/api/v1/chatbot.py
```

其中 `get_current_session` 会校验 Session Token。

## 11. 第三步：普通聊天

接口：

```text
POST /api/v1/chatbot/chat
```

示例请求：

```bash
curl -X POST http://localhost:8000/api/v1/chatbot/chat \
  -H "Authorization: Bearer <session token>" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "你好，请介绍一下你自己"}]}'
```

这个请求会进入完整 Agent 链路：

```text
FastAPI Router -> Session 校验 -> LangGraphAgent -> Memory Search -> LLM -> Response
```

如果模型需要工具，还会进入工具调用循环。

## 12. 第四步：流式聊天

接口：

```text
POST /api/v1/chatbot/chat/stream
```

示例请求：

```bash
curl -X POST http://localhost:8000/api/v1/chatbot/chat/stream \
  -H "Authorization: Bearer <session token>" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "用三句话解释 LangGraph"}]}'
```

流式响应适合前端逐字展示。大模型生成回答可能需要几秒，如果等完整回答再返回，用户会感觉卡顿。流式输出可以显著改善体验。

当前项目使用 FastAPI 的 `StreamingResponse` 和 Server-Sent Events 返回流式内容。

核心文件：

```text
app/api/v1/chatbot.py
app/core/langgraph/graph.py
```

## 13. 查询历史消息

接口：

```text
GET /api/v1/chatbot/messages
```

示例请求：

```bash
curl -X GET http://localhost:8000/api/v1/chatbot/messages \
  -H "Authorization: Bearer <session token>"
```

这个接口读取当前 Session 对应的 LangGraph 状态，并返回用户和助手消息。

为什么能读到历史？因为当前项目使用 Checkpointer 把会话状态保存到了 PostgreSQL。

## 14. 清空历史消息

接口：

```text
DELETE /api/v1/chatbot/messages
```

示例请求：

```bash
curl -X DELETE http://localhost:8000/api/v1/chatbot/messages \
  -H "Authorization: Bearer <session token>"
```

它会清理当前 Session 对应的 Checkpoint 表数据。

当前项目会处理这些表：

```text
checkpoint_blobs
checkpoint_writes
checkpoints
```

这说明 LangGraph 的状态不是简单存在某个 Python 变量里，而是持久化到了数据库中。

## 15. 启动时应用做了什么

很多初学者只关注接口调用，但生产项目启动时也很重要。

当前项目的启动逻辑在：

```text
app/main.py
```

FastAPI `lifespan` 中会做几件事：

- 初始化 cache service。
- 预热 LangGraph agent。
- 创建 graph 和数据库连接池。
- 预热 memory service。
- 应用关闭时释放缓存和数据库连接。

为什么要预热？

如果第一次用户请求才初始化 graph、连接数据库、检查 memory schema，那么第一个用户会遇到明显延迟。生产系统通常会在启动阶段完成这些准备工作。

## 16. 为什么需要数据库迁移

数据库结构会随着项目迭代变化。例如：

- 增加用户字段。
- 增加 Session 名称。
- 新增索引。
- 调整表结构。

如果手动改数据库，很容易造成环境不一致。Alembic 迁移的作用是让数据库结构版本化。

所以启动后执行：

```bash
make migrate
```

不是可选步骤，而是生产工程的基本习惯。

## 17. 常见启动问题

### 17.1 API 启动失败

优先检查：

- `.env.development` 是否存在。
- `OPENAI_API_KEY` 是否填写。
- `JWT_SECRET_KEY` 是否填写。
- 端口 8000 是否被占用。

### 17.2 数据库连接失败

优先检查：

- Docker 是否启动。
- PostgreSQL 容器是否运行。
- `POSTGRES_HOST`、`POSTGRES_PORT` 是否正确。
- 是否执行过迁移。

### 17.3 Memory 没有效果

优先检查：

- OpenAI Key 是否可用。
- pgvector 是否启用。
- 是否有用户 ID。
- 是否已经发生过足够的对话让 mem0 抽取记忆。

### 17.4 Langfuse 报错

如果只是本地学习，可以先关闭：

```text
LANGFUSE_TRACING_ENABLED=false
```

## 18. 生产级启动流程背后的设计思想

你会发现当前项目的启动流程比一个普通 Demo 复杂。但这些复杂度不是无意义的，而是在为生产环境做准备。

### 18.1 配置外置

环境变量让应用可以在不同环境使用不同配置，而不用改代码。

### 18.2 数据持久化

PostgreSQL 让用户、Session、状态、记忆都不会因为服务重启而丢失。

### 18.3 认证先行

所有聊天请求都绑定用户和 Session，避免数据混乱。

### 18.4 预热关键服务

减少第一次请求延迟。

### 18.5 文档自动生成

Swagger UI 让前后端协作更容易。

这些都是生产级框架需要具备的基本素养。

## 19. 本篇总结

这一篇你完成了从 0 到第一次对话的流程，也理解了背后的设计原因：

- 环境变量用于管理不同环境配置和敏感信息。
- Docker 降低本地启动数据库和 pgvector 的复杂度。
- Alembic 保证数据库结构可迁移。
- 用户注册和 Session 创建保证多用户、多会话隔离。
- 普通聊天和流式聊天对应不同用户体验。
- Checkpoint 让历史消息可以持久化。
- 启动预热减少第一次请求延迟。

下一篇我们会进入目录结构，学习当前项目为什么要这样分层，以及每个目录在生产级 Agent 平台中承担什么职责。

## 20. 本篇练习

请完成以下练习：

1. 创建 `.env.development`，并找出其中最关键的 5 个配置项。
2. 启动项目并打开 `http://localhost:8000/docs`。
3. 完成注册用户、创建 Session、普通聊天三个接口调用。
4. 再调用一次流式聊天接口，观察响应格式有什么不同。
5. 调用 `GET /messages`，确认历史消息存在。
6. 调用 `DELETE /messages`，再查询一次历史消息。
7. 打开 `app/main.py`，找出启动时预热了哪些服务。

完成这些练习后，你就不仅仅是“跑起来了项目”，而是理解了生产级 Agent 后端启动流程的基本逻辑。
