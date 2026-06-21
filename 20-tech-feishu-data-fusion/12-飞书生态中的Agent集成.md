# 12 - 飞书生态中的 Agent 集成

> **TL;DR**：把飞书数据融合系统**封装成 Agent 工具**（MCP），让外部 Agent 能调用它——这是 AI Agent 时代的关键能力。**未来不是"每个系统都做 Agent"，而是"每个系统都成为 Agent 可调用的工具"**。

---

## 一、为什么把飞书系统封装成 Agent 工具

### 趋势判断

- 飞书生态有完整的业务数据流。
- 但飞书 API 调用有学习成本（权限、限流、签名）。
- AI Agent 是未来用户与系统交互的**新入口**。
- **未来不是"用户用飞书 API"，而是"Agent 调用飞书 API"**。

### 价值

把飞书系统封装成 Agent 工具：

- **降低 Agent 开发者的接入成本**。
- **统一最佳实践**（限流、重试、缓存）。
- **业务方能直接用 Agent 完成工作**（如"帮我把上周客户会议提取任务"）。

---

## 二、什么是 MCP（Model Context Protocol）

MCP 是 Anthropic 提出的**模型上下文协议**，用来标准化 AI 模型与外部工具的交互。

### MCP 三个核心概念

| 概念 | 说明 |
|---|---|
| **Tool（工具）** | Agent 可调用的函数 |
| **Resource（资源）** | Agent 可访问的数据 |
| **Prompt（提示词）** | 预设的提示词模板 |

### MCP 工作流

```
Agent（如 Claude）
    ↓ 决定调用哪个 Tool
MCP Client
    ↓ 协议请求
MCP Server（飞书集成）
    ↓ 调用飞书 API
Feishu API
    ↓ 返回结果
MCP Server
    ↓ 格式化响应
MCP Client
    ↓ 解析结果
Agent 继续推理
```

---

## 三、封装飞书系统为 MCP Server

### 完整实现

```python
# mcp_server.py
import asyncio
import json
from mcp.server import Server, stdio_server
from mcp.types import Tool, TextContent, Resource
from clients.feishu_client import FeishuClient
from services.minutes_extractor import MinutesExtractor
from services.task_creator import TaskCreator

app = Server("feishu-data-fusion")

# 全局客户端
feishu_client: FeishuClient = None


@app.list_tools()
async def list_tools() -> list[Tool]:
    """列出可用工具"""
    return [
        Tool(
            name="extract_meeting_todos",
            description=(
                "从会议妙记中提取 AI 待办事项。"
                "输入 minute_token（妙记 ID），返回结构化的待办列表。"
                "每个待办包含 content、owner、due、priority 字段。"
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "minute_token": {
                        "type": "string",
                        "description": "飞书妙记 ID（从会议结束事件获取）"
                    },
                    "wait_timeout_minutes": {
                        "type": "integer",
                        "description": "等待妙记 AI 处理的最长时间（默认 30 分钟）",
                        "default": 30
                    }
                },
                "required": ["minute_token"]
            }
        ),
        Tool(
            name="create_feishu_task",
            description=(
                "创建飞书任务。"
                "输入任务摘要、描述、负责人、截止时间、优先级、origin。"
                "返回 task_guid（任务 ID）。"
                "origin 必须包含 source、meeting_id、minute_token、bitable_record_id（如果关联多维表格）。"
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "summary": {
                        "type": "string",
                        "description": "任务标题（≤200 字）"
                    },
                    "description": {
                        "type": "string",
                        "description": "任务描述"
                    },
                    "assignee_open_ids": {
                        "type": "array",
                        "items": {"type": "string"},
                        "description": "负责人 open_id 列表"
                    },
                    "due_timestamp_ms": {
                        "type": "integer",
                        "description": "截止时间（毫秒时间戳）"
                    },
                    "priority": {
                        "type": "string",
                        "enum": ["low", "medium", "high", "urgent"],
                        "description": "优先级"
                    },
                    "origin": {
                        "type": "object",
                        "description": (
                            "溯源信息。必须包含："
                            "source (manual/minutes_artifact)、"
                            "meeting_id (会议 ID)、"
                            "minute_token (妙记 ID)、"
                            "bitable_record_id (多维表格记录 ID)"
                        )
                    }
                },
                "required": ["summary", "origin"]
            }
        ),
        Tool(
            name="update_bitable_record",
            description=(
                "更新多维表格记录。"
                "输入 app_token、table_id、record_id 和要更新的字段。"
                "返回是否成功。"
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "app_token": {"type": "string"},
                    "table_id": {"type": "string"},
                    "record_id": {"type": "string"},
                    "fields": {
                        "type": "object",
                        "description": "要更新的字段（key-value）"
                    }
                },
                "required": ["app_token", "table_id", "record_id", "fields"]
            }
        ),
        Tool(
            name="list_bitable_records",
            description=(
                "列出多维表格的所有记录。"
                "支持过滤条件。"
                "返回记录列表（每条包含 record_id 和 fields）。"
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "app_token": {"type": "string"},
                    "table_id": {"type": "string"},
                    "max_records": {
                        "type": "integer",
                        "description": "最大返回记录数（默认 100）",
                        "default": 100
                    }
                },
                "required": ["app_token", "table_id"]
            }
        ),
        Tool(
            name="search_user",
            description=(
                "通过名字搜索飞书用户，返回 open_id。"
                "用于把任务负责人从名字转成 open_id。"
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "name": {
                        "type": "string",
                        "description": "用户姓名（支持模糊搜索）"
                    }
                },
                "required": ["name"]
            }
        ),
        Tool(
            name="complete_task",
            description=(
                "标记飞书任务为完成状态。"
                "完成后会自动回写关联的多维表格记录。"
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "task_guid": {
                        "type": "string",
                        "description": "任务 GUID"
                    },
                    "completion_note": {
                        "type": "string",
                        "description": "完成说明"
                    }
                },
                "required": ["task_guid"]
            }
        )
    ]


@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    """调用工具"""
    try:
        if name == "extract_meeting_todos":
            return await _extract_meeting_todos(arguments)
        elif name == "create_feishu_task":
            return await _create_feishu_task(arguments)
        elif name == "update_bitable_record":
            return await _update_bitable_record(arguments)
        elif name == "list_bitable_records":
            return await _list_bitable_records(arguments)
        elif name == "search_user":
            return await _search_user(arguments)
        elif name == "complete_task":
            return await _complete_task(arguments)
        else:
            raise ValueError(f"Unknown tool: {name}")
    except Exception as e:
        return [TextContent(
            type="text",
            text=json.dumps({"error": str(e)}, ensure_ascii=False)
        )]


async def _extract_meeting_todos(args: dict) -> list[TextContent]:
    """提取会议待办"""
    extractor = MinutesExtractor(feishu_client)
    artifact = await extractor.extract_with_retry(
        minute_token=args["minute_token"],
        max_wait_minutes=args.get("wait_timeout_minutes", 30)
    )
    
    return [TextContent(
        type="text",
        text=json.dumps({
            "summary": artifact.get("summary"),
            "todo_list": artifact.get("todo_list", []),
            "sections": artifact.get("sections", [])
        }, ensure_ascii=False, indent=2)
    )]


async def _create_feishu_task(args: dict) -> list[TextContent]:
    """创建飞书任务"""
    task_guid = await feishu_client.create_task(
        summary=args["summary"],
        description=args.get("description", ""),
        assignee_open_ids=args.get("assignee_open_ids", []),
        due_timestamp_ms=args.get("due_timestamp_ms"),
        priority=args.get("priority", "medium"),
        origin=args["origin"]
    )
    
    return [TextContent(
        type="text",
        text=json.dumps({"task_guid": task_guid, "status": "created"})
    )]


async def _update_bitable_record(args: dict) -> list[TextContent]:
    """更新多维表格"""
    success = await feishu_client.update_bitable_record(
        app_token=args["app_token"],
        table_id=args["table_id"],
        record_id=args["record_id"],
        fields=args["fields"]
    )
    
    return [TextContent(type="text", text=json.dumps({"success": success}))]


async def _list_bitable_records(args: dict) -> list[TextContent]:
    """列出多维表格记录"""
    records = await feishu_client.list_bitable_records(
        app_token=args["app_token"],
        table_id=args["table_id"]
    )
    
    # 限制返回数量
    max_records = args.get("max_records", 100)
    records = records[:max_records]
    
    return [TextContent(
        type="text",
        text=json.dumps({
            "records": [
                {
                    "record_id": r["record_id"],
                    "fields": r["fields"]
                }
                for r in records
            ],
            "count": len(records)
        }, ensure_ascii=False, indent=2)
    )]


async def _search_user(args: dict) -> list[TextContent]:
    """搜索用户"""
    data = await feishu_client._request(
        "GET",
        "/contact/v3/users",
        params={"query": args["name"], "page_size": 5},
        api_name="default"
    )
    
    items = data.get("data", {}).get("items", [])
    return [TextContent(
        type="text",
        text=json.dumps({
            "users": [
                {
                    "open_id": u.get("open_id"),
                    "name": u.get("name"),
                    "email": u.get("email"),
                    "department": u.get("department")
                }
                for u in items
            ]
        }, ensure_ascii=False, indent=2)
    )]


async def _complete_task(args: dict) -> list[TextContent]:
    """完成任务"""
    await feishu_client.complete_task(args["task_guid"])
    return [TextContent(type="text", text=json.dumps({"status": "completed"}))]


@app.list_resources()
async def list_resources() -> list[Resource]:
    """列出资源"""
    return [
        Resource(
            uri="feishu://config",
            name="飞书系统配置",
            description="当前飞书数据融合系统的配置信息",
            mimeType="application/json"
        ),
        Resource(
            uri="feishu://bitables",
            name="可访问的多维表格列表",
            description="列出所有可访问的多维表格及其元数据",
            mimeType="application/json"
        )
    ]


@app.read_resource()
async def read_resource(uri: str) -> str:
    """读取资源"""
    if uri == "feishu://config":
        return json.dumps({
            "app_id": CONFIG.feishu.app_id,
            "bitable_app_token": CONFIG.bitable.app_token,
            "tables": {
                "project": CONFIG.bitable.project_table_id,
                "meeting": CONFIG.bitable.meeting_table_id,
                "task": CONFIG.bitable.task_table_id
            }
        }, ensure_ascii=False, indent=2)
    
    elif uri == "feishu://bitables":
        # 列出所有多维表格
        return json.dumps({
            "tables": [
                {"name": "项目主表", "table_id": CONFIG.bitable.project_table_id},
                {"name": "会议登记表", "table_id": CONFIG.bitable.meeting_table_id},
                {"name": "任务跟踪表", "table_id": CONFIG.bitable.task_table_id}
            ]
        }, ensure_ascii=False, indent=2)
    
    raise ValueError(f"Unknown resource: {uri}")


async def main():
    """启动 MCP Server"""
    global feishu_client
    
    # 初始化客户端
    token_manager = TokenManager(CONFIG.feishu.app_id, CONFIG.feishu.app_secret)
    feishu_client = FeishuClient(token_manager)
    
    # 启动 stdio server
    async with stdio_server() as (read_stream, write_stream):
        await app.run(
            read_stream,
            write_stream,
            app.create_initialization_options()
        )


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 四、Agent 调用示例

### Claude Code 调用 MCP

```bash
# Claude Code 配置 MCP server
claude mcp add feishu-data-fusion \
  --command "python mcp_server.py" \
  --env FEISHU_APP_ID=cli_xxx \
  --env FEISHU_APP_SECRET=xxx
```

### Claude 调用示例

```
用户：上周的客户反馈会议有妙记 ID 是 obcn1234567，帮我提取里面的待办事项。

Claude（调用 extract_meeting_todos 工具）：
{"minute_token": "obcn1234567"}

工具返回：
{
  "summary": "本次客户反馈会议讨论了...",
  "todo_list": [
    {"content": "修复登录 Bug", "owner": "张三", "due": "2026-06-25", "priority": "high"},
    ...
  ]
}

Claude：会议中提取到 3 个待办：
1. 修复登录 Bug（张三，截止 6/25，高优先级）
2. ...
要帮您创建为飞书任务吗？

用户：好，创建任务。

Claude（依次调用）：
1. search_user("张三") → 获得 open_id
2. create_feishu_task(summary="修复登录 Bug", assignee_open_ids=["ou_xxx"], origin={...})

Claude：已创建 3 个任务：task_001、task_002、task_003。
```

---

## 五、在 OpenClaw 多 Agent 编排中使用

把飞书 MCP 集成到 OpenClaw 的某个 Specialist Agent：

```json
{
  "agents": {
    "feishu_ops": {
      "type": "specialist",
      "role": "飞书业务运营 Agent",
      "workspace": "agents/feishu_ops",
      "model": "gpt-4",
      "memory_enabled": true,
      "tools": [
        "mcp:feishu-data-fusion:extract_meeting_todos",
        "mcp:feishu-data-fusion:create_feishu_task",
        "mcp:feishu-data-fusion:update_bitable_record",
        "mcp:feishu-data-fusion:list_bitable_records",
        "mcp:feishu-data-fusion:search_user"
      ],
      "mcp_servers": {
        "feishu-data-fusion": {
          "command": "python /path/to/mcp_server.py",
          "env": {
            "FEISHU_APP_ID": "${FEISHU_APP_ID}",
            "FEISHU_APP_SECRET": "${FEISHU_APP_SECRET}"
          }
        }
      }
    }
  }
}
```

---

## 六、典型场景

### 场景 1：会议 Agent 自动跟进

```
Orchestrator Agent 检测到会议结束
    ↓
调度 feishu_ops Agent
    ↓
feishu_ops 调用 extract_meeting_todos 获取妙记 AI 产物
    ↓
对每个 todo，调用 search_user 获取 open_id
    ↓
调用 create_feishu_task 创建任务
    ↓
调用 update_bitable_record 回写多维表格进展
    ↓
任务完成
```

### 场景 2：客服 Agent 处理反馈

```
用户提交反馈（多维表格）
    ↓
feishu_ops Agent 接收
    ↓
调用 list_bitable_records 拉取新反馈
    ↓
LLM 分析反馈，自动分类、分配优先级
    ↓
调用 create_feishu_task 创建处理任务
    ↓
调用 update_bitable_record 更新反馈状态
```

### 场景 3：老板看数据

```
老板问："上周延期最多的项目是哪些？"
    ↓
飞书 Agent 调用 list_bitable_records
    ↓
LLM 分析数据，生成回答
    ↓
"上周有 3 个项目延期：项目 A、项目 B、项目 C..."
```

---

## 七、最佳实践

### 1. 工具命名要清晰

```python
# ✅ 清晰的命名
extract_meeting_todos
create_feishu_task
update_bitable_record

# ❌ 模糊的命名
extract  # 提取什么？
create_task  # 哪个系统的任务？
update  # 更新什么？
```

### 2. 描述要包含使用场景

```python
Tool(
    name="extract_meeting_todos",
    description=(
        "从会议妙记中提取 AI 待办事项。"
        "输入 minute_token（妙记 ID），返回结构化的待办列表。"
        "【使用场景】会议结束后，自动提取会议中讨论的待办事项。"
        "【返回字段】每个待办包含 content、owner、due、priority。"
        "【限制】妙记 AI 处理需要 5-30 分钟，调用会自动等待。"
    ),
    ...
)
```

### 3. 输入参数要严格校验

```python
inputSchema={
    "type": "object",
    "properties": {
        "minute_token": {
            "type": "string",
            "minLength": 10,
            "description": "妙记 ID（必须是 obcn 开头的字符串）"
        }
    },
    "required": ["minute_token"]
}
```

### 4. 错误处理要友好

```python
async def _extract_meeting_todos(args):
    try:
        artifact = await extractor.extract_with_retry(...)
        return [TextContent(type="text", text=json.dumps(...))]
    except TimeoutError:
        return [TextContent(
            type="text",
            text=json.dumps({
                "error": "妙记 AI 处理超时",
                "suggestion": "请稍后重试，或手动指定会议内容"
            }, ensure_ascii=False)
        )]
    except Exception as e:
        return [TextContent(
            type="text",
            text=json.dumps({"error": str(e)}, ensure_ascii=False)
        )]
```

### 5. 提供 Resources 给 Agent 上下文

```python
@app.read_resource()
async def read_resource(uri: str) -> str:
    if uri == "feishu://config":
        # 让 Agent 知道系统配置
        return json.dumps({...})
    
    if uri == "feishu://schema/project_table":
        # 让 Agent 知道多维表格的字段
        return json.dumps({
            "fields": {
                "项目名称": "Text",
                "项目状态": "SingleSelect",
                ...
            }
        }, ensure_ascii=False)
```

---

## 八、未来展望

### 短期（1-2 年）

- **MCP 标准化**：越来越多的工具支持 MCP，飞书系统可以无缝接入任何 Agent。
- **Agent Marketplace**：用户可以从市场选择"飞书会议 Agent"、"飞书任务 Agent"等。

### 中期（3-5 年）

- **多 Agent 协作**：飞书 Agent 接入更广泛的 Agent 生态（如 Notion、Slack、GitHub）。
- **企业 Agent 网络**：企业内部多套系统（飞书 + 自研 + SaaS）通过 Agent 互联互通。

### 长期（5+ 年）

- **AI-native 飞书**：飞书本身可能变成"Agent 平台"，每个用户都有私人 Agent。
- **业务自动化从"配置"变成"对话"**：用户不再需要配置工作流，直接告诉 Agent "每周一把上周会议任务总结发给我"。

---

## 九、本章关键 takeaway

- 把飞书系统封装成 MCP 工具，让任何 Agent 可调用。
- MCP 三大核心：Tool（工具）、Resource（资源）、Prompt（提示词）。
- 工具命名清晰、描述详细、参数严格、错误友好。
- 通过 Resources 给 Agent 提供上下文（配置、Schema）。
- 与 OpenClaw 多 Agent 编排系统集成。
- 未来趋势：所有系统都将成为 Agent 可调用的工具。

---

**上一篇**：[11 - 签名校验与安全实践](./11-签名校验与安全实践.md)
**返回**：[20-tech-feishu-data-fusion README](./README.md)
