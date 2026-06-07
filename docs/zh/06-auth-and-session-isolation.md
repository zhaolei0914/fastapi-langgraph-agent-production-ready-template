# 06 认证与 Session：多用户、多会话隔离机制

Agent 平台一旦面向真实用户，就必须解决身份和隔离问题。谁在使用？他有哪些会话？这个请求属于哪个会话？长期记忆应该属于谁？这些问题都不能靠前端约定，必须在后端架构中解决。

这一篇讲当前项目的认证与 Session 设计。

## 1. 为什么 Agent 平台需要认证

如果没有认证，系统会出现严重问题：

- 用户可以访问别人的会话。
- 长期记忆可能混入不同用户数据。
- 日志无法定位具体用户。
- 无法做用户级限流和审计。
- 无法删除或管理个人会话。
- 生产环境无法满足基本安全要求。

Agent 往往会处理大量个人化上下文，安全隔离比普通应用更重要。

## 2. 当前项目的认证流程

当前项目采用 JWT 认证，并区分两类 token：

- **User Token**：表示用户身份。
- **Session Token**：表示某个聊天会话身份。

流程是：

```text
注册/登录 -> 得到 User Token
User Token -> 创建 Session -> 得到 Session Token
Session Token -> 调用聊天接口
```

这不是多余设计，而是为了把“用户身份”和“对话状态”分开。

## 3. User 和 Session 的关系

一个用户可以拥有多个 Session。

Session 模型定义在：

```text
app/models/session.py
```

关键字段包括：

- `id`：Session 主键。
- `user_id`：关联用户。
- `name`：会话名称。
- `username`：创建会话时复制的用户显示名。

这说明 Session 是独立实体，不只是前端里的一个临时变量。

## 4. 为什么要有 Session

没有 Session 的 Agent 会遇到这些问题：

- 所有对话混在一起。
- 用户无法开启多个话题。
- 无法独立清空某一段上下文。
- LangGraph 无法知道该用哪个 thread。
- 历史消息无法精确恢复。

Session 的作用是把一次连续对话变成可管理的资源。

## 5. Session 和 LangGraph Thread 的关系

当前项目把：

```text
session.id
```

作为 LangGraph 的：

```text
configurable.thread_id
```

这意味着：

- 每个 Session 对应一条 LangGraph 状态线。
- Checkpointer 根据 thread_id 保存状态。
- 获取历史消息时也根据 thread_id 读取状态。
- 清空消息时清理该 thread_id 对应的 checkpoint。

这是当前项目会话隔离的核心设计。

## 6. get_current_user 做什么

`get_current_user` 定义在：

```text
app/api/v1/auth.py
```

它用于需要用户身份的接口，例如创建 Session、查询 Session 列表。

它会：

1. 读取 Bearer Token。
2. 清洗 token。
3. 验证 JWT。
4. 得到 user_id。
5. 查询数据库确认用户存在。
6. 绑定日志上下文。
7. 返回 User 对象。

这个依赖确保用户级操作必须由合法用户发起。

## 7. get_current_session 做什么

聊天接口依赖的是：

```text
get_current_session
```

它也定义在：

```text
app/api/v1/auth.py
```

它会：

1. 读取 Session Token。
2. 验证 JWT。
3. 从 token 的 `sub` 中得到 session_id。
4. 查询数据库确认 Session 存在。
5. 绑定 `user_id` 到日志上下文。
6. 返回 Session 对象。

这让聊天接口天然知道：

- 当前会话 ID。
- 当前用户 ID。
- 当前用户名。

## 8. JWT 中保存了什么

JWT 创建逻辑在：

```text
app/utils/auth.py
```

`create_access_token` 会把传入的 thread_id 写入：

```text
sub
```

还会写入：

- `exp`：过期时间。
- `iat`：签发时间。
- `jti`：唯一 token id。

User Token 的 `sub` 是用户 ID，Session Token 的 `sub` 是 Session ID。

这也是为什么同一个 `verify_token` 可以服务两类 token，区别在于调用方如何解释 `sub`。

## 9. 为什么聊天接口使用 Session Token

聊天接口需要的不只是“用户是谁”，还要知道“当前对话是哪一个”。

如果只使用 User Token，每次请求还必须额外传 `session_id`，这会带来风险：

- 用户可能传错 Session ID。
- 前端可能把别的 Session ID 发过来。
- 后端还要额外验证 Session 是否属于当前用户。
- API 使用更复杂。

使用 Session Token 后，Session ID 已经在签名 token 中，后端可以直接信任验证后的结果。

## 10. 长期记忆如何隔离

长期记忆通过 `user_id` 隔离，而不是通过 Session ID。

原因是：

- Session 是短期会话。
- 记忆是跨会话的用户长期上下文。
- 同一个用户的不同 Session 应该共享长期偏好。
- 不同用户之间绝对不能共享记忆。

在聊天请求中，`session.user_id` 会传给：

```text
memory_service.search(user_id, query)
memory_service.add(user_id, messages, metadata)
```

这保证记忆按用户命名空间存储和检索。

## 11. username 为什么复制到 Session

Session 模型中有一个字段：

```text
username
```

它是在创建 Session 时从用户信息复制过来的。

这样做的好处是：聊天请求读取 Session 时就能拿到用户名，不需要每次再查 User 表。

生产价值：

- 减少数据库查询。
- 降低请求延迟。
- 保持 Prompt 个性化。
- 避免高频聊天接口额外 DB 压力。

`docs/architecture.md` 中也提到：用户名通过 Session 流转，而不是每次请求查数据库。

## 12. Session 自动命名

当前项目还支持 Session 自动命名。

聊天请求到来时，如果 Session 名称为空，项目会根据第一条用户消息生成标题。

这属于产品体验能力：

- 用户能更容易识别历史会话。
- 不影响主聊天响应。
- 后台异步执行，降低主链路延迟。

## 13. 安全设计细节

当前项目在认证中还做了这些安全处理：

- 使用 bcrypt 哈希密码。
- JWT 使用 `JWT_SECRET_KEY` 签名。
- Token 有过期时间。
- Token 包含 `jti`。
- 对 token 和 session_id 做字符串清洗。
- 注册和登录接口有限流。
- 输入字段有校验和清洗。

这些都是生产级认证系统的基本要求。

## 14. 常见错误设计

### 14.1 所有人共用一个会话

这会导致上下文混乱和数据泄漏。

### 14.2 只用 user_id，不用 session_id

用户无法同时维护多个话题。

### 14.3 前端直接传 session_id

容易被篡改，后端必须额外验证。

### 14.4 长期记忆按 session_id 存

这样新会话无法继承用户偏好。

### 14.5 每次聊天都查用户表

高频接口不必要地增加数据库负担。

## 15. 来源与依据

本篇依据当前项目以下文件：

- **docs/authentication.md**：认证流程、User Token、Session Token、端点说明。
- **app/api/v1/auth.py**：`get_current_user`、`get_current_session`、注册、登录、Session 管理实现。
- **app/utils/auth.py**：JWT 创建和验证逻辑。
- **app/models/session.py**：Session 数据模型。
- **app/api/v1/chatbot.py**：聊天接口依赖 `get_current_session`。
- **app/core/langgraph/graph.py**：Session ID 作为 `thread_id` 传入 LangGraph。
- **docs/architecture.md**：用户名通过 Session 流转、Session 标题后台生成等设计说明。

## 16. 依据示例

### 16.1 Session 模型依据

`app/models/session.py` 中定义了 `id`、`user_id`、`name`、`username`，说明 Session 是数据库实体。

### 16.2 聊天依赖依据

`app/api/v1/chatbot.py` 中聊天函数参数包含：

```text
session: Session = Depends(get_current_session)
```

说明聊天接口必须先解析 Session。

### 16.3 JWT 依据

`app/utils/auth.py` 中 `create_access_token` 把传入 ID 写入 `sub`，`verify_token` 从 `sub` 取回 ID。

### 16.4 LangGraph Thread 依据

`app/core/langgraph/graph.py` 中 config 包含：

```text
configurable: {"thread_id": session_id}
```

说明 Session ID 是 Graph 状态隔离依据。

## 17. 本篇总结

当前项目的认证与 Session 设计解决了生产级 Agent 的核心问题：多用户、多会话、长期记忆和状态隔离。

- User Token 解决用户身份。
- Session Token 解决当前会话身份。
- Session ID 映射 LangGraph thread_id。
- user_id 隔离长期记忆。
- username 通过 Session 流转，减少数据库查询。

理解这套机制后，你才能真正理解为什么聊天接口不是简单的 `/chat`。

## 18. 本篇练习

1. 打开 `app/api/v1/auth.py`，比较 `get_current_user` 和 `get_current_session`。
2. 打开 `app/models/session.py`，解释每个字段的作用。
3. 打开 `app/utils/auth.py`，找出 JWT 的 `sub` 字段。
4. 打开 `app/api/v1/chatbot.py`，找出 Session 如何传给 Agent。
5. 思考：如果一个用户有 5 个 Session，LangGraph 如何区分它们？
6. 思考：为什么长期记忆按 user_id 隔离，而不是按 session_id 隔离？
