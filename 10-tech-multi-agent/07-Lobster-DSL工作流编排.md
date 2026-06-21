# 07 - Lobster DSL 工作流编排

> **TL;DR**：Lobster DSL 是 OpenClaw 的工作流编排语言。它把多 Agent 协作流程写成**可执行 YAML**，支持串行、并行、波次、条件、人工审批等模式。

---

## 一、什么是 Lobster DSL

Lobster（OpenClaw 多 Agent 工作流编排 DSL）是一种**声明式 YAML 语言**，用来描述：

- 多 Agent 之间的协作流程。
- 每个 Agent 在何时执行、执行什么任务。
- 任务间的依赖关系。
- 错误处理、超时、重试策略。
- 人工审批节点。

它的设计目标是：

> **让非程序员也能读懂多 Agent 协作流程；让流程可版本化、可审计、可重放。**

---

## 二、Lobster 工作流的基本结构

```yaml
name: "工作流名称"
description: "工作流描述"
mode: wave  # sequential / parallel / wave / conditional / hitl

variables:
  project_name: "..."
  approver: "user"

waves:
  - wave: 1
    mode: sequential
    tasks: [...]

error_handling:
  on_agent_timeout:
    action: "retry_with_different_model"
    max_retries: 2

completion:
  on_success:
    - action: "send_notification"
  on_failure:
    - action: "escalate_to_human"
```

---

## 三、完整示例：电商小程序

```yaml
# lobsters/ecommerce_project.yaml
name: "E-commerce Mini Program"
description: "从用户需求到部署上线的完整流程"
mode: wave

variables:
  project_name: "ecommerce-miniapp"
  approver: "user@example.com"

# 工作流变量（可被任务引用）
context:
  target_platform: "wechat_mini"
  tech_preferences: ["typescript", "node.js", "react"]

# 人工检查点配置
hitl:
  default_timeout: "24h"
  on_timeout: "escalate"

# 错误处理
error_handling:
  on_agent_timeout:
    action: "retry_with_different_model"
    max_retries: 2
  on_quality_gate_failure:
    action: "reassign_to_owner_agent"
    max_retries: 3
  on_unrecoverable_error:
    action: "escalate_to_human"
    notify: ["{{ approver }}"]

waves:
  # ============ Wave 1: 需求澄清 ============
  - wave: 1
    mode: sequential
    tasks:
      - id: "T1.1"
        agent: "product"
        task: "Write PRD"
        inputs:
          - "{{ user_requirement }}"
          - "shared/00_Context/project_brief.md"
        outputs:
          - "shared/01_PRD/PRD.md"
          - "shared/01_PRD/UserStories.md"
        estimated_duration: "2h"
        guardrails:
          - "min_word_count: 500"
          - "must_contain: [User Stories, Acceptance Criteria]"

  # ============ Wave 2: PRD 审批 + 设计 + 架构 ============
  - wave: 2
    mode: hitl
    steps:
      # PRD 完成后等用户审批
      - id: "T2.0_approval"
        action: "pause_for_approval"
        artifact: "shared/01_PRD/PRD.md"
        approvers: ["{{ approver }}"]
        approval_criteria:
          - "用户故事覆盖所有核心功能"
          - "验收标准可量化"
        on_reject:
          - action: "reassign_to"
            agent: "product"
            reason: "{{ rejection_reason }}"

      # 设计系统和系统架构并行
      - id: "T2.1"
        agent: "ui_designer"
        task: "Define Design System"
        inputs:
          - "shared/01_PRD/PRD.md"
        outputs:
          - "shared/02_UI/DesignSystem.md"
        estimated_duration: "1h"
        depends_on_approval: "T2.0_approval"

      - id: "T3.1"
        agent: "architect"
        task: "Design System Architecture"
        inputs:
          - "shared/01_PRD/PRD.md"
        outputs:
          - "shared/03_Arch/Architecture.md"
        estimated_duration: "2h"
        depends_on_approval: "T2.0_approval"

  # ============ Wave 3: 详细设计 + API Spec ============
  - wave: 3
    mode: parallel
    tasks:
      - id: "T2.2"
        agent: "ui_designer"
        task: "Design Home Page"
        inputs:
          - "shared/02_UI/DesignSystem.md"
        outputs:
          - "shared/02_UI/Wireframes/HomePage.md"
        estimated_duration: "1.5h"

      - id: "T2.3"
        agent: "ui_designer"
        task: "Design Product Detail Page"
        inputs:
          - "shared/02_UI/DesignSystem.md"
        outputs:
          - "shared/02_UI/Wireframes/ProductDetail.md"
        estimated_duration: "1.5h"

      - id: "T2.4"
        agent: "ui_designer"
        task: "Design Cart & Checkout Flow"
        inputs:
          - "shared/02_UI/DesignSystem.md"
        outputs:
          - "shared/02_UI/Wireframes/CartCheckout.md"
        estimated_duration: "2h"

      - id: "T3.2"
        agent: "architect"
        task: "Design API Spec"
        inputs:
          - "shared/03_Arch/Architecture.md"
        outputs:
          - "shared/03_Arch/APISpec.yaml"
          - "shared/03_Arch/Schema.sql"
        estimated_duration: "2h"

  # ============ Wave 4: 实现（前端 + 后端并行） ============
  - wave: 4
    mode: parallel
    tasks:
      - id: "T4.1"
        agent: "coder_backend"
        task: "Implement Backend API"
        inputs:
          - "shared/03_Arch/APISpec.yaml"
          - "shared/03_Arch/Schema.sql"
        outputs:
          - "/backend/"
        estimated_duration: "4h"
        quality_gates:
          - "lint_score > 80"
          - "test_coverage > 80"
          - "all_api_endpoints_implemented"

      - id: "T4.2"
        agent: "coder_frontend"
        task: "Implement Frontend UI"
        inputs:
          - "shared/02_UI/Wireframes/HomePage.md"
          - "shared/02_UI/Wireframes/ProductDetail.md"
          - "shared/02_UI/Wireframes/CartCheckout.md"
          - "shared/03_Arch/APISpec.yaml"
        outputs:
          - "/frontend/"
        estimated_duration: "4h"
        quality_gates:
          - "lint_score > 80"
          - "all_pages_implemented"
          - "responsive_on_mobile"

  # ============ Wave 5: 测试验证 ============
  - wave: 5
    mode: sequential
    tasks:
      - id: "T5.1"
        agent: "qa"
        task: "Run Tests"
        inputs:
          - "/backend/"
          - "/frontend/"
          - "shared/01_PRD/AcceptanceCriteria.md"
        outputs:
          - "shared/05_QA/TestReport.md"
          - "shared/05_QA/BugReport.md"
        estimated_duration: "3h"

  # ============ Wave 6: 测试通过则部署，失败则修复 ============
  - wave: 6
    mode: conditional
    branches:
      - condition: "tests_passed == true AND critical_bugs == 0"
        then:
          - id: "T6.1"
            agent: "devops"
            task: "Build & Deploy"
            inputs:
              - "/backend/"
              - "/frontend/"
            outputs:
              - "shared/06_Deploy/DeploymentGuide.md"
            estimated_duration: "1h"

      - condition: "tests_passed == false OR critical_bugs > 0"
        then:
          - id: "T5.2_fix"
            agent: "coder_backend"  # 或 coder_frontend，根据 bug 归属
            task: "Fix Bugs"
            inputs:
              - "shared/05_QA/BugReport.md"
            outputs:
              - "/backend/"  # 或 /frontend/
            loop_until: "tests_passed == true AND critical_bugs == 0"
            max_iterations: 3
          - if_loop_fails:
              - action: "escalate_to_human"
                notify: ["{{ approver }}"]

# 工作流级错误处理
error_handling:
  on_agent_timeout:
    action: "retry_with_different_model"
    max_retries: 2

# 完成后的聚合动作
completion:
  on_success:
    - action: "send_notification"
      channel: "email"
      to: "{{ approver }}"
      template: "project_completed"
    - action: "generate_summary"
      output: "PROJECT_SUMMARY.md"

  on_failure:
    - action: "send_notification"
      channel: "email"
      to: "{{ approver }}"
      template: "project_failed"
    - action: "save_state"
      path: "shared/_state/failed_state.yaml"
```

---

## 四、Lobster 关键语法详解

### 1. 工作流变量

```yaml
variables:
  project_name: "ecommerce-miniapp"
  approver: "user@example.com"

# 引用方式：{{ variable_name }}
inputs:
  - "shared/{{ project_name }}/PRD.md"
```

### 2. 工作流模式

```yaml
mode: sequential  # 串行
mode: parallel    # 并行
mode: wave        # 波次（混合）
mode: conditional # 条件分支
mode: hitl        # 人工介入
```

### 3. 任务定义

```yaml
- id: "T1.1"
  agent: "product"  # 必须已在 openclaw.json 中定义
  task: "Write PRD"  # 任务描述（自然语言）
  inputs: [...]  # 输入产物路径
  outputs: [...]  # 输出产物路径
  estimated_duration: "2h"
  guardrails: [...]  # 任务约束
  quality_gates: [...]  # 完成后必须通过的检查
  retry_policy:
    max_retries: 3
    on_failure: "reassign"  # 或 "escalate"
```

### 4. 依赖与审批

```yaml
# 依赖审批节点
- id: "T2.1"
  depends_on_approval: "T2.0_approval"

# 依赖其他任务完成
- id: "T4.2"
  depends_on: ["T2.2", "T2.3", "T2.4"]

# 依赖条件
- id: "T5.2_fix"
  condition: "tests_passed == false"
```

### 5. 循环与分支

```yaml
# 循环（带最大次数）
- id: "T5.2_fix"
  loop_until: "tests_passed == true"
  max_iterations: 3

# 循环失败后升级
- if_loop_fails:
    - action: "escalate_to_human"

# 条件分支
- mode: conditional
  branches:
    - condition: "..."
      then: [...]
    - condition: "..."
      then: [...]

# 默认分支（所有条件都不满足时）
- default:
    - action: "escalate_to_human"
```

### 6. 人工介入节点

```yaml
- id: "T2.0_approval"
  action: "pause_for_approval"
  artifact: "shared/01_PRD/PRD.md"
  approvers: ["user@example.com"]
  approval_criteria:
    - "用户故事覆盖所有核心功能"
  timeout: "24h"
  on_timeout: "escalate"
  on_reject:
    - action: "reassign_to"
      agent: "product"
      reason: "{{ rejection_reason }}"
```

### 7. 错误处理

```yaml
error_handling:
  on_agent_timeout:
    action: "retry_with_different_model"
    max_retries: 2
  on_quality_gate_failure:
    action: "reassign_to_owner_agent"
    max_retries: 3
  on_unrecoverable_error:
    action: "escalate_to_human"
    notify: ["user@example.com"]
```

### 8. 完成动作

```yaml
completion:
  on_success:
    - action: "send_notification"
    - action: "generate_summary"
  on_failure:
    - action: "save_state"
    - action: "send_notification"
```

---

## 五、Bug 修复循环工作流

```yaml
# lobsters/bug_fix_cycle.yaml
name: "Bug Fix Cycle"
description: "测试发现 Bug 后，触发修复循环"
mode: conditional

variables:
  project_root: "."

branches:
  # 如果测试通过
  - condition: "tests_passed == true"
    then:
      - action: "log"
        message: "All tests passed, proceeding to deploy"

  # 如果测试失败
  - condition: "tests_passed == false"
    then:
      # 修复循环
      - id: "fix_loop"
        loop_until: "tests_passed == true"
        max_iterations: 3
        steps:
          # 分析 Bug 归属
          - id: "analyze"
            agent: "orchestrator"
            task: "Analyze BugReport and assign to owner agent"
            inputs:
              - "shared/05_QA/BugReport.md"

          # 修复 Bug
          - id: "fix"
            agent: "{{ bug_owner }}"  # coder_backend / coder_frontend
            task: "Fix Bugs"
            inputs:
              - "shared/05_QA/BugReport.md"
            outputs:
              - "{{ bug_files }}"

          # 重新测试
          - id: "retest"
            agent: "qa"
            task: "Re-run Tests"
            inputs:
              - "{{ bug_files }}"

      # 循环失败后升级
      - if_loop_fails:
          - action: "escalate_to_human"
            notify: ["{{ approver }}"]
            context:
              failed_iterations: "{{ fix_loop.iterations }}"
              remaining_bugs: "{{ bug_count }}"

completion:
  on_success:
    - action: "log"
      message: "All bugs fixed"
  on_failure:
    - action: "save_state"
      path: "shared/_state/bug_fix_failed.yaml"
```

---

## 六、Lobster 引擎实现要点

### 状态机驱动

```python
class LobsterEngine:
    """Lobster DSL 执行引擎"""
    
    def __init__(self, workflow_yaml: str, openclaw_client):
        self.workflow = yaml.safe_load(workflow_yaml)
        self.openclaw = openclaw_client
        self.context = {}  # 工作流变量
    
    async def execute(self):
        mode = self.workflow["mode"]
        
        if mode == "sequential":
            await self._execute_sequential()
        elif mode == "parallel":
            await self._execute_parallel()
        elif mode == "wave":
            await self._execute_wave()
        elif mode == "conditional":
            await self._execute_conditional()
        elif mode == "hitl":
            await self._execute_hitl()
    
    async def _execute_wave(self):
        for wave in self.workflow["waves"]:
            await self._execute_single_wave(wave)
    
    async def _execute_single_wave(self, wave):
        tasks = wave["tasks"]
        mode = wave.get("mode", "sequential")
        
        if mode == "sequential":
            for task in tasks:
                await self._execute_task(task)
        elif mode == "parallel":
            await asyncio.gather(*[self._execute_task(t) for t in tasks])
        elif mode == "conditional":
            await self._execute_conditional_wave(wave)
    
    async def _execute_task(self, task):
        # 1. 解析 inputs（替换变量）
        inputs = [self._resolve(i) for i in task.get("inputs", [])]
        
        # 2. 检查依赖（审批、其他任务）
        if "depends_on_approval" in task:
            await self._wait_approval(task["depends_on_approval"])
        if "depends_on" in task:
            await self._wait_tasks(task["depends_on"])
        
        # 3. 调用 Agent
        agent = self.openclaw.get_agent(task["agent"])
        result = await agent.execute(
            task_id=task["id"],
            description=task["task"],
            inputs=inputs,
            expected_outputs=task.get("outputs", [])
        )
        
        # 4. 质量门禁
        if not self._pass_quality_gates(task, result):
            raise QualityGateFailure(task_id=task["id"])
        
        # 5. 更新上下文
        self.context[task["id"]] = result
```

---

## 七、Lobster 工作流的最佳实践

### 实践 1：从最小工作流开始

不要一开始就写完整工作流。从一个 3-5 任务的最小工作流跑通，再逐步加任务。

### 实践 2：HITL 节点不要太多

HITL 节点（人工审批）会显著拖慢流程。仅在"关键决策点"加 HITL：
- PRD 完成后（开始 UI/架构前）。
- 架构评审完成后（开始编码前）。
- 测试通过后（部署前）。
- 任何重大变更。

### 实践 3：错误处理必须显式

每个工作流都必须有 `error_handling` 段。不要假设"一切都会成功"。

### 实践 4：变量与上下文分离

- `variables` 是**工作流启动时定义**的常量。
- `context` 是**运行时累积**的状态。

不要把运行时数据放进 variables。

### 实践 5：可重放性

工作流必须可以从任意中间状态恢复：
- 每个 wave 完成时 commit 状态。
- Orchestrator 重启后从最后一个完成的 wave 继续。

---

## 八、本章关键 takeaway

- Lobster DSL 是声明式 YAML 工作流语言。
- 支持 sequential、parallel、wave、conditional、hitl 五种模式。
- 关键语法：变量、模式、任务定义、依赖、审批、循环、分支、错误处理、完成动作。
- 引擎用状态机驱动，每个 wave 内按模式执行。
- 最佳实践：从最小工作流开始、HITL 不要太多、错误处理显式、可重放。

---

**上一篇**：[06 - Agent 间通信协议与产物驱动](./06-Agent间通信协议与产物驱动.md)
**下一篇**：[08 - 质量门禁与自愈机制](./08-质量门禁与自愈机制.md)
