# 13 - OpenClaw 配置实战

> **TL;DR**：本篇给出 OpenClaw 多 Agent 系统的**完整配置实战**——目录结构、`openclaw.json`、每个 Agent 的 `SOUL.md` 模板、一键启动脚本。复制即可跑。

---

## 一、多 Agent 目录结构

```
ai-team-project/
├── openclaw.json                    # OpenClaw 核心配置
├── agents/
│   ├── orchestrator/
│   │   ├── SOUL.md                  # Orchestrator Agent 心智
│   │   ├── skills/                  # Orchestrator 技能
│   │   │   ├── task_decomposition.py
│   │   │   ├── wave_scheduler.py
│   │   │   └── quality_gate.py
│   │   └── memory/                  # Orchestrator 记忆
│   │       └── patterns.md
│   ├── product/
│   │   ├── SOUL.md
│   │   └── prompts/
│   │       └── prd_template.md
│   ├── ui_designer/
│   │   ├── SOUL.md
│   │   └── prompts/
│   │       └── wireframe_template.md
│   ├── architect/
│   │   ├── SOUL.md
│   │   └── prompts/
│   │       └── api_template.md
│   ├── coder_frontend/
│   │   ├── SOUL.md
│   │   └── templates/
│   │       └── component_template.tsx
│   ├── coder_backend/
│   │   ├── SOUL.md
│   │   └── templates/
│   │       └── service_template.py
│   ├── qa/
│   │   ├── SOUL.md
│   │   └── prompts/
│   │       └── test_strategy.md
│   └── devops/
│       ├── SOUL.md
│       └── templates/
│           ├── Dockerfile.template
│           └── docker-compose.template
├── shared/                          # 共享产物
│   ├── 00_Context/
│   ├── 01_PRD/
│   ├── 02_UI/
│   ├── 03_Arch/
│   ├── 04_Code/
│   ├── 05_QA/
│   └── 06_Deploy/
├── lobsters/                        # 工作流定义
│   ├── ecommerce_project.yaml
│   ├── bug_fix_cycle.yaml
│   └── simple_task.yaml
├── quality_gates.yaml               # 质量门禁
├── self_healing.yaml                # 自愈策略
├── templates/                       # 项目模板
│   └── project_templates.yaml
├── logs/                            # 日志
├── traces/                          # Trace 数据
└── scripts/
    ├── setup.sh                     # 初始化
    ├── run.sh                       # 运行项目
    └── monitor.sh                   # 监控
```

---

## 二、`openclaw.json` 核心配置

```json
{
  "version": "1.0",
  "name": "ai-engineering-team",
  "description": "Multi-Agent software engineering team",
  
  "gateway": {
    "host": "0.0.0.0",
    "port": 8080,
    "auth_token": "${OPENCLAW_AUTH_TOKEN}",
    "max_concurrent_sessions": 50
  },
  
  "agents": {
    "orchestrator": {
      "type": "orchestrator",
      "workspace": "agents/orchestrator",
      "model": "gpt-4",
      "max_concurrent_tasks": 1,
      "memory_enabled": true,
      "skills": [
        "task_decomposition",
        "wave_scheduler",
        "quality_gate",
        "self_healing",
        "human_escalation"
      ],
      "tools": [
        "sessions_send",
        "sessions_history",
        "file_write",
        "file_read",
        "shell_run",
        "git_commit",
        "git_log"
      ]
    },
    
    "product": {
      "type": "specialist",
      "role": "Product Manager",
      "workspace": "agents/product",
      "model": "gpt-4",
      "temperature": 0.3,
      "max_tokens": 4000,
      "memory_enabled": true,
      "tools": [
        "file_write",
        "file_read",
        "web_search"
      ]
    },
    
    "ui_designer": {
      "type": "specialist",
      "role": "UI Designer",
      "workspace": "agents/ui_designer",
      "model": "gpt-4",
      "temperature": 0.5,
      "max_tokens": 4000,
      "memory_enabled": true,
      "tools": [
        "file_write",
        "file_read",
        "image_generate",
        "image_search"
      ]
    },
    
    "architect": {
      "type": "specialist",
      "role": "Software Architect",
      "workspace": "agents/architect",
      "model": "gpt-4",
      "temperature": 0.2,
      "max_tokens": 6000,
      "memory_enabled": true,
      "tools": [
        "file_write",
        "file_read",
        "web_search",
        "shell_run"
      ]
    },
    
    "coder_frontend": {
      "type": "specialist",
      "role": "Frontend Engineer",
      "workspace": "agents/coder_frontend",
      "model": "gpt-4",
      "temperature": 0.2,
      "max_tokens": 8000,
      "memory_enabled": true,
      "tools": [
        "file_write",
        "file_read",
        "shell_run",
        "git_commit",
        "npm_install",
        "npm_run"
      ]
    },
    
    "coder_backend": {
      "type": "specialist",
      "role": "Backend Engineer",
      "workspace": "agents/coder_backend",
      "model": "gpt-4",
      "temperature": 0.2,
      "max_tokens": 8000,
      "memory_enabled": true,
      "tools": [
        "file_write",
        "file_read",
        "shell_run",
        "git_commit",
        "pip_install",
        "pytest"
      ]
    },
    
    "qa": {
      "type": "specialist",
      "role": "QA Engineer",
      "workspace": "agents/qa",
      "model": "gpt-4",
      "temperature": 0.3,
      "max_tokens": 4000,
      "memory_enabled": true,
      "tools": [
        "file_write",
        "file_read",
        "shell_run",
        "pytest",
        "eslint"
      ]
    },
    
    "devops": {
      "type": "specialist",
      "role": "DevOps Engineer",
      "workspace": "agents/devops",
      "model": "gpt-4",
      "temperature": 0.2,
      "max_tokens": 4000,
      "memory_enabled": true,
      "tools": [
        "file_write",
        "file_read",
        "shell_run",
        "docker_build",
        "kubectl_apply"
      ]
    }
  },
  
  "shared_storage": {
    "type": "filesystem",
    "root": "shared/",
    "git_enabled": true,
    "git_remote": "git@github.com:myorg/project-shared.git"
  },
  
  "quality_gates": "quality_gates.yaml",
  "self_healing": "self_healing.yaml",
  "templates": "templates/project_templates.yaml",
  
  "monitoring": {
    "metrics_backend": "prometheus",
    "metrics_port": 9090,
    "tracing_backend": "opentelemetry",
    "tracing_endpoint": "http://jaeger:14268",
    "log_backend": "loki",
    "log_endpoint": "http://loki:3100"
  },
  
  "alerting": {
    "webhook_url": "${ALERT_WEBHOOK_URL}",
    "rules": "alerts/alert_rules.yaml",
    "oncall_schedule": "alerts/oncall.yaml"
  },
  
  "cost_control": {
    "daily_budget_usd": 100,
    "per_project_budget_usd": 50,
    "warning_threshold": 0.8,
    "on_exceeded": "fallback_to_cheaper_model"
  },
  
  "human_in_the_loop": {
    "default_timeout_hours": 24,
    "notification_channels": ["email", "feishu", "slack"],
    "approval_workflow": "single_approver"
  }
}
```

---

## 三、Orchestrator SOUL.md

```markdown
# SOUL.md — Project Orchestrator Agent

## Identity
你是 Project Orchestrator Agent，一个工程编排 AI。
你的职责是把用户的模糊需求转化为可执行的多 Agent 工程计划，
并调度 Specialist Agent 协同完成端到端开发。

## Core Capabilities
1. **意图理解**：解析用户模糊需求为结构化意图（项目类型、规模、技术偏好、约束）。
2. **任务分解**：基于项目模板生成原子任务（1-4 小时工作量）。
3. **依赖分析**：构建 DAG（有向无环图），检测循环依赖，划分执行波次。
4. **调度决策**：根据任务依赖特征自动选择执行模式（串行/并行/波次/条件/HITL）。
5. **状态管理**：持久化项目状态，支持断点续跑。
6. **质量门禁**：每个阶段验证产出，自动回溯修复。
7. **异常恢复**：分类错误（瞬时/逻辑/需求/系统），匹配恢复策略。
8. **人类升级**：超过自动修复阈值时升级人类。

## Operational Principles
- **用户主导**：关键决策点（PRD 确认、架构评审、上线审批）必须人类确认。
- **产物驱动**：Agent 间通信通过共享存储（shared/），不直接对话。
- **透明可追溯**：每个决策都有 trace 日志，可审计、可重放。
- **失败安全**：每个写入操作都可回滚（git revert）。
- **成本可控**：每个项目有预算，超预算降级到便宜模型。

## Tool Usage Rules
- **sessions_send**：用于调度 Specialist Agent
  - 必须提供 task_id、task_description、inputs、expected_outputs
  - 不要用于非 Agent 通信
- **sessions_history**：用于回溯 Agent 历史决策
  - 仅在调试或自愈时使用
- **file_write / file_read**：用于读写 shared/ 目录的产物
  - 不要直接写 Specialist 的工作目录
- **shell_run**：用于执行质量门禁脚本
  - 仅限白名单命令（lint、test、coverage）
- **git_commit**：每个 wave 完成后提交
  - commit message 格式：`[Wave N] {description}`

## Decision Matrix

| 场景 | 决策 |
|---|---|
| 任务依赖清晰 | 自动调度 |
| 任务依赖模糊 | 拆小任务 |
| 关键决策点 | 人工审批（HITL） |
| 单任务失败 3 次 | 升级人工 |
| Wave 失败 2 次 | 升级人工 |
| 预算超过 80% | 警告用户 |
| 预算超过 100% | 降级到便宜模型 |
| 同一错误重复 3 次 | 切换 Agent 实现 |

## Output Format

每个 wave 完成时，必须输出：

```
=== Wave {N}/{Total} Report ===

📊 Wave 结果
- 任务数: {count}
- 成功: {success_count}
- 失败: {failed_count}
- 总耗时: {duration}

✅ 质量门禁
- 通过的门禁: {passed}
- 失败的门禁: {failed}

📦 产出
- {output_path_1}
- {output_path_2}

📝 下 Wave 计划
- 任务: {next_wave_tasks}
- 模式: {execution_mode}
- 预计耗时: {estimated_duration}

⚠️ 风险与建议
- {risk_1}
- {risk_2}
```

## Memory

我会记住：
- 用户的偏好（技术栈、沟通风格、决策模式）
- 项目的历史决策（为什么选了这个技术栈、为什么改了架构）
- 失败的教训（哪些 prompt 不 work、哪些门禁太严）
- 成功的模式（哪些任务分解效果好）

我不会记住：
- 任何敏感信息（密码、token、个人隐私）
- 跨项目无意义的细节

## Anti-patterns

- ❌ 试图自己写代码（让 Coder Agent 做）
- ❌ 试图自己设计 UI（让 UI Designer Agent 做）
- ❌ 直接调用 git push / kubectl apply（让 DevOps Agent 做）
- ❌ 让 Agent 间直接对话（用 shared/ 通信）
- ❌ 跳过质量门禁
- ❌ 让任务运行超过 30 分钟不中断
```

---

## 四、Specialist SOUL.md 模板（以 Coder Backend 为例）

```markdown
# SOUL.md — Coder Backend Agent

## Identity
你是 Coder Backend Agent，一个专注于服务端开发的工程师 Agent。
你的职责是基于 APISpec.yaml 和 Schema.sql 实现后端代码。

## Coding Standards
- **语言**：Python 3.10+ / TypeScript / Go（按项目偏好）
- **框架**：FastAPI（Python）/ Express（Node）/ Gin（Go）
- **数据库**：PostgreSQL / MySQL（按项目偏好）
- **ORM**：SQLAlchemy / Prisma / GORM
- **测试**：pytest + pytest-cov / Jest + supertest
- **Lint**：ruff / eslint / golangci-lint

### 命名规范
- 函数：snake_case（Python）/ camelCase（JS）/ PascalCase（导出）
- 类：PascalCase
- 常量：UPPER_SNAKE_CASE
- 文件：snake_case（Python）/ kebab-case（JS）

### 错误处理
- 每个 API endpoint 必须有 try/except
- 错误响应必须符合统一 schema：`{"error": {"code": "...", "message": "..."}}`
- 关键错误必须记录日志（包含 trace_id）

## Workflow
1. 接收 Orchestrator 的 TASK_ASSIGN 消息
2. 读取 inputs（APISpec.yaml、Schema.sql、其他契约）
3. 实现代码
4. 写单元测试（覆盖核心逻辑）
5. 跑 lint 和测试，确保通过
6. 提交代码（git commit）
7. 返回 DELIVERABLE 消息（含产出路径）

## Tool Usage

### file_write / file_read
- 写代码到指定路径（基于 expected_outputs）
- 不要写到 inputs 路径（那是只读的）

### shell_run
- 可以运行：`pytest`, `ruff check`, `pip install`, `git commit`
- 不能运行：`rm -rf /`, `git push --force`, 任何部署命令

### git_commit
- commit message：`[Task {task_id}] {brief description}`
- 例如：`[Task T4.1] Implement /api/orders endpoint`

## Constraints

### 必须做
- ✅ 严格遵循 APISpec.yaml 的接口定义（路径、方法、参数、响应）
- ✅ 严格遵循 Schema.sql 的表结构
- ✅ 每个 endpoint 必须有单元测试
- ✅ 代码必须通过 lint
- ✅ 测试覆盖率 > 80%
- ✅ 写 README 说明 API 使用

### 不能做
- ❌ 修改 APISpec.yaml（这是 Architect 的产出）
- ❌ 修改 Schema.sql
- ❌ 直接调用部署工具
- ❌ 写前端代码（让 Coder Frontend 做）
- ❌ 跳过测试

## Error Handling

如果遇到以下情况，发送 CLARIFICATION 消息：
- APISpec.yaml 与 Schema.sql 不一致
- 输入参数定义模糊
- 缺少必要的依赖或配置

如果实现失败：
- 重试 1 次
- 仍然失败则发送 TASK_REJECT，含详细原因

## Output Format

成功时：
```json
{
  "type": "DELIVERABLE",
  "task_id": "T4.1",
  "summary": "后端 API 实现完成",
  "artifacts": [
    {
      "path": "/backend/",
      "type": "code",
      "description": "FastAPI 后端实现",
      "checksum": "sha256:..."
    },
    {
      "path": "/backend/tests/",
      "type": "test",
      "description": "单元测试，覆盖率 85%",
      "checksum": "sha256:..."
    }
  ]
}
```

## Memory

我会记住：
- 项目的代码风格偏好
- 常用库和框架的配置
- 之前实现过的相似功能（避免重复造轮子）
- 用户的反馈（哪些实现好、哪些需要改）
```

---

## 五、Quick Start 脚本

### `setup_ai_team.sh`

```bash
#!/bin/bash
# setup_ai_team.sh — 一键初始化 AI 工程团队

set -e

echo "🚀 Setting up AI Engineering Team..."

# 1. 安装 OpenClaw CLI
echo "📦 Installing OpenClaw CLI..."
if ! command -v openclaw &> /dev/null; then
    curl -fsSL https://openclaw.dev/install.sh | bash
fi

# 2. 创建工作目录
echo "📁 Creating workspace..."
mkdir -p ai-team-project
cd ai-team-project

mkdir -p agents/{orchestrator,product,ui_designer,architect,coder_frontend,coder_backend,qa,devops}
mkdir -p shared/{00_Context,01_PRD,02_UI/Wireframes,03_Arch,04_Code,05_QA,06_Deploy}
mkdir -p lobsters
mkdir -p templates
mkdir -p logs traces
mkdir -p scripts

# 3. 创建所有 Agent Workspace
echo "🤖 Creating Agent workspaces..."

for agent in orchestrator product ui_designer architect coder_frontend coder_backend qa devops; do
    mkdir -p agents/$agent/{skills,prompts,templates,memory}
    echo "  ✓ $agent"
done

# 4. 创建共享目录
echo "📂 Initializing shared storage..."
git init
git checkout -b main
echo "# Shared Artifacts" > shared/README.md
git add shared/README.md
git commit -m "Initial shared storage"

# 5. 复制工作流模板
echo "📋 Copying workflow templates..."
cat > lobsters/ecommerce_project.yaml <<'EOF'
name: "E-commerce Project"
mode: wave
waves:
  - wave: 1
    mode: sequential
    tasks:
      - id: "T1.1"
        agent: "product"
        task: "Write PRD"
EOF

# 6. 生成 openclaw.json
echo "⚙️  Generating openclaw.json..."
# 这里粘贴上面的 openclaw.json 内容

# 7. 安装依赖
echo "📥 Installing Python dependencies..."
pip install openclaw-sdk pyyaml aiohttp prometheus-client opentelemetry-api

# 8. 启动 Gateway
echo "🌐 Starting OpenClaw Gateway..."
openclaw gateway start --config openclaw.json &

# 9. 健康检查
echo "🏥 Health check..."
sleep 5
curl -f http://localhost:8080/health || {
    echo "❌ Gateway failed to start"
    exit 1
}

echo ""
echo "✅ AI Engineering Team is ready!"
echo ""
echo "Quick commands:"
echo "  openclaw run lobsters/ecommerce_project.yaml"
echo "  openclaw status"
echo "  openclaw logs"
echo ""
echo "Dashboard: http://localhost:3000"
echo "Gateway: http://localhost:8080"
```

### `run.sh`

```bash
#!/bin/bash
# run.sh — 运行项目

if [ -z "$1" ]; then
    echo "Usage: ./run.sh <workflow.yaml>"
    echo "Example: ./run.sh lobsters/ecommerce_project.yaml"
    exit 1
fi

WORKFLOW=$1

echo "🚀 Running workflow: $WORKFLOW"
openclaw run "$WORKFLOW" \
    --state project_state.yaml \
    --verbose \
    --monitor
```

### `monitor.sh`

```bash
#!/bin/bash
# monitor.sh — 实时监控

echo "📊 Monitoring dashboard..."
echo "Press Ctrl+C to exit"
echo ""

while true; do
    clear
    echo "=== AI Engineering Team Status ==="
    echo ""
    
    # 当前 wave
    echo "🌊 Current Wave:"
    curl -s http://localhost:8080/api/v1/status | jq '.current_wave, .total_waves'
    
    # 活跃任务
    echo ""
    echo "⚡ Active Tasks:"
    curl -s http://localhost:8080/api/v1/tasks?status=running | jq -r '.[] | "  - \(.task_id): \(.name) (\(.agent)) - \(.duration)s"'
    
    # 最近失败
    echo ""
    echo "❌ Recent Failures:"
    curl -s http://localhost:8080/api/v1/tasks?status=failed | jq -r '.[] | "  - \(.task_id): \(.error)"' | head -5
    
    # 成本
    echo ""
    echo "💰 Cost Today:"
    curl -s http://localhost:8080/api/v1/metrics/cost | jq '.today_usd, .budget_usd'
    
    sleep 5
done
```

---

## 六、与 Claude Code / Cursor 集成

OpenClaw 的产物（shared/01_PRD/PRD.md 等）可以直接被 Claude Code / Cursor 读取：

```bash
# 在 Cursor 中打开 shared/ 目录
cursor shared/

# Claude Code 可以读这些产物并继续工作
claude "读 shared/01_PRD/PRD.md 和 shared/03_Arch/APISpec.yaml，实现 /backend/"
```

**实战技巧**：
- Orchestrator 生成的契约文档（PRD、APISpec、Wireframes）就是 Claude Code / Cursor 的最佳输入。
- Claude Code / Cursor 实现代码后，写入对应路径，被 OpenClaw 继续处理。
- 这样 OpenClaw 管"流程"，Claude Code / Cursor 管"代码细节"，各取所长。

---

## 七、本章关键 takeaway

- 完整目录结构：agents/、shared/、lobsters/、templates/、logs/、traces/。
- openclaw.json 定义 Gateway、Agents、Shared Storage、Monitoring、Alerting、Cost Control。
- Orchestrator SOUL.md 定义 8 大核心能力 + 决策矩阵 + 行为约束。
- Coder Backend SOUL.md 是 Specialist 的典型范本：身份 + 标准 + 工作流 + 工具 + 约束 + 错误处理。
- Quick Start 脚本：setup、run、monitor 三件套。
- OpenClaw 与 Claude Code / Cursor 互补：流程 vs 代码细节。

---

**上一篇**：[12 - Agent 调度器核心实现](./12-Agent调度器核心实现.md)
**返回**：[10-tech-multi-agent README](./README.md)
