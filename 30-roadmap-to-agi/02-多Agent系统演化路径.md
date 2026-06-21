# 02 - 多 Agent 系统演化路径

> **TL;DR**：多 Agent 系统不是"多个 Agent 简单相加"，是**演化路径**——单体 Agent → 协作 Agent → 角色分工 → 自治网络 → Agent Society。每一步都解决前一步的局限性。

---

## 一、为什么需要多 Agent

单体 Agent 在很多场景下够用，但遇到以下情况会"卡壳"：

### 卡壳 1：上下文超载

把"做一个完整电商系统"丢给单体 Agent，它要把产品、UI、架构、前后端、测试、部署**所有领域知识**塞进上下文。结果：

- 关键信息被淹没。
- Token 成本爆炸。
- 推理质量下降。

### 卡壳 2：专业化不足

单体 Agent 试图"通吃所有领域"，每个领域都只能做浅。表现是"看起来做了，每个环节都不专业"。

### 卡壳 3：无法并行

10 个步骤串行执行。3 个小时才能跑完。

### 卡壳 4：错误扩散

单体 Agent 出错时，从头再来。

---

## 二、五个演化阶段

```
阶段 1：Single Agent（单体 Agent）
  - 1 个 Agent 处理所有事
  - 优点：简单、上下文统一
  - 缺点：上下文爆炸、专业化不足

阶段 2：Multi-Agent Collaboration（协作 Agent）
  - 多个 Agent 协作，Orchestrator 调度
  - 优点：专业化、可并行
  - 缺点：通信开销、契约不一致

阶段 3：Role-based Multi-Agent（角色分工）
  - 每个 Agent 有明确职责（产品/UI/架构/编码/测试/部署）
  - 优点：职责清晰、可维护
  - 缺点：需要标准化 SOP

阶段 4：Self-Organizing Multi-Agent（自治网络）
  - Agent 自己组织、自己协商
  - 优点：灵活性高
  - 缺点：难以预测、难以控制

阶段 5：Agent Society（智能体社会）
  - Agent 形成类似社会的结构（角色、文化、规范）
  - 优点：涌现行为、跨领域迁移
  - 缺点：理论探索阶段
```

---

## 三、阶段 1：Single Agent

### 形态

一个 LLM + 工具调用 + 记忆 + 工作流。处理端到端任务。

### 典型实现

```python
# Single Agent
agent = Agent(
    llm=llm_client,
    tools=[search, calculator, code_runner],
    memory=ConversationBufferMemory()
)

# 处理任务
result = agent.run("设计一个电商网站")
```

### 适用场景

- 任务能在 8K-32K token 内完成。
- 不需要跨领域深度知识。
- 用户接受"通才"输出。

### 局限

- 长任务上下文超载。
- 跨领域任务质量下降。
- 无法并行。

---

## 四、阶段 2：Multi-Agent Collaboration

### 形态

多个 Agent 通过 Orchestrator 协调，每个 Agent 关注任务的一个方面。

### 典型架构

```
Orchestrator（调度）
    ├── Research Agent（调研）
    ├── Code Agent（编码）
    ├── Test Agent（测试）
    └── Review Agent（审查）
```

### 关键设计

- **Orchestrator 唯一决策**：避免多头决策冲突。
- **共享存储**：Agent 间通过共享存储通信。
- **契约明确**：每个 Agent 的输入输出契约清晰。

### 适用场景

- 任务可分解为多个子任务。
- 子任务间有明确依赖。
- 每个子任务需要的工具/知识不同。

### 局限

- Orchestrator 是单点瓶颈。
- Agent 间通信开销。
- 需要精细的契约管理。

---

## 五、阶段 3：Role-based Multi-Agent

### 形态

基于真实世界角色分工的多 Agent 团队。每个 Agent 模拟一个真实岗位。

### 典型架构

```
Orchestrator（项目经理）
    ├── Product Agent（产品经理）
    ├── UI Designer（UI 设计师）
    ├── Architect（架构师）
    ├── Coder Frontend（前端开发）
    ├── Coder Backend（后端开发）
    ├── QA（测试工程师）
    └── DevOps（运维工程师）
```

### 关键设计

- **SOUL.md**：每个 Agent 模拟真实岗位的心智（详见 [`../10-tech-multi-agent/03-Agent角色分工与职责边界.md`](../10-tech-multi-agent/03-Agent角色分工与职责边界.md)）。
- **专业工具**：每个 Agent 有自己领域的工具集。
- **质量门禁**：每个 Agent 的产出都有质量标准。
- **工作流驱动**：Lobster DSL 把流程写成可执行 YAML。

### 适用场景

- 标准化、有 SOP 的任务。
- 跨领域需要深度专业知识的任务。
- 团队已有成熟分工的领域（软件开发、市场营销、客户服务等）。

### 局限

- 角色固定，不灵活。
- 需要投入大量精力设计 SOUL.md。
- 任务分解严重依赖模板。

### 当前主流

OpenClaw、MetaGPT、AutoGen、LangGraph 都在这个阶段。

---

## 六、阶段 4：Self-Organizing Multi-Agent

### 形态

Agent 自己协商、分工、重组，没有固定的角色。

### 关键能力

- **动态角色**：根据任务动态选择角色。
- **动态协商**：Agent 之间协商谁做什么。
- **动态重组**：任务变化时 Agent 重新组队。

### 典型实验

```python
# AgentVerse 等学术项目的雏形
agents = [agent_a, agent_b, agent_c, agent_d]
task = "解决这个复杂问题"

# 让 Agent 自己组织
result = self_organize(agents, task)
# → Agent a 提议方案 → Agent b 评估 → Agent c 补充 → Agent d 总结
```

### 适用场景

- 任务类型多变、难以模板化。
- 需要跨领域创新。
- 探索性任务。

### 局限

- 行为难以预测。
- 调试困难。
- 资源消耗大（Agent 间多次对话）。
- 当前还在研究阶段。

---

## 七、阶段 5：Agent Society

### 形态

Agent 形成类似人类社会的结构：

- **角色（Roles）**：教师、学生、批评者、决策者。
- **文化（Culture）**：Agent 共享的价值观和行为规范。
- **规范（Norms）**：Agent 遵守的规则。
- **制度（Institutions）**：Agent 间的组织形式。

### 关键能力

- **涌现行为**：单个 Agent 没有，组合后涌现。
- **跨领域迁移**：在一个领域学到的能迁移到另一个领域。
- **自我进化**：Agent 网络能持续改进。

### 典型研究

- **Camel**：角色扮演型多 Agent。
- **AgentVerse**：动态多 Agent 协作。
- **MetaGPT**：软件工程多 Agent 团队。

### 当前状态

理论探索阶段。尚无成熟产品。

---

## 八、演化的关键驱动

### 驱动 1：任务复杂度

任务越复杂，越需要多 Agent。

### 驱动 2：上下文窗口

LLM 上下文窗口越大，单体 Agent 能处理的任务越多。但**专业化的价值不会消失**。

### 驱动 3：工具生态

工具越丰富，每个 Agent 能做的事越多。多 Agent 的价值在"组合"而非"单个能力"。

### 驱动 4：成本下降

LLM API 成本下降 → 多 Agent 的额外开销变得可接受。

### 驱动 5：工程化能力

可观测性、调试工具、契约框架的成熟 → 多 Agent 的工程门槛降低。

---

## 九、当前所处阶段 + 未来路径

**当前主流**：阶段 3（Role-based Multi-Agent）。

**1-2 年内**：阶段 4 的某些特性会进入产品（动态协商、动态角色）。

**3-5 年内**：阶段 5 的某些概念会进入实验（涌现、跨领域迁移）。

**5-10 年内**：AGI 雏形可能从多 Agent 网络涌现。

---

## 十、对实践者的建议

### 现在做多 Agent 系统

- **从阶段 3 起步**：基于角色分工的团队。
- **不要追求阶段 4、5**：技术不成熟。
- **重视工程化**：质量门禁、可观测、自愈。
- **垂直深耕**：先在一个领域做深。

### 未来 1-2 年

- 关注动态角色、动态协商的产品化。
- 关注 MCP（Model Context Protocol）标准化。
- 关注跨 Agent 通信协议。

### 未来 3-5 年

- 关注 Agent Society 概念的产品化。
- 关注 Agent 自我进化能力。
- 关注跨领域 Agent 网络。

---

## 十一、本章关键 takeaway

- 多 Agent 系统演化分五个阶段：Single → Collaboration → Role-based → Self-Organizing → Society。
- 当前主流是阶段 3（基于角色），OpenClaw 是典型代表。
- 演化的关键驱动：任务复杂度、上下文窗口、工具生态、成本下降、工程化能力。
- 现在做：从阶段 3 起步，重视工程化，垂直深耕。
- 未来关注：动态角色、MCP 标准化、跨领域 Agent 网络。

---

**上一篇**：[01 - AI Agent 发展阶段论](./01-AI-Agent发展阶段论.md)
**下一篇**：[03 - 未来 3 年关键技术演进](./03-未来3年关键技术演进.md)
