# 03 - Agent 角色分工与职责边界

> **TL;DR**：多 Agent 系统的核心是**"按职责分工"**。典型软件工程团队有 6 类 Agent：产品、UI、架构、编码、测试、部署。每类 Agent 的职责、输入、输出、契约都是**显式定义**的。

---

## 一、为什么 Agent 角色分工如此重要

如果所有 Agent 都"通才"：
- 每个 Agent 都要懂全部领域 → 上下文爆炸。
- 没有明确职责 → Agent 互相推诿。
- 调试困难 → 不知道哪个 Agent 该负责哪个产出。

**角色分工是降低系统复杂度的核心机制**。每个 Agent 只关心自己的领域，对其他领域一无所知。

---

## 二、6 类 Agent 的标准角色

### 角色 1：Product Agent（产品 Agent）

**职责**：把用户需求转化为结构化的产品需求文档（PRD）。

**输入**：
- 用户原始需求。
- Orchestrator 提供的项目上下文。

**输出**：
- `PRD.md`：产品需求文档（用户故事、验收标准、业务规则）。
- `UserStories.md`：用户故事列表。
- `AcceptanceCriteria.md`：验收标准。

**契约**：

```yaml
inputs:
  user_requirement: string
  project_context: object

outputs:
  PRD.md:
    format: markdown
    required_sections: ["Overview", "User Stories", "Acceptance Criteria", "Business Rules"]
    min_word_count: 500
  UserStories.md:
    format: markdown
    structure: list of {id, as_a, i_want, so_that}
  
guardrails:
  - "如果需求模糊，必须输出 ambiguity_flags 而不是猜测"
  - "不输出技术方案（这是 Architect 的职责）"
  - "不输出 UI 设计（这是 UI Designer 的职责）"
```

**典型 SOUL.md 关键片段**：

```markdown
## Identity
你是 Product Agent，专注于产品需求分析。

## Coding Standards
- 用户故事使用 "As a [role], I want [feature], so that [benefit]" 格式
- 验收标准使用 Given/When/Then 格式
- 不涉及技术实现细节

## Constraints
- 不写代码
- 不做技术选型
- 不设计 UI
```

---

### 角色 2：UI Designer Agent（UI 设计 Agent）

**职责**：把 PRD 转化为 UI 设计稿（线框图 + 交互规范 + 设计系统）。

**输入**：
- `PRD.md`：产品需求。
- `UserStories.md`：用户故事。

**输出**：
- `DesignSystem.md`：色彩、字体、间距、组件库。
- `Wireframes/`：每个页面的线框图（Markdown + ASCII 图）。
- `InteractionSpecs.md`：交互状态机、手势、动画。

**契约**：

```yaml
inputs:
  PRD.md: required
  UserStories.md: required

outputs:
  DesignSystem.md:
    sections: ["Colors", "Typography", "Spacing", "Components"]
  Wireframes/[PageName].md:
    required_elements: ["布局结构", "组件清单", "响应式规则", "空/加载/错误状态"]
  InteractionSpecs.md:
    coverage: "覆盖所有用户故事的交互路径"

guardrails:
  - "不输出 API 或数据库设计（这是 Architect 的职责）"
  - "不写业务逻辑代码"
  - "必须覆盖 PRD 中所有页面"
```

---

### 角色 3：Architect Agent（架构设计 Agent）

**职责**：设计技术架构、API、数据模型、技术选型。

**输入**：
- `PRD.md`：产品需求。
- `UserStories.md`：用户故事。

**输出**：
- `Architecture.md`：系统架构图、技术选型、部署方案。
- `TechStack.md`：技术栈清单（前后端框架、数据库、第三方服务）。
- `APISpec.yaml`：API 接口规范（OpenAPI 3.0）。
- `Schema.sql`：数据库 Schema。

**契约**：

```yaml
inputs:
  PRD.md: required
  UserStories.md: required

outputs:
  Architecture.md:
    required_sections: ["系统架构图", "技术选型理由", "部署方案", "风险与权衡"]
  APISpec.yaml:
    format: "OpenAPI 3.0"
    coverage: "覆盖 PRD 中所有核心功能"
  Schema.sql:
    format: "PostgreSQL/MySQL DDL"

guardrails:
  - "API 必须有清晰的版本号"
  - "数据库 Schema 必须有索引设计"
  - "技术选型必须有理由，不能随便选"
```

---

### 角色 4：Coder Agent（编码 Agent）

**职责**：基于 PRD、UI Spec、API Spec 实现代码。

Coder 内部可以再细分为 **Frontend Coder** 和 **Backend Coder**：

#### Frontend Coder

**输入**：
- `Wireframes/`：UI 线框图。
- `DesignSystem.md`：设计系统。
- `APISpec.yaml`：API 规范。

**输出**：
- `/frontend/`：前端代码。
- `frontend/README.md`：使用说明。

#### Backend Coder

**输入**：
- `APISpec.yaml`：API 规范。
- `Schema.sql`：数据库 Schema。

**输出**：
- `/backend/`：后端代码。
- `backend/README.md`：API 使用说明。

**共同契约**：

```yaml
guardrails:
  - "必须跑通 lint"
  - "必须有关键单元测试"
  - "必须写 README"
  - "API 实现必须严格匹配 APISpec.yaml"
```

---

### 角色 5：QA / Tester Agent（测试 Agent）

**职责**：执行测试，输出测试报告和 Bug 报告。

**输入**：
- `/frontend/`、`/backend/`：待测试代码。
- `APISpec.yaml`：API 规范。
- `AcceptanceCriteria.md`：验收标准。

**输出**：
- `TestReport.md`：测试执行报告。
- `BugReport.md`：Bug 列表（按严重程度排序）。
- `CodeReview.md`：代码审查意见。

**契约**：

```yaml
outputs:
  TestReport.md:
    required_sections: ["覆盖范围", "测试结果", "失败用例"]
  BugReport.md:
    structure: list of {
      id, severity, title, description,
      reproduction_steps, expected, actual, related_files
    }

guardrails:
  - "Bug 报告必须可复现"
  - "测试覆盖率必须 > 80%"
  - "critical bug 必须 0 个才能通过门禁"
```

---

### 角色 6：DevOps / Deployer Agent（部署 Agent）

**职责**：打包、部署、CI/CD 配置、运维脚本。

**输入**：
- `/frontend/`、`/backend/`：代码。
- `Architecture.md`：部署方案。

**输出**：
- `Dockerfile`：镜像构建。
- `docker-compose.yml`：本地编排。
- `.github/workflows/`：CI/CD 配置。
- `DeploymentGuide.md`：部署文档。

**契约**：

```yaml
guardrails:
  - "部署脚本必须幂等（重复执行结果一致）"
  - "必须有健康检查端点"
  - "必须有日志和监控配置"
```

---

## 三、角色边界 vs 角色重叠

### 边界清晰的场景

- **PRD vs 架构**：PRD 写"做什么"，架构写"怎么做"。不重叠。
- **UI vs 架构**：UI 写"长什么样"，架构写"数据怎么流转"。不重叠。
- **编码 vs 测试**：编码写"怎么实现"，测试写"是否符合预期"。互补。

### 边界模糊的场景

- **PRD vs 架构**：PRD 偶尔要提技术约束（如"必须支持 iOS 12"），架构要基于这些约束。
- **UI vs 前端**：UI 写线框图，前端写代码实现。但 UI 经常要包含"组件交互逻辑"。
- **编码 vs 架构**：编码时可能发现架构不合理的细节，反馈回架构调整。

### 应对方法

**明确"决策权归属"**：
- PRD 模糊时，**产品 Agent 主动澄清**，不替架构做决定。
- UI 与前端冲突时，**UI Spec 是真理**，前端必须按 UI Spec 实现。
- 编码发现架构问题，**通过 BUG_REPORT 反馈**，让架构 Agent 决定是否改架构。

**共享契约文档**：
- `CONTRACT.md` 显式列出每个 Agent 的输入输出契约。
- 任何 Agent 修改自己的契约必须发版通知。

---

## 四、Agent 数量与团队规模

### 最小团队（3 Agent）

```
Orchestrator
├── Coder (通才，前端后端都做)
└── Reviewer (QA + 部署)
```

适合：MVP、小项目。

### 标准团队（5 Agent）

```
Orchestrator
├── Product
├── Coder (通才)
├── Tester
└── Reviewer
```

适合：中等规模全栈应用。

### 完整团队（7+ Agent）

```
Orchestrator
├── Product
├── UI Designer
├── Architect
├── Coder Frontend
├── Coder Backend
├── Tester
└── DevOps
```

适合：大型项目、多团队协作。

### 我的判断

- **不要从 7 Agent 起步**。从 3 Agent 跑通，逐步加。
- **每个新 Agent 必须解决"明确问题"**。不要为了"架构优雅"加 Agent。
- **Agent 数量超过 10 个时，通信开销开始大于专业化收益**。考虑合并角色。

---

## 五、角色间的协作流程（典型软件项目）

```
Wave 1: Product Agent → PRD.md
                  ↓
Wave 2: (并行) UI Designer Agent → Wireframes/
        (并行) Architect Agent → Architecture.md
                  ↓
Wave 3: Architect Agent → APISpec.yaml + Schema.sql
                  ↓
Wave 4: (并行) Coder Frontend → /frontend/
        (并行) Coder Backend → /backend/
                  ↓
Wave 5: Tester Agent → TestReport.md + BugReport.md
        (如果有 bug) → 回溯到对应 Agent 修复
                  ↓
Wave 6: DevOps Agent → Dockerfile + 部署
                  ↓
Wave 7: 人工最终验收
```

每一波次的产物都是下一波次的输入，**形成清晰的依赖链**。

---

## 六、Agent 角色 vs 人类角色

注意一个关键区别：

> **人类角色是"职能"，Agent 角色是"职责"。**

- 一个人类产品经理可以同时做 PRD、写需求文档、做用户研究、跟进开发进度（多种职能）。
- 一个 Agent Product Agent 只做 PRD 这一种职责。

Agent 角色是**单一职责**，人类角色是**复合职能**。这是 Agent 设计的关键约束。

但反过来：
- 一个人类编码可以同时做前端、后端、数据库（跨领域）。
- 一个 Coder Agent 也可能跨前后端，但通常我们会拆分 Frontend Coder 和 Backend Coder。

**取舍原则**：
- 跨领域会导致 Agent 上下文变大 → 拆分。
- 跨领域任务量很小 → 不拆分。
- 跨领域需要频繁切换 → 拆分。

---

## 七、本章关键 takeaway

- 6 类标准 Agent：Product、UI Designer、Architect、Coder、QA、DevOps。
- 每类 Agent 有明确的输入、输出、契约、guardrails。
- 边界模糊时靠"决策权归属 + 共享契约文档"处理。
- 团队规模从 3 Agent 起步，逐步扩展。
- Agent 角色是单一职责，人类角色是复合职能。

---

**上一篇**：[02 - Orchestrator 设计模式详解](./02-Orchestrator设计模式详解.md)
**下一篇**：[04 - 任务分解与依赖图构建](./04-任务分解与依赖图构建.md)
