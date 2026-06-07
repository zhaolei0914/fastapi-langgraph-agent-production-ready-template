# 03 项目目录结构与分层设计

学习生产级项目时，目录结构就是地图。你如果看不懂目录结构，就会在代码里迷路：不知道请求从哪里进来，不知道业务逻辑放在哪里，不知道 Agent 图在哪里，不知道数据库模型和 API Schema 有什么区别。

这一篇会带你从零理解当前项目的分层设计。重点不是背目录，而是理解每一层为什么存在、解决什么问题、如何互相协作。

## 1. 为什么生产项目必须分层

一个小 Demo 可以把所有代码写在一个文件里：

- 路由。
- Prompt。
- 模型调用。
- 工具函数。
- 数据库操作。
- 配置。

这样写启动很快，但很快会失控。

当项目变大后，你会遇到这些问题：

- 改一个接口会影响 Agent 逻辑。
- 改一个模型调用会影响业务接口。
- 数据库代码散落在各处。
- Prompt 和工具注册没有统一位置。
- 配置写死在代码里。
- 日志和错误处理风格不一致。
- 新人不知道应该在哪里加代码。

分层设计的目的就是让每类代码各归其位。

## 2. 当前项目的总体结构

项目核心结构如下：

```text
app/
  api/v1/          # HTTP 路由层
  core/            # 核心基础设施
    langgraph/     # Agent 图与工具系统
    prompts/       # 系统 Prompt
    config.py      # 配置管理
    cache.py       # 缓存
    logging.py     # 日志
    limiter.py     # 限流
    metrics.py     # 指标
    middleware.py  # 中间件
  models/          # 数据库模型
  schemas/         # Pydantic 数据结构
  services/        # 业务服务层
  utils/           # 通用工具函数
alembic/           # 数据库迁移
docs/              # 文档
evals/             # 评估框架
scripts/           # 脚本
```

如果用一句话概括：

```text
api 接请求，schemas 定协议，services 做业务，core 提供基础设施，models 映射数据库，langgraph 编排 Agent。
```

## 3. 第一层：应用入口 `app/main.py`

`app/main.py` 是 FastAPI 应用入口。它不是业务逻辑文件，而是应用装配文件。

它主要负责：

- 加载环境变量。
- 初始化 Langfuse。
- 创建 FastAPI app。
- 设置应用生命周期。
- 初始化缓存服务。
- 预热 LangGraph Agent。
- 预热 Memory Service。
- 注册中间件。
- 注册路由。
- 设置 Prometheus metrics。
- 处理全局异常。

为什么入口文件不应该写业务逻辑？

因为入口层应该只负责“把系统组装起来”。如果把聊天逻辑、数据库逻辑、工具逻辑都塞进 `main.py`，项目很快会变得不可维护。

## 4. 第二层：API 路由层 `app/api/v1`

API 层负责 HTTP 协议相关的事情。

主要文件包括：

```text
app/api/v1/api.py
app/api/v1/auth.py
app/api/v1/chatbot.py
```

### 4.1 `api.py`

通常负责汇总和注册各个子路由。它让 `main.py` 不需要知道每个业务路由的细节。

### 4.2 `auth.py`

负责认证相关接口，例如：

- 注册用户。
- 登录。
- 创建 Session。
- 查询 Session。
- 删除 Session。
- 修改 Session 名称。
- 提供 `get_current_session` 等依赖。

### 4.3 `chatbot.py`

负责聊天相关接口，例如：

- 普通聊天。
- 流式聊天。
- 查询消息历史。
- 清空消息历史。

API 层的重点是：

- 接收请求。
- 校验依赖。
- 调用服务或 Agent。
- 返回响应。
- 做接口级日志和异常处理。
- 加限流装饰器。

它不应该承担复杂业务编排。

## 5. 第三层：Schema 层 `app/schemas`

Schema 层定义数据结构，主要使用 Pydantic。

它解决的问题是：系统内部和外部到底传递什么格式的数据。

例如聊天请求需要定义：

- 消息 role。
- 消息 content。
- 请求体结构。
- 响应体结构。

Graph State 也需要 Schema，因为 LangGraph 的状态不能随意用字典乱传。状态结构清晰，才能保证节点之间协作稳定。

Schema 层的价值：

- 自动参数校验。
- 自动生成 API 文档。
- 降低字典字段拼错的风险。
- 给 pyright 等类型检查工具提供信息。
- 让接口协议稳定。

生产级系统中，Schema 不是形式主义，而是接口契约。

## 6. 第四层：Model 层 `app/models`

Model 层定义数据库表结构，当前项目使用 SQLModel。

Schema 和 Model 很容易混淆，但它们职责不同：

- **Schema**：面向 API 和内部数据传输。
- **Model**：面向数据库表结构和 ORM 映射。

例如 User Model 可能包含：

- 用户 ID。
- 邮箱。
- 密码哈希。
- 用户名。
- 创建时间。

但 API 返回给前端时绝不能返回密码哈希，所以需要单独的响应 Schema。

这就是为什么生产项目通常区分 Model 和 Schema。

## 7. 第五层：Service 层 `app/services`

Service 层负责业务能力封装。

当前项目的重要服务包括：

```text
app/services/database.py
app/services/memory.py
app/services/session_naming.py
app/services/llm/
```

### 7.1 Database Service

负责用户、Session 等数据库操作。

API 层不应该直接到处写 SQL 或 ORM 查询。把数据库操作集中到 Service 层，可以让逻辑更清晰，也更容易测试和复用。

### 7.2 Memory Service

负责长期记忆：

- 初始化 mem0。
- 搜索用户相关记忆。
- 缓存搜索结果。
- 后台写入新记忆。

关键文件：

```text
app/services/memory.py
```

### 7.3 LLM Service

负责模型调用可靠性：

- 模型注册。
- 调用模型。
- 重试。
- fallback。
- 超时控制。
- 工具绑定。
- 结构化输出。

关键目录：

```text
app/services/llm/
```

### 7.4 Session Naming Service

负责后台生成会话标题。

为什么这是单独服务？因为它是一个独立业务能力：当用户第一次发消息时，主聊天响应不应该被标题生成阻塞，所以项目把标题生成放到后台任务中。

## 8. 第六层：Core 层 `app/core`

`core` 放的是项目基础设施，不是具体业务。

主要包括：

```text
app/core/config.py
app/core/cache.py
app/core/logging.py
app/core/limiter.py
app/core/metrics.py
app/core/middleware.py
app/core/observability.py
app/core/prompts/
app/core/langgraph/
```

### 8.1 Config

`config.py` 读取环境变量，形成统一配置对象。

所有模块都通过配置对象读取设置，而不是直接到处 `os.getenv`。这样可以让配置逻辑集中、可维护。

### 8.2 Logging

`logging.py` 配置 structlog。

生产环境中，日志要能被机器检索和聚合，所以当前项目使用结构化日志，而不是随意 `print`。

### 8.3 Limiter

`limiter.py` 配置 slowapi 限流。

Agent 接口通常成本较高，因为每次请求都可能调用大模型。限流可以防止滥用，也能保护系统稳定性。

### 8.4 Metrics

`metrics.py` 定义 Prometheus 指标。

生产系统不能只知道“能不能访问”，还要知道请求量、延迟、错误率、模型耗时。

### 8.5 Middleware

`middleware.py` 负责请求级基础能力，例如：

- 绑定日志上下文。
- 记录请求指标。
- 性能分析。

中间件的好处是这些能力可以自动作用于所有请求，而不用每个路由重复写。

## 9. 第七层：LangGraph 层 `app/core/langgraph`

这是 Agent 编排的核心位置。

主要包括：

```text
app/core/langgraph/graph.py
app/core/langgraph/tools/
```

### 9.1 `graph.py`

这是当前项目最重要的文件之一。它定义了 `LangGraphAgent`，负责：

- 初始化 LLM Service 并绑定工具。
- 创建 PostgreSQL 连接池。
- 创建 `StateGraph`。
- 定义 `chat` 节点。
- 定义 `tool_call` 节点。
- 编译 graph。
- 处理普通响应。
- 处理流式响应。
- 读取历史消息。
- 清空 Checkpoint。

### 9.2 `tools/`

这里放可被 LLM 调用的工具。

生产级 Agent 的工具系统不能随便散落在业务代码中，必须集中注册、统一暴露给模型。

## 10. Prompt 层 `app/core/prompts`

Prompt 是 Agent 行为的重要组成部分，但它不应该硬编码在业务函数里。

当前项目把系统 Prompt 放在：

```text
app/core/prompts/system.md
```

并通过代码加载、格式化后注入：

- 用户名。
- 当前时间。
- 长期记忆。

这样设计的好处是：

- Prompt 可以独立维护。
- 不需要改业务代码就能调整 Agent 行为。
- Prompt 变量来源清晰。
- 更容易做 Prompt 版本管理和评估。

## 11. Utils 层 `app/utils`

Utils 放通用工具函数，例如消息转换、文本提取、响应处理等。

注意：Utils 不应该变成“垃圾桶”。只有真正跨模块复用、没有明确业务归属的函数，才适合放到这里。

如果一个函数只服务于 Memory，就放 Memory Service。
如果一个函数只服务于 LLM，就放 LLM Service。
如果一个函数只服务于 API，就放 API 层或对应 Service。

## 12. Alembic 迁移目录

`alembic/` 负责数据库迁移。

为什么迁移单独放？

因为数据库结构是生产系统的一部分，需要版本管理。你不能靠手动执行 SQL 来维护不同环境的表结构。

迁移让你可以：

- 在本地创建表。
- 在测试环境复现结构。
- 在生产环境安全升级。
- 回溯数据库变更历史。

## 13. Evals 评估目录

`evals/` 是 Agent 项目中非常重要但容易被忽略的部分。

传统后端主要用单元测试判断正确性。但 Agent 输出具有概率性，需要评估体系持续判断质量。

当前项目的评估框架可以：

- 从 Langfuse 拉取 traces。
- 用指标 Prompt 评估回答。
- 使用 LLM Judge 打分。
- 生成 JSON 报告。

这说明当前项目不只关心“能回答”，也关心“回答质量是否可持续改进”。

## 14. Docs 文档目录

`docs/` 是项目说明文档，包括：

- 启动说明。
- 架构说明。
- 配置说明。
- 认证说明。
- 数据库说明。
- LLM 服务说明。
- 记忆系统说明。
- 可观测性说明。
- 评估说明。
- Docker 部署说明。

现在新增的 `docs/zh/` 是中文学习教程，目标是帮助你从零理解这些设计。

## 15. Scripts 脚本目录

`scripts/` 通常放自动化脚本，例如环境准备、构建、部署辅助命令等。

生产项目中，脚本可以减少人为操作错误。例如：

- 初始化环境。
- 构建镜像。
- 执行检查。
- 启动服务。

## 16. 一次请求如何穿过这些层

以普通聊天接口为例：

```text
Client
  -> app/main.py 注册的 FastAPI 应用
  -> middleware 处理日志、metrics、request_id
  -> app/api/v1/chatbot.py 路由
  -> get_current_session 校验 Session
  -> ChatRequest Schema 校验请求体
  -> LangGraphAgent.get_response
  -> MemoryService.search 检索长期记忆
  -> LangGraph StateGraph 执行 chat 节点
  -> LLMService.call 调用模型
  -> 如有工具调用，进入 tool_call 节点
  -> Checkpointer 保存状态
  -> MemoryService.add 后台写入长期记忆
  -> ChatResponse Schema 返回响应
```

这条链路就是项目分层协作的最好例子。

## 17. 为什么这种分层适合生产级 Agent

### 17.1 清晰职责

每层负责自己的事情，降低耦合。

### 17.2 易于扩展

想新增工具，只改 `langgraph/tools`。
想换模型策略，只改 `services/llm`。
想加接口，只改 `api/v1` 和相关 Schema/Service。
想改 Prompt，只改 `core/prompts`。

### 17.3 易于测试

Service 可以单独测试，Schema 可以单独校验，API 可以集成测试，Agent 可以用 evals 评估。

### 17.4 易于排查问题

如果请求失败，可以按层排查：

- 是 HTTP 参数错？
- 是认证失败？
- 是数据库失败？
- 是 Memory 失败？
- 是 LLM 超时？
- 是工具异常？
- 是 Graph 状态异常？

### 17.5 易于多人协作

后端工程师可以维护 API 和数据库，AI 工程师可以维护 Prompt、Graph、Tool，平台工程师可以维护部署和监控。

## 18. 初学者读代码的推荐顺序

不要按文件名随机读。建议按这条路径：

### 第一步：入口和路由

```text
app/main.py
app/api/v1/api.py
app/api/v1/chatbot.py
```

先知道请求从哪里进入。

### 第二步：请求和响应结构

```text
app/schemas/
```

理解接口传什么、返回什么。

### 第三步：Agent 核心

```text
app/core/langgraph/graph.py
```

理解 LangGraph 如何执行。

### 第四步：LLM 和 Memory

```text
app/services/llm/
app/services/memory.py
```

理解模型调用和长期记忆。

### 第五步：认证和数据库

```text
app/api/v1/auth.py
app/services/database.py
app/models/
```

理解多用户和多会话。

### 第六步：生产能力

```text
app/core/logging.py
app/core/middleware.py
app/core/metrics.py
app/core/limiter.py
app/core/cache.py
```

理解日志、监控、限流、缓存。

## 19. 你以后应该把新代码放在哪里

### 19.1 新增 HTTP 接口

放在：

```text
app/api/v1/
```

同时定义请求/响应 Schema。

### 19.2 新增数据库表

放在：

```text
app/models/
```

并创建 Alembic 迁移。

### 19.3 新增业务逻辑

放在：

```text
app/services/
```

### 19.4 新增 Agent 工具

放在：

```text
app/core/langgraph/tools/
```

并注册到工具列表。

### 19.5 修改 Agent 人设

修改：

```text
app/core/prompts/system.md
```

### 19.6 新增配置项

修改：

```text
app/core/config.py
.env.example
```

### 19.7 新增评估指标

放在：

```text
evals/metrics/prompts/
```

## 20. 常见错误理解

### 20.1 “Agent 逻辑都应该写在 API 里”

不对。API 只负责接入，Agent 逻辑应该在 LangGraph 层和 Service 层。

### 20.2 “Schema 和 Model 是一回事”

不对。Schema 是接口和数据传输结构，Model 是数据库结构。

### 20.3 “Prompt 写在 Python 字符串里更方便”

短期方便，长期难维护。生产项目应该把 Prompt 独立管理。

### 20.4 “工具函数哪里用就写哪里”

不对。Agent 工具要统一注册，才能稳定暴露给 LLM。

### 20.5 “日志用 print 就够了”

不对。生产环境需要结构化日志和上下文。

## 21. 本篇总结

当前项目采用清晰的分层设计：

- `main.py` 负责应用装配。
- `api/v1` 负责 HTTP 接入。
- `schemas` 负责数据契约。
- `models` 负责数据库映射。
- `services` 负责业务能力。
- `core` 负责基础设施。
- `core/langgraph` 负责 Agent 编排。
- `core/prompts` 负责 Prompt 管理。
- `evals` 负责质量评估。
- `docs` 负责文档说明。

理解目录结构，本质上是在理解生产级 Agent 平台的职责边界。

下一篇会进入一次聊天请求的完整链路，从 HTTP 请求开始，一步步追到 LangGraph、LLM、Memory 和 Response。

## 22. 本篇练习

请完成以下练习：

1. 打开 `app/main.py`，列出它完成了哪些应用装配工作。
2. 打开 `app/api/v1/chatbot.py`，找出普通聊天和流式聊天接口。
3. 找到聊天请求和响应对应的 Schema。
4. 打开 `app/core/langgraph/graph.py`，找到 `chat` 节点和 `tool_call` 节点。
5. 打开 `app/services/memory.py`，找出 `search` 和 `add` 方法。
6. 打开 `app/services/llm/service.py`，找出 LLM 调用入口方法。
7. 用自己的话解释 Schema 和 Model 的区别。
8. 思考：如果你要新增一个“订单查询工具”，应该在哪些目录中添加或修改代码？

完成这些练习后，你就具备了继续深入请求链路和 LangGraph 机制的基础。
