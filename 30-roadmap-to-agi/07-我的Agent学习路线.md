# 07 - 我的 Agent 学习路线

> **TL;DR**：这是我个人（一个深度使用 AI Agent 的工程师）的实际学习路线，**不空谈**。从基础到进阶，分阶段给出具体的学习目标、资源、实战项目。

---

## 一、我的基础情况

- 角色：AI 工程师 + 产品负责人。
- 背景：10+ 年软件开发经验。
- 当前 AI 使用深度：重度（每天 4-6 小时 AI 协作）。
- 当前 Agent 实践：OpenClaw 多 Agent 编排 + 飞书 Agent 集成。

**前提**：这个路线适合**有编程基础**的人。如果你不是工程师，可以跳过技术细节，专注于 AI 工具使用和工作流重塑。

---

## 二、阶段 0：基础准备（0-3 个月）

### 目标

熟练使用 2-3 个 AI 工具，建立日常 AI 工作流。

### 学习内容

#### 1. ChatGPT / Claude 基础使用

**学习目标**：
- 熟练对话交互。
- 掌握基础 Prompt 技巧（角色设定、Few-shot、思维链）。
- 理解 LLM 的能力边界。

**实战项目**：
- 用 ChatGPT 重写 10 封邮件。
- 用 Claude 翻译 5 篇技术文档。
- 用 LLM 总结 5 个长报告。

#### 2. Cursor 基础使用

**学习目标**：
- 熟练 Composer（跨文件编辑）。
- 掌握 Tab 补全。
- 学会用 @ 符号引用上下文。

**实战项目**：
- 用 Cursor 重构一个 1000 行代码的小项目。
- 用 Composer 实现一个新功能。
- 用 Cursor 写测试用例。

#### 3. Claude Code 基础使用

**学习目标**：
- 熟悉 CLI 工作流。
- 掌握脚本化 AI 调用。
- 学会批处理任务。

**实战项目**：
- 用 Claude Code 批量改 30 个文件的命名。
- 用 Claude Code 写自动化脚本。
- 用 Claude Code 跑数据清洗。

### 验收标准

- 每天使用 AI 工具 2+ 小时。
- 提效 3 倍以上。
- 建立个人"AI 工具栈"。

---

## 三、阶段 1：进阶使用（3-6 个月）

### 目标

深入掌握 Prompt 工程，建立 AI-native 工作流。

### 学习内容

#### 1. 高级 Prompt 工程

**学习主题**：
- ReAct 模式（推理 + 行动）。
- Chain-of-Thought（思维链）。
- Self-Consistency（自我一致性）。
- Tool-use（Function Calling）。

**实战项目**：
- 实现一个"研究助手"（搜索 + 总结 + 引用）。
- 实现一个"代码审查助手"（自动 PR review）。
- 实现一个"数据分析助手"（SQL 自动生成 + 执行 + 解释）。

#### 2. RAG（检索增强生成）

**学习主题**：
- 向量数据库（Pinecone、Weaviate、Milvus）。
- Embedding 模型选型。
- 检索策略（语义检索、混合检索）。
- 文档分块与索引。

**实战项目**：
- 构建企业文档问答系统。
- 构建个人知识库问答助手。
- 构建代码库问答助手。

#### 3. Memory 与上下文管理

**学习主题**：
- 短期记忆 vs 长期记忆。
- Memory Bank 设计。
- 上下文压缩技术。

**实战项目**：
- 构建一个有"记忆"的客服 Agent。
- 构建一个有"学习能力"的个人助手。

### 验收标准

- 能独立设计复杂的 Prompt。
- 能实现一个完整的 RAG 系统。
- 能构建有"记忆"的 Agent。

---

## 四、阶段 2：Agent 工程化（6-12 个月）

### 目标

掌握 Agent 工程化的核心技术，能构建生产级 Agent 系统。

### 学习内容

#### 1. Agent 框架

**学习框架**：
- LangGraph（LangChain 团队）。
- OpenClaw（多 Agent 编排）。
- AutoGen（微软）。
- CrewAI（角色扮演多 Agent）。

**实战项目**：
- 用 LangGraph 实现一个"研究 Agent"。
- 用 OpenClaw 实现一个"软件开发多 Agent 团队"。
- 用 CrewAI 实现一个"营销活动编排 Agent"。

#### 2. Agent 工程化模式

**学习主题**：
- 质量门禁（Quality Gate）。
- 可观测性（Tracing、Metrics、Logging）。
- 异常恢复（自愈机制）。
- 错误分类与重试策略。

**实战项目**：
- 给 Agent 添加完整的可观测性。
- 实现自动质量门禁。
- 实现异常恢复机制。

#### 3. 工具生态与 MCP

**学习主题**：
- MCP（Model Context Protocol）。
- Function Calling 标准化。
- 工具市场设计。

**实战项目**：
- 把自己的 API 封装为 MCP Server。
- 集成第三方 MCP Server。
- 构建 Agent 工具市场。

### 验收标准

- 能构建生产级 Agent（7×24 稳定运行）。
- 能设计 Agent 工程化体系（监控、调试、恢复）。
- 能封装 MCP Server。

---

## 五、阶段 3：多 Agent 编排（12-18 个月）

### 目标

深入掌握多 Agent 编排，能设计复杂的多 Agent 系统。

### 学习内容

#### 1. 多 Agent 架构

**学习主题**：
- Orchestrator + Specialist 模式。
- 角色分工与职责边界。
- 任务分解与依赖图。
- 调度决策引擎。

**实战项目**：
- 构建完整的"AI 软件开发团队"（基于 OpenClaw）。
- 构建"AI 营销团队"。
- 构建"AI 法务团队"。

#### 2. 工作流编排

**学习主题**：
- Lobster DSL（OpenClaw）。
- 工作流可视化。
- 错误处理与重试。
- HITL（人工介入）节点。

**实战项目**：
- 设计复杂工作流（10+ 步骤）。
- 实现条件分支与循环。
- 实现 HITL 审批流程。

#### 3. 跨 Agent 通信与契约

**学习主题**：
- 产物驱动通信。
- 契约设计（Schema、Quality Gates）。
- 共享存储策略。
- 版本管理。

**实战项目**：
- 设计完整的 Agent 契约体系。
- 实现产物溯源系统。

### 验收标准

- 能设计完整的多 Agent 系统架构。
- 能编排复杂工作流。
- 能处理多 Agent 的工程化挑战。

---

## 六、阶段 4：前沿探索（18 个月+）

### 目标

跟踪前沿研究，尝试实验性方向。

### 学习主题

#### 1. Self-improvement（自我改进）

- Agent 自动调优 Prompt。
- 模型蒸馏与压缩。
- Self-play 训练。

#### 2. Agent Society（智能体社会）

- 动态角色分工。
- 涌现行为。
- 跨领域迁移。

#### 3. 多模态 Agent

- 视觉理解。
- 语音交互。
- 视频处理。

### 实战项目

- 实验 Self-improvement 框架。
- 构建多模态 Agent Demo。
- 跟踪最新论文，开源自己的实验。

---

## 七、推荐学习资源

### 必读论文

1. **"ReAct: Synergizing Reasoning and Acting in Language Models"**（Yao et al., 2022）
2. **"Chain-of-Thought Prompting Elicits Reasoning in Large Language Models"**（Wei et al., 2022）
3. **"Toolformer: Language Models Can Teach Themselves to Use Tools"**（Schick et al., 2023）
4. **"AutoGPT: An Autonomous GPT-4 Experiment"**（Significant Gravitas, 2023）
5. **"MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework"**（Hong et al., 2023）

### 必看视频

1. **Andrej Karpathy: "Intro to Large Language Models"**
2. **LangChain: "Agents and Multi-Agent Systems"**
3. **OpenAI: "Function Calling and Tool Use"**
4. **Anthropic: "Model Context Protocol"**

### 必用工具

- **Cursor**：AI 增强的 IDE。
- **Claude Code**：AI 命令行。
- **OpenClaw**：多 Agent 编排。
- **LangSmith**：Agent 可观测性。
- **MCP Servers**：工具生态。

### 必关注社区

- LangChain Discord。
- Anthropic Discord。
- r/LocalLLaMA（开源模型）。
- Hacker News（AI 板块）。
- X / Twitter（关注 AI 研究者）。

---

## 八、实战项目清单

按优先级排序：

### 项目 1：个人 AI 助手（入门）

**目标**：用 LangChain + RAG 构建个人知识库问答助手。

**时间**：2-4 周。

**价值**：掌握 RAG 基础。

### 项目 2：自动代码审查 Agent（进阶）

**目标**：用 LangGraph + GitHub API 构建自动 PR Review Agent。

**时间**：4-6 周。

**价值**：掌握 Agent 工程化。

### 项目 3：会议 Agent（实战）

**目标**：用飞书 API + OpenClaw 构建会议 → 任务闭环 Agent。

**时间**：4-8 周。

**价值**：掌握企业级场景。

### 项目 4：多 Agent 软件开发团队（高阶）

**目标**：用 OpenClaw 构建完整的"AI 软件开发团队"。

**时间**：8-12 周。

**价值**：掌握多 Agent 编排。

### 项目 5：垂直 Agent 产品化（创业级）

**目标**：选择一个垂直场景（如法律、销售、医疗），构建完整的 Agent 产品。

**时间**：3-6 个月。

**价值**：掌握 Agent 产品化全流程。

---

## 九、学习时间分配建议

按周分配（10 小时/周）：

```
Prompt 工程基础：2 小时
工具使用练习：2 小时
阅读/视频学习：2 小时
实战项目：3 小时
社区交流：1 小时
```

按月分配：

```
本月主题深入：60% 时间
下月主题预览：20% 时间
实战项目：20% 时间
```

按季度分配：

```
Q1：基础（阶段 0-1）
Q2：进阶（阶段 1-2）
Q3：工程化（阶段 2-3）
Q4：实战项目（端到端）
```

---

## 十、避免的学习陷阱

### 陷阱 1：只学不用

学了一堆框架和工具，但没在真实项目用过。

**避免**：每个学习主题必须有实战项目。

### 陷阱 2：只追新技术

每个新框架、新工具都学一遍，结果什么都不深入。

**避免**：选 1-2 个核心框架深入。

### 陷阱 3：忽视工程化

只关注模型能力，不关注生产稳定性。

**避免**：把可观测性、监控、错误恢复作为核心技能。

### 陷阱 4：忽视业务场景

学了技术但找不到应用场景。

**避免**：每个学习阶段都绑定一个真实业务问题。

### 陷阱 5：单兵作战

自己一个人学，没有交流。

**避免**：加入社区、找学习伙伴、参与开源。

---

## 十一、本章关键 takeaway

- 学习路线分 5 个阶段：基础 → 进阶 → 工程化 → 编排 → 前沿。
- 每个阶段有明确的学习目标和实战项目。
- 必读 5 篇论文，必看 4 个视频，必用 5 个工具。
- 学习时间分配：60% 深入 + 20% 预览 + 20% 实战。
- 避免 5 个陷阱：只学不用、追新、忽视工程化、忽视业务、单兵作战。

---

**上一篇**：[06 - 从工具到同事到自主体](./06-从工具到同事到自主体.md)
**返回**：[30-roadmap-to-agi README](./README.md)
