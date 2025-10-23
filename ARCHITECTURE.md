# JoyAgent-JDGenie 项目架构解析

> 深入解析 JoyAgent-JDGenie 的系统架构、技术栈和核心设计

## 目录

- [项目概述](#项目概述)
- [整体架构](#整体架构)
- [技术栈](#技术栈)
- [核心模块](#核心模块)
- [数据流](#数据流)
- [Agent 工作机制](#agent-工作机制)
- [扩展性设计](#扩展性设计)

---

## 项目概述

JoyAgent-JDGenie 是由京东开源的**端到端多智能体产品**，与其他仅提供 SDK 或框架的项目不同，它是一个完整的、可直接使用的生产级应用。

### 核心特点

- **完整产品** - 包含前端 UI、后端服务、工具服务的完整解决方案
- **多智能体架构** - 支持 React、Plan-Execute、ReAct 等多种智能体模式
- **数据智能** - 内置 DataAgent，支持自然语言查询数据库
- **开箱即用** - 无需额外开发即可部署使用
- **高度可扩展** - 支持自定义工具、MCP 协议集成

---

## 整体架构

### 三层架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户交互层                                 │
│                    Frontend (React + Vite)                       │
│  - 对话界面 (ChatView)                                            │
│  - 任务规划展示 (PlanView)                                        │
│  - 文件管理 (FileList, ActionView)                               │
│  - 结果可视化 (ECharts, Markdown)                                 │
│                      Port: 3000                                  │
└───────────────────────┬─────────────────────────────────────────┘
                        │ HTTP/SSE (Server-Sent Events)
                        ↓
┌─────────────────────────────────────────────────────────────────┐
│                    智能体编排层                                   │
│              Backend (Spring Boot + Java 17)                     │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Controller Layer (GenieController, DataAgentController)  │   │
│  │ - SSE 流式输出                                           │   │
│  │ - 请求路由和参数验证                                      │   │
│  └────────────────────┬─────────────────────────────────────┘   │
│                       │                                          │
│  ┌────────────────────▼─────────────────────────────────────┐   │
│  │ Agent Orchestration Layer                                │   │
│  │                                                           │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │ ReactAgent   │  │PlanningAgent │  │ExecutorAgent │   │   │
│  │  │ (单步推理)    │  │(任务拆解)     │  │(工具调用)     │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  │                                                           │   │
│  │  ┌──────────────┐  ┌──────────────┐                      │   │
│  │  │ SummaryAgent │  │  DataAgent   │                      │   │
│  │  │ (结果汇总)    │  │(数据分析)     │                      │   │
│  │  └──────────────┘  └──────────────┘                      │   │
│  └────────────────────┬─────────────────────────────────────┘   │
│                       │                                          │
│  ┌────────────────────▼─────────────────────────────────────┐   │
│  │ Tool Collection                                           │   │
│  │ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐     │   │
│  │ │CodeTool  │ │ReportTool│ │SearchTool│ │FileTool  │     │   │
│  │ └──────────┘ └──────────┘ └──────────┘ └──────────┘     │   │
│  │ ┌──────────┐ ┌──────────┐ ┌──────────┐                  │   │
│  │ │DataTool  │ │PlanTool  │ │MCP Tools │                  │   │
│  │ └──────────┘ └──────────┘ └──────────┘                  │   │
│  └────────────────────┬─────────────────────────────────────┘   │
│                       │                                          │
│  ┌────────────────────▼─────────────────────────────────────┐   │
│  │ LLM Layer (OpenAI/Claude/DeepSeek API)                   │   │
│  │ - Token 计数和管理                                        │   │
│  │ - 多模型支持和配置                                        │   │
│  │ - 流式响应处理                                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│                      Port: 8080                                  │
└───────────────────────┬─────────────────────────────────────────┘
                        │ HTTP REST API
                        ↓
┌─────────────────────────────────────────────────────────────────┐
│                      工具服务层                                   │
│            Python Tools (FastAPI + Python 3.11)                 │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Report Generation Service                                 │   │
│  │ - HTML 报告生成 (Jinja2)                                  │   │
│  │ - PPT 生成                                                │   │
│  │ - Markdown 渲染                                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Search & Analysis Service                                 │   │
│  │ - 深度搜索 (DeepSearch)                                   │   │
│  │ - 数据分析 (Pandas, NumPy, Scikit-learn)                 │   │
│  │ - 图表生成 (Matplotlib, ECharts)                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Table RAG Service                                         │   │
│  │ - 表格检索增强生成                                        │   │
│  │ - 向量化和语义匹配                                        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                      Port: 1601                                  │
└───────────────────────┬─────────────────────────────────────────┘
                        │
        ┌───────────────┴──────────────────┐
        ↓                                  ↓
┌──────────────────┐            ┌──────────────────┐
│  MCP Client      │            │  Data Storage     │
│  (Port 8188)     │            │                   │
│  - MCP 协议适配   │            │  - MySQL 数据库   │
│  - 外部工具集成   │            │  - Qdrant 向量库  │
└──────────────────┘            │  - Elasticsearch  │
                                │  - 文件存储       │
                                └──────────────────┘
```

---

## 技术栈

### 前端技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| **React** | 19.0 | UI 框架 |
| **TypeScript** | 5.7 | 类型安全 |
| **Vite** | 6.1 | 构建工具 |
| **Tailwind CSS** | 4.1 | 样式框架 |
| **Ant Design** | 5.26 | UI 组件库 |
| **ECharts** | 6.0 | 数据可视化 |
| **React Router** | 7.6 | 路由管理 |
| **React Markdown** | 9.0 | Markdown 渲染 |

**关键特性：**
- 实时流式输出（SSE）
- 代码高亮（Prism.js）
- Markdown 渲染
- 文件上传下载
- 响应式设计

### 后端技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| **Spring Boot** | 3.2.2 | 应用框架 |
| **Java** | 17 | 编程语言 |
| **Maven** | 3.8 | 构建工具 |
| **MyBatis Plus** | 3.5.14 | ORM 框架 |
| **HikariCP** | 4.0.3 | 连接池 |
| **OkHttp** | 4.9.3 | HTTP 客户端（支持 SSE） |
| **FastJSON** | 1.2.83 | JSON 处理 |
| **MySQL Connector** | 8.3.0 | 数据库驱动 |
| **Qdrant Client** | 1.10.0 | 向量数据库客户端 |
| **Elasticsearch** | 7.17 | 搜索引擎 |

**关键特性：**
- SSE 流式响应
- 多 Agent 编排
- 工具动态加载
- Token 管理
- DAG 执行引擎

### Python 工具服务技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| **FastAPI** | 0.115.14 | Web 框架 |
| **Python** | 3.11+ | 编程语言 |
| **LiteLLM** | 1.74+ | LLM 抽象层 |
| **SmolaAgents** | 1.19+ | 轻量级 Agent 框架 |
| **Pandas** | 2.3+ | 数据处理 |
| **Scikit-learn** | 1.7+ | 机器学习 |
| **Matplotlib** | 3.10+ | 可视化 |
| **Jinja2** | 3.1+ | 模板引擎 |
| **Qdrant Client** | 1.5 | 向量数据库 |

**关键特性：**
- 报告生成（HTML/PPT/Markdown）
- 数据分析和可视化
- Table RAG
- 深度搜索

---

## 核心模块

### 1. Agent 模块 (`genie-backend/agent/`)

#### BaseAgent - 智能体基类

所有 Agent 的抽象基类，定义了核心接口：

```java
public abstract class BaseAgent {
    // Agent 运行主方法
    public abstract AgentResult run(AgentRequest request);

    // LLM 调用
    protected LLMResponse callLLM(Message message);

    // 工具调用
    protected Object callTool(ToolCall toolCall);
}
```

#### ReactImplAgent - 单步反应式 Agent

**特点：**
- 单次推理和行动
- 适合简单、快速的任务
- 低延迟响应

**工作流程：**
```
用户输入 → 思考 → 工具调用 → 返回结果
```

#### PlanningAgent - 规划 Agent

**特点：**
- 将复杂任务拆解为子任务
- 生成有序的执行计划
- 支持计划更新和调整

**工作流程：**
```
复杂任务 → 深度推理 → 生成计划列表 → 逐个执行
```

**Planning 工具参数：**
```json
{
  "command": "create",
  "title": "任务标题",
  "steps": [
    "执行顺序1. 子任务1：详细描述",
    "执行顺序2. 子任务2：详细描述"
  ]
}
```

#### ExecutorAgent - 执行 Agent

**特点：**
- 专注于单个子任务的执行
- 调用具体工具完成任务
- 返回结构化结果

**工作流程：**
```
接收子任务 → 分析需求 → 选择工具 → 执行 → 返回结果
```

#### SummaryAgent - 总结 Agent

**特点：**
- 汇总多个任务的执行结果
- 生成最终答案
- 提取关键文件名

**输出格式：**
```
答案内容 $$$ 文件名1、文件名2、文件名3
```

### 2. 工具模块 (`genie-backend/agent/tool/`)

#### 工具接口设计

```java
public interface BaseTool {
    String getName();           // 工具名称
    String getDescription();    // 工具描述
    Map<String, Object> toParams(); // 参数 schema
    Object execute(Object input);   // 执行方法
}
```

#### 内置工具列表

| 工具 | 功能 | 实现位置 |
|------|------|----------|
| **CodeInterpreterTool** | Python 代码执行 | Java → Python 服务 |
| **ReportTool** | 报告生成 | Java → Python 服务 |
| **FileTool** | 文件读写 | Java 本地实现 |
| **DeepSearchTool** | 互联网搜索 | Java → Python 服务 |
| **DataAnalysisTool** | 数据分析 | Java → Python 服务 |
| **PlanningTool** | 任务规划 | Java 本地实现 |
| **MCP Tools** | 外部工具 | MCP 协议适配 |

#### 工具调用流程

```
Agent 决策
    ↓
识别需要的工具
    ↓
构造工具调用参数
    ↓
┌─────────────────┐
│ 本地工具？      │
└────┬──────┬─────┘
     YES   NO
      ↓     ↓
   直接执行  HTTP 调用 Python 服务
      ↓     ↓
    返回结果
      ↓
   Agent 处理结果
```

### 3. DataAgent 模块 (`genie-backend/data/`)

#### 核心功能

1. **数据治理（DGP 协议）**
   - 表级元数据管理
   - 字段级语义标注
   - 值域枚举同步

2. **NL2SQL（自然语言转 SQL）**
   - 表召回（Table RAG）
   - 字段召回（Column RAG）
   - SQL 生成和优化
   - 结果解释

3. **智能诊断分析**
   - 趋势分析
   - 异常检测
   - 相关性分析

#### 架构组件

```
┌─────────────────────────────────────────────────┐
│ DataAgent Controller                            │
│ - /data/queryModelInfo (获取表信息)             │
│ - /data/apiChatQuery (数据查询)                 │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│ JDBC Catalog Manager                            │
│ - MySQL Catalog                                 │
│ - H2 Catalog                                    │
│ - ClickHouse Catalog                            │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│ SQL Generator                                   │
│ - Dialect Support (MySQL/H2/ClickHouse)        │
│ - Query Optimization                            │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│ Vector Store (Qdrant)                           │
│ - 表级向量                                       │
│ - 字段级向量                                     │
└─────────────────────────────────────────────────┘
```

### 4. LLM 层 (`genie-backend/agent/llm/`)

#### 多模型支持

**配置示例：**
```yaml
llm:
  default:
    base_url: 'https://api.openai.com/v1'
    apikey: 'sk-xxx'
    model: 'gpt-4.1'
    max_tokens: 16384

  settings: '{
    "claude-3-7-sonnet": {
      "model": "claude-3-7-sonnet-v1",
      "max_tokens": 8192,
      "base_url": "https://api.anthropic.com",
      "apikey": "sk-ant-xxx"
    },
    "deepseek-chat": {
      "model": "deepseek-chat",
      "max_tokens": 8192,
      "base_url": "https://api.deepseek.com"
    }
  }'
```

#### Token 管理

```java
public class TokenCounter {
    // 计算消息 token 数
    public int countTokens(List<Message> messages);

    // 截断超长消息
    public List<Message> truncateMessages(
        List<Message> messages,
        int maxTokens
    );
}
```

---

## 数据流

### 1. 用户查询流程

```
用户输入
    ↓
Frontend 发送请求 (POST /api/v1/chat)
    ↓
GenieController 接收
    ↓
创建 SSE 连接
    ↓
选择 Agent 模式 (React/Plan-Execute)
    ↓
Agent 开始推理
    ↓
┌─────────────────────────────┐
│ 需要工具调用？              │
└────┬────────────────┬───────┘
     YES              NO
      ↓                ↓
调用 Tool         直接生成答案
      ↓                ↓
Tool 返回结果        │
      ↓                ↓
Agent 处理 ──────────→ 生成响应
      ↓
通过 SSE 流式发送到前端
      ↓
前端实时显示
```

### 2. Plan-Execute 模式数据流

```
用户复杂任务
    ↓
PlanningAgent 接收
    ↓
深度推理，生成任务列表
    ↓
┌──────────────────────────┐
│ Planning Tool 调用       │
│ {                        │
│   "command": "create",   │
│   "steps": [...]         │
│ }                        │
└──────────┬───────────────┘
           ↓
任务列表持久化
    ↓
逐个任务执行
    ↓
┌──────────────────────────┐
│ ExecutorAgent 执行子任务 │
│ - 子任务 1 → 工具调用    │
│ - 子任务 2 → 工具调用    │
│ - 子任务 3 → 工具调用    │
└──────────┬───────────────┘
           ↓
收集所有执行结果
    ↓
SummaryAgent 总结
    ↓
返回最终答案和文件
```

### 3. DataAgent 查询流程

```
自然语言查询 (如："查询最近三个月销售额")
    ↓
DataAgent 接收
    ↓
┌─────────────────────────────┐
│ 表召回 (Table RAG)          │
│ - 查询向量化                │
│ - 向量相似度匹配            │
│ - 返回相关表                │
└──────────┬──────────────────┘
           ↓
┌─────────────────────────────┐
│ 字段召回 (Column RAG)       │
│ - 从召回的表中选择字段      │
│ - 字段语义匹配              │
└──────────┬──────────────────┘
           ↓
┌─────────────────────────────┐
│ SQL 生成                    │
│ - LLM 生成 SQL              │
│ - SQL 优化和验证            │
└──────────┬──────────────────┘
           ↓
执行 SQL
    ↓
返回结果 + 解释
    ↓
前端展示（表格/图表）
```

---

## Agent 工作机制

### React 模式详解

**适用场景：**
- 简单、单一的查询
- 需要快速响应
- 不需要复杂规划

**Prompt 结构：**
```
System Prompt
    ↓
用户问题
    ↓
可用工具列表
    ↓
思考 (Thought)
    ↓
行动 (Action) - 工具调用或直接回答
    ↓
观察 (Observation) - 工具返回结果
    ↓
（可能的多轮迭代）
    ↓
完成 (Finish)
```

### Plan-Execute 模式详解

**适用场景：**
- 复杂、多步骤任务
- 需要系统性规划
- 多个子任务有依赖关系

**两阶段设计：**

**阶段 1：Planning**
```
System Prompt (规划专家)
    ↓
任务分析
    ↓
识别子任务
    ↓
排序和组织
    ↓
生成计划列表
```

**阶段 2：Execution**
```
For each 子任务:
    ↓
ExecutorAgent 接收子任务
    ↓
分析需要的工具
    ↓
调用工具
    ↓
验证结果
    ↓
标记任务完成
```

### Prompt 工程

**System Prompt 设计原则：**

1. **明确角色定位**
```
你是一个智能助手，名叫 Genie。
你擅长任务规划、工具调用和问题解决。
```

2. **清晰的工作流程**
```
请按以下步骤工作：
1. 思考：分析问题，确定方法
2. 行动：调用工具或生成答案
3. 观察：查看工具返回结果
4. 反思：评估是否需要继续
```

3. **约束和规范**
```
- 禁止编造信息
- 必须使用工具列表中的工具
- 输出格式必须是 JSON
- 思考过程不超过 200 字
```

4. **示例和模板**
```
示例：
用户："分析最近的销售数据"
思考：需要先查询数据，再进行分析
行动：调用 DataAnalysisTool
...
```

---

## 扩展性设计

### 1. 添加自定义 Agent

```java
// 1. 继承 BaseAgent
public class CustomAgent extends BaseAgent {
    @Override
    public AgentResult run(AgentRequest request) {
        // 自定义逻辑
        // 调用 LLM
        LLMResponse response = callLLM(message);

        // 解析工具调用
        List<ToolCall> toolCalls = parseToolCalls(response);

        // 执行工具
        for (ToolCall call : toolCalls) {
            Object result = callTool(call);
            // 处理结果
        }

        return new AgentResult(...);
    }
}

// 2. 在 Controller 中注册
@RestController
public class CustomController {
    @PostMapping("/api/custom/chat")
    public SseEmitter customChat(@RequestBody ChatRequest request) {
        CustomAgent agent = new CustomAgent(config);
        return agent.run(request);
    }
}
```

### 2. 添加自定义工具

```java
// 1. 实现 BaseTool 接口
public class WeatherTool implements BaseTool {
    @Override
    public String getName() {
        return "weather_query";
    }

    @Override
    public String getDescription() {
        return "查询指定城市的天气信息";
    }

    @Override
    public Map<String, Object> toParams() {
        return Map.of(
            "type", "object",
            "properties", Map.of(
                "city", Map.of(
                    "type", "string",
                    "description", "城市名称"
                )
            ),
            "required", List.of("city")
        );
    }

    @Override
    public Object execute(Object input) {
        // 实现查询逻辑
        String city = extractCity(input);
        return queryWeatherAPI(city);
    }
}

// 2. 注册工具
toolCollection.addTool(new WeatherTool());
```

### 3. 集成 MCP 工具

**MCP (Model Context Protocol) 支持：**

```yaml
# application.yml
mcp_server_url: "http://service1:port/sse,http://service2:port/sse"
```

MCP 工具会自动被发现和注册，无需额外代码。

### 4. 自定义 Prompt 模板

```yaml
autobots:
  autoagent:
    planner:
      system_prompt: '{
        "default": "你是一个专业的任务规划专家..."
      }'
    executor:
      system_prompt: '{
        "default": "你是一个高效的任务执行专家..."
      }'
```

### 5. 扩展数据源

**添加新的数据库支持：**

```java
// 1. 实现 JdbcCatalog 接口
public class PostgreSQLCatalog implements JdbcCatalog {
    @Override
    public String getDialect() {
        return "postgresql";
    }

    @Override
    public List<TableInfo> listTables(Connection conn) {
        // 查询 PostgreSQL 表信息
    }

    @Override
    public List<ColumnInfo> listColumns(
        Connection conn,
        String tableName
    ) {
        // 查询列信息
    }
}

// 2. 注册到 CatalogFactory
@AutoService(JdbcCatalogFactory.class)
public class PostgreSQLCatalogFactory
    implements JdbcCatalogFactory {

    @Override
    public JdbcCatalog create() {
        return new PostgreSQLCatalog();
    }
}
```

---

## 性能优化

### 1. Token 管理

- **自动截断**：超长对话历史自动截断
- **智能压缩**：保留关键信息，压缩冗余内容
- **分层缓存**：频繁调用的 prompt 缓存

### 2. 并发执行

- **DAG 执行引擎**：识别可并行的工具调用
- **异步 I/O**：所有外部调用使用异步
- **连接池**：数据库和 HTTP 连接池复用

### 3. 流式输出

- **SSE 推送**：实时推送 Agent 思考过程
- **分块传输**：大文件分块发送
- **心跳保活**：10 秒心跳防止连接断开

---

## 安全性设计

### 1. 输入验证

- **参数校验**：所有输入参数严格验证
- **SQL 注入防护**：使用参数化查询
- **文件路径检查**：防止路径遍历

### 2. 权限控制

- **API 密钥管理**：环境变量存储，不硬编码
- **工具权限**：敏感工具需要额外授权
- **数据访问控制**：基于用户的数据权限

### 3. 错误处理

- **优雅降级**：工具失败不影响整体流程
- **重试机制**：临时失败自动重试（3 次）
- **错误隐藏**：敏感错误信息不暴露给用户

---

## 监控和日志

### 日志级别

```yaml
logging:
  level:
    root: INFO
    com.jd.genie: DEBUG
```

### 关键日志点

1. **请求日志**：记录所有 API 调用
2. **Agent 日志**：记录 Agent 决策过程
3. **工具调用日志**：记录工具调用参数和结果
4. **错误日志**：记录所有异常和错误

### 监控指标

- **响应时间**：API 平均响应时间
- **成功率**：任务完成率
- **工具使用率**：各工具调用频率
- **Token 消耗**：LLM token 使用量

---

## 总结

JoyAgent-JDGenie 采用**分层架构**，将用户交互、智能体编排和工具服务清晰分离，实现了：

✅ **高度模块化** - 每个模块职责单一，易于维护
✅ **强扩展性** - 支持自定义 Agent、工具和数据源
✅ **生产就绪** - 完整的错误处理、日志和监控
✅ **性能优化** - Token 管理、并发执行、流式输出
✅ **开发友好** - 清晰的接口、丰富的示例

这使得它不仅是一个 Agent 框架，更是一个**可直接部署的企业级多智能体产品**。
