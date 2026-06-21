# 15 - 飞书项目（Projex）应用

> **TL;DR**：飞书项目（原 Projex）是飞书生态的**专业项目管理工具**。本篇讲它与多维表格的区别、适用场景、与 Agent 系统的集成。

---

## 一、飞书项目 vs 多维表格

### 定位差异

| 维度 | 多维表格（Bitable） | 飞书项目（Projex） |
|---|---|---|
| **本质** | 轻量级数据库 | 专业 PM 工具 |
| **目标** | 灵活的数据管理 | 项目全生命周期 |
| **视图** | 表格 + 看板 + 画册 | 甘特图 + 看板 + 列表 |
| **依赖** | 手动关联 | 自动依赖图 |
| **资源** | 不支持 | 资源管理 |
| **成本** | 较低 | 较高 |
| **复杂度** | 低 | 中-高 |
| **学习成本** | 低 | 中 |

### 何时用哪个

#### 用多维表格

- 简单数据管理（订单、客户反馈）。
- 灵活但不需要复杂 PM 功能。
- 业务简单，跨部门协作少。

#### 用飞书项目

- 中-大型项目（软件工程、产品发布）。
- 需要甘特图、依赖图、资源管理。
- 标准项目管理流程。

#### 两者结合

- 多维表格存业务数据。
- 飞书项目管项目流程。
- 双向同步（业务状态 ↔ 项目状态）。

---

## 二、飞书项目的核心功能

### 1. 项目空间

- 项目空间 = 一个项目的容器。
- 包含：视图、成员、设置、组件。

### 2. 视图

- **列表视图**：任务列表。
- **看板视图**：按状态分列。
- **甘特图**：时间线 + 依赖。
- **表格视图**：类似多维表格。
- **时间线**：里程碑视图。

### 3. 任务管理

- **任务**：基本单元。
- **子任务**：任务可拆分。
- **依赖**：FS / SS / FF / SF。
- **负责人 + 关注人**。
- **优先级、截止时间、标签**。
- **自定义字段**。

### 4. 工作流

- **触发器**：状态变更、字段变更等。
- **自动化动作**：发消息、改状态、调 API。
- **审批节点**：HITL。

### 5. 资源管理

- 工时登记。
- 资源分配。
- 容量规划。

---

## 三、飞书项目 API

### 主要端点

```
项目空间：
  GET /project/v1/project/{project_key}
  
任务：
  GET /project/v1/project/{project_key}/work_item
  POST /project/v1/project/{project_key}/work_item
  PATCH /project/v1/project/{project_key}/work_item/{work_item_id}
  
视图：
  GET /project/v1/project/{project_key}/view/{view_id}
  
工作流：
  POST /project/v1/workflow/{workflow_id}/trigger
```

### 权限

- `project:project:readonly`：读取。
- `project:project:write`：读写。
- 类似多维表格，需要审核。

---

## 四、飞书项目 + 多维表格 双向同步

### 架构

```
业务数据（多维表格）
    ↕ 双向同步
项目流程（飞书项目）
    ↕ 双向同步
任务执行（飞书任务）
```

### 同步策略

#### 方向 1：多维表格 → 飞书项目

- **触发**：多维表格记录新增。
- **动作**：在飞书项目创建对应任务。
- **频率**：实时（Automation Webhook）。

#### 方向 2：飞书项目 → 多维表格

- **触发**：飞书项目任务状态变更。
- **动作**：更新多维表格记录的"状态"字段。
- **频率**：实时（Webhook）。

#### 方向 3：飞书项目 → 飞书任务

- **触发**：飞书项目任务创建。
- **动作**：自动创建对应的飞书任务。
- **目的**：让任务能在"我的任务"里看到。

### 实现

```python
class ProjexBitableSyncer:
    """飞书项目 ↔ 多维表格双向同步"""
    
    def __init__(self, feishu_client):
        self.feishu = feishu_client
    
    async def on_bitable_record_created(self, record):
        """多维表格记录创建 → 飞书项目任务"""
        projex_task = await self.feishu.create_projex_task(
            project_key=record.fields["项目Key"],
            name=record.fields["任务名称"],
            owner=record.fields.get("负责人"),
            due=record.fields.get("截止时间"),
            priority=record.fields.get("优先级", "medium"),
            origin={
                "source": "bitable",
                "bitable_record_id": record.record_id,
                "bitable_app_token": record.app_token,
                "bitable_table_id": record.table_id
            }
        )
        # 回填 Projex 任务 ID
        await self.update_bitable_record(record.record_id, {
            "Projex任务ID": projex_task.work_item_id
        })
    
    async def on_projex_task_updated(self, work_item):
        """飞书项目任务变更 → 多维表格"""
        # 找到对应的多维表格记录
        record_id = work_item.origin.get("bitable_record_id")
        if not record_id:
            return
        
        # 更新多维表格
        await self.update_bitable_record(record_id, {
            "任务状态": self._map_status(work_item.status),
            "进展": self._map_progress(work_item.status),
            "最后更新时间": int(time.time() * 1000),
            "数据来源": "projex_sync"
        })
```

---

## 五、与 Agent 系统的集成

### 场景

- **Orchestrator** 分解任务时，把任务**同时**写入飞书项目 + 飞书任务。
- **Coder Agent** 完成代码后，更新飞书项目状态。
- **QA Agent** 测试通过后，更新飞书项目 + 飞书任务。

### 实现

```python
class ProjectAgentIntegration:
    """Agent 系统与飞书项目集成"""
    
    async def create_work_item(self, task, project_key):
        """Agent 任务 → 飞书项目工作项"""
        work_item = await self.feishu.create_projex_task(
            project_key=project_key,
            name=task.name,
            description=task.description,
            owner=task.assignee,
            due=task.due,
            priority=task.priority,
            estimated_hours=task.estimated_duration_minutes / 60,
            origin={
                "source": "agent_orchestrator",
                "agent_task_id": task.id,
                "agent_id": task.agent,
                "wave_number": task.wave_number
            }
        )
        return work_item
    
    async def update_work_item_status(self, work_item_id, status):
        """Agent 完成任务 → 更新飞书项目"""
        await self.feishu.update_projex_task(work_item_id, {
            "status": status,
            "completed_at": datetime.utcnow().isoformat()
        })
```

---

## 六、飞书项目的局限

### 局限 1：API 不如多维表格丰富

- 多维表格 API 较完整。
- 飞书项目 API 部分功能缺失（如工作流触发）。

### 局限 2：跨项目操作复杂

- 多项目数据汇总需要分别查询。
- 多维表格的"跨表关联"更灵活。

### 局限 3：成本

- 飞书项目高级功能需要付费。
- 多维表格基础功能免费。

### 局限 4：移动端体验

- 多维表格移动端较好。
- 飞书项目移动端相对复杂。

---

## 七、我的实战建议

### 建议 1：业务数据用多维表格，项目流程用飞书项目

- 业务数据（订单、客户）→ 多维表格。
- 项目流程（任务、依赖）→ 飞书项目。
- 双向同步。

### 建议 2：小项目用多维表格即可

- 5 人以下、3 个月内、无复杂依赖 → 多维表格。
- 不需要引入飞书项目的复杂度。

### 建议 3：大项目必须用飞书项目

- 10 人以上、跨季度、复杂依赖 → 飞书项目。
- 甘特图、依赖图、资源管理必需。

### 建议 4：飞书项目 + Agent 系统是未来

- Agent 编排天然需要"项目管理"工具。
- 飞书项目 + Agent = "AI 项目经理"。

---

## 八、本章关键 takeaway

- **多维表格** vs **飞书项目**：业务数据 vs 项目流程。
- 飞书项目核心功能：项目空间、视图、任务、依赖、工作流、资源管理。
- **双向同步**：多维表格 ↔ 飞书项目 ↔ 飞书任务。
- 与 Agent 系统的集成：任务分解时同步写入、状态变更时同步更新。
- **实战选择**：小项目用多维表格、大项目用飞书项目 + 双向同步。

---

**返回**：[20-tech-feishu-data-fusion README](./README.md)
**上一篇**：[14 - 数据同步中的最终一致性实战](./14-数据同步中的最终一致性实战.md)
