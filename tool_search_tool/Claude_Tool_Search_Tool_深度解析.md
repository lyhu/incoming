# Claude Tool Search Tool 深度解析：动态工具发现与按需加载机制

> 本文深入剖析 Anthropic Claude 平台的 Tool Search Tool 机制，详解 `defer_loading` 工具加载策略、关键协议消息格式，以及 MCP Server 在工具托管场景中的核心作用。

---

## 一、问题背景：大规模工具库的挑战

### 1.1 Token 消耗困境

随着 AI Agent 应用场景的扩展，模型需要与越来越多的工具进行交互。一个典型的企业级 Agent 可能需要连接：

| 工具来源 | 工具数量 | Token 消耗（估算） |
| -------- | -------- | ------------------ |
| GitHub   | 35 tools | ~26K tokens        |
| Slack    | 11 tools | ~21K tokens        |
| Jira     | 15 tools | ~17K tokens        |
| Sentry   | 5 tools  | ~3K tokens         |
| Grafana  | 5 tools  | ~3K tokens         |

当工具定义在每次 API 调用时全量加载，**58 个工具即可消耗约 55K tokens**——这还未计入用户对话内容。Anthropic 内部测试显示，优化前的工具定义曾消耗高达 **134K tokens**。

### 1.2 工具选择准确性下降

Token 消耗并非唯一问题。当大量工具定义涌入上下文窗口时，模型面临：

- **选择困难**：相似命名的工具（如 `notification-send-user` vs `notification-send-channel`）容易混淆
- **参数错误**：工具数量增加导致参数格式记忆模糊
- **上下文污染**：真正有用的对话历史被工具定义挤占

Anthropic 的评测数据显示，在大型工具库场景下：

- Claude Opus 4 准确率仅为 **49%**
- 启用 Tool Search Tool 后提升至 **74%**（+25%）

---

## 二、Tool Search Tool：按需发现机制

### 2.1 核心设计理念

Tool Search Tool 的设计哲学是**「按需加载，动态发现」**：

```
传统模式：所有工具定义 → 全量加载到上下文 → 模型选择
新模式：  仅搜索工具可见 → 模型搜索 → 动态加载匹配工具 → 模型选择
```

这种模式将上下文消耗从 **~77K tokens** 降至 **~8.7K tokens**，实现 **85% 的 Token 节省**。

### 2.2 两种搜索变体

Claude 平台提供两种开箱即用的搜索工具：

| 搜索类型       | 类型标识                          | 适用场景                                   |
| -------------- | --------------------------------- | ------------------------------------------ |
| **Regex 搜索** | `tool_search_tool_regex_20251119` | 精确匹配、模式匹配，适合命名规范的工具     |
| **BM25 搜索**  | `tool_search_tool_bm25_20251119`  | 语义理解、自然语言查询，适合描述丰富的工具 |

开发者也可基于 Embedding 等技术实现自定义搜索策略。

---

## 三、defer_loading 机制详解

### 3.1 工作原理

`defer_loading` 是控制工具可见性的核心参数。其工作逻辑如下：

```javascript
// 工具定义示例
{
  "tools": [
    // 始终可见：搜索工具
    {
      "type": "tool_search_tool_regex_20251119",
      "name": "tool_search_tool_regex"
    },
    // 延迟加载：标记为 defer_loading: true
    {
      "name": "get_weather",
      "description": "Get the weather at a specific location",
      "input_schema": {
        "type": "object",
        "properties": {
          "location": { "type": "string" },
          "unit": { "type": "string", "enum": ["celsius", "fahrenheit"] }
        },
        "required": ["location"]
      },
      "defer_loading": true  // 关键标记
    }
  ]
}
```

### 3.2 可见性控制矩阵

| 工具类型         | `defer_loading` | 初始可见性    | 加载时机   |
| ---------------- | --------------- | ------------- | ---------- |
| Tool Search Tool | N/A             | ✅ 始终可见   | 请求开始   |
| 高频核心工具     | `false`         | ✅ 始终可见   | 请求开始   |
| 低频/大量工具    | `true`          | ❌ 初始不可见 | 搜索命中后 |

### 3.3 与 Prompt Caching 的兼容性

一个关键设计优势：**defer_loading 不会破坏 Prompt Caching**。

原因在于：延迟加载的工具在初始 Prompt 中完全不存在，仅在搜索触发后才被注入。因此：

- System Prompt 可被缓存
- 核心工具定义（`defer_loading: false`）可被缓存
- 仅动态注入的工具定义不参与缓存

---

## 四、核心协议消息解析

Tool Search Tool 引入了多种新的协议消息类型，理解这些消息对于调试和集成至关重要。

### 4.1 server_tool_use：服务端工具调用

当模型需要搜索工具时，会生成 `server_tool_use` 类型的输出：

```json
{
  "type": "server_tool_use",
  "id": "srvtoolu_01ABC123",
  "name": "tool_search_tool_regex",
  "input": {
    "query": "weather"
  }
}
```

**关键特征**：

- `type` 固定为 `server_tool_use`
- `id` 使用 `srvtoolu_` 前缀（区别于普通工具的 `toolu_` 前缀）
- 该调用由**服务端自动执行**，无需客户端干预

### 4.2 tool_search_tool_result：搜索结果注入

服务端执行搜索后，将结果作为 `tool_search_tool_result` 注入模型上下文：

```json
{
  "type": "tool_search_tool_result",
  "tool_use_id": "srvtoolu_01ABC123",
  "content": {
    "type": "tool_search_tool_search_result",
    "tool_references": [
      {
        "type": "tool_reference",
        "tool_name": "get_weather"
      }
    ]
  }
}
```

**关键特征**：

- `tool_use_id` 与对应的 `server_tool_use` 关联
- `tool_references` 包含匹配工具的引用列表
- 被引用的工具 Schema 会**自动展开**到模型上下文

### 4.3 tool_use 与 tool_result：标准工具调用

工具 Schema 注入后，模型可生成标准的工具调用：

```json
// 工具调用
{
  "type": "tool_use",
  "id": "toolu_01XYZ789",
  "name": "get_weather",
  "input": {
    "location": "San Francisco",
    "unit": "fahrenheit"
  }
}

// 工具结果
{
  "type": "tool_result",
  "tool_use_id": "toolu_01XYZ789",
  "content": "San Francisco, CA: 72°F, Sunny"
}
```

---

## 五、交互式演示动画

读者可以访问以下链接查看 Tool Search Tool 的交互式动画演示：

**演示地址**：[https://lyhu.github.io/incoming/tool_search_tool/](https://lyhu.github.io/incoming/tool_search_tool/)

该动画展示了完整的 11 步时序流程，帮助理解各参与方（Client、Service、Model、Registry、Real Tool）之间的交互过程。通过可视化的时序图和实时协议数据展示，您可以：

- **逐步查看**每个交互步骤的详细说明
- **查看协议数据**：每一步的完整 JSON 消息格式
- **理解数据流向**：观察 `server_tool_use`、`tool_search_tool_result`、`tool_use` 等消息在各参与方之间的传递
- **对比两种模式**：演示展示的是 MCP Server 托管工具场景（Server-Side Execution）

> **注意**：演示假设工具由 MCP Server 托管，实现服务端自动执行。标准 API 模式下，自定义工具需要客户端执行，涉及多次请求/响应往返。

---

## 六、MCP Server 的角色与价值

### 6.1 什么是 MCP (Model Context Protocol)

MCP 是 Anthropic 推出的模型上下文协议，用于标准化 AI 模型与外部工具/数据源的连接方式。MCP Server 作为工具的托管方，可以：

- 提供工具定义（Schema）
- 执行工具调用
- 返回执行结果

### 6.2 MCP 托管模式的优势

当工具由 MCP Server 托管时，整个交互流程可实现**服务端闭环**：

```
Client → Anthropic API → Model → MCP Server → 工具执行 → Model → Client
                    ↑_________________________↓
                       服务端自动完成，无需客户端参与
```

**核心优势**：

1. **单次请求/响应**：客户端仅需发送一次请求
2. **延迟更低**：无需多次网络往返
3. **架构更简洁**：工具执行逻辑集中在服务端

### 6.3 MCP 工具集配置示例

```json
{
  "type": "mcp_toolset",
  "mcp_server_name": "google-drive",
  "default_config": {
    "defer_loading": true // 整个 MCP Server 的工具默认延迟加载
  },
  "configs": {
    "search_files": {
      "defer_loading": false // 高频工具保持始终可见
    }
  }
}
```

---

## 七、两种执行模式对比

### 7.1 标准 API 模式（Client-Side Execution）

在不使用 MCP Server 的标准场景下，自定义工具需要**客户端执行**：

```
请求1: Client → API (用户问题 + 工具定义)
响应1: API → Client (tool_use, stop_reason: "tool_use")
       ↓
       客户端执行工具
       ↓
请求2: Client → API (tool_result)
响应2: API → Client (最终回复)
```

**特点**：

- 至少 2 次请求/响应往返
- 工具执行逻辑在客户端
- 客户端需处理 `stop_reason: "tool_use"` 的中间状态

### 7.2 MCP 托管模式（Server-Side Execution）

当工具由 MCP Server 托管时：

```
请求: Client → API (用户问题 + 工具定义)
                  ↓
           [服务端内部流程]
           Model → Tool Search → 工具注入 → 工具调用 → MCP 执行 → 结果注入
                  ↓
响应: API → Client (完整回复，包含所有中间步骤)
```

**特点**：

- 1 次请求/响应完成
- 工具执行逻辑在服务端
- 响应中包含完整的交互轨迹

### 7.3 模式选择决策树

```
是否使用 MCP Server 托管工具？
├── 是 → Server-Side Execution（单次请求）
└── 否 → 工具是否为 Anthropic 内置 Server Tool？
         ├── 是（如 code_execution）→ Server-Side Execution
         └── 否 → Client-Side Execution（多次请求）
```

---

## 八、11 步时序流程详解

以下基于一个天气查询场景（MCP 托管模式），详细解析完整的交互流程。

### 参与方说明

| 参与方             | 角色                      |
| ------------------ | ------------------------- |
| **Client**         | 调用方应用程序            |
| **Service (API)**  | Anthropic Claude API 网关 |
| **Model**          | Claude 语言模型           |
| **Registry (MCP)** | 工具注册中心 / MCP Server |
| **Real Tool**      | 实际的天气服务            |

### 步骤详解

#### 步骤 1: [REQUEST] 初始请求

**Client → Service**

客户端发送 API 请求，包含搜索工具和延迟加载的业务工具：

```json
{
  "model": "claude-sonnet-4-5-20250929",
  "max_tokens": 2048,
  "messages": [
    { "role": "user", "content": "What is the weather in San Francisco?" }
  ],
  "tools": [
    {
      "type": "tool_search_tool_regex_20251119",
      "name": "tool_search_tool_regex"
    },
    {
      "name": "get_weather",
      "description": "Get the weather at a specific location",
      "input_schema": { ... },
      "defer_loading": true
    }
  ]
}
```

#### 步骤 2: 转发推理

**Service → Model**

API 网关对请求进行预处理，**过滤掉 `defer_loading: true` 的工具**，仅将搜索工具传递给模型：

```json
{
  "messages": [
    { "role": "user", "content": "What is the weather in San Francisco?" }
  ],
  "tools": [
    {
      "type": "tool_search_tool_regex_20251119",
      "name": "tool_search_tool_regex"
    }
  ]
}
```

**关键点**：此时模型**完全看不到** `get_weather` 工具。

#### 步骤 3: 触发工具搜索

**Model → Service**

模型识别到用户需要天气信息，但当前上下文中没有相关工具。于是调用 Tool Search Tool 进行搜索：

```json
{
  "type": "server_tool_use",
  "id": "srvtoolu_01ABC123",
  "name": "tool_search_tool_regex",
  "input": { "query": "weather" }
}
```

#### 步骤 4: 检索工具定义

**Service → Registry/MCP**

服务层根据搜索查询，在 `defer_loading: true` 的工具列表中进行匹配：

```json
{
  "search_query": "weather",
  "matching_tools": ["get_weather"]
}
```

#### 步骤 5: 返回 Schema

**Registry → Service**

注册中心返回匹配工具的完整定义：

```json
{
  "name": "get_weather",
  "description": "Get the weather at a specific location",
  "input_schema": {
    "type": "object",
    "properties": {
      "location": { "type": "string" },
      "unit": { "type": "string", "enum": ["celsius", "fahrenheit"] }
    },
    "required": ["location"]
  }
}
```

#### 步骤 6: 注入定义到上下文

**Service → Model**

服务层将搜索结果封装为 `tool_search_tool_result`，注入模型上下文。同时，被引用的工具 Schema 会被**自动展开**：

```json
{
  "type": "tool_search_tool_result",
  "tool_use_id": "srvtoolu_01ABC123",
  "content": {
    "type": "tool_search_tool_search_result",
    "tool_references": [
      { "type": "tool_reference", "tool_name": "get_weather" }
    ]
  }
}
```

**此时模型已可看到完整的 `get_weather` 工具定义。**

#### 步骤 7: 生成工具调用

**Model → Service**

模型基于注入的 Schema，生成正式的工具调用请求：

```json
{
  "type": "tool_use",
  "id": "toolu_01XYZ789",
  "name": "get_weather",
  "input": {
    "location": "San Francisco",
    "unit": "fahrenheit"
  }
}
```

#### 步骤 8: 调用实际天气 API

**Service → Real Tool**

服务层（通过 MCP Server）将调用转发给实际的天气服务：

```json
{
  "tool_name": "get_weather",
  "input": {
    "location": "San Francisco",
    "unit": "fahrenheit"
  }
}
```

#### 步骤 9: 返回天气数据

**Real Tool → Service**

天气服务返回实时数据：

```json
{
  "location": "San Francisco, CA",
  "temperature": 72,
  "unit": "fahrenheit",
  "conditions": "Sunny"
}
```

#### 步骤 10: 提交结果并生成回复

**Service → Model**

服务层将工具执行结果封装为 `tool_result`，提交给模型生成最终回复：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01XYZ789",
  "content": "San Francisco, CA: 72°F, Sunny"
}
```

#### 步骤 11: [RESPONSE] 返回最终回复

**Service → Client**

API 返回完整响应，包含整个交互过程的轨迹：

```json
{
  "id": "msg_01XYZ123",
  "type": "message",
  "role": "assistant",
  "model": "claude-sonnet-4-5-20250929",
  "content": [
    {
      "type": "text",
      "text": "I'll search for tools to help with the weather information."
    },
    {
      "type": "server_tool_use",
      "id": "srvtoolu_01ABC123",
      "name": "tool_search_tool_regex",
      "input": { "query": "weather" }
    },
    {
      "type": "tool_search_tool_result",
      "tool_use_id": "srvtoolu_01ABC123",
      "content": {
        "type": "tool_search_tool_search_result",
        "tool_references": [
          { "type": "tool_reference", "tool_name": "get_weather" }
        ]
      }
    },
    {
      "type": "text",
      "text": "I found a weather tool. Let me get the weather for San Francisco."
    },
    {
      "type": "tool_use",
      "id": "toolu_01XYZ789",
      "name": "get_weather",
      "input": { "location": "San Francisco", "unit": "fahrenheit" }
    },
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01XYZ789",
      "content": "San Francisco, CA: 72°F, Sunny"
    },
    {
      "type": "text",
      "text": "The weather in San Francisco is currently **72°F** and **Sunny**. Perfect weather for outdoor activities!"
    }
  ],
  "stop_reason": "end_turn",
  "usage": {
    "input_tokens": 1024,
    "output_tokens": 256
  }
}
```

---

## 九、实际应用场景

### 9.1 多 MCP Server 集成

企业级 Agent 通常需要连接多个数据源：

```json
{
  "tools": [
    { "type": "tool_search_tool_bm25_20251119", "name": "tool_search" },
    {
      "type": "mcp_toolset",
      "mcp_server_name": "salesforce",
      "default_config": { "defer_loading": true }
    },
    {
      "type": "mcp_toolset",
      "mcp_server_name": "jira",
      "default_config": { "defer_loading": true }
    },
    {
      "type": "mcp_toolset",
      "mcp_server_name": "slack",
      "default_config": { "defer_loading": true },
      "configs": {
        "send_message": { "defer_loading": false } // 高频工具保持可见
      }
    }
  ]
}
```

### 9.2 大规模工具库管理

当工具数量超过 100 个时，推荐的配置策略：

| 工具类别     | 数量     | defer_loading | 理由             |
| ------------ | -------- | ------------- | ---------------- |
| 核心高频工具 | 3-5 个   | `false`       | 避免搜索开销     |
| 业务工具     | 10-50 个 | `true`        | 按需加载         |
| 长尾工具     | 50+ 个   | `true`        | 几乎不影响上下文 |

### 9.3 与 Programmatic Tool Calling 结合

对于复杂工作流，可结合 Programmatic Tool Calling 进一步优化：

```json
{
  "tools": [
    { "type": "tool_search_tool_regex_20251119", "name": "tool_search" },
    { "type": "code_execution_20250825", "name": "code_execution" },
    {
      "name": "get_expenses",
      "defer_loading": true,
      "allowed_callers": ["code_execution_20250825"] // 允许代码执行环境调用
    }
  ]
}
```

---

## 十、技术实现要点

### 10.1 Beta 功能启用

Tool Search Tool 目前处于 Beta 阶段，需要添加特定 Header：

```python
import anthropic

client = anthropic.Anthropic()

response = client.beta.messages.create(
    betas=["advanced-tool-use-2025-11-20"],
    model="claude-sonnet-4-5-20250929",
    max_tokens=4096,
    tools=[
        {"type": "tool_search_tool_regex_20251119", "name": "tool_search_tool_regex"},
        # ... 其他工具定义
    ],
    messages=[{"role": "user", "content": "..."}]
)
```

### 10.2 工具定义最佳实践

为提高搜索准确性，工具定义应遵循：

```json
// ✅ 推荐：清晰的名称和详细描述
{
  "name": "search_customer_orders",
  "description": "Search for customer orders by date range, status, or total amount. Returns order details including items, shipping, and payment info."
}

// ❌ 避免：模糊的名称和简短描述
{
  "name": "query_db_orders",
  "description": "Execute order query"
}
```

### 10.3 搜索限制

| 限制项           | 值                            |
| ---------------- | ----------------------------- |
| 最大工具数量     | 10,000                        |
| 单次搜索返回结果 | 3-5 个                        |
| Regex 模式长度   | 最大 200 字符                 |
| 支持的模型       | Claude Sonnet 4.5+, Opus 4.5+ |

### 10.4 Token 使用监控

```python
# 监控 Token 消耗
print(f"Input tokens: {response.usage.input_tokens}")
print(f"Output tokens: {response.usage.output_tokens}")

# 如果支持，还可监控搜索请求次数
if hasattr(response.usage, 'server_tool_use'):
    print(f"Tool search requests: {response.usage.server_tool_use.tool_search_requests}")
```

---

## 十一、总结

### 11.1 核心价值

| 指标            | 传统模式    | Tool Search Tool |
| --------------- | ----------- | ---------------- |
| 上下文消耗      | ~77K tokens | ~8.7K tokens     |
| Token 节省      | —           | **85%**          |
| Opus 4 准确率   | 49%         | **74%**          |
| Opus 4.5 准确率 | 79.5%       | **88.1%**        |

### 11.2 适用场景判断

**推荐使用 Tool Search Tool**：

- 工具定义超过 10K tokens
- 工具数量超过 10 个
- 存在工具选择准确性问题
- 构建 MCP 驱动的多服务系统

**可不使用**：

- 工具数量少于 10 个
- 所有工具高频使用
- 工具定义紧凑

### 11.3 技术展望

Tool Search Tool 代表了 AI Agent 工具管理的新范式——从「全量加载」走向「智能发现」。结合 MCP 协议的标准化能力，开发者可以构建真正可扩展的企业级 AI 应用，连接数千个工具而不受上下文窗口限制。

---

**参考资料**：

- [Anthropic Engineering Blog - Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use)
- [Claude Platform Docs - Tool Search Tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)
- [Claude Cookbook - Tool Search with Embeddings](https://github.com/anthropics/claude-cookbooks/blob/main/tool_use/tool_search_with_embeddings.ipynb)
