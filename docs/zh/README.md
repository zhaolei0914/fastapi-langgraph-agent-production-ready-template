# 生产级 Agent 平台开发中文教程

这是一套面向零基础学习者的中文长文教程，目标不是只教你“怎么调用一次大模型”，而是带你理解一个生产级 Agent 后端为什么需要这些模块、这些模块解决什么问题、在当前项目中如何落地，以及你应该如何逐步练习到能独立开发 Agent 平台。

当前项目是一个基于 FastAPI、LangGraph、PostgreSQL、pgvector、mem0、Langfuse、Prometheus、JWT、Docker 的生产级 Agent 后端模板。它不只是一个 Demo，而是把真实业务上线会遇到的会话状态、长期记忆、工具调用、流式输出、认证、限流、观测、评估、部署等能力都放进了同一个框架中。

## 适合谁阅读

- **Python 初学者**：希望从 Web API、数据库、配置、日志开始建立后端工程基础。
- **FastAPI 使用者**：希望理解如何把 FastAPI 扩展成生产级 AI 后端。
- **LangGraph 学习者**：希望理解 State、Node、Command、Checkpointer、Thread 等概念在真实项目中的用法。
- **Agent 开发者**：希望从简单 Agent Demo 进阶到可上线、可观测、可评估、可扩展的 Agent 平台。
- **后端工程师**：希望系统掌握 LLM 应用在生产环境中的架构设计。

## 学习方法

建议不要一开始就跳进 `app/core/langgraph/graph.py`。生产级 Agent 框架不是一个函数，也不是一个 Prompt，而是一组能力组合：

1. 先理解项目解决什么问题。
2. 再理解一次请求从 HTTP 到 Agent 再到数据库的完整链路。
3. 再逐个拆解 Agent 图、工具、记忆、模型服务。
4. 最后学习认证、限流、日志、监控、评估、部署这些生产能力。

每篇教程都尽量按下面结构讲解：

- **这一篇解决什么问题**
- **如果没有这个设计会发生什么**
- **核心原理是什么**
- **当前项目如何实现**
- **生产环境为什么需要它**
- **你应该怎么练习**

## 18 篇教程规划

### 第一阶段：建立整体认知

1. [为什么需要生产级 Agent 后端框架](./01-why-production-agent-framework.md)
2. [从 0 启动项目并完成第一次对话](./02-getting-started-first-chat.md)
3. [项目目录结构与分层设计](./03-project-structure-and-layering.md)

### 第二阶段：理解一次请求的生命线

4. [一次聊天请求的完整链路：从 FastAPI 到 LangGraph](./04-chat-request-lifecycle.md)
5. [API 协议设计：普通响应、流式响应与消息管理](./05-api-protocol-design.md)
6. [认证与 Session：多用户、多会话隔离机制](./06-auth-and-session-isolation.md)

### 第三阶段：掌握 LangGraph Agent 核心

7. [LangGraph 基础：State、Node、Edge、Command](./07-langgraph-state-node-command.md)
8. [当前 Agent 图详解：chat 与 tool_call 双节点循环](./08-current-agent-graph-chat-tool-loop.md)
9. Loop 控制机制：工具调用如何驱动多轮推理
10. Checkpoint 与 Thread：Agent 状态如何持久化

### 第四阶段：掌握 Agent 能力模块

11. Prompt 管理：系统提示词、用户信息与动态上下文
12. 工具系统：如何定义、注册、调用和并发执行工具
13. 长期记忆系统：mem0、pgvector 与个性化上下文
14. LLM 服务：重试、fallback、超时与结构化输出

### 第五阶段：走向生产级平台

15. 配置、数据库与迁移：让项目适配多环境
16. 可观测性：日志、Langfuse、Prometheus 与 Grafana
17. 评估体系：如何判断 Agent 是否真的变好了
18. 从模板到产品：扩展业务 Agent 的完整实践路线

## 当前已完成

- 第 1 篇：为什么需要生产级 Agent 后端框架
- 第 2 篇：从 0 启动项目并完成第一次对话
- 第 3 篇：项目目录结构与分层设计
- 第 4 篇：一次聊天请求的完整链路：从 FastAPI 到 LangGraph
- 第 5 篇：API 协议设计：普通响应、流式响应与消息管理
- 第 6 篇：认证与 Session：多用户、多会话隔离机制
- 第 7 篇：LangGraph 基础：State、Node、Edge、Command
- 第 8 篇：当前 Agent 图详解：chat 与 tool_call 双节点循环

## 你最终应该掌握什么

读完并练完这套教程后，你应该能够：

- **理解架构**：说清楚 FastAPI、LangGraph、LLM Service、Memory Service、PostgreSQL、Langfuse 分别负责什么。
- **读懂代码**：能从一个 API 入口一路追踪到 Agent 图执行和状态保存。
- **改造 Agent**：能修改 Prompt、添加工具、扩展业务 API、调整模型策略。
- **处理生产问题**：知道为什么要限流、重试、fallback、日志、监控、评估。
- **设计平台**：能基于当前模板设计客服 Agent、数据分析 Agent、个人助理 Agent 或多 Agent 系统。
