# AI Agent 提效平台 - 学习笔记

## 一、框架使用方式：扩展 vs 二次开发

### 1.1 基本概念区分

#### 扩展（Extension）

**定义**：在框架预留的扩展点上添加功能，但不修改框架本身的代码。

**特点**：

- ✅ 使用框架提供的 API、接口、钩子
- ✅ 实现框架定义的 Interface/Protocol
- ✅ 通过插件机制、配置机制添加功能
- ✅ 框架源码保持不变（作为 npm/maven 包引入）
- ✅ 遵循框架的设计规范和约束

**项目实现示例**：

```typescript
// 1. 引入框架的核心组件（不修改源码）
import { EditorRenderer, FreeLayoutEditorProvider } from '@flowgram.ai/free-layout-editor';

// 2. 实现框架提供的节点注册接口
export const nodeRegistries = {
  start: StartNode,
  end: EndNode,
  model: ModelNode,
  tool_mcp: ToolMcpNode,
  // ...
};

// 3. 使用框架的插件机制
"@flowgram.ai/free-container-plugin": "0.1.28"

// 4. 集成自己的业务逻辑
AiAgentDrawService.getDrawConfig()  // 后端接口集成
```

#### 二次开发（Secondary Development）

**定义**：对现有系统进行修改、定制，可能涉及修改源码。

**特点**：

- ⚠️ 可能 fork 原项目仓库
- ⚠️ 修改框架的源代码
- ⚠️ 改变框架的核心实现逻辑
- ⚠️ 定制化程度更深
- ⚠️ 升级框架版本时可能遇到冲突

**典型做法**：

```bash
# 二次开发的典型流程
git clone git@github.com:flowgram/flowgram.git
cd flowgram
# 修改框架源码，比如改变渲染逻辑、增加新的核心功能等
# 然后发布为自己的版本
```

### 1.2 对比总结

| 维度         | 扩展（Extension）    | 二次开发（Secondary Development） |
| ------------ | -------------------- | --------------------------------- |
| **代码位置** | 独立项目，引用框架   | Fork 或修改框架源码               |
| **修改范围** | 仅在扩展点添加功能   | 可以修改框架核心逻辑              |
| **依赖方式** | npm/maven 包依赖     | 源码级依赖                        |
| **升级成本** | 低（直接升级包版本） | 高（需要合并代码冲突）            |
| **耦合度**   | 松耦合               | 强耦合                            |
| **维护性**   | 好维护               | 较难维护                          |

### 1.3 判断标准

**简单判断方法**：

- 如果 FlowGram 在 `package.json` 的 `dependencies` 里 → **扩展**
- 如果项目里有 FlowGram 的源码文件夹，并且在修改它 → **二次开发**

**本项目定位**：标准的**框架扩展应用**，利用 FlowGram 提供的"拖拉拽"流程编辑能力。

---

## 二、MCP 协议与通信机制

### 2.1 核心概念

#### SSE (Server-Sent Events)

**定义**：基于 HTTP 连接到**已运行的服务**并接收推送

**核心区别**：

- **SSE 模式**：连接一个**已经在运行**的服务（不启动新进程）
- **stdio 模式**：**启动一个新的子进程**（由你的程序管理）

**通信模式对比**：

```java
传统 HTTP：
客户端 → 请求 → 服务器
客户端 ← 响应 ← 服务器
（一问一答，连接关闭）

SSE：
客户端 → 建立连接 → 服务器（已运行的独立服务）
客户端 ← 持续推送 ← 服务器（连接保持打开）
客户端 ← 持续推送 ← 服务器
客户端 ← 持续推送 ← 服务器
```

**特点**：

- 连接已部署的独立服务
- 服务可独立扩缩容、重启
- 支持多客户端同时连接
- 基于 HTTP，可跨网络
- 自动重连机制
- 适合场景：远程服务、微服务、第三方 API

**项目实际应用**（docker-compose-app.yml）：

```yaml
# CSDN 服务 - 独立部署的 Docker 容器
mcp-server-csdn-app:
  ports:
    - "8101:8101"  # 服务已运行，Java 通过 SSE 连接

# 微信服务 - 独立部署的 Docker 容器
mcp-server-weixin-app:
  ports:
    - "8102:8102"  # 服务已运行，Java 通过 SSE 连接
```

**代码实现**（MCPTest.java:92-96）：

```java
// 连接到已运行的 MCP Server
HttpClientSseClientTransport sseClientTransport = HttpClientSseClientTransport
    .builder("http://127.0.0.1:9999")  // 服务已经在运行
    .sseEndpoint("/sse?apikey=DElk89iu8Ehhnbu")
    .build();
```

#### stdio (Standard Input/Output)

**定义**：通过标准输入输出流启动本地子进程并与之通信

**核心区别**：

- **SSE 模式**：连接一个**已经在运行**的服务（HTTP 调用）
- **stdio 模式**：**启动一个新的子进程**并管理其生命周期

**通信模式**：

```java
你的 Java 程序 (PID: 1234)
   │
   └─ 启动子进程 ──> Node.js/Python/其他 (PID: 5678)
                    ├─ stdin  ← Java 写入命令
                    └─ stdout → Java 读取结果
```

**特点**：

- 由你的程序启动和管理子进程
- 进程生命周期跟随主程序
- 不需要网络，本地进程通信
- 1对1 通信（不能多客户端共享）
- 适合场景：本地工具、脚本、NPM 包

**实际例子**（MCPTest.java:70-73）：

```java
// 启动 NPM 包作为 MCP Server
var stdioParams = ServerParameters.builder("npx")
    .args("-y", "fetcher-mcp")  // 相当于执行: npx -y fetcher-mcp
    .build();
var mcpClient = McpClient.sync(new StdioClientTransport(stdioParams))

// 其他例子：
// 1. 启动本地文件系统 MCP
ServerParameters.builder("npx")
    .args("-y", "@modelcontextprotocol/server-filesystem", "/data/uploads")

// 2. 启动本地 SQLite MCP
ServerParameters.builder("npx")
    .args("-y", "@modelcontextprotocol/server-sqlite", "--db-path", "/data/local.db")

// 3. 启动 Python 脚本
ServerParameters.builder("python3")
    .args("-m", "mcp_server_local")
```

#### SSE vs stdio 决策树

```go
需要使用 MCP Server？
   │
   ├─ 这个服务已经在运行吗？（独立部署的服务）
   │   ├─ 是 → 使用 SSE 模式
   │   │   ✓ Docker 容器服务（CSDN、微信）
   │   │   ✓ 第三方 API（百度搜索、高德地图）
   │   │   ✓ 微服务架构中的其他服务
   │   │
   │   └─ 否（需要临时启动工具） → 使用 stdio 模式
   │       ✓ NPM 包（fetcher-mcp、filesystem）
   │       ✓ 本地脚本（Python、Shell）
   │       ✓ 本地工具（SQLite、Git）
```

**对比表格**：

| 对比维度       | SSE 模式           | stdio 模式       |
| -------------- | ------------------ | ---------------- |
| **服务状态**   | 已运行（独立部署） | 需启动（子进程） |
| **谁启动服务** | 运维/Docker        | 你的 Java 程序   |
| **生命周期**   | 独立（可随时重启） | 跟随主程序       |
| **通信方式**   | HTTP + SSE         | stdin/stdout     |
| **网络需求**   | 需要（可跨机器）   | 不需要（本地）   |
| **多客户端**   | 支持               | 不支持（1对1）   |
| **适用场景**   | 微服务、第三方 API | 本地工具、脚本   |
| **扩展性**     | 可负载均衡         | 不可扩展         |

**实际场景举例**：

```java
生产环境架构：
AI Agent 平台 (8091)
   │
   ├─ SSE 模式 ─────> Nginx Gateway (9999)
   │                   ├─> CSDN 服务 (8101) - Docker 容器
   │                   ├─> 微信服务 (8102) - Docker 容器
   │                   └─> 百度搜索 - 外部 API
   │
   └─ stdio 模式 ─────> 启动本地子进程
                       ├─> npx fetcher-mcp (网页抓取)
                       ├─> npx filesystem (文件操作)
                       └─> python script.py (本地脚本)
```

### 2.2 为什么需要双向通信？

#### MCP 协议工作流程

```java
┌─────────────┐          ┌─────────────┐          ┌─────────────┐
│  AI 模型    │ ←──────→ │ MCP Client  │ ←──────→ │ MCP Server  │
│  (GPT-4)    │  调用工具  │ (你的系统)  │  双向通信  │ (外部工具)  │
└─────────────┘          └─────────────┘          └─────────────┘
```

#### 双向通信的必要性

**1. 请求-响应循环**:

- AI → MCP Client：我需要调用搜索工具
- MCP Client → MCP Server：执行搜索("天气")
- MCP Server → MCP Client：搜索结果...
- MCP Client → AI：这是搜索结果

**2. 状态同步**:

- 工具列表查询 (initialize)
- 能力协商
- 心跳保活

**3. 流式响应**:

- 长时间任务的进度反馈
- 实时数据流

**代码体现**（MCPTest.java:59-64）：

```java
chatModel.call(Prompt.builder()
    .messages(new UserMessage("有哪些工具可以使用"))  // ① AI 发起请求
    .build());
// ② MCP Client 通过 SSE/stdio 向 MCP Server 查询工具列表
// ③ MCP Server 返回工具信息
// ④ MCP Client 返回给 AI
```

### 2.3 双向通信组件实现

#### SSE 双向通信

虽然 SSE 本身是单向的，但 MCP 协议通过**结合 HTTP POST** 实现双向：

```java
客户端 → POST /sse (请求)  → 服务器
客户端 ← SSE 流 (持续响应) ← 服务器
```

**代码位置**：

- `HttpClientSseClientTransport`
- `McpClient.sync(sseClientTransport)` - 封装了双向通信逻辑

#### stdio 双向通信

```java
// AiClientToolMcpNode.java:107
var mcpClient = McpClient.sync(new StdioClientTransport(stdioParams))
```

`StdioClientTransport` 内部：

- **stdin 流** → 向子进程发送命令
- **stdout 流** → 从子进程读取响应

#### Function Calling 机制

**代码位置**（MCPTest.java:59）：

```java
.toolCallbacks(new SyncMcpToolCallbackProvider(sseMcpClient2auth()).getToolCallbacks())
```

**调用链路**：

```java
AI 模型 (Function Calling)
    ↓
SyncMcpToolCallbackProvider (适配器)
    ↓
MCP Client (HttpClientSseClientTransport / StdioClientTransport)
    ↓
MCP Server (外部工具：搜索、微信、数据库等)
```

---

## 三、Nginx MCP Gateway 架构设计

### 3.1 配置文件位置

`/ai-agent-station/docs/dev-ops-v2/ai-agent-mcp-gateway/conf/conf.d/mcp.localhost.conf`

### 3.2 核心功能实现

#### 1. Token 校验（第 13-15 行）

```nginx
if ($arg_apikey != "DElk89iu8Ehhnbu") {
    return 403; # 如果apikey不正确，返回403禁止访问
}
```

#### 2. 路由转发（第 2-4, 20 行）

```nginx
upstream backend_servers {
    server 192.168.1.108:8101;  # 可配置多个服务器实现负载均衡
}

proxy_pass http://backend_servers;
```

#### 3. SSE 专用配置（第 26-28 行）

```nginx
chunked_transfer_encoding off;  # 禁用分块传输
proxy_buffering off;            # 禁用缓冲，确保实时推送
proxy_cache off;                # 禁用缓存
```

### 3.3 完整调用链路

```java
┌─────────────────────────────────────────────────────────────────────┐
│                         用户在前端拖拽编排                              │
│  ┌────┐    ┌────┐    ┌────────┐    ┌─────────┐    ┌────┐           │
│  │开始│ → │模型│ → │MCP工具│ → │条件判断│ → │结束│           │
│  └────┘    └────┘    └────────┘    └─────────┘    └────┘           │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
              用户点击"运行"或通过 API 调用
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│  前端 (React) 发起 HTTP 请求                                          │
│  POST http://your-server:8091/api/v1/ai/agent/chat_agent            │
│  {                                                                   │
│    "aiAgentId": 1,                                                   │
│    "message": "帮我发一篇关于 Spring AI 的 CSDN 文章"                   │
│  }                                                                   │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│  后端 Java (Spring Boot)                                             │
│  AiAgentController.chatAgent()  (第 77 行)                           │
│      ↓                                                               │
│  AIAgentChatService.aiAgentChat()                                    │
│      ↓                                                               │
│  构建 Agent 工作流（执行各个节点）                                       │
│      ↓                                                               │
│  遇到 Tool MCP 节点 (AiClientToolMcpNode.java:66)                    │
│  调用 createMcpSyncClient()                                          │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
                    判断传输类型（SSE 还是 stdio）
                              ↓
        ┌─────────────────────┴─────────────────────┐
        ↓                                           ↓
     SSE 方式                                    stdio 方式
        ↓                                           ↓
┌──────────────────────────┐          ┌─────────────────────────┐
│  HttpClientSseClientTransport │          │ StdioClientTransport    │
│  baseUri: http://127.0.0.1:9999│          │ 启动本地子进程           │
│  sseEndpoint: /sse?apikey=XXX  │          │ command: java -jar ...  │
└──────────────────────────┘          └─────────────────────────┘
        ↓                                           ↓
┌──────────────────────────┐          ┌─────────────────────────┐
│  Nginx MCP Gateway       │          │  本地 MCP Server         │
│  (端口 9999)              │          │  (stdio 进程)            │
│                          │          └─────────────────────────┘
│  ① Token 校验 (第 13 行)  │
│     apikey == "DElk..."? │
│     NO → 返回 403        │
│     YES ↓                │
│                          │
│  ② 路由转发 (第 20 行)    │
│     proxy_pass           │
│     http://backend_servers│
└──────────────────────────┘
        ↓
┌──────────────────────────┐
│  远程 MCP Server          │
│  (CSDN/微信/百度搜索等)     │
│  - mcp-server-csdn-app    │
│    (8101端口)             │
│  - mcp-server-weixin-app  │
│    (8102端口)             │
└──────────────────────────┘
        ↓
    返回执行结果
        ↓
┌──────────────────────────┐
│  AI 模型 (GPT-4)         │
│  通过 Function Calling    │
│  获取工具执行结果          │
│  生成最终回复              │
└──────────────────────────┘
        ↓
    返回给前端显示
```

### 3.4 "跨系统调用"详解

#### 什么是跨系统？

```java
┌──────────────────┐
│ AI Agent 平台     │
│ (你的系统)        │
└──────────────────┘
         ↓
    Nginx Gateway (9999)
    ├─ Token 校验
    ├─ 限流保护
    └─ 路由转发
         ↓
    ┌────┴─────────────────────────────────┐
    ↓                ↓                     ↓
┌─────────┐   ┌──────────┐   ┌──────────────────┐
│CSDN系统 │   │微信系统  │   │百度搜索系统        │
│(8101)   │   │(8102)    │   │(外部API)          │
└─────────┘   └──────────┘   └──────────────────┘
  系统A          系统B            系统C
```

**"跨系统"包含**：

1. **内部系统**：CSDN 发文服务、微信通知服务（Docker 容器）
2. **外部 API**：百度搜索 MCP、高德地图 MCP
3. **第三方服务**：其他部门、合作伙伴提供的服务

#### 两种请求路径的区别

**① 用户 Prompt 请求（不经过 MCP Gateway）**:

```java
前端 → 8091 (Spring Boot) → AI 模型处理
```

**② AI 调用 MCP 工具（经过 MCP Gateway）**:

```java
Spring Boot → 9999 (Nginx Gateway) → 8101 (MCP Server CSDN)
```

#### 为什么需要 Gateway？

| 场景         | 没有 Gateway                     | 有 Gateway                |
| ------------ | -------------------------------- | ------------------------- |
| **鉴权**     | 每个 MCP Server 都要实现鉴权逻辑 | 统一在 Nginx 校验 Token   |
| **安全**     | 直接暴露后端服务 IP 和端口       | 只暴露 Gateway 端口 9999  |
| **限流**     | 恶意调用可能打爆某个服务         | Nginx 统一限流            |
| **监控**     | 分散在各个服务的日志             | Nginx access.log 统一记录 |
| **负载均衡** | 单点故障                         | upstream 配置多个后端     |

---

## 四、面试问答准备

### 4.1 基础理解类

**Q1: SSE 和 WebSocket 的区别是什么？为什么选 SSE？**

A:

- **SSE**：单向（服务器→客户端），基于 HTTP，轻量级，自动重连
- **WebSocket**：双向，独立协议，更复杂，需要维护长连接
- **选择原因**：MCP 协议大部分场景是服务器推送，SSE 更简单且易于穿透防火墙

**Q2: stdio 通信和 HTTP 通信的优缺点？**

A:

- **stdio**：
  - 优点：本地、高效、无网络开销
  - 缺点：只能本地进程，无法跨机器
- **HTTP/SSE**：
  - 优点：跨网络、分布式部署
  - 缺点：有网络延迟、需要处理网络异常

**Q3: 什么是 MCP (Model Context Protocol)？**

A: 一个标准化协议，让 AI 模型能够调用外部工具/服务。类似于函数调用，但规范化了通信格式和传输方式。

### 4.2 架构设计类

**Q4: 为什么需要 Nginx 作为 Gateway？**

A:

- **鉴权统一**：避免每个 MCP Server 都实现鉴权逻辑
- **限流保护**：防止恶意调用
- **负载均衡**：upstream 配置多个后端
- **SSL 终结**：统一处理 HTTPS
- **可观测性**：统一日志记录

**Q5: 如何保证 MCP 通信的安全性？**

A:

- **Nginx 层**：Token 校验、IP 白名单、HTTPS
- **应用层**：requestTimeout 防止长时间占用、异常处理
- **网络层**：内网隔离，只暴露必要端口

**Q6: 如果 MCP Server 宕机，如何处理？**

A:

- Nginx upstream 健康检查
- 重试机制
- 降级策略（使用默认工具或返回友好提示）
- 监控告警

### 4.3 实现细节类

**Q7: `McpSyncClient` 为什么用同步客户端？异步会更好吗？**

A:

- **同步**：代码简单，适合顺序执行的工作流
- **异步**：高并发场景更好，但需要处理回调复杂度
- 工作流场景多是串行执行，同步客户端更适合

**Q8: Function Calling 是怎么工作的？**

A:

1. AI 模型分析用户意图，判断需要调用工具
2. 返回 function_call JSON（包含工具名和参数）
3. `SyncMcpToolCallbackProvider` 解析并调用对应工具
4. 将结果返回给 AI，AI 继续生成回复

**Q9: 代码中的 `beanName("AiClientToolMcp_" + id)` 是做什么的？**

A: 动态注册 Spring Bean。每个 MCP 配置对应一个独立的 `McpSyncClient` 实例，便于多租户隔离和独立管理。

### 4.4 生产实践类

**Q10: SSE 连接断线如何处理？**

A:

- MCP Client 有自动重连机制
- `requestTimeout` 配置超时时间
- 应用层捕获异常并记录日志

**Q11: 如何监控 MCP 调用的性能和成功率？**

A:

- Nginx access_log 记录所有请求
- 应用层 log.info 记录初始化和调用详情
- 可接入 Prometheus + Grafana 进行可视化监控

**Q12: 如果要支持多个 MCP Server，如何扩展？**

A:

- 数据库表 `ai_client_tool_mcp` 配置化管理
- `createMcpSyncClient` 方法已支持动态创建
- Nginx upstream 配置多个后端实现负载均衡

---

## 五、简历优化建议

### 原文

> 集成 MCP 去封装基于 SSE 与 stdio 的双向通信组件，通过 Function Calling 机制打通了大模型与运维基础设施的调用链路。在 Nginx 层设计了自定义 Token 校验与路由转发逻辑，作为 MCP Gateway 统一管理 Agent 调用的鉴权与限流，确保了生产环境下跨系统调用的安全性。

### 优化建议

> 基于 MCP 协议实现了 AI Agent 工具调用中台，封装了 **SSE（远程服务）** 和 **stdio（本地进程）** 两种传输层，通过 Spring AI 的 Function Calling 机制打通了 GPT-4 与微信通知、CSDN 发文、联网搜索等外部能力的调用链路。在 Nginx 层设计了 **Token 校验**、**路由转发**、**限流保护**，作为统一 MCP Gateway 确保生产环境下跨系统调用的安全性与可观测性。

**优化理由**：

- 更具体（提到了微信、CSDN 等实际场景）
- 体现技术深度（传输层、中台、可观测性）
- 便于面试官提问（每个关键词都有话可说）

---

## 六、关键代码位置索引

### 前端部分

- **FlowGram 配置**：`ai-agent-station-front/package.json`
- **编辑器组件**：`ai-agent-station-front/src/editor.tsx`
- **节点注册**：`ai-agent-station-front/src/nodes/`

### 后端部分

- **MCP 测试**：`ai-agent-station/ai-agent-station-app/src/test/java/cn/bugstack/ai/test/mcp/MCPTest.java`
- **MCP 节点实现**：`ai-agent-station/ai-agent-station-domain/src/main/java/cn/bugstack/ai/domain/agent/service/armory/node/AiClientToolMcpNode.java`
- **Agent 控制器**：`ai-agent-station/ai-agent-station-trigger/src/main/java/cn/bugstack/ai/trigger/http/AiAgentController.java`

### DevOps 部分

- **Nginx Gateway 配置**：`ai-agent-station/docs/dev-ops-v2/ai-agent-mcp-gateway/conf/conf.d/mcp.localhost.conf`
- **Docker Compose**：`ai-agent-station/docs/dev-ops-v2/docker-compose-app.yml`

---

## 七、学习要点总结

1. **扩展 vs 二次开发**：理解框架使用的两种方式，本项目采用扩展方式
2. **MCP 协议**：标准化的 AI 工具调用协议，支持 SSE 和 stdio 两种传输方式
3. **双向通信**：理解为什么需要双向通信，以及如何通过 SSE + HTTP POST 实现
4. **Function Calling**：AI 模型调用外部工具的核心机制
5. **Gateway 架构**：Nginx 作为统一网关，实现鉴权、路由、限流、监控
6. **跨系统调用**：理解生产环境中多系统协作的安全性和可维护性
