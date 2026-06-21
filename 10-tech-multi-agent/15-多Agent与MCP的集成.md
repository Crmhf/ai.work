# 15 - 多 Agent 与 MCP 的集成

> **TL;DR**：MCP（Model Context Protocol）是 Agent 工具调用的**事实标准**。把多 Agent 系统的能力**封装为 MCP Server**，让任何 Agent 可调用。

---

## 一、为什么需要 MCP

### 问题

- 每个 Agent 框架都有自己的 Tool 定义（LangChain、AutoGen、CrewAI 各一套）。
- 工具市场无法统一。
- 同一个工具要实现 N 个版本。

### MCP 的解决方案

- **统一协议**：所有 Agent 用同一种方式调用工具。
- **MCP Server**：工具开发者只需实现一次。
- **MCP Client**：Agent 框架只需实现 Client。

```
MCP Client（Agent） →  MCP 协议  →  MCP Server（工具）
```

### 类比 USB-C

- USB-C 之前：每个设备都有自己的接口。
- USB-C 之后：所有设备通用。
- MCP 之于 Agent = USB-C 之于设备。

---

## 二、MCP 的三大核心概念

### 1. Tool（工具）

Agent 可调用的函数。

```typescript
{
  name: "extract_meeting_todos",
  description: "从妙记 AI 提取待办事项",
  inputSchema: {
    type: "object",
    properties: {
      minute_token: { type: "string" }
    },
    required: ["minute_token"]
  }
}
```

### 2. Resource（资源）

Agent 可访问的数据（类似 GET 端点）。

```typescript
{
  uri: "feishu://config",
  name: "飞书系统配置",
  mimeType: "application/json"
}
```

### 3. Prompt（提示词）

预定义的提示词模板。

```typescript
{
  name: "summarize_meeting",
  description: "总结会议要点",
  arguments: [
    { name: "transcript", required: true }
  ]
}
```

---

## 三、把多 Agent 系统封装为 MCP Server

### 架构

```
┌─────────────────────────────────────┐
│  MCP Server（多 Agent 系统）          │
│                                      │
│  ┌──────────────────────────────┐  │
│  │  Orchestrator                │  │
│  │  (任务分解、调度)             │  │
│  └──────────────────────────────┘  │
│           ↓                          │
│  ┌──────┬──────┬──────┬──────┐     │
│  │Spec A│Spec B│Spec C│Spec D│     │
│  └──────┴──────┴──────┴──────┘     │
│           ↓                          │
│  ┌──────────────────────────────┐  │
│  │  共享存储 + 产物              │  │
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘
            ↑ MCP 协议
            │
   ┌────────┴────────┐
   │  Agent Client    │
   │  (Claude / GPT)  │
   └──────────────────┘
```

### MCP Server 实现

```python
import asyncio
from mcp.server import Server, stdio_server
from mcp.types import Tool, TextContent, Resource

app = Server("multi-agent-orchestrator")

# 全局：Orchestrator 实例
orchestrator = None


@app.list_tools()
async def list_tools():
    """列出可用工具"""
    return [
        Tool(
            name="decompose_and_execute_project",
            description=(
                "分解用户需求为多 Agent 任务图，并调度执行。"
                "输入：项目描述（如"做一个电商小程序"）"
                "返回：执行结果摘要 + 产物路径列表。"
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "user_requirement": {
                        "type": "string",
                        "description": "用户的项目需求描述"
                    },
                    "project_type": {
                        "type": "string",
                        "enum": ["webapp", "mobile_app", "mini_program", "api_service"],
                        "description": "项目类型"
                    }
                },
                "required": ["user_requirement"]
            }
        ),
        Tool(
            name="decompose_task",
            description="仅分解任务，不执行。返回任务 DAG。",
            inputSchema={
                "type": "object",
                "properties": {
                    "user_requirement": {"type": "string"},
                    "project_type": {"type": "string"}
                },
                "required": ["user_requirement"]
            }
        ),
        Tool(
            name="list_available_agents",
            description="列出所有可用的 Specialist Agent 及其能力",
            inputSchema={"type": "object", "properties": {}}
        ),
        Tool(
            name="get_task_status",
            description="查询任务执行状态",
            inputSchema={
                "type": "object",
                "properties": {
                    "task_id": {"type": "string"}
                },
                "required": ["task_id"]
            }
        ),
        Tool(
            name="get_project_artifacts",
            description="获取项目的所有产物（PRD、API Spec、代码等）",
            inputSchema={
                "type": "object",
                "properties": {
                    "project_id": {"type": "string"}
                }
            }
        )
    ]


@app.call_tool()
async def call_tool(name: str, arguments: dict):
    """调用工具"""
    if name == "decompose_and_execute_project":
        return await _decompose_and_execute(arguments)
    elif name == "decompose_task":
        return await _decompose_only(arguments)
    elif name == "list_available_agents":
        return await _list_agents()
    elif name == "get_task_status":
        return await _get_status(arguments["task_id"])
    elif name == "get_project_artifacts":
        return await _get_artifacts(arguments["project_id"])


async def _decompose_and_execute(args):
    """分解并执行项目"""
    try:
        # 调用 Orchestrator
        result = await orchestrator.execute_project(
            user_requirement=args["user_requirement"],
            project_type=args.get("project_type", "webapp")
        )
        
        return [TextContent(
            type="text",
            text=json.dumps({
                "status": "completed",
                "tasks_executed": len(result["tasks"]),
                "artifacts": result["artifacts"],
                "duration_seconds": result["duration"]
            }, ensure_ascii=False, indent=2)
        )]
    except Exception as e:
        return [TextContent(
            type="text",
            text=json.dumps({"status": "failed", "error": str(e)}, ensure_ascii=False)
        )]


async def _decompose_only(args):
    """仅分解任务"""
    tasks = await orchestrator.decompose(
        user_requirement=args["user_requirement"],
        project_type=args.get("project_type", "webapp")
    )
    return [TextContent(
        type="text",
        text=json.dumps({
            "tasks": [
                {
                    "id": t.id,
                    "name": t.name,
                    "agent": t.agent,
                    "dependencies": t.dependencies,
                    "estimated_duration_minutes": t.estimated_duration_minutes
                }
                for t in tasks
            ]
        }, ensure_ascii=False, indent=2)
    )]
```

---

## 四、Agent 集成 MCP

### Claude / Cursor 集成

```bash
# Claude Code
claude mcp add multi-agent \
  --command "python mcp_server.py" \
  --env OPENCLAW_CONFIG=/path/to/openclaw.json

# Cursor（通过 mcp.json）
{
  "mcpServers": {
    "multi-agent": {
      "command": "python",
      "args": ["/path/to/mcp_server.py"]
    }
  }
}
```

### Claude 调用示例

```
用户：我想做一个电商小程序，包含商品展示、购物车、下单支付功能。

Claude（调用 decompose_and_execute_project 工具）：
{
  "user_requirement": "电商小程序：商品展示、购物车、下单支付",
  "project_type": "mini_program"
}

工具返回：
{
  "status": "completed",
  "tasks_executed": 12,
  "artifacts": [
    {"path": "shared/01_PRD/PRD.md", "type": "document"},
    {"path": "shared/02_UI/Wireframes/HomePage.md", "type": "document"},
    {"path": "shared/03_Arch/APISpec.yaml", "type": "document"},
    {"path": "/backend/", "type": "code"},
    {"path": "/frontend/", "type": "code"}
  ],
  "duration_seconds": 14400
}

Claude：项目已完成。包含 PRD、UI 线框图、API Spec、后端和前端代码。
关键产物：
- PRD：shared/01_PRD/PRD.md
- API Spec：shared/03_Arch/APISpec.yaml
- 后端代码：/backend/
- 前端代码：/frontend/

需要我帮你 review 哪个部分？
```

---

## 五、OpenClaw + MCP 的双向集成

### 方向 1：OpenClaw 调用外部 MCP Server

```json
{
  "agents": {
    "feishu_ops": {
      "tools": [
        "mcp:feishu:create_task",
        "mcp:feishu:update_bitable"
      ],
      "mcp_servers": {
        "feishu": {
          "command": "python /path/to/feishu_mcp_server.py"
        }
      }
    }
  }
}
```

### 方向 2：把 OpenClaw 暴露为 MCP Server

让 Claude / Cursor / 其他 Agent 框架可以调用 OpenClaw 的多 Agent 能力。

详见本篇前面"MCP Server 实现"部分。

### 双向集成的优势

- **OpenClaw 作为编排核心**，调用多个 MCP 工具。
- **其他 Agent 框架**也能调用 OpenClaw。
- **工具市场形成**：所有 Agent 都能用所有工具。

---

## 六、MCP Server 实战清单

### 开发期

- [ ] 定义清晰的 Tool / Resource / Prompt。
- [ ] 输入 schema 严格（JSON Schema）。
- [ ] 输出标准化（JSON）。
- [ ] 错误友好（错误信息明确）。
- [ ] 日志完整（trace_id 贯穿）。

### 测试期

- [ ] 单元测试（每个 Tool 独立测试）。
- [ ] 集成测试（与 Agent 集成测试）。
- [ ] 性能测试（响应时间、并发）。
- [ ] 错误测试（异常情况处理）。

### 上线期

- [ ] 部署（Docker / K8s）。
- [ ] 监控（调用量、错误率、延迟）。
- [ ] 告警（异常时通知）。
- [ ] 文档（Tool 说明 + 示例）。

---

## 七、本章关键 takeaway

- **MCP** 是 Agent 工具调用的**事实标准**（类比 USB-C）。
- **三大核心**：Tool（工具）、Resource（资源）、Prompt（提示词）。
- **把多 Agent 封装为 MCP Server**，让任何 Agent 可调用。
- **OpenClaw 与 MCP 双向集成**：OpenClaw 调外部工具 + OpenClaw 暴露给外部 Agent。
- **MCP 工具市场**是未来趋势，所有 Agent 能用所有工具。

---

**返回**：[10-tech-multi-agent README](./README.md)
**上一篇**：[14 - Agent 系统的错误处理策略](./14-Agent系统的错误处理策略.md)
