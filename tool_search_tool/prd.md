# PRD: Claude Tool Search Tool 原理演示网页

## 1. 文档信息

| 项目         | 内容                       |
| ------------ | -------------------------- |
| **文档版本** | v1.0                       |
| **创建日期** | 2026-01-18                 |
| **状态**     | 已完成                     |
| **产出**     | [index.html](./index.html) |

---

## 2. 需求背景

### 2.1 业务目标

通过可视化时序图帮助开发者理解 Claude API 的 **Tool Search Tool** 工作原理，特别是：

- **按需加载机制**：`defer_loading: true` 工具不会立即加载到模型上下文
- **动态发现机制**：模型通过 `server_tool_use` 触发工具搜索
- **Server-Side Execution**：工具执行在 Service 层完成（MCP Server 托管场景）

### 2.2 参考文档

- [官方文档](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)

---

## 3. 功能需求

### 3.1 核心功能：11 步时序图动画演示

#### 参与方（5 泳道）

| 泳道           | 角色               | 职责                       |
| -------------- | ------------------ | -------------------------- |
| Client         | 客户端             | 发起一次请求，接收一次响应 |
| Service (API)  | Anthropic API 网关 | 编排所有内部流程           |
| Model          | Claude 模型        | 推理核心                   |
| Registry (MCP) | 工具定义仓库       | 提供工具 Schema            |
| Weather Tool   | 实际天气服务       | 执行业务逻辑               |

#### 交互流程

| 步骤 | 方向                   | 描述                                            |
| ---- | ---------------------- | ----------------------------------------------- |
| 1    | Client → Service       | **[REQUEST]** 初始请求（含 defer_loading 工具） |
| 2    | Service → Model        | 转发推理（Model 仅看到 tool_search_tool）       |
| 3    | Model → Service        | 触发 `server_tool_use` 请求检索                 |
| 4    | Service → Registry     | 检索工具定义                                    |
| 5    | Registry → Service     | 返回 Schema                                     |
| 6    | Service → Model        | 注入定义（tool_search_tool_result）             |
| 7    | Model → Service        | 生成 `tool_use(get_weather)`                    |
| 8    | Service → Weather Tool | 调用实际 API                                    |
| 9    | Weather Tool → Service | 返回真实数据                                    |
| 10   | Service → Model        | 提交 tool_result，生成最终回复                  |
| 11   | Service → Client       | **[RESPONSE]** 返回润色后的完整响应             |

### 3.2 协议数据展示

每个步骤需展示对应的 JSON 协议数据，格式严格按照官方文档：

- `server_tool_use`：模型发起的工具搜索请求
- `tool_search_tool_result`：包含 `tool_references` 的搜索结果
- `tool_use`：正式的工具调用
- `tool_result`：工具执行结果

### 3.3 交互要求

- 支持"上一步"、"下一步"、"重置"按钮控制
- 点击时序图箭头可直接跳转到对应步骤
- 当前激活步骤高亮显示
- Request（步骤 1）使用绿色箭头，Response（步骤 11）使用橙色箭头

---

## 4. 技术规格

### 4.1 技术栈

| 技术               | 用途               |
| ------------------ | ------------------ |
| Vanilla HTML/CSS   | 页面结构与样式     |
| Vanilla JavaScript | 动画控制与交互逻辑 |
| Flexbox            | 5 列均分布局       |
| CSS 变量           | 主题色管理         |

### 4.2 视觉设计

- **深色主题**：`#0f172a` 背景
- **主色调**：紫色 `#7c4dff` + 青色 `#00e5ff`
- **JSON 语法高亮**：Key/String/Number/Boolean 分色显示
- **玻璃拟态**：卡片使用半透明背景

### 4.3 单文件交付

所有样式和逻辑内联在 `tool_search_tool.html` 中，无外部依赖。

---

## 5. 验收标准

- [x] 5 条生命线正确显示并对齐
- [x] 11 步交互箭头可点击切换
- [x] Step 2 协议数据仅包含 `tool_search_tool`（不含 `get_weather`）
- [x] Step 11 协议数据完整包含 `server_tool_use`、`tool_search_tool_result`、`tool_use`、`tool_result`
- [x] 所有协议数据无自定义注释字段（如 `_note`）
- [x] JSON 语法高亮正常工作
- [x] 响应式布局在 1400px+ 宽度下正常显示

---

## 6. 附录：协议数据示例

### 6.1 初始请求（Step 1）

```json
{
  "model": "claude-sonnet-4-5-20250929",
  "max_tokens": 2048,
  "messages": [{ "role": "user", "content": "What is the weather in San Francisco?" }],
  "tools": [
    { "type": "tool_search_tool_regex_20251119", "name": "tool_search_tool_regex" },
    { "name": "get_weather", "description": "Get the weather at a specific location", "input_schema": {...}, "defer_loading": true }
  ]
}
```

### 6.2 最终响应（Step 11）

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
  "usage": { "input_tokens": 1024, "output_tokens": 256 }
}
```
