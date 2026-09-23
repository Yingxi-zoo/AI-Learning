# Model Context Protocol（MCP）深度学习

**版本基准：MCP 2025-11-25 → MCP 2026-07-28；Python SDK v1 → v2**

> 文中“v1/v2”主要指 Python SDK 的编程模型；协议本身应使用官方日期版本号，例如 `2025-11-25` 和 `2026-07-28`。两套版本号不能混用。

## 摘要

Model Context Protocol（MCP）是连接 AI 应用与外部工具、数据源及服务的开放协议。它不规定大语言模型如何推理，而是规定 AI 应用如何以统一方式发现能力、读取上下文、调用工具，并在不同传输方式和部署环境中保持一致的交互语义。

MCP 建立在 JSON-RPC 2.0 消息模型之上，主要由以下几层组成：

```text
┌──────────────────────────────────────┐
│              MCP primitives          │
│       Tools / Resources / Prompts    │
├──────────────────────────────────────┤
│       Lifecycle / Capabilities       │
│       Errors / Authorization         │
├──────────────────────────────────────┤
│              JSON-RPC 2.0             │
│       Request / Response / Notify    │
├──────────────────────────────────────┤
│              Transport                │
│       stdio / Streamable HTTP        │
└──────────────────────────────────────┘
```

从应用架构看，MCP 解决的是“能力连接”问题：

```text
Host（AI 应用、IDE、Agent）
  └── Client（协议连接与调用）
        └── MCP Server
              ├── Tools      可执行动作
              ├── Resources  可读取上下文
              └── Prompts    可复用提示模板
```

2026-07-28 版本的核心方向，是把协议核心进一步调整为更适合现代 HTTP 基础设施的请求驱动模型：减少对长期连接状态和反向请求通道的依赖，增强路由、缓存、扩展和长任务能力。Python SDK v2 则在实现层提供更高层的 `Client`、`MCPServer` 和新的服务器 API。

# 1. MCP 要解决什么问题

## 1.1 没有 MCP 时的连接问题

一个 AI 应用如果需要使用数据库、文件系统、搜索服务、支付系统和企业内部 API，传统做法通常是为每个服务单独编写：

- 认证和连接代码；
- 参数描述和工具定义；
- 错误处理；
- 上下文格式转换；
- 不同模型或 Agent 框架的适配器。

服务数量增加后，连接关系容易变成“点对点集成”：

```text
AI 应用 A ── 自定义适配 ── 数据库
AI 应用 A ── 自定义适配 ── 搜索服务
AI 应用 B ── 另一套适配 ── 数据库
AI 应用 B ── 另一套适配 ── 搜索服务
```

MCP 的目标是把能力描述和调用方式标准化，使服务提供方实现一次 MCP Server，多个支持 MCP 的 Host 或 Agent 都可以使用它。

## 1.2 MCP 不是什么

MCP 不是：

- 模型本身；
- Agent 编排框架；
- 数据库协议；
- 传输层协议的单一实现；
- 自动保证工具调用安全的系统。

MCP 提供协议约束，但权限、身份、数据脱敏、审批和业务审计仍然需要由 Host、Server 以及部署基础设施共同完成。

## 1.3 MCP 的基本价值

MCP 主要提供四种统一能力：

1. **发现**：Client 可以列出 Server 暴露的工具、资源和提示模板。
2. **描述**：能力带有名称、说明、参数 schema 和结果信息。
3. **调用**：Client 通过统一的 JSON-RPC 方法执行操作。
4. **演进**：协议可通过 capabilities、extensions 和版本协商逐步扩展。

# 2. MCP 的整体架构

## 2.1 Host、Client 和 Server

```text
┌──────────────────────────────────────┐
│                Host                  │
│  AI 应用 / IDE / Agent / 自动化平台   │
│                                      │
│   ┌──────────┐      ┌──────────┐     │
│   │ Client 1 │      │ Client 2 │ ... │
│   └────┬─────┘      └────┬─────┘     │
└────────┼──────────────────┼───────────┘
         │ MCP              │ MCP
         ▼                  ▼
   Server A             Server B
```

|角色|职责|
|---|---|
|Host|承载用户体验、模型、Agent 循环和多个 Client 的应用|
|Client|Host 内与某个 MCP Server 建立协议连接的组件|
|Server|向 Client 暴露 Tools、Resources、Prompts 等能力|

一个 Host 可以同时管理多个 Client，每个 Client 通常对应一个 Server。Server 不需要知道模型内部如何推理，只需要提供清晰、可调用的能力。

## 2.2 一次典型调用

```text
1. Host 创建 Client
2. Client 与 Server 完成 initialize
3. Client 请求 tools/list
4. Host 将工具描述交给模型
5. 模型产生工具调用意图
6. Client 发送 tools/call
7. Server 执行业务逻辑
8. Server 返回结构化结果或错误
9. Host 将结果放回模型上下文
```

MCP 只负责第 2、3、6、7、8 步的协议交互，以及相关的生命周期和能力声明；模型是否选择工具、如何组合工具，是 Host 或 Agent 层的职责。

# 3. MCP 的消息与调用机制

## 3.1 JSON-RPC 2.0 是消息基础

MCP 使用 JSON-RPC 2.0 的三类消息：

|消息|是否有 `id`|是否需要响应|用途|
|---|---:|---:|---|
|Request|是|是|请求另一方执行方法|
|Response|对应 Request|否|返回成功结果或错误|
|Notification|否|否|发送单向事件或状态通知|

一个请求的基本形式：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}
```

成功响应：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": []
  }
}
```

失败响应：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Invalid params"
  }
}
```

Notification 不带 `id`，例如能力变化通知：

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

## 3.2 MCP 方法

常见 MCP 方法包括：

```text
tools/list
tools/call

resources/list
resources/read
resources/templates/list

prompts/list
prompts/get

initialize
notifications/initialized
```

协议方法由 MCP 规范定义；具体 SDK 可能将它们包装成不同的 Python 方法名。

## 3.3 初始化与能力协商

传统生命周期通常包括：

```text
Client                         Server
  │                              │
  │ initialize                   │
  │ capabilities + version      │
  │ ───────────────────────────> │
  │                              │
  │ initialize result            │
  │ <─────────────────────────── │
  │                              │
  │ notifications/initialized    │
  │ ───────────────────────────> │
```

初始化阶段的作用不是登录业务系统，而是协商：

- 协议版本；
- 双方支持的 capabilities；
- Client 或 Server 的实现信息；
- 后续允许使用的协议特性。

## 3.4 请求、业务结果与协议错误

调用工具时需要区分三类情况：

1. **传输错误**：连接中断、HTTP 错误、进程退出。
2. **协议错误**：方法不存在、参数结构错误、请求状态不合法。
3. **业务错误**：工具本身执行了，但业务条件不满足，例如余额不足或文件不存在。

业务错误可以作为工具结果返回，并通过 `isError` 或 SDK 对应字段表达；协议错误通常位于 JSON-RPC 的 `error` 对象中。两者不能混为“工具调用失败”后直接丢弃，因为 Host 可能需要向模型展示不同的恢复策略。

# 4. MCP Server 的核心 Primitives

MCP Server 主要暴露三种 primitives：Tools、Resources、Prompts。

## 4.1 Tools：模型可以执行的动作

Tool 是可被模型或 Host 调用的动作，例如：

- 查询订单；
- 创建日历事件；
- 执行 SQL；
- 搜索文档；
- 修改配置。

一个工具通常包括：

|字段|作用|
|---|---|
|`name`|稳定的工具名称|
|`description`|帮助模型理解何时使用|
|`inputSchema`|输入参数的 JSON Schema|
|`outputSchema`|可选的结构化输出描述|

列出工具：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list",
  "params": {}
}
```

调用工具：

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "add",
    "arguments": {
      "a": 10,
      "b": 20
    }
  }
}
```

工具设计应尽量做到：

- 输入字段语义明确；
- 对副作用进行说明；
- 对权限边界进行说明；
- 返回稳定的结构化结果；
- 业务失败时给出可行动的错误信息。

## 4.2 Resources：可读取的上下文

Resource 表示 Server 提供给 Client 读取的上下文，例如：

- 文件；
- 数据库记录；
- 文档；
- API 返回的数据；
- 动态生成的报告。

资源通常由 URI 标识：

```text
file:///workspace/readme.md
db://customers/123
https://example.com/document/abc
```

读取资源：

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "resources/read",
  "params": {
    "uri": "file:///workspace/readme.md"
  }
}
```

Resources 与 Tools 的区别是：Tool 表达“执行动作”，Resource 表达“读取上下文”。有些系统可以同时提供二者，例如通过 Tool 搜索资源，再通过 Resource 读取具体内容。

## 4.3 Prompts：可复用的提示模板

Prompt 是 Server 提供的可复用提示模板。它可以包含参数和消息角色，用于将领域知识或工作流模板化。

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "method": "prompts/get",
  "params": {
    "name": "summarize_document",
    "arguments": {
      "style": "academic"
    }
  }
}
```

Prompt 不等于模型回复。它只是为 Host 或用户提供一组结构化消息，最终如何提交给模型由 Host 决定。

## 4.4 三种 primitives 的关系

```text
Resource  ──提供上下文──> Host / Model
Tool      ──执行动作────> 外部系统
Prompt    ──提供模板────> Host / Model
```

一个完整的 Agent 工作流可能是：

```text
Prompt：生成代码审查任务
  ↓
Resource：读取代码和规范
  ↓
Tool：运行测试或查询缺陷系统
  ↓
Resource：读取测试报告
  ↓
Model：综合生成结果
```

# 5. Transport 与部署方式

Transport 只解决“消息如何到达”，不改变 Tools、Resources 和 Prompts 的语义。

## 5.1 stdio

stdio 适合本地进程模型：

```text
Host
 ├── Client
 └── 启动 Server 子进程
       ├── stdin  接收消息
       └── stdout 发送消息
```

特点：

- 启动简单；
- 不需要监听端口；
- 适合本地工具、桌面应用和开发环境；
- 进程生命周期通常由 Host 管理；
- Server 的日志应写到 stderr，避免污染 stdout 上的协议消息。

## 5.2 Streamable HTTP

Streamable HTTP 面向远程或服务化部署。它允许 MCP 消息通过 HTTP 请求传输，并可使用流式响应承载长响应或事件。

典型结构：

```text
Host / Client
      │ HTTP
      ▼
反向代理 / 网关 / 负载均衡
      │
      ▼
MCP Server
```

它更容易接入：

- TLS；
- 身份认证；
- 反向代理；
- 负载均衡；
- 访问日志和审计；
- 云平台和容器平台。

## 5.3 SSE 的定位

早期远程 MCP 部署常使用 SSE。随着 Streamable HTTP 成为主要方向，SSE 应视为 legacy 兼容方式，而不是新系统的默认设计。已有 SSE Server 仍可能需要继续维护，但新项目应优先评估 Streamable HTTP。

## 5.4 Transport 选择

|场景|建议|
|---|---|
|本地桌面工具|stdio|
|本地开发和调试|stdio 或 Streamable HTTP|
|远程服务|Streamable HTTP|
|需要网关、审计和扩缩容|Streamable HTTP|
|兼容旧客户端|保留 SSE 适配层|

# 6. MCP 协议与 Python SDK 的演进

## 6.1 两条版本演进线

### 协议版本线

协议使用日期作为版本标识：

```text
MCP 2025-11-25  ───────────────>  MCP 2026-07-28
```

它描述的是 wire protocol 的能力、消息、生命周期、Transport 和扩展规则。

### Python SDK 版本线

SDK 使用软件版本号：

```text
Python SDK v1  ────────────────>  Python SDK v2
```

它描述 Python 实现的 API、抽象层、类型、辅助函数和开发体验。

因此下面两句话含义不同：

- “协议从 2025-11-25 演进到 2026-07-28”：讨论通信规范。
- “Python SDK 从 v1 演进到 v2”：讨论 Python 开发接口。

SDK v2 可以实现和兼容多个协议版本；协议新版本也不等于所有语言 SDK 同时升级为“v2”。

## 6.2 总体变化概览

|特点|MCP 2025-11-25 / SDK v1 常见模型|MCP 2026-07-28 / SDK v2 方向|
|---|---|---|
|协议核心|初始化后依赖连接上下文|更偏向 stateless protocol core 和请求驱动|
|交互方向|Server-to-Client Request 能力较强|用 Multi Round-Trip Requests（MRTR）覆盖需要输入的交互|
|远程 Transport|SSE 与 Streamable HTTP 并存|Streamable HTTP 成为主要方向，SSE 为 legacy|
|HTTP 路由|主要依赖会话或部署层约定|引入 Header Routing 语义|
|列表数据|每次请求可能重新计算|支持 List Result Caching|
|扩展|特性分散在核心或具体实现中|提供 Extensions Framework|
|长任务|普通请求与通知为主|通过 Tasks Extension 表达延迟结果和生命周期|
|Python Client|底层 stream 与 `ClientSession`|高层 `Client` 封装常见连接和调用|
|Python Server|`FastMCP` 等旧 API|`MCPServer` 等更明确的服务器抽象|

“新版”并不是把旧模型全部废弃，而是把核心路径重新设计为更适合 HTTP、代理、网关、无粘性负载均衡和异步任务的形式。

## 6.3 生命周期：从初始化握手到请求驱动

### 旧模型：连接上下文较重要

典型 v1 Client 代码：

```python
async with streamable_http_client(url) as (read_stream, write_stream):
    async with ClientSession(read_stream, write_stream) as session:
        await session.initialize()
        tools = await session.list_tools()
        result = await session.call_tool(
            "add",
            arguments={"a": 1, "b": 2},
        )
```

在这种模型中，开发者需要显式处理：

- Transport 上下文；
- 读写 stream；
- Session；
- 初始化；
- session 内的后续请求。

这并不意味着 v1 无法使用 HTTP，而是它更接近“建立会话，再在会话中调用”的编程模型。

### 新模型：请求本身承载更多上下文

2026-07-28 的方向是 stateless protocol core。这里的 stateless 不是说 Server 永远不能保存业务状态，而是协议核心尽量不要求 Server 依赖一个长期存在的协议 Session 才能理解每个请求。

这样设计的原因包括：

- HTTP 请求天然是独立的；
- 请求可以被网关、代理和负载均衡转发；
- Server 可以无粘性扩展；
- 连接断开后更容易恢复；
- 认证、审计和重放边界更清晰。

需要区分：

```text
协议层无 Session 依赖
        ≠
业务层不能有状态
```

例如任务结果、用户授权、数据库事务仍然可能需要持久化状态，只是这些状态不再必然依附于一条长连接。

## 6.4 Server-to-Client Requests 与 Multi Round-Trip Requests

### 旧问题：Server 需要 Client 反向请求

某些工具执行到一半时，Server 可能需要：

- 请求用户确认；
- 请求模型补充信息；
- 请求 Host 提供采样能力；
- 请求 Client 选择一个资源。

旧模型可以通过 Server-to-Client Request 表达：

```text
Client ── tools/call ──> Server
Client <─ Server request ─ Server
Client ── answer ───────> Server
Client <─ final result ── Server
```

这种交互能力强，但对基础设施要求高。HTTP 网关、反向代理、负载均衡器和权限系统通常更擅长处理“Client 发起请求”，而不是让 Server 在任意时刻反向调用 Client。

### MRTR 是什么

Multi Round-Trip Requests（MRTR，多轮往返请求）不是普通重试，也不是网络层自动重发。它指的是：

> 一次逻辑操作由多个 Client → Server 的请求/响应往返共同完成；Server 在某一轮返回“还需要什么输入”，Client 获取输入后，再发起下一轮请求。

示意：

```text
第 1 轮：Client ── operation ──> Server
         Client <─ need input ─── Server

第 2 轮：Client ── input ────────> Server
         Client <─ need input ─── Server

第 3 轮：Client ── input ────────> Server
         Client <─ final result ── Server
```

### 为什么要重新设计为 MRTR

官方方向的核心原因是降低对 Server → Client 反向通道的依赖：

1. **适配 HTTP**：每一轮都由 Client 发起，更符合普通 HTTP 请求模型。
2. **适配网关**：代理和网关更容易记录、认证和转发 Client 发起的请求。
3. **支持无粘性扩展**：不同轮次可以由不同实例处理，只要状态有明确的外部存储或可恢复表示。
4. **改善审计**：每一次补充输入都成为显式请求，便于审计和权限判断。
5. **增强恢复能力**：某一轮失败后，Client 可以根据协议状态决定继续、恢复或终止。

因此，MRTR 把“Server 主动打回来问问题”的隐式反向交互，改造成“Server 返回下一步需要的输入，Client 再显式提交”的请求驱动交互。

### MRTR 与 Tasks 的区别

两者解决的是不同问题：

|机制|解决的问题|关键特征|
|---|---|---|
|MRTR|一次交互需要补充输入|同一个逻辑操作需要多次往返|
|Tasks|结果需要稍后产生|需要状态、查询、等待或取消生命周期|

一个操作可以同时使用二者：先通过 MRTR 获取用户确认，再创建一个 Task 执行长时间任务。

## 6.5 Transport：从 SSE 到 Streamable HTTP

### v1 常见状态

早期远程部署中，SSE 提供了 Server 到 Client 的事件流，适合持续推送，但它带来了连接管理、代理兼容性和反向消息路径等复杂度。

### 2026-07-28 的方向

Streamable HTTP 更适合作为远程 MCP 的主要 Transport，因为它保留流式能力，同时使用更通用的 HTTP 请求语义。设计目标包括：

- 让普通 HTTP 基础设施能够处理 MCP；
- 兼容短请求和长响应；
- 减少对固定长连接的依赖；
- 方便接入认证、路由、缓存、监控和负载均衡。

SSE 并非“立即失效”，而是被定位为旧式兼容路径。迁移时应同时确认：

- Client 是否支持 Streamable HTTP；
- 网关是否允许流式响应；
- 超时和断线恢复策略；
- 认证头和 CORS 配置；
- 是否仍有旧客户端依赖 SSE。

## 6.6 HTTP Header Routing

在多租户、多版本或多实例部署中，单靠 URL 路径不一定足够。例如同一个 MCP endpoint 可能需要根据协议方法、能力名称或请求类型选择处理路径。

Header Routing 的价值是让 HTTP 层能够使用明确的 MCP header 信息进行路由，而不必解析完整业务 payload。典型 header 名称包括：

```text
Mcp-Method
Mcp-Name
```

它可以帮助基础设施：

- 将不同方法转给不同后端；
- 根据工具或资源名称做策略控制；
- 在网关层执行审计和限流；
- 在不理解完整 MCP 业务语义的情况下完成基础路由。

这并不意味着业务 Server 可以忽略 JSON-RPC 中的 `method` 和参数校验。Header 是 HTTP 层辅助信息，最终协议语义仍由 MCP 消息决定。

## 6.7 List Result Caching

`tools/list`、`resources/list` 和 `prompts/list` 常被频繁调用，但列表内容通常不会随每个请求变化。每次都重新计算会增加：

- Server 计算成本；
- 数据库或外部 API 访问；
- 网络传输；
- Host 初始化延迟。

List Result Caching 的目标是允许列表结果带有缓存语义，例如：

```text
ttlMs      缓存有效时间
cacheScope 缓存作用范围
```

缓存设计时需要问三个问题：

1. 列表多久可能变化？
2. 缓存属于哪个主体：连接、用户、租户还是公共范围？
3. 工具或资源变化时如何失效？

缓存只适用于可接受短暂陈旧的发现结果。对于权限、租户隔离和高风险工具，不能因为缓存而绕过实时授权检查。

## 6.8 Extensions Framework

如果所有新能力都直接写入核心协议，协议会变得越来越难以演进。Extensions Framework 的作用是提供一个可协商、可隔离的扩展机制：

```text
核心协议：所有实现都应理解
扩展协议：只有声明支持的实现才启用
```

扩展通常需要解决：

- 如何命名；
- 如何声明支持；
- 如何协商版本；
- 如何定义消息和错误；
- 不支持扩展的一方如何安全降级。

这样可以让新能力在不破坏核心互操作性的情况下演进。

## 6.9 Tasks Extension

普通请求假设结果可以在一次请求生命周期内返回，但实际系统中可能存在：

- 大文件处理；
- 批量数据分析；
- 远程部署；
- 需要排队的任务；
- 需要用户稍后查看的结果。

Tasks Extension 用于表达“请求已经产生一个可追踪任务，但最终结果还没有立即完成”。常见状态包括：

```text
running
completed
failed
cancelled
```

典型生命周期：

```text
创建任务
  ↓
查询任务状态
  ├── running      继续等待或稍后查询
  ├── completed    获取最终结果
  ├── failed       获取错误
  └── cancelled    结束
```

Tasks 与普通请求的区别：

|普通请求|Task|
|---|---|
|结果通常随响应返回|响应可能只返回任务标识|
|请求生命周期短|任务跨越多个请求|
|状态常由连接上下文承载|状态需要显式持久化和查询|
|适合快速操作|适合异步、排队和长时间操作|

Tasks 不等于后台线程的简单包装。可靠实现还需要考虑任务持久化、幂等、取消、权限、过期和结果清理。

## 6.10 Python SDK Client：从底层会话到高层 Client

### v1：显式 Session

v1 常见写法直接暴露 Transport 和 Session：

```python
async with streamable_http_client(url) as (read_stream, write_stream):
    async with ClientSession(read_stream, write_stream) as session:
        await session.initialize()
        tools = await session.list_tools()
        result = await session.call_tool(
            "add",
            arguments={"a": 1, "b": 2},
        )
```

这种 API 适合学习协议内部过程，也便于精细控制，但业务代码需要管理较多底层对象。

### v2：高层 Client

SDK v2 提供更接近业务意图的 Client 使用方式：

```python
from mcp import Client

async with Client("http://localhost:8000/mcp") as client:
    tools = await client.list_tools()
    result = await client.call_tool(
        "add",
        {"a": 1, "b": 2},
    )
```

高层 Client 的设计目的不是隐藏协议，而是把常见流程集中封装：

- 建立连接；
- 初始化；
- 方法调用；
- 结果转换；
- 错误处理；
- Transport 生命周期。

当需要调试原始消息、实现自定义 Transport 或研究协议细节时，仍然应使用底层 API。具体类名和参数应以当前安装版本的官方 SDK 文档及类型签名为准。

## 6.11 Python SDK Server API

### v1：以旧式封装为主

旧代码中常见 `FastMCP` 等高层封装。它降低了创建 Server 的门槛，但随着协议能力增加，名称和职责可能不够清晰。

### v2：更明确的 `MCPServer`

SDK v2 的方向是使用更明确的 Server 抽象，并让注册 Tool、Resource、Prompt 的 API 更接近协议概念：

```python
from mcp.server import MCPServer

server = MCPServer("demo")

@server.tool()
def add(a: int, b: int) -> int:
    return a + b
```

Server API 的核心职责仍然是：

- 注册 primitives；
- 生成 schema；
- 响应列表请求；
- 执行工具；
- 返回结构化结果；
- 管理生命周期和 Transport。

示例只展示概念，不应被理解为所有 SDK v2 小版本都保证完全相同的导入路径。课堂实践应以安装版本的官方类型定义为准。

## 6.12 错误处理模型

新版设计并没有消除错误，反而更强调区分错误边界：

```text
Transport error
    ↓
Protocol error
    ↓
Tool business error
    ↓
Task failure
```

工程上应分别记录：

- 网络状态码和连接原因；
- JSON-RPC 错误码；
- 工具业务错误；
- Task 的失败状态和可重试性；
- 用户是否已经确认过某个副作用操作。

重试也必须谨慎。网络超时不一定代表 Server 没有执行成功；对于创建订单、发送邮件、扣款等副作用操作，应通过幂等键或业务查询避免重复执行。

## 6.13 演进总结

这次演进可以概括为四个变化：

```text
连接中心       → 请求中心
反向通道       → 多轮显式往返
单一 Transport → 面向 HTTP 基础设施的部署模型
核心能力堆积   → 核心协议 + 可协商扩展
```

其目标不是让协议更复杂，而是让协议在真实基础设施中更容易部署、路由、扩展、审计和恢复。

# 7. 最小可运行示例

## 7.1 一个简单的 Server

下面示例展示 Server 的概念结构。实际运行时，请根据安装的 Python SDK 版本调整导入路径和启动方式。

```python
from mcp.server import MCPServer

server = MCPServer("calculator")


@server.tool()
def add(a: int, b: int) -> int:
    """Add two integers."""
    return a + b


@server.resource("config://demo")
def demo_config() -> str:
    return '{"environment": "development"}'
```

## 7.2 一个简单的 Client

```python
from mcp import Client


async def main() -> None:
    async with Client("http://localhost:8000/mcp") as client:
        tools = await client.list_tools()
        print(tools)

        result = await client.call_tool(
            "add",
            {"a": 10, "b": 20},
        )
        print(result)
```

## 7.3 从协议角度展开

上面的高层调用大致对应：

```text
Client("...")
  ├── initialize
  ├── tools/list
  └── tools/call
        └── name = add
        └── arguments = {a: 10, b: 20}
```

高层 SDK 只是把这些步骤包装起来。理解底层映射后，开发者才能判断问题究竟出在：

- Python API；
- JSON-RPC；
- Transport；
- Server 业务函数；
- 外部依赖。

# 8. 概念辨析与总结

## 8.1 MCP、JSON-RPC 和 Transport

```text
MCP       定义能力和交互语义
JSON-RPC  定义消息格式
Transport 定义消息如何传输
```

JSON-RPC 不知道什么是 Tool；Transport 不知道什么是模型；MCP 把二者组合起来，形成面向 AI 能力连接的协议。

## 8.2 Tool、Resource 和 Prompt

```text
Tool      做事情
Resource  读东西
Prompt    提供表达任务的模板
```

一个系统可以同时拥有三者，但不要因为都通过 MCP 暴露，就把它们当成相同类型的接口。

## 8.3 Session、MRTR 和 Task

```text
Session  连接或协议生命周期上下文
MRTR     一次逻辑操作的多次请求往返
Task     延迟结果的可追踪生命周期
```

三者关注点不同：

- Session 关注“连接如何开始和维持”；
- MRTR 关注“交互还需要哪些输入”；
- Task 关注“结果什么时候完成以及如何查询”。

## 8.4 学习 MCP 应形成的完整模型

```text
Host
  └── Client
        ├── Lifecycle / Capability
        ├── JSON-RPC
        ├── Transport
        └── Server
              ├── Tools
              ├── Resources
              ├── Prompts
              ├── Extensions
              └── Tasks
```

最终，MCP 的价值不在于“少写几行函数调用代码”，而在于为 AI 应用和外部能力之间建立可发现、可描述、可调用、可扩展、可治理的共同协议。

# 附录：官方参考资料

以下链接均指向官方规范、官方仓库或官方文档。阅读和实践时，应以当前版本的规范和 SDK 类型签名为准。

## MCP 官方资料

- [MCP 官方网站](https://modelcontextprotocol.io/)
- [MCP 2025-11-25 规范](https://modelcontextprotocol.io/specification/2025-11-25)
- [MCP 2026-07-28 规范](https://modelcontextprotocol.io/specification/2026-07-28)
- [MCP 2026-07-28 规范源码目录](https://github.com/modelcontextprotocol/modelcontextprotocol/tree/main/docs/docs/2026-07-28)
- [2026-07-28 版本发布说明](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/)

## Python SDK

- [官方 Python SDK 仓库](https://github.com/modelcontextprotocol/python-sdk)
- [Python SDK 官方文档](https://py.sdk.modelcontextprotocol.io/)
- [Python SDK 协议版本说明](https://github.com/modelcontextprotocol/python-sdk/blob/main/docs/protocol-versions.md)
- [Python SDK 版本与支持策略](https://github.com/modelcontextprotocol/python-sdk/blob/main/VERSIONING.md)
- [Python SDK 路线图](https://github.com/modelcontextprotocol/python-sdk/blob/main/ROADMAP.md)

## 基础规范

- [JSON-RPC 2.0 规范](https://www.jsonrpc.org/specification)

由于协议和 SDK 都在持续演进，示例中的具体导入路径、参数名称和可用扩展可能随安装版本变化。课堂实验应记录实际使用的协议版本、SDK 版本和 Transport，以便复现。
