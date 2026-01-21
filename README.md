# Incoming - 技术学习知识库

## 项目简介

本项目是一个系统化的技术学习知识库，专注于记录前沿技术深度研究、原理解析与交互式演示。通过文档与可视化演示相结合的方式，提供对复杂技术概念的全方位理解。

## 项目背景与目标

随着人工智能技术的快速发展，大型语言模型（LLM）与外部工具的集成方式日益复杂。本项目旨在：

1. **系统性记录**：对前沿技术进行深度研究与文档化整理
2. **可视化演示**：通过交互式 HTML 页面将抽象概念具象化
3. **知识沉淀**：构建个人技术知识体系，便于回顾与分享
4. **实践驱动**：结合理论分析与实际应用场景

## 技术栈

### 文档撰写

- **Markdown**：技术文档编写
- **技术写作规范**：严谨、逻辑严密、术语准确

### 交互式演示

- **HTML5 + CSS3**：页面结构与视觉呈现
- **Vanilla JavaScript**：动画控制与交互逻辑
- **Flexbox/Grid 布局**：响应式设计
- **CSS 变量**：主题化设计系统
- **单文件架构**：无外部依赖，便于分发与部署

### 可视化技术

- **时序图动画**：展示系统交互流程
- **JSON 语法高亮**：协议数据可视化
- **玻璃拟态设计**：现代化 UI 风格

## 当前内容概览

### 1. Palantir: 终极创始人孵化器

#### 🎨 交互式演示

- **交互式演示**：[https://lyhu.github.io/incoming/palantir/palantir_visualization.html](https://lyhu.github.io/incoming/palantir/palantir_visualization.html)
- **主题**：Palantir 如何从咨询公司蜕变为千亿美金巨头
- **核心内容**：
  - 多角色视角（部门负责人、产品经理、架构师、研发人员）
  - 80% 利润率：从咨询到软件平台的商业模式跃迁
  - 30% 创业转化率：员工成为创始人的比例
  - FDE 一线部署工程师模式
  - Ontology 本体层架构设计

#### 📖 原文链接

- **链接**：[Palantir 创始人孵化器（微信公众号）](https://mp.weixin.qq.com/s/r8PtewR1ORyCJaVyYFSbsw)

### 2. Claude Tool Search Tool 深度解析

#### 📄 技术文档

- **文件**：[Claude*Tool_Search_Tool*深度解析.md](./tool_search_tool/Claude_Tool_Search_Tool_深度解析.md)
- **主题**：Anthropic Claude 平台的动态工具发现与按需加载机制
- **核心内容**：
  - Token 消耗优化策略（85% Token 节省）
  - `defer_loading` 机制工作原理
  - 协议消息格式详解（`server_tool_use`、`tool_search_tool_result` 等）
  - MCP Server 托管模式 vs 标准 API 模式
  - 11 步完整时序流程解析
  - 工具选择准确率提升（49% → 74%）

#### 🎨 交互式演示

- **项目主页**：[https://lyhu.github.io/incoming](https://lyhu.github.io/incoming)
- **Palantir 可视化**：[https://lyhu.github.io/incoming/palantir/palantir_visualization.html](https://lyhu.github.io/incoming/palantir/palantir_visualization.html)
- **Tool Search Tool 演示**：[https://lyhu.github.io/incoming/tool_search_tool](https://lyhu.github.io/incoming/tool_search_tool)

#### 📋 产品需求文档（AI 生成）

- **文件**：[tool_search_tool/prd.md](./tool_search_tool/prd.md)
- **内容**：演示页面的完整技术规格与验收标准

## 未来规划

### 短期目标

- [ ] 添加更多 AI Agent 相关技术解析（如 Prompt Engineering、RAG 架构等）
- [ ] 扩展交互式演示库（支持更多技术场景）
- [ ] 增加代码示例仓库（Python/JavaScript/Go 实现）
- [ ] 建立主题索引与标签分类系统

### 中期目标

- [ ] 构建完整的 AI 工具链知识图谱
- [ ] 添加技术对比分析系列（如不同 LLM 工具调用方案对比）
- [ ] 集成在线代码运行环境（CodePen 嵌入）
- [ ] 建立技术术语词汇表

## 贡献指南

本项目目前为个人学习记录项目。如您有改进建议或发现问题，欢迎通过以下方式参与：

1. **提交 Issue**：报告错误或提出功能建议
2. **Pull Request**：贡献代码或文档改进
3. **技术讨论**：在 Discussions 区域分享见解

## 许可证

本项目采用 [MIT License](LICENSE)，允许自由使用、修改与分发。

---

## 项目元信息

- **创建日期**：2026-01-18
- **主要语言**：中文
- **更新频率**：持续更新
- **联系方式**：通过 GitHub Issues 联系

---

**技术驱动，持续精进 | Technology-Driven, Continuous Improvement**
