# 02 - Orchestrator 设计模式详解

> **TL;DR**：Orchestrator 是多 Agent 系统的"大脑"。它的核心职责是**意图理解、任务分解、依赖分析、调度决策、状态管理、质量门禁、异常恢复**。这 7 个职责不是 7 个函数，是 7 个**互相耦合的能力模块**。

---

## 一、Orchestrator 的角色定位

Orchestrator（工程编排 Agent）在多 Agent 系统中扮演三种角色：

### 角色 1：翻译官

把用户的**模糊需求**（"做一个电商小程序"）翻译成**可执行的任务图**。

翻译需要：
- 意图理解：识别项目类型、规模、技术偏好。
- 需求澄清：识别歧义点，主动追问用户。
- 模板匹配：匹配预定义的"项目模板"（Web 模板、App 模板、API 模板）。

### 角色 2：调度官

把任务图**按依赖关系分波次执行**，每一波次并行跑。

调度需要：
- 依赖分析：构建 DAG，检测循环依赖。
- 波次划分：同层任务并行，跨层串行。
- 资源管理：避免 Agent 抢占、Token 超支。

### 角色 3：质量官

确保每个阶段产出**符合质量标准**，不符合则回溯修复。

质量需要：
- 门禁规则：每个阶段的可量化标准。
- 自动验证：脚本化检查（lint、test、schema 校验）。
- 升级机制：超过自动修复阈值则人工介入。

---

## 二、Orchestrator 的内部构造

```
┌─────────────────────────────────────────────┐
│          Orchestrator Agent                  │
│                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────┐ │
│  │ 意图理解器  │  │ 任务分解器  │  │模板库  │ │
│  └────────────┘  └────────────┘  └────────┘ │
│  ┌────────────┐  ┌────────────┐  ┌────────┐ │
│  │ 依赖分析器  │  │ 调度决策器  │  │执行器  │ │
│  └────────────┘  └────────────┘  └────────┘ │
│  ┌────────────┐  ┌────────────┐  ┌────────┐ │
│  │ 状态管理器  │  │ 质量门禁    │  │恢复器  │ │
│  └────────────┘  └────────────┘  └────────┘ │
│  ┌─────────────────────────────────────┐    │
│  │     共享存储 + 项目状态             │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

每个模块的职责：

### 意图理解器（Intent Analyzer）
- **输入**：用户原始需求文本。
- **输出**：结构化意图 JSON（项目类型、规模、技术偏好、约束、歧义点）。
- **实现**：调用 LLM 解析 + 模板匹配。

### 任务分解器（Task Decomposer）
- **输入**：结构化意图。
- **输出**：原子任务列表（每个 1-4 小时工作量）。
- **实现**：基于"项目模板 + LLM 实例化"。

### 模板库（Template Library）
- 预定义的"项目骨架"，如 `webapp_template.yaml`、`mobile_app_template.yaml`。
- 每个模板包含典型阶段、典型任务、典型依赖。

### 依赖分析器（Dependency Analyzer）
- **输入**：任务列表。
- **输出**：DAG（有向无环图）+ 拓扑排序。
- **实现**：DFS 检环 + 波次划分。

### 调度决策器（Scheduler）
- **输入**：DAG + 波次。
- **输出**：执行计划（哪一波次并行、串行、需审批）。
- **实现**：基于依赖特征自动选择模式（详见 `05-调度决策引擎实现.md`）。

### 执行器（Executor）
- **输入**：执行计划。
- **输出**：每个任务的执行结果。
- **实现**：通过 OpenClaw `sessions_send` 调度 Specialist Agent。

### 状态管理器（State Manager）
- 持久化项目状态（当前 wave、已完成任务、失败任务）。
- 支持断点续跑（Orchestrator 重启后从上次状态继续）。

### 质量门禁（Quality Gate）
- 每个阶段的可量化标准。
- 自动验证脚本（lint、test、schema）。
- 不通过则触发修复循环。

### 恢复器（Recovery Handler）
- 错误分类（瞬时错误、逻辑错误、需求错误）。
- 对应策略（重试、回溯、升级人工）。
- 3 次自动修复失败后升级人工。

---

## 三、Orchestrator 的 7 大核心职责详解

### 职责 1：意图理解（Intent Understanding）

**目标**：从用户模糊输入中提取结构化信息。

**典型实现**：

```python
def analyze_intent(requirement: str) -> Dict:
    prompt = f"""
    分析以下用户需求：
    
    需求: {requirement}
    
    输出 JSON：
    {{
        "project_type": "webapp|mobile_app|mini_program|api_service",
        "target_platform": ["web", "ios", "android"],
        "core_features": ["feature1", "feature2"],
        "user_scale": "small|medium|large",
        "tech_preferences": ["react", "node.js"],
        "constraints": ["budget_limit", "timeline"],
        "ambiguity_flags": ["unclear_auth_flow", "missing_payment_detail"]
    }}
    """
    return json.loads(llm.generate(prompt, format="json"))
```

**关键点**：
- 必须显式提取 `ambiguity_flags`，主动追问而非猜测。
- 项目类型必须从有限集合中选，便于匹配模板。
- 技术偏好若有冲突（如"既要 React 又要 Vue"），标记为 ambiguity。

### 职责 2：任务分解（Task Decomposition）

**目标**：从结构化意图生成原子任务列表。

**典型实现**：

```python
def decompose(intent: Dict) -> List[Task]:
    template = templates.get(intent["project_type"], templates["generic"])
    prompt = f"""
    基于模板 {template} 和项目意图 {intent}，
    生成具体的任务列表（每个 1-4 小时工作量）。
    输出 YAML。
    """
    tasks_yaml = llm.generate(prompt, format="yaml")
    return parse_tasks(tasks_yaml)
```

**关键点**：
- 每个任务必须"原子化"（1-4 小时）。
- 必须显式标注输入依赖（其他任务的输出）。
- 必须显式标注输出产物（其他任务的输入）。

### 职责 3：依赖分析（Dependency Analysis）

**目标**：构建 DAG，检测循环，划分波次。

**核心算法**（拓扑排序 + 波次划分）：

```python
def build_waves(tasks: List[Task]) -> List[List[Task]]:
    """入度为 0 的任务组成一个 wave，每轮减下游入度"""
    in_degree = {t.id: len(t.dependencies) for t in tasks}
    waves = []
    remaining = set(t.id for t in tasks)
    
    while remaining:
        current = [tid for tid in remaining if in_degree[tid] == 0]
        if not current:
            raise CycleError("依赖循环")
        waves.append([get_task(tid) for tid in current])
        remaining -= set(current)
        for tid in current:
            for downstream in get_downstreams(tid):
                in_degree[downstream] -= 1
    
    return waves
```

**关键点**：
- 必须检测循环依赖（任务 A 依赖 B，B 依赖 A）。
- 必须检测"过早依赖"（任务 B 只需要 A 的某个字段，但标注了"全依赖 A"）。

### 职责 4：调度决策（Scheduling Decision）

**目标**：决定每个 wave 的执行模式（串行/并行/条件/人工审批）。

详见 [`05-调度决策引擎实现.md`](./05-调度决策引擎实现.md)。

### 职责 5：状态管理（State Management）

**目标**：持久化项目状态，支持断点续跑。

**状态数据结构**：

```yaml
# project_state.yaml
project:
  id: "ecommerce-miniapp"
  status: "RUNNING"  # IDLE / RUNNING / PAUSED / COMPLETED / FAILED
  current_wave: 3
  total_waves: 7

tasks:
  T1.1:
    status: "COMPLETED"
    agent: "product"
    started_at: "2026-06-21T10:00:00Z"
    completed_at: "2026-06-21T11:30:00Z"
    outputs: ["shared/01_PRD/PRD.md"]
  
  T2.1:
    status: "RUNNING"
    agent: "ui_designer"
    started_at: "2026-06-21T11:35:00Z"
    retry_count: 0
```

**关键点**：
- 状态必须每次变化都持久化（避免 Agent 崩溃丢失进度）。
- 状态变更必须可审计（谁在什么时候改了什么）。

### 职责 6：质量门禁（Quality Gate）

**目标**：每个阶段都有可量化的通过标准。

**示例门禁规则**：

```yaml
gates:
  - name: "prd_completeness"
    condition: |
      file_exists("shared/01_PRD/PRD.md") AND
      word_count > 500 AND
      contains("User Stories") AND
      contains("Acceptance Criteria")
    auto_retry: 2
  
  - name: "code_quality"
    condition: |
      lint_score > 80 AND
      test_coverage > 80 AND
      no_critical_security_issues()
    auto_retry: 3
    on_fail: "escalate_to_human"
```

**关键点**：
- 每个门禁必须是**可机器验证的**（不是"看起来不错"）。
- 失败时先自动重试，超过阈值才升级人工。

### 职责 7：异常恢复（Recovery）

**目标**：分类错误，做对应处理。

**错误分类**：

| 错误类型 | 表现 | 策略 |
|---|---|---|
| 瞬时错误 | 网络超时、API 限流 | 重试 3 次，指数退避 |
| 逻辑错误 | Agent 输出不符合契约 | 回溯到上游 Agent 修复 |
| 需求错误 | Agent 误解用户意图 | 升级人工澄清 |
| 系统错误 | 存储满、磁盘坏 | 暂停并报警 |

详见 [`08-质量门禁与自愈机制.md`](./08-质量门禁与自愈机制.md)。

---

## 四、Orchestrator 的 SOUL.md 模板

```markdown
# SOUL.md — Project Orchestrator Agent

## Identity
你是 Project Orchestrator Agent，负责把用户需求转化为可执行的多 Agent 工程计划，
并调度 Specialist Agent 协同完成端到端开发。

## Core Capabilities
1. 意图理解：解析用户模糊需求为结构化意图
2. 任务分解：基于项目模板生成原子任务
3. 依赖分析：构建 DAG，划分执行波次
4. 调度决策：选择串行/并行/条件/审批模式
5. 状态管理：持久化项目状态，支持断点续跑
6. 质量门禁：每阶段验证产出，自动回溯修复
7. 异常恢复：分类错误，匹配恢复策略

## Operational Principles
- 用户主导：关键决策点（PRD 确认、上线审批）必须人工确认
- 产物驱动：Agent 间通信通过共享存储，不直接对话
- 透明可追溯：每个决策都有 trace 日志
- 失败安全：每个写入操作都可回滚

## Tool Usage Rules
- sessions_send：用于调度 Specialist Agent
- sessions_history：用于回溯 Agent 历史
- file_write/file_read：用于读写共享产物
- shell_run：用于执行质量门禁脚本

## Decision Matrix
- 任务依赖清晰 → 自动调度
- 任务依赖模糊 → 拆小任务
- 关键决策点 → 人工审批
- 错误超过 3 次 → 升级人工

## Output Format
每个 wave 完成时输出：
1. 本 wave 任务列表与结果
2. 质量门禁检查结果
3. 下 wave 计划
4. 风险与建议
```

---

## 五、我的实战建议

1. **Orchestrator 不要做得太"聪明"**。意图理解和任务分解用 LLM，但调度决策、质量门禁用规则（规则更稳定、更可解释）。
2. **Orchestrator 必须有"自检"机制**。每次决策前问自己"这个决策的依据是什么"，保留可审计的 trace。
3. **不要让 Orchestrator 直接调用工具**（如 git push、deploy）。这些是高风险操作，必须 Specialist 执行 + 人类审批。
4. **给 Orchestrator 设"健康检查"**。定期检查项目状态、子 Agent 状态、Token 消耗，避免失控。

---

## 六、本章关键 takeaway

- Orchestrator 三大角色：翻译官、调度官、质量官。
- 7 大核心职责：意图理解、任务分解、依赖分析、调度决策、状态管理、质量门禁、异常恢复。
- 核心算法：拓扑排序 + 波次划分 + 错误分类恢复。
- SOUL.md 模板定义了 Orchestrator 的"心智"。
- 实战建议：不要做得太聪明、必须有自检、不要直接执行高风险工具。

---

**上一篇**：[01 - 多 Agent 编排总览与定位](./01-多Agent编排总览与定位.md)
**下一篇**：[03 - Agent 角色分工与职责边界](./03-Agent角色分工与职责边界.md)
