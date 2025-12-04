# MCP服务

<cite>
**本文档引用的文件**   
- [MCPService.ts](file://src/main/services/MCPService.ts)
- [factory.ts](file://src/main/mcpServers/factory.ts)
- [brave-search.ts](file://src/main/mcpServers/brave-search.ts)
- [dify-knowledge.ts](file://src/main/mcpServers/dify-knowledge.ts)
- [fetch.ts](file://src/main/mcpServers/fetch.ts)
- [memory.ts](file://src/main/mcpServers/memory.ts)
- [python.ts](file://src/main/mcpServers/python.ts)
- [sequentialthinking.ts](file://src/main/mcpServers/sequentialthinking.ts)
- [mcp.ts](file://src/main/apiServer/routes/mcp.ts)
- [mcp.ts](file://src/main/apiServer/services/mcp.ts)
- [mcp.ts](file://src/main/utils/mcp.ts)
- [types.ts](file://src/renderer/src/types/mcp.ts)
</cite>

## 目录
1. [简介](#简介)
2. [架构设计](#架构设计)
3. [核心功能](#核心功能)
4. [内置MCP服务器实现](#内置mcp服务器实现)
5. [接口定义与数据流](#接口定义与数据流)
6. [错误处理策略](#错误处理策略)
7. [性能优化建议](#性能优化建议)
8. [系统集成](#系统集成)
9. [常见问题与调试](#常见问题与调试)

## 简介
MCP（Model Control Protocol）服务是Cherry Studio的核心组件，用于扩展AI模型的功能。它通过标准化的协议与各种外部服务和工具进行交互，为AI模型提供丰富的功能支持。MCP服务支持多种通信方式，包括stdio、HTTP流式传输和内存内通信，能够灵活地集成各种类型的服务器。

## 架构设计

```mermaid
graph TD
A[MCP客户端] --> B[MCPService]
B --> C{服务器类型}
C --> |内置服务器| D[InMemoryTransport]
C --> |HTTP服务器| E[StreamableHTTPClientTransport]
C --> |SSE服务器| F[SSEClientTransport]
C --> |子进程服务器| G[StdioClientTransport]
D --> H[内置MCP服务器]
E --> I[远程MCP服务器]
F --> I
G --> J[本地MCP服务器]
H --> K[brave-search]
H --> L[dify-knowledge]
H --> M[fetch]
H --> N[filesystem]
H --> O[memory]
H --> P[python]
H --> Q[sequentialthinking]
```

**图示来源**
- [MCPService.ts](file://src/main/services/MCPService.ts#L136-L587)
- [factory.ts](file://src/main/mcpServers/factory.ts#L17-L54)

**本节来源**
- [MCPService.ts](file://src/main/services/MCPService.ts#L1-L923)

## 核心功能

### 服务注册与管理
MCP服务通过`MCPService`类提供服务注册和管理功能。系统支持多种类型的MCP服务器，包括内置服务器和外部服务器。内置服务器通过内存内传输（InMemoryTransport）直接集成，而外部服务器则通过stdio、SSE或HTTP流式传输进行通信。

```mermaid
sequenceDiagram
participant Client as "客户端"
participant MCPService as "MCPService"
participant Server as "MCP服务器"
Client->>MCPService : 注册服务器
MCPService->>MCPService : 生成服务器密钥
MCPService->>MCPService : 检查现有客户端
alt 客户端已存在
MCPService->>MCPService : 验证连接状态
MCPService->>Server : 发送ping请求
Server-->>MCPService : 响应ping
MCPService-->>Client : 返回现有客户端
else 客户端不存在
MCPService->>MCPService : 创建新客户端
MCPService->>MCPService : 初始化传输层
MCPService->>Server : 建立连接
Server-->>MCPService : 连接成功
MCPService->>MCPService : 存储客户端
MCPService-->>Client : 返回新客户端
end
```

**图示来源**
- [MCPService.ts](file://src/main/services/MCPService.ts#L171-L459)

### 工具调用机制
MCP服务提供统一的工具调用接口，支持缓存机制以提高性能。工具调用时会自动处理身份验证、超时和进度报告等功能。

```mermaid
flowchart TD
Start([开始调用工具]) --> ValidateInput["验证输入参数"]
ValidateInput --> InputValid{"参数有效?"}
InputValid --> |否| ReturnError["返回错误"]
InputValid --> |是| CheckCache["检查缓存"]
CheckCache --> CacheHit{"缓存命中?"}
CacheHit --> |是| ReturnCache["返回缓存结果"]
CacheHit --> |否| InitClient["初始化客户端"]
InitClient --> AuthCheck["检查身份验证"]
AuthCheck --> |需要认证| OAuthFlow["执行OAuth流程"]
OAuthFlow --> Connect["重新连接"]
AuthCheck --> |无需认证| Connect
Connect --> CallTool["调用工具"]
CallTool --> Progress["报告进度"]
Progress --> ProcessResult["处理结果"]
ProcessResult --> UpdateCache["更新缓存"]
UpdateCache --> ReturnResult["返回结果"]
ReturnCache --> End([结束])
ReturnResult --> End
ReturnError --> End
```

**图示来源**
- [MCPService.ts](file://src/main/services/MCPService.ts#L660-L716)
- [mcp.ts](file://src/main/apiServer/services/mcp.ts#L122-L176)

**本节来源**
- [MCPService.ts](file://src/main/services/MCPService.ts#L111-L134)
- [MCPService.ts](file://src/main/services/MCPService.ts#L660-L716)

## 内置MCP服务器实现

### Brave Search服务器
Brave Search MCP服务器提供网络搜索和本地搜索功能，通过Brave Search API获取信息。

```mermaid
classDiagram
class BraveSearchServer {
+server : Server
-apiKey : string
+constructor(apiKey : string)
+initialize() : void
}
class Fetcher {
+html(requestPayload : RequestPayload) : Promise<Response>
+json(requestPayload : RequestPayload) : Promise<Response>
+txt(requestPayload : RequestPayload) : Promise<Response>
+markdown(requestPayload : RequestPayload) : Promise<Response>
}
BraveSearchServer --> Fetcher : "使用"
BraveSearchServer --> Server : "实现"
```

**图示来源**
- [brave-search.ts](file://src/main/mcpServers/brave-search.ts#L291-L374)

### Dify Knowledge服务器
Dify Knowledge MCP服务器用于访问Dify平台的知识库，支持列出知识库和搜索知识内容。

```mermaid
classDiagram
class DifyKnowledgeServer {
+server : Server
-config : DifyKnowledgeServerConfig
+constructor(difyKey : string, args : string[])
+initialize() : void
-performListKnowledges(difyKey : string, apiHost : string) : Promise<McpResponse>
-performSearchKnowledge(id : string, query : string, topK : number, difyKey : string, apiHost : string) : Promise<McpResponse>
}
DifyKnowledgeServer --> Server : "实现"
```

**图示来源**
- [dify-knowledge.ts](file://src/main/mcpServers/dify-knowledge.ts#L53-L258)

### Fetch服务器
Fetch MCP服务器提供网页内容抓取功能，支持HTML、JSON、纯文本和Markdown格式的提取。

```mermaid
classDiagram
class FetchServer {
+server : Server
+constructor()
}
class Fetcher {
+html(requestPayload : RequestPayload) : Promise<Response>
+json(requestPayload : RequestPayload) : Promise<Response>
+txt(requestPayload : RequestPayload) : Promise<Response>
+markdown(requestPayload : RequestPayload) : Promise<Response>
}
FetchServer --> Fetcher : "使用"
FetchServer --> Server : "实现"
```

**图示来源**
- [fetch.ts](file://src/main/mcpServers/fetch.ts#L227-L233)

### Memory服务器
Memory MCP服务器提供知识图谱管理功能，支持实体、关系和观察的创建、查询和删除。

```mermaid
classDiagram
class MemoryServer {
+server : Server
-knowledgeGraphManager : KnowledgeGraphManager
-initializationPromise : Promise<void>
+constructor(envPath : string)
+setupRequestHandlers() : void
}
class KnowledgeGraphManager {
-memoryPath : string
-entities : Map<string, Entity>
-relations : Set<string>
-fileMutex : Mutex
+create(memoryPath : string) : Promise<KnowledgeGraphManager>
+createEntities(entities : Entity[]) : Promise<Entity[]>
+createRelations(relations : Relation[]) : Promise<Relation[]>
+addObservations(observations : { entityName : string; contents : string[] }[]) : Promise<{ entityName : string; addedObservations : string[] }[]>
+deleteEntities(entityNames : string[]) : Promise<void>
+deleteObservations(deletions : { entityName : string; observations : string[] }[]) : Promise<void>
+deleteRelations(relations : Relation[]) : Promise<void>
+readGraph() : Promise<KnowledgeGraph>
+searchNodes(query : string) : Promise<KnowledgeGraph>
+openNodes(names : string[]) : Promise<KnowledgeGraph>
}
MemoryServer --> KnowledgeGraphManager : "使用"
MemoryServer --> Server : "实现"
```

**图示来源**
- [memory.ts](file://src/main/mcpServers/memory.ts#L339-L714)

### Python服务器
Python MCP服务器提供在沙箱环境中执行Python代码的功能，使用Pyodide引擎。

```mermaid
classDiagram
class PythonServer {
+server : Server
+constructor()
+setupRequestHandlers() : void
}
PythonServer --> Server : "实现"
PythonServer --> pythonService : "使用"
```

**图示来源**
- [python.ts](file://src/main/mcpServers/python.ts#L11-L115)

### Sequential Thinking服务器
Sequential Thinking MCP服务器提供逐步思考功能，支持动态问题解决和反思性分析。

```mermaid
classDiagram
class ThinkingServer {
+server : Server
-thinkingServer : SequentialThinkingServer
+constructor()
+initialize() : void
}
class SequentialThinkingServer {
-thoughtHistory : ThoughtData[]
-branches : Record<string, ThoughtData[]>
+processThought(input : unknown) : { content : Array<{ type : string; text : string }>; isError? : boolean }
}
ThinkingServer --> SequentialThinkingServer : "使用"
ThinkingServer --> Server : "实现"
```

**图示来源**
- [sequentialthinking.ts](file://src/main/mcpServers/sequentialthinking.ts#L251-L294)

**本节来源**
- [brave-search.ts](file://src/main/mcpServers/brave-search.ts#L1-L375)
- [dify-knowledge.ts](file://src/main/mcpServers/dify-knowledge.ts#L1-L260)
- [fetch.ts](file://src/main/mcpServers/fetch.ts#L1-L234)
- [memory.ts](file://src/main/mcpServers/memory.ts#L1-L715)
- [python.ts](file://src/main/mcpServers/python.ts#L1-L116)
- [sequentialthinking.ts](file://src/main/mcpServers/sequentialthinking.ts#L1-L295)

## 接口定义与数据流

### API接口
MCP服务通过REST API提供服务，主要接口包括：

| 接口 | 方法 | 描述 | 请求参数 | 响应格式 |
|------|------|------|----------|----------|
| /v1/mcps | GET | 列出所有MCP服务器 | 无 | {success: boolean, data: McpServersResp} |
| /v1/mcps/{server_id} | GET | 获取MCP服务器信息 | server_id: string | {success: boolean, data: MCPServer} |
| /v1/mcps/{server_id}/mcp | ALL | 处理MCP请求 | server_id: string, 请求体包含MCP消息 | MCP协议响应 |

```mermaid
sequenceDiagram
participant Client as "客户端"
participant ApiServer as "API服务器"
participant MCPService as "MCPService"
Client->>ApiServer : GET /v1/mcps
ApiServer->>MCPService : 获取所有服务器
MCPService-->>ApiServer : 返回服务器列表
ApiServer-->>Client : 返回JSON响应
Client->>ApiServer : GET /v1/mcps/{server_id}
ApiServer->>MCPService : 获取服务器信息
MCPService-->>ApiServer : 返回服务器信息
ApiServer-->>Client : 返回JSON响应
Client->>ApiServer : POST /v1/mcps/{server_id}/mcp
ApiServer->>MCPService : 处理MCP请求
MCPService-->>ApiServer : 返回MCP响应
ApiServer-->>Client : 返回MCP响应
```

**图示来源**
- [mcp.ts](file://src/main/apiServer/routes/mcp.ts#L45-L156)
- [mcp.ts](file://src/main/apiServer/services/mcp.ts#L56-L176)

### 数据流
MCP服务的数据流遵循以下模式：

```mermaid
flowchart LR
A[客户端请求] --> B[API路由]
B --> C[API服务]
C --> D[MCP服务]
D --> E{服务器类型}
E --> |内置| F[内存传输]
E --> |HTTP| G[HTTP流式传输]
E --> |SSE| H[SSE传输]
E --> |子进程| I[stdio传输]
F --> J[内置服务器]
G --> K[远程服务器]
H --> K
I --> L[本地服务器]
J --> M[处理请求]
K --> M
L --> M
M --> N[返回结果]
N --> O[API服务]
O --> P[API路由]
P --> Q[客户端响应]
```

**图示来源**
- [mcp.ts](file://src/main/apiServer/routes/mcp.ts#L140-L153)
- [mcp.ts](file://src/main/apiServer/services/mcp.ts#L122-L176)
- [MCPService.ts](file://src/main/services/MCPService.ts#L171-L459)

**本节来源**
- [mcp.ts](file://src/main/apiServer/routes/mcp.ts#L1-L157)
- [mcp.ts](file://src/main/apiServer/services/mcp.ts#L1-L186)

## 错误处理策略

### 错误分类
MCP服务采用分层的错误处理策略，主要错误类型包括：

```mermaid
erDiagram
ERROR {
string type PK
string code
string message
object details
timestamp timestamp
}
CONNECTION_ERROR ||--o{ ERROR : "属于"
AUTH_ERROR ||--o{ ERROR : "属于"
VALIDATION_ERROR ||--o{ ERROR : "属于"
SERVER_ERROR ||--o{ ERROR : "属于"
RATE_LIMIT_ERROR ||--o{ ERROR : "属于"
CONNECTION_ERROR {
string type PK
string host
number port
string protocol
}
AUTH_ERROR {
string type PK
string provider
string scope
}
VALIDATION_ERROR {
string type PK
string field
string expected
string actual
}
SERVER_ERROR {
string type PK
number status_code
string service
}
RATE_LIMIT_ERROR {
string type PK
number limit
number remaining
timestamp reset
}
```

### 错误处理流程
MCP服务的错误处理流程如下：

```mermaid
flowchart TD
Start([开始]) --> ReceiveError["接收错误"]
ReceiveError --> ErrorType{"错误类型?"}
ErrorType --> |连接错误| HandleConnection["处理连接错误"]
ErrorType --> |认证错误| HandleAuth["处理认证错误"]
ErrorType --> |验证错误| HandleValidation["处理验证错误"]
ErrorType --> |服务器错误| HandleServer["处理服务器错误"]
ErrorType --> |速率限制| HandleRateLimit["处理速率限制"]
ErrorType --> |其他错误| HandleOther["处理其他错误"]
HandleConnection --> LogError["记录错误日志"]
HandleAuth --> LogError
HandleValidation --> LogError
HandleServer --> LogError
HandleRateLimit --> LogError
HandleOther --> LogError
LogError --> CheckRetry{"可重试?"}
CheckRetry --> |是| Retry["重试操作"]
CheckRetry --> |否| FormatError["格式化错误"]
Retry --> ReceiveError
FormatError --> ReturnError["返回用户错误"]
ReturnError --> End([结束])
```

**本节来源**
- [MCPService.ts](file://src/main/services/MCPService.ts#L660-L716)
- [brave-search.ts](file://src/main/mcpServers/brave-search.ts#L157-L240)
- [dify-knowledge.ts](file://src/main/mcpServers/dify-knowledge.ts#L135-L256)

## 性能优化建议

### 缓存策略
MCP服务采用多级缓存策略来提高性能：

```mermaid
graph TD
A[请求] --> B{缓存检查}
B --> |命中| C[返回缓存结果]
B --> |未命中| D[执行操作]
D --> E[获取结果]
E --> F[存储到缓存]
F --> G[返回结果]
C --> H[结束]
G --> H
subgraph "缓存层级"
I[内存缓存] --> J[持久化缓存]
J --> K[分布式缓存]
end
```

**图示来源**
- [MCPService.ts](file://src/main/services/MCPService.ts#L111-L134)
- [MCPService.ts](file://src/main/services/MCPService.ts#L640-L652)

### 连接管理
MCP服务通过连接池和连接复用优化性能：

```mermaid
classDiagram
class ConnectionPool {
-connections : Map<string, Client>
-pendingClients : Map<string, Promise<Client>>
+getClient(server : MCPServer) : Promise<Client>
+releaseClient(serverKey : string) : void
+closeAll() : Promise<void>
}
class MCPService {
-clients : Map<string, Client>
-pendingClients : Map<string, Promise<Client>>
+initClient(server : MCPServer) : Promise<Client>
+closeClient(serverKey : string) : Promise<void>
}
MCPService --> ConnectionPool : "使用"
```

**图示来源**
- [MCPService.ts](file://src/main/services/MCPService.ts#L137-L138)
- [MCPService.ts](file://src/main/services/MCPService.ts#L171-L459)

**本节来源**
- [MCPService.ts](file://src/main/services/MCPService.ts#L111-L134)
- [MCPService.ts](file://src/main/services/MCPService.ts#L171-L459)

## 系统集成

### 与AI核心的集成
MCP服务与AI核心的集成方式如下：

```mermaid
graph LR
A[AI核心] --> B[MCP服务]
B --> C[内置MCP服务器]
B --> D[外部MCP服务器]
C --> E[brave-search]
C --> F[dify-knowledge]
C --> G[fetch]
C --> H[memory]
C --> I[python]
C --> J[sequentialthinking]
D --> K[HTTP服务器]
D --> L[SSE服务器]
D --> M[子进程服务器]
subgraph "AI核心功能"
N[模型推理]
O[上下文管理]
P[工具调用]
end
N --> P
O --> P
P --> B
```

**本节来源**
- [MCPService.ts](file://src/main/services/MCPService.ts#L1-L923)
- [aiCore](file://packages/aiCore/src/core/)

### 与插件系统的集成
MCP服务与插件系统的集成方式如下：

```mermaid
graph TD
A[插件系统] --> B[MCP服务]
B --> C[插件MCP服务器]
C --> D[插件功能]
D --> E[扩展AI能力]
E --> F[AI核心]
subgraph "插件生命周期"
G[安装] --> H[激活]
H --> I[配置]
I --> J[使用]
J --> K[停用]
K --> L[卸载]
end
G --> C
H --> C
I --> C
J --> C
K --> C
L --> C
```

**本节来源**
- [MCPService.ts](file://src/main/services/MCPService.ts#L1-L923)
- [plugins](file://packages/aiCore/src/core/plugins/)

## 常见问题与调试

### 常见问题
以下是使用MCP服务时可能遇到的常见问题及解决方案：

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| 服务器连接失败 | 网络问题、服务器未启动、认证失败 | 检查网络连接，确认服务器状态，重新进行认证 |
| 工具调用超时 | 服务器响应慢、网络延迟、请求复杂度过高 | 增加超时时间，优化请求，检查服务器性能 |
| 缓存不一致 | 缓存过期、数据更新未同步 | 清除缓存，重启服务，检查数据同步机制 |
| 身份验证失败 | 凭据过期、权限不足、OAuth流程中断 | 重新获取凭据，检查权限设置，完成OAuth流程 |
| 内存泄漏 | 连接未正确关闭、对象未释放 | 确保连接正确关闭，检查资源释放逻辑 |

### 调试技巧
调试MCP服务时可以使用以下技巧：

1. **启用详细日志**：通过配置日志级别获取更详细的调试信息
2. **使用进度事件**：监听`Mcp_Progress`事件监控工具调用进度
3. **检查缓存状态**：使用`CacheService`检查和清除缓存
4. **验证服务器配置**：确保服务器配置正确无误
5. **测试连接**：使用`checkMcpConnectivity`方法测试服务器连接状态

```mermaid
flowchart TD
A[发现问题] --> B[收集信息]
B --> C{信息是否充分?}
C --> |否| D[启用详细日志]
D --> E[重现问题]
E --> B
C --> |是| F[分析日志]
F --> G{找到根本原因?}
G --> |否| H[使用调试工具]
H --> I[设置断点]
I --> J[逐步执行]
J --> F
G --> |是| K[制定解决方案]
K --> L[实施修复]
L --> M[验证修复]
M --> N{问题解决?}
N --> |否| A
N --> |是| O[记录解决方案]
```

**本节来源**
- [MCPService.ts](file://src/main/services/MCPService.ts#L592-L612)
- [MCPService.ts](file://src/main/services/MCPService.ts#L660-L716)
- [loggerService](file://src/main/services/LoggerService.ts)