# 04 - 任务管理与 origin 溯源

> **TL;DR**：飞书任务 V2 API 提供完整 CRUD 能力。**origin 字段是闭环的关键**——它让任务可追溯到会议来源、妙记产物、多维表格记录。**没有 origin，闭环就是黑盒**。

---

## 一、飞书任务 V2 API 概览

### 核心端点

| 操作 | 端点 | 频率限制 |
|---|---|---|
| 创建任务 | `POST /task/v2/tasks` | 10 QPS |
| 获取任务详情 | `GET /task/v2/tasks/:task_guid` | 10 QPS |
| 更新任务 | `PATCH /task/v2/tasks/:task_guid` | 10 QPS |
| 删除任务 | `DELETE /task/v2/tasks/:task_guid` | 10 QPS |
| 完成任务 | `POST /task/v2/tasks/:task_guid/complete` | 10 QPS |
| 列出任务 | `GET /task/v2/tasks` | 10 QPS |
| 添加成员 | `POST /task/v2/tasks/:task_guid/add_members` | 10 QPS |

### 关键权限

- `task:task:readonly`：读取任务
- `task:task:write`：读写任务
- 都需要在自建应用后台申请并通过审核（3-5 个工作日）

---

## 二、任务数据模型

```typescript
interface Task {
  guid: string;                    // 任务唯一 ID
  summary: string;                 // 任务标题（200 字内）
  description?: string;            // 任务描述（5000 字内）
  due?: {
    timestamp: string;             // 截止时间（毫秒）
    is_all_day: boolean;           // 是否全天
  };
  status: "todo" | "doing" | "done" | "archived";
  priority: "low" | "medium" | "high" | "urgent";
  assignees: Array<{
    id: string;                    // open_id
    role: "assignee" | "follower";
  }>;
  
  // 🔑 关键字段：origin 溯源
  origin?: {
    source: string;                // 来源标识
    [key: string]: any;            // 自定义数据
  };
  
  created_at: string;
  updated_at: string;
  completed_at?: string;
  creator: {
    id: string;
    name: string;
  };
}
```

---

## 三、origin 字段：闭环的关键

### 为什么 origin 至关重要

闭环系统要回答的核心问题：

> "这个任务是从哪里来的？对应的会议是哪场？关联的多维表格记录是哪条？"

`origin` 字段就是答案。

### origin 的设计

```python
origin = {
    # 来源标识
    "source": "minutes_artifact",  # 来源类型
    
    # 关联到会议
    "meeting_id": "vc_1234567890",  # 飞书会议 ID
    "meeting_title": "Q3 OKR 进度对齐",  # 会议标题（可选）
    "meeting_start_time": "2026-06-21T10:00:00Z",
    "meeting_end_time": "2026-06-21T11:00:00Z",
    
    # 关联到妙记
    "minute_token": "obcnxxxxxx",  # 妙记 ID
    "artifact_version": 1,  # 妙记 AI 产物版本
    
    # 关联到妙记产物中的具体 todo
    "todo_content": "通知客户 A 功能推迟到 9/25",
    "todo_index": 2,  # 在 todo_list 中的索引
    
    # 关联到多维表格
    "bitable_app_token": "bitcnxxxxxx",
    "bitable_table_id": "tblxxxxxx",
    "bitable_record_id": "recxxxxxx",
    
    # 提取元数据
    "extracted_by": "minutes_artifact_extractor",
    "extracted_at": "2026-06-21T11:35:00Z",
    
    # 原始数据（用于调试）
    "raw_todo": {
        "content": "通知客户 A 功能推迟",
        "owner": "王五",
        "due": "2026-09-16"
    }
}
```

### origin 的作用

#### 作用 1：正向追溯（任务 → 会议）

```python
async def trace_task_to_meeting(self, task_guid: str) -> Dict:
    """追溯任务到会议"""
    task = await self.get_task(task_guid)
    origin = json.loads(task.origin)
    
    if origin.get("source") == "minutes_artifact":
        return {
            "meeting_id": origin["meeting_id"],
            "minute_token": origin["minute_token"],
            "extracted_from": "妙记 AI 产物"
        }
    elif origin.get("source") == "manual":
        return {
            "creator": task.creator,
            "created_at": task.created_at
        }
```

#### 作用 2：反向追溯（会议 → 任务）

```python
async def list_tasks_from_meeting(self, meeting_id: str) -> List[Task]:
    """列出一个会议产生的所有任务"""
    all_tasks = await self.list_all_tasks(limit=1000)
    
    return [
        task for task in all_tasks
        if self._task_from_meeting(task, meeting_id)
    ]

def _task_from_meeting(self, task: Task, meeting_id: str) -> bool:
    origin = json.loads(task.origin) if task.origin else {}
    return (
        origin.get("source") == "minutes_artifact" and
        origin.get("meeting_id") == meeting_id
    )
```

#### 作用 3：关联多维表格记录

任务完成后，origin 让我们知道应该**更新多维表格的哪条记录**。

```python
async def sync_task_completion_to_bitable(self, task_guid: str):
    """任务完成后同步到多维表格"""
    task = await self.get_task(task_guid)
    origin = json.loads(task.origin)
    
    # 从 origin 拿到关联的多维表格记录
    bitable_record_id = origin.get("bitable_record_id")
    if not bitable_record_id:
        return  # 没有关联，跳过
    
    # 更新多维表格
    await self.update_bitable_record(
        app_token=origin["bitable_app_token"],
        table_id=origin["bitable_table_id"],
        record_id=bitable_record_id,
        fields={
            "进展": 1.0,
            "进展说明": f"任务已完成：{task.summary}",
            "最后更新时间": int(time.time() * 1000)
        }
    )
```

---

## 四、任务 CRUD 实战

### 创建任务

```python
import json
from datetime import datetime
from lark_oapi import Client
from lark_oapi.api.task.v2 import *

class TaskManager:
    """飞书任务管理器"""
    
    def __init__(self, app_id: str, app_secret: str):
        self.client = Client.builder() \
            .app_id(app_id).app_secret(app_secret).build()
    
    async def create_task(
        self,
        summary: str,
        description: str = "",
        assignee_open_ids: List[str] = None,
        due_timestamp: int = None,
        priority: str = "medium",
        origin: Dict = None,
        bitable_link: Dict = None
    ) -> str:
        """创建任务"""
        
        # 构造 origin（关键）
        if origin is None:
            origin = {
                "source": "manual",
                "created_by": "manual",
                "created_at": datetime.utcnow().isoformat()
            }
        
        # 添加多维表格关联到 origin
        if bitable_link:
            origin.update({
                "bitable_app_token": bitable_link["app_token"],
                "bitable_table_id": bitable_link["table_id"],
                "bitable_record_id": bitable_link["record_id"]
            })
        
        # 构造请求
        builder = CreateTaskRequestBody.builder() \
            .summary(summary[:200]) \
            .description(description[:5000] if description else None) \
            .priority(self._map_priority(priority)) \
            .origin(json.dumps(origin, ensure_ascii=False))
        
        if due_timestamp:
            builder = builder.due(
                TaskDue.builder()
                .timestamp(str(due_timestamp))
                .is_all_day(False)
                .build()
            )
        
        if assignee_open_ids:
            builder = builder.assignees([
                TaskAssignee.builder().id(open_id).build()
                for open_id in assignee_open_ids
            ])
        
        req = CreateTaskRequest.builder() \
            .request_body(builder.build()) \
            .build()
        
        resp = self.client.task.v2.task.create(req)
        
        if resp.success():
            task_guid = resp.data.task.guid
            print(f"✓ 任务创建成功: {summary} → {task_guid}")
            return task_guid
        else:
            raise TaskCreationError(resp.code, resp.msg)
    
    async def complete_task(
        self,
        task_guid: str,
        completion_note: str = ""
    ) -> bool:
        """完成任务"""
        req = CompleteTaskRequest.builder() \
            .task_guid(task_guid) \
            .request_body(
                CompleteTaskRequestBody.builder()
                .description(completion_note)
                .build()
            ) \
            .build()
        
        resp = self.client.task.v2.task.complete(req)
        
        if resp.success():
            return True
        else:
            raise TaskCompletionError(resp.code, resp.msg)
    
    async def update_task_progress(
        self,
        task_guid: str,
        progress: float,
        note: str = ""
    ) -> bool:
        """更新任务进度"""
        # 任务的"进度"通常通过 description 或自定义字段表达
        # 这里我们用追加评论的方式表达进度
        
        # 1. 获取当前任务
        task = await self.get_task(task_guid)
        
        # 2. 在描述里追加进度更新
        new_desc = (task.description or "") + \
            f"\n\n---\n**进度更新 ({datetime.now().isoformat()})**: {progress*100:.0f}% - {note}"
        
        # 3. 更新任务
        req = UpdateTaskRequest.builder() \
            .task_guid(task_guid) \
            .request_body(
                UpdateTaskRequestBody.builder()
                .description(new_desc[:5000])
                .build()
            ) \
            .build()
        
        resp = self.client.task.v2.task.update(req)
        
        if resp.success():
            return True
        else:
            raise TaskUpdateError(resp.code, resp.msg)
    
    async def get_task(self, task_guid: str) -> Task:
        """获取任务详情"""
        req = GetTaskRequest.builder() \
            .task_guid(task_guid) \
            .build()
        
        resp = self.client.task.v2.task.get(req)
        
        if resp.success():
            return self._parse_task(resp.data.task)
        else:
            raise TaskNotFoundError(task_guid)
    
    async def list_tasks(
        self,
        assignee_open_id: str = None,
        status: str = None,
        completed: bool = None,
        limit: int = 100
    ) -> List[Task]:
        """列出任务"""
        builder = ListTaskRequest.builder().limit(limit)
        
        if assignee_open_id:
            builder = builder.assignee(assignee_open_id)
        if completed is not None:
            builder = builder.completed(completed)
        
        req = builder.build()
        resp = self.client.task.v2.task.list(req)
        
        if resp.success():
            return [self._parse_task(t) for t in resp.data.items]
        else:
            raise TaskListError(resp.code, resp.msg)
    
    def _map_priority(self, priority: str) -> str:
        mapping = {"low": "low", "medium": "medium", "high": "high", "urgent": "urgent"}
        return mapping.get(priority, "medium")
```

---

## 五、任务与会议纪要的联动

### 完整链路

```
会议结束
    ↓
妙记 AI 处理（5-30 分钟）
    ↓
轮询获取妙记 AI 产物
    ↓
提取 todo_list
    ↓
[任务同步引擎] 对每个 todo：
    1. 解析负责人（name → open_id）
    2. 解析截止时间（string → timestamp）
    3. 构造 origin（含会议/妙记/多维表格关联）
    4. 创建任务
    ↓
任务创建成功 → 触发多维表格回写
```

### 联动代码

```python
async def sync_minutes_to_tasks(
    self,
    minute_token: str,
    meeting_id: str,
    bitable_record_id: str
) -> List[str]:
    """把妙记产物转换为任务"""
    
    # 1. 提取妙记 AI 产物
    artifact = await self.minutes_extractor.extract_with_retry(minute_token)
    
    # 2. 对每个 todo 创建任务
    task_guids = []
    for idx, todo in enumerate(artifact.todo_list):
        # 去重检查
        if await self._is_duplicate_task(todo, meeting_id):
            print(f"跳过重复任务: {todo.content}")
            continue
        
        # 解析负责人
        owner_open_id = await self.resolve_owner_to_open_id(todo.owner_name)
        
        # 解析截止时间
        due_timestamp = self.parse_due_to_timestamp(todo.due)
        
        # 构造 origin
        origin = {
            "source": "minutes_artifact",
            "meeting_id": meeting_id,
            "minute_token": minute_token,
            "artifact_version": artifact.version,
            "todo_content": todo.content,
            "todo_index": idx,
            "raw_todo": {
                "content": todo.content,
                "owner": todo.owner_name,
                "due": todo.due
            },
            "extracted_by": "minutes_artifact_extractor",
            "extracted_at": datetime.utcnow().isoformat()
        }
        
        # 创建任务
        try:
            task_guid = await self.task_manager.create_task(
                summary=todo.content,
                description=(
                    f"**原始提取内容**: {todo.content}\n"
                    f"**来源会议**: {meeting_id}\n"
                    f"**妙记**: {minute_token}\n"
                    f"**关联多维表格记录**: {bitable_record_id}\n\n"
                    f"---\n\n"
                    f"**会议总结**:\n{artifact.summary}"
                ),
                assignee_open_ids=[owner_open_id] if owner_open_id else [],
                due_timestamp=due_timestamp,
                priority=todo.priority,
                origin=origin
            )
            task_guids.append(task_guid)
        except TaskCreationError as e:
            print(f"任务创建失败: {todo.content} → {e}")
    
    return task_guids

async def _is_duplicate_task(self, todo: TodoItem, meeting_id: str) -> bool:
    """检查是否重复任务"""
    # 列出该会议产生的所有任务
    existing_tasks = await self.task_manager.list_tasks(limit=100)
    
    for task in existing_tasks:
        if not task.origin:
            continue
        origin = json.loads(task.origin)
        if (
            origin.get("source") == "minutes_artifact" and
            origin.get("meeting_id") == meeting_id and
            origin.get("todo_content") == todo.content
        ):
            return True
    
    return False
```

---

## 六、进展回写多维表格

任务状态变更时，回写多维表格：

```python
async def on_task_status_change(
    self,
    task_guid: str,
    new_status: str
):
    """任务状态变更时回写多维表格"""
    
    # 1. 获取任务（含 origin）
    task = await self.task_manager.get_task(task_guid)
    origin = json.loads(task.origin) if task.origin else {}
    
    # 2. 提取多维表格关联
    bitable_record_id = origin.get("bitable_record_id")
    if not bitable_record_id:
        return  # 没有关联，不回写
    
    # 3. 计算进度
    progress_map = {
        "todo": 0.0,
        "doing": 0.5,
        "done": 1.0,
        "archived": 1.0
    }
    progress = progress_map.get(new_status, 0.0)
    
    # 4. 更新多维表格
    await self.bitable_client.update_record(
        app_token=origin["bitable_app_token"],
        table_id=origin["bitable_table_id"],
        record_id=bitable_record_id,
        fields={
            "进展": progress,
            "进展说明": f"任务状态变更为 {new_status}",
            "最后更新时间": int(time.time() * 1000)
        }
    )
```

---

## 七、我的实战经验

### 经验 1：origin 字段一定要传

很多人创建任务时**忘记传 origin** 或 **origin 简化**，导致任务无法追溯。**origin 是闭环的基础**，必须详细记录。

### 经验 2：去重要前置

妙记提取的 todo_list 经常有重复（同一件事被提了两次）。在创建任务前先做去重检查，避免任务泛滥。

### 经验 3：失败任务要留痕

创建任务失败时（可能是 owner 解析失败、QPS 超限），不要直接跳过，要把失败的 todo 写到 "失败队列" 里，方便人工补单。

### 经验 4：origin 的 schema 要稳定

origin 是 JSON 字符串，字段命名要**稳定且向后兼容**。不要轻易改字段名，否则历史任务的 origin 解析会失败。

### 经验 5：定期清理 origin 隐私数据

origin 里如果有敏感信息（如个人 token），**定期清理**。origin 会被 API 响应返回，前端可能展示。

---

## 八、本章关键 takeaway

- 飞书任务 V2 API 提供完整 CRUD，频率限制 10 QPS。
- **origin 字段是闭环关键**：让任务可追溯到会议、妙记、多维表格。
- 任务创建 → 状态变更 → 多维表格回写，是完整的链路。
- 负责人匹配、截止时间解析、去重检查是实战关键。
- 失败任务要留痕，便于人工补单。
- origin schema 要稳定，敏感信息要定期清理。

---

**上一篇**：[03 - 妙记AI产物提取实战](./03-妙记AI产物提取实战.md)
**下一篇**：[05 - 多维表格数据模型设计](./05-多维表格数据模型设计.md)
