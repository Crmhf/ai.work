# 06 - Agent 间通信协议与产物驱动

> **TL;DR**：多 Agent 系统的通信不是"对话"，是**"产物驱动"**。Agent A 完成工作把产物写到共享存储，Agent B 读这个产物继续。这种设计把通信开销降到最低，把系统复杂度降到最低。

---

## 一、通信模式对比

### 模式 1：直接对话（Chat-based）

Agent A 和 Agent B 直接发消息：

```
A → B: "请帮我实现登录功能"
B → A: "好的，我先看看 API Spec"
B → A: "已经实现，请查看 PR #123"
```

**问题**：
- 通信开销大（每次对话都重传上下文）。
- 调试困难（消息链路长）。
- 耦合紧（Agent A 必须等 Agent B 回复）。

### 模式 2：产物驱动（Artifact-driven）

Agent A 完成工作把产物写到共享存储，Agent B 自己去读：

```
A: 写入 shared/03_Arch/APISpec.yaml（API Spec 文档）
   ↓
B: 读取 shared/03_Arch/APISpec.yaml，自己实现
B: 写入 /backend/（代码）
```

**优势**：
- 通信开销最小（不传上下文，只传产物）。
- 调试容易（看共享存储就知道每个 Agent 做了什么）。
- 解耦（A 不需要等 B，只需写完产物即可）。
- 可重放（B 可以重读产物重新开始）。

**这是 OpenClaw 多 Agent 编排的核心通信模式。**

---

## 二、OpenClaw 通信协议栈

```
┌─────────────────────────────────────────────┐
│           应用层：项目专用消息格式              │
│    {"type": "TASK_ASSIGN", "payload": {...} }│
├─────────────────────────────────────────────┤
│           会话层：OpenClaw sessions_send      │
│    跨 Agent 会话消息路由                      │
├─────────────────────────────────────────────┤
│           传输层：agentToAgent 工具调用        │
│    Gateway WebSocket 路由                     │
├─────────────────────────────────────────────┤
│           网络层：Gateway 进程间通信           │
│    本地 IPC / 远程 WebSocket                  │
└─────────────────────────────────────────────┘
```

### 应用层：项目专用消息

Orchestrator 与 Specialist 之间用结构化消息通信：

```typescript
interface AgentMessage {
  message_id: string;        // UUID
  timestamp: string;         // ISO 8601
  correlation_id: string;    // 关联上游消息（追踪链）
  
  type:
    | "TASK_ASSIGN"      // Orchestrator → Worker：分配任务
    | "TASK_ACCEPT"      // Worker → Orchestrator：接受任务
    | "TASK_REJECT"      // Worker → Orchestrator：拒绝任务（附原因）
    | "PROGRESS_UPDATE"  // Worker → Orchestrator：进度汇报（0-100%）
    | "DELIVERABLE"      // Worker → Orchestrator：产物交付
    | "QUALITY_REPORT"   // QA → Orchestrator：质量检查结果
    | "BUG_REPORT"       // QA → Coder：Bug 报告
    | "CLARIFICATION"    // Any → Orchestrator：需要澄清/阻塞
    | "APPROVAL_REQUEST" // Orchestrator → Human：需要人工审批
    | "HUMAN_FEEDBACK";  // Human → Orchestrator：人工反馈
  
  from: string;    // Agent ID
  to: string;      // Agent ID or "orchestrator" or "human"
  
  payload: {
    // TASK_ASSIGN 示例
    task_id?: string;
    task_description?: string;
    inputs?: string[];          // 输入产物路径列表
    expected_outputs?: string[]; // 期望产出物路径列表
    deadline?: string;          // 建议完成时间
    
    // DELIVERABLE 示例
    artifacts?: Artifact[];
    summary?: string;
    
    // BUG_REPORT 示例
    severity?: "critical" | "major" | "minor";
    description?: string;
    reproduction_steps?: string[];
    expected_behavior?: string;
    actual_behavior?: string;
    related_files?: string[];
  };
}

interface Artifact {
  path: string;         // 文件路径
  type: "document" | "code" | "image" | "config" | "test";
  description: string;  // 内容摘要
  checksum: string;       // 完整性校验（sha256:...）
}
```

### 会话层：OpenClaw sessions_send

Orchestrator 通过 `sessions_send` 把消息路由到目标 Agent：

```python
# Orchestrator 调用 UI Designer Agent
await openclaw.sessions_send(
    from_session="orchestrator",
    to_session="ui_designer",
    message={
        "type": "TASK_ASSIGN",
        "task_id": "T2.2",
        "task_description": "...",
        "inputs": ["shared/01_PRD/PRD.md"],
        "expected_outputs": ["shared/02_UI/Wireframes/HomePage.md"]
    }
)
```

### 传输层：agentToAgent 工具

OpenClaw Gateway 提供 `agentToAgent` 工具调用，底层是 WebSocket 路由。

---

## 三、典型通信场景

### 场景 1：任务分配

**Orchestrator → UI Designer**：

```yaml
message:
  type: "TASK_ASSIGN"
  from: "orchestrator"
  to: "ui_designer"
  correlation_id: "wave_2_batch_1"
  payload:
    task_id: "T2.2"
    task_description: |
      基于已批准的 PRD 和 DesignSystem，设计电商小程序首页。
      要求：
      1. 包含商品轮播、分类入口、推荐商品列表
      2. 适配微信小程序安全区域
      3. 输出线框图 + 交互说明文档
    inputs:
      - "shared/01_PRD/PRD.md"
      - "shared/02_UI/DesignSystem.md"
    expected_outputs:
      - "shared/02_UI/Wireframes/HomePage.md"
      - "shared/02_UI/Wireframes/HomePage_Interaction.md"
    deadline: "2026-06-21T14:00:00Z"
```

### 场景 2：产物交付

**UI Designer → Orchestrator**：

```yaml
message:
  type: "DELIVERABLE"
  from: "ui_designer"
  to: "orchestrator"
  correlation_id: "wave_2_batch_1"
  payload:
    task_id: "T2.2"
    summary: "首页设计完成，包含3种状态（加载/空/有数据）"
    artifacts:
      - path: "shared/02_UI/Wireframes/HomePage.md"
        type: "document"
        description: "首页线框图，含布局、组件、响应式规则"
        checksum: "sha256:abc..."
      - path: "shared/02_UI/Wireframes/HomePage_Interaction.md"
        type: "document"
        description: "交互状态机、手势、动画说明"
        checksum: "sha256:def..."
```

### 场景 3：Bug 报告

**QA → Coder（经 Orchestrator 路由）**：

```yaml
message:
  type: "BUG_REPORT"
  from: "qa"
  to: "coder_frontend"
  correlation_id: "test_cycle_1"
  payload:
    task_id: "T5.1"
    severity: "major"
    title: "购物车数量更新后总价未同步刷新"
    description: |
      在商品详情页添加商品到购物车后，进入购物车页面修改数量，
      页面总价显示未随数量变化而更新，需手动下拉刷新后才正确。
    reproduction_steps:
      - "进入商品详情页，点击'加入购物车'"
      - "切换到购物车 Tab"
      - "点击某商品数量 '+' 按钮增加数量"
      - "观察底部总价区域"
    expected_behavior: "总价应实时随数量变化重新计算"
    actual_behavior: "总价保持旧值，下拉刷新后更新"
    related_files:
      - "frontend/pages/cart/index.tsx"
      - "frontend/stores/cartStore.ts"
```

### 场景 4：澄清请求

**Coder → Orchestrator**：

```yaml
message:
  type: "CLARIFICATION"
  from: "coder_backend"
  to: "orchestrator"
  correlation_id: "wave_4_T4.1"
  payload:
    task_id: "T4.1"
    question: |
      APISpec.yaml 中 /api/orders 接口的返回结构与 Schema.sql 中的 orders 表不一致。
      请问是按 API Spec 实现还是按 Schema.sql 实现？
    context:
      api_spec: "shared/03_Arch/APISpec.yaml:lines 145-167"
      schema: "shared/03_Arch/Schema.sql:lines 89-102"
```

---

## 四、共享存储策略

### 目录结构

```
shared/
├── 01_PRD/
│   ├── PRD.md
│   ├── UserStories.md
│   └── AcceptanceCriteria.md
├── 02_UI/
│   ├── DesignSystem.md
│   ├── Wireframes/
│   │   ├── HomePage.md
│   │   ├── ProductDetail.md
│   │   └── CartCheckout.md
│   └── InteractionSpecs.md
├── 03_Arch/
│   ├── Architecture.md
│   ├── APISpec.yaml
│   └── Schema.sql
├── 04_Code/
│   ├── frontend/
│   └── backend/
├── 05_QA/
│   ├── TestReport.md
│   └── BugReport.md
└── 06_Deploy/
    ├── Dockerfile
    ├── docker-compose.yml
    └── DeploymentGuide.md
```

### 存储策略

| 数据类型 | 存储位置 | 访问模式 | 生命周期 |
|---|---|---|---|
| 结构化产物（PRD/设计/架构） | `shared/` + Git | 只读（下游）/ 追加（上游） | 项目全程 |
| 代码（前端/后端） | `/frontend/` `/backend/` + Git | 读写（编码 Agent） | 项目全程 |
| 临时产物（中间计算、临时文件） | `/tmp/` | 读写 | 单次任务 |
| 测试报告 | `shared/05_QA/` + Git | 只读（Orchestrator） | 项目全程 |
| 部署产物 | `shared/06_Deploy/` + Git | 只读（DevOps） | 项目全程 |

### 访问规则

1. **下游只读上游产物**：UI Designer 不能修改 PRD.md。
2. **上游可追加自己的产物**：UI Designer 可以修改 DesignSystem.md，但不能修改别人的产物。
3. **Orchestrator 全权管理**：Orchestrator 可以读取所有产物，但**不直接修改**任何 Specialist 的产物（避免破坏责任归属）。

---

## 五、产物契约（Artifact Contract）

每个产物必须有明确的契约：

```yaml
# shared/02_UI/Wireframes/_contract.yaml
artifact: "shared/02_UI/Wireframes/HomePage.md"
owner: "ui_designer"
schema:
  type: "object"
  required:
    - "layout_structure"
    - "components"
    - "responsive_rules"
    - "states"  # 加载/空/有数据
  properties:
    layout_structure:
      type: "string"
      description: "页面整体布局（Markdown 描述 + ASCII 图）"
    components:
      type: "array"
      items:
        type: "object"
        required: ["name", "props", "states"]
    responsive_rules:
      type: "object"
      properties:
        mobile: { type: "string" }
        tablet: { type: "string" }
        desktop: { type: "string" }
    states:
      type: "array"
      items:
        enum: ["loading", "empty", "data", "error"]

quality_gates:
  - "layout_structure word_count >= 100"
  - "components.length >= 3"
  - "states.length == 4"

consumers:
  - "coder_frontend"  # 前端编码 Agent 读这个产物
```

契约由 Orchestrator 在启动任务前发布，Agent 按契约产出。如果 Agent 输出不符合契约，Orchestrator 不接收，要求重做。

---

## 六、通信的可观测性

### Trace 日志

每个 Agent 调用都有完整 trace：

```json
{
  "trace_id": "uuid-12345",
  "span_id": "uuid-67890",
  "parent_span_id": "uuid-11111",
  "agent": "ui_designer",
  "operation": "design_homepage",
  "task_id": "T2.2",
  "correlation_id": "wave_2_batch_1",
  "start_time": "2026-06-21T11:00:00Z",
  "end_time": "2026-06-21T11:25:00Z",
  "duration_seconds": 1500,
  "inputs": ["shared/01_PRD/PRD.md", "shared/02_UI/DesignSystem.md"],
  "outputs": ["shared/02_UI/Wireframes/HomePage.md"],
  "tokens": {"input": 4500, "output": 3200, "total": 7700},
  "status": "success",
  "messages": [
    {"type": "TASK_ASSIGN", "at": "2026-06-21T11:00:00Z"},
    {"type": "PROGRESS_UPDATE", "at": "2026-06-21T11:10:00Z", "progress": 50},
    {"type": "DELIVERABLE", "at": "2026-06-21T11:25:00Z"}
  ]
}
```

### Metrics

| 指标 | 说明 |
|---|---|
| 任务成功率 | COMPLETED / (COMPLETED + FAILED) |
| 平均任务时长 | 各任务 duration_seconds 的平均值 |
| Token 消耗 | 输入 + 输出 token 总量 |
| 重试率 | 重试次数 / 总任务数 |
| 通信延迟 | TASK_ASSIGN 到 DELIVERABLE 的时间 |

---

## 七、产物的版本管理

### Git 是产物存储的核心

- 所有 `shared/` 目录下的文件都进 Git。
- 每次 Orchestrator 调度 wave 前，先 commit 当前状态。
- Agent 修改产物后，commit 信息包含 task_id（如 `[T2.2] design homepage`）。
- 出错时可以 `git revert` 回滚到上一个稳定状态。

### 产物的不可变性

- 产物一旦交付（DELIVERABLE），Orchestrator 不再允许上游 Agent 修改。
- 修改必须通过"创建新版本"或"创建新任务"完成。
- 这样保证调试时可以复现 Agent 行为。

---

## 八、产物驱动 vs 直接对话：何时选哪个

| 场景 | 推荐模式 |
|---|---|
| Agent 间传递任务分配 | 应用层消息（TASK_ASSIGN） |
| Agent 间传递数据 | 产物驱动（共享存储） |
| Agent 间需要讨论 | 直接对话（但要短） |
| Agent 向 Orchestrator 报告进度 | 应用层消息（PROGRESS_UPDATE） |
| Agent 需要澄清需求 | 应用层消息（CLARIFICATION） |

**原则**：能产物驱动就别对话。能异步就别同步。

---

## 九、本章关键 takeaway

- 多 Agent 通信首选**产物驱动**（共享存储），不选直接对话。
- OpenClaw 提供 4 层协议栈：应用层消息、sessions_send、agentToAgent、Gateway。
- 典型消息类型：TASK_ASSIGN、DELIVERABLE、BUG_REPORT、CLARIFICATION、HITL。
- 共享存储按目录分类（PRD/UI/Arch/Code/QA/Deploy）。
- 每个产物有契约（schema、quality_gates、consumers）。
- 通信必须有 trace + metrics，便于调试。
- Git 是产物存储的核心，产物一旦交付不可修改。

---

**上一篇**：[05 - 调度决策引擎实现](./05-调度决策引擎实现.md)
**下一篇**：[07 - Lobster DSL 工作流编排](./07-Lobster-DSL工作流编排.md)
