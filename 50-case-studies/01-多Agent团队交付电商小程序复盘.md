# 01 - 多 Agent 团队交付电商小程序复盘

> **TL;DR**：用 OpenClaw 多 Agent 团队**完整跑通**一个中型电商小程序项目（PRD → UI → 架构 → 编码 → 测试 → 部署），总耗时 8 小时，节省人工约 70%。本篇是完整复盘。

---

## 一、项目背景

### 项目概况

- **项目**：电商小程序 MVP（类似"小红书 + 淘宝"轻量版）。
- **规模**：5 个核心页面（首页、商品列表、商品详情、购物车、个人中心）。
- **技术栈**：微信小程序 + TypeScript + Node.js + PostgreSQL + Redis。
- **原人工估算**：2 个工程师 × 2 周 = 160 工时。
- **目标**：用多 Agent 团队跑通 MVP，验证可行性。

### 团队情况

- **Orchestrator**：1 个 Agent（GPT-4）。
- **Specialist**：6 个 Agent（产品 / UI / 架构 / 前端 / 后端 / QA / DevOps）。
- **HITL 节点**：3 个（PRD 确认、架构评审、上线审批）。

---

## 二、完整工作流

### Wave 1：PRD 编写

**Agent**：Product Agent。
**耗时**：12 分钟。
**输入**：用户需求"做一个电商小程序"。
**输出**：

- `PRD.md`（约 3500 字）
- `UserStories.md`（8 个用户故事）
- `AcceptanceCriteria.md`（24 条验收标准）

**质量门禁**：
- ✅ 字数 > 500
- ✅ 包含 User Stories
- ✅ 包含 Acceptance Criteria

**HITL**：人工 review PRD，通过。

### Wave 2：设计系统 + 架构

**Agents**：UI Designer + Architect（并行）。
**耗时**：25 分钟。
**输出**：

- `DesignSystem.md`（颜色、字体、间距、组件清单）
- `Architecture.md`（系统架构图、技术栈说明）
- `TechStack.md`（前后端框架、数据库、第三方服务）

**质量门禁**：
- ✅ 设计系统定义完整
- ✅ 架构图清晰

**HITL**：人工 review 架构，通过。

### Wave 3：详细设计 + API Spec

**Agents**：UI Designer（页面线框图）+ Architect（API Spec）。
**耗时**：35 分钟。
**输出**：

- `Wireframes/`（5 个页面线框图）
- `APISpec.yaml`（OpenAPI 3.0，18 个端点）
- `Schema.sql`（8 张表的 DDL）

**质量门禁**：
- ✅ Wireframes 覆盖所有页面
- ✅ API Spec 是有效的 OpenAPI 3.0
- ✅ Schema 有主键和外键

### Wave 4：编码实现

**Agents**：Coder Frontend + Coder Backend（并行）。
**耗时**：2 小时 15 分钟。
**输出**：

- `/frontend/`（微信小程序代码，约 1500 行 TypeScript）
- `/backend/`（Node.js 代码，约 2000 行）
- 测试用例（约 500 行）

**质量门禁**：
- ✅ Lint 通过
- ✅ Type check 通过
- ✅ 测试覆盖率 87%

**踩坑**：前端 Coder 第一次输出有 2 个页面没实现完整，被门禁拦截，1 次重试后通过。

### Wave 5：测试验证

**Agent**：QA Agent。
**耗时**：18 分钟。
**输出**：

- `TestReport.md`
- `BugReport.md`（3 个 minor bug，无 critical）

**质量门禁**：
- ✅ 所有验收标准有对应测试
- ✅ critical bug = 0
- ✅ major bug ≤ 2

**踩坑**：1 个 bug 是 Orchestrator 路由错误（QA → Coder Backend，但其实是 Coder Frontend 的问题）。修正路由后修复。

### Wave 6：部署上线

**Agent**：DevOps Agent。
**耗时**：15 分钟。
**输出**：

- `Dockerfile`
- `docker-compose.yml`
- `DeploymentGuide.md`

**质量门禁**：
- ✅ Docker 镜像构建成功
- ✅ 健康检查端点存在
- ✅ 环境变量文档完整

**HITL**：人工最终 review，部署到测试环境。

---

## 三、时间线

```
09:00  项目启动，Orchestrator 开始任务分解
09:12  Wave 1 完成（PRD）
09:37  Wave 2 完成（设计 + 架构）
10:12  Wave 3 完成（详细设计 + API）
12:27  Wave 4 完成（编码实现）  ← 最长
12:45  Wave 5 完成（测试）
13:00  Wave 6 完成（部署）
13:00  项目交付

总耗时：4 小时
总等待 + 异步：约 8 小时（含 HITL、人工 review、门禁等待等）
```

---

## 四、量化结果

### 节省成本

| 项目 | 传统人工 | 多 Agent 团队 | 节省 |
|---|---|---|---|
| PRD 编写 | 4 小时 | 12 分钟 | 95% |
| UI 设计 | 16 小时 | 35 分钟 | 96% |
| 架构设计 | 8 小时 | 35 分钟 | 93% |
| 编码实现 | 80 小时 | 2 小时 15 分钟 | 97% |
| 测试验证 | 16 小时 | 18 分钟 | 98% |
| 部署上线 | 4 小时 | 15 分钟 | 94% |
| **总计** | **128 小时** | **4 小时（执行）** | **97%** |

### 实际成本

- **Token 成本**：约 $35（GPT-4 调用约 80 次）
- **HITL 人工成本**：约 30 分钟（PRD review + 架构 review + 最终 review）
- **总时间**：约 4 小时执行 + 人工 review

### 质量对比

| 指标 | 传统人工 | 多 Agent 团队 |
|---|---|---|
| 代码规范一致性 | 中（取决于工程师） | 高（Agent 严格遵循 prompt） |
| 测试覆盖率 | 70-80% | 87% |
| 文档完整度 | 中 | 高（强制输出） |
| 架构合理性 | 取决于架构师 | 中（依赖模板） |

---

## 五、踩过的坑

### 坑 1：上下文爆炸导致 PRD 漏字段

**现象**：第一版 PRD 漏掉了"会员等级"功能。

**原因**：LLM 上下文有限，复杂需求被"截断"。

**解决**：
- 任务拆分更细（先收集功能，再写 PRD）。
- 增加"功能 checklist" 步骤。

### 坑 2：API Spec 与 Schema 不一致

**现象**：前端 Coder 报告 `orders` 表字段与 API 不一致。

**原因**：Architect Agent 输出 Schema 和 APISpec 时，分别调用了两次 LLM，没保证一致性。

**解决**：
- 改为 Architect Agent 一次性输出（包含 API + Schema + 一致性检查）。
- 增加"自动一致性校验"门禁。

### 坑 3：HITL review 太久

**现象**：PRD review 等待 1 小时（人忙别的事）。

**原因**：HITL 节点是阻塞的。

**解决**：
- HITL 节点设置 SLA（24 小时内必须 review，否则自动通过或升级）。
- 多级审批人（主审批 + 备份审批）。

### 坑 4：成本失控风险

**现象**：Wave 4 编码实现时，Agent 多次重试，单次任务 token 消耗超预期。

**原因**：质量门禁失败导致多次重试。

**解决**：
- 增加 max_retries 限制（默认 3 次）。
- 失败时升级到 HITL，而不是无限重试。

---

## 六、关键教训

### 教训 1：模板质量决定结果

Orchestrator 使用的项目模板如果质量差，分解出的任务就差。

**实践**：投资时间打磨模板。

### 教训 2：质量门禁要"严"

如果门禁太松，下游 Agent 会被错误产物带偏。

**实践**：每个门禁都要"机器可验证"，不能"看起来不错"。

### 教训 3：HITL 节点必须有限流

否则流程会被"人工等待"拖慢。

**实践**：所有 HITL 节点都有 SLA + 升级路径。

### 教训 4：可观测性救命

4 小时跑通，但中间出了 2 次问题（漏字段、不一致），靠 Trace 日志快速定位。

**实践**：从第一天起，所有 Agent 调用都有 Trace。

---

## 七、可复用资产

### 1. 项目模板

```yaml
# templates/ecommerce_miniapp.yaml
project_type: "mini_program"
phases:
  - id: "P1"
    name: "Product Planning"
    tasks: [...]
  # ... 6 个阶段
```

### 2. 工作流定义

```yaml
# lobsters/ecommerce_project.yaml
waves: [...]
```

### 3. 质量门禁规则

```yaml
# quality_gates.yaml
gates:
  - prd_completeness
  - api_spec_validity
  - code_quality
  # ...
```

### 4. SOUL.md 模板

每个 Agent 的 `SOUL.md` 都可复用。

---

## 八、本章关键 takeaway

- 多 Agent 团队 4 小时跑通中型电商小程序（vs 传统 128 小时）。
- Token 成本约 $35，可接受。
- 质量比传统人工略高（一致性、文档、测试覆盖率）。
- 关键挑战：模板质量、门禁严格度、HITL SLA、可观测性。
- 复盘后整理出可复用的模板、工作流、门禁、SOUL.md。

---

**返回**：[50-case-studies README](./README.md)
**下一篇**：[02 - 飞书会议 Agent 在创业公司的应用](./02-飞书会议Agent在创业公司的应用.md)
