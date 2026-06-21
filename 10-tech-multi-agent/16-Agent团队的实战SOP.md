# 16 - Agent 团队的实战 SOP

> **TL;DR**：SOP（标准操作流程）是 Agent 团队稳定运行的**关键**。本篇给出 6 类 Agent 团队的实战 SOP 模板。

---

## 一、什么是 Agent SOP

**SOP**（Standard Operating Procedure）= 把"最佳实践"变成**可重复执行的步骤**。

### 传统 SOP vs Agent SOP

| 维度 | 传统 SOP | Agent SOP |
|---|---|---|
| 读者 | 人类员工 | AI Agent + 人类监督 |
| 形式 | 文档 | Prompt + Workflow + Skill |
| 灵活性 | 较低（人执行） | 中（LLM 推理） |
| 可执行性 | 培训后 | 自动化 |

### Agent SOP 的核心

- **明确的输入**：每个步骤的输入是什么。
- **明确的输出**：每个步骤的输出是什么（schema）。
- **明确的标准**：什么算"做好"（guardrails）。
- **明确的异常处理**：出错怎么办。

---

## 二、SOP 模板结构

```markdown
# [SOP 名称]

## 目标
做这件事要达到什么目标。

## 输入
需要什么输入（用户需求、上下文）。

## 输出
输出什么（产物 + schema）。

## 步骤
1. 步骤 1：...
2. 步骤 2：...
3. 步骤 3：...

## 质量标准
- 标准 1
- 标准 2

## 异常处理
- 异常 1：处理方式
- 异常 2：处理方式

## 工具
- 工具 1
- 工具 2
```

---

## 三、6 类 Agent 团队的实战 SOP

### SOP 1：Product Agent（产品 Agent）

#### 目标

把用户需求转化为结构化的产品需求文档（PRD）。

#### 输入

- 用户原始需求。
- 项目上下文（行业、用户规模、约束）。

#### 输出

- PRD.md（包含 Overview、User Stories、Acceptance Criteria、Business Rules）。
- UserStories.md。
- AcceptanceCriteria.md。

#### 步骤

```yaml
1. 意图解析
   - 提取：项目类型、目标平台、核心功能、用户规模、技术偏好、约束
   - 识别：ambiguity_flags（必须显式标记）
   - 输出：intent.json

2. 任务原子化
   - 把模糊需求拆为原子功能
   - 每个功能是 1-4 小时工作量
   - 输出：feature_list.json

3. 用户故事编写
   - 每个功能 → 1-3 个用户故事
   - 格式："As a [role], I want [feature], so that [benefit]"
   - 输出：UserStories.md

4. 验收标准
   - 每个用户故事 → 1-3 条验收标准
   - 格式：Given/When/Then
   - 输出：AcceptanceCriteria.md

5. 业务规则
   - 提取业务规则（如"会员等级 >= 3 才能用 XX"）
   - 输出：BusinessRules.md

6. 整合 PRD
   - 把所有内容整合为 PRD.md
   - 输出：PRD.md
```

#### 质量标准

- PRD 字数 > 500。
- 包含 User Stories、Acceptance Criteria、Business Rules 三节。
- 所有核心功能都有对应用户故事。
- 验收标准可量化。

#### 异常处理

- 需求模糊 → 输出 ambiguity_flags，触发用户澄清。
- 用户故事不清晰 → 重新生成或人工补充。

---

### SOP 2：UI Designer Agent（UI 设计 Agent）

#### 目标

基于 PRD 设计 UI 线框图、交互规范、组件库。

#### 输入

- PRD.md。
- 用户故事。
- 验收标准。

#### 输出

- DesignSystem.md（颜色、字体、间距、组件）。
- Wireframes/（每个页面一个文件）。
- InteractionSpecs.md。

#### 步骤

```yaml
1. 设计系统定义
   - 颜色：主色、辅色、状态色
   - 字体：标题、正文、辅助
   - 间距：基础单位、间距梯度
   - 组件：按钮、输入框、列表、卡片等
   - 输出：DesignSystem.md

2. 页面拆分
   - 从 PRD 提取所有页面
   - 每个页面是独立文件
   - 输出：page_list.json

3. 线框图设计
   - 每个页面：
     - 布局结构（Markdown + ASCII 图）
     - 组件清单
     - 响应式规则（mobile / tablet / desktop）
     - 状态（加载/空/有数据/错误）
   - 输出：Wireframes/[PageName].md

4. 交互规范
   - 状态机（哪些状态、转换条件）
   - 手势 / 动画
   - 输出：InteractionSpecs.md

5. 一致性检查
   - 所有页面使用统一的 Design System
   - 命名规范一致
   - 间距、对齐一致
```

#### 质量标准

- DesignSystem 完整（4 节齐全）。
- 每个页面有 mobile / tablet / desktop 规则。
- 每个页面有 4 个状态（加载/空/有数据/错误）。
- 组件命名规范一致。

#### 异常处理

- PRD 不完整 → 反馈给 Orchestrator，要求补充。
- 页面过多 → 分批设计，先核心页面。

---

### SOP 3：Architect Agent（架构师 Agent）

#### 目标

基于 PRD 设计系统架构、API、数据库 Schema。

#### 输入

- PRD.md。
- UserStories.md。

#### 输出

- Architecture.md（系统架构图、技术选型、部署方案）。
- TechStack.md。
- APISpec.yaml（OpenAPI 3.0）。
- Schema.sql（数据库 DDL）。

#### 步骤

```yaml
1. 技术选型
   - 前端框架（React / Vue / 小程序原生）
   - 后端框架（Node.js / Python / Go）
   - 数据库（PostgreSQL / MySQL / MongoDB）
   - 第三方服务（认证、支付、消息）
   - 输出：TechStack.md（含选型理由）

2. 系统架构
   - 整体架构图（microservices / monolith）
   - 模块划分
   - 数据流
   - 部署方案（云服务、容器）
   - 输出：Architecture.md

3. API 设计
   - RESTful / GraphQL
   - 端点列表
   - 请求/响应 schema
   - 错误处理
   - 输出：APISpec.yaml（OpenAPI 3.0）

4. 数据模型
   - 实体关系图（ERD）
   - 表结构
   - 索引设计
   - 输出：Schema.sql

5. 一致性检查
   - API 与 Schema 对应（每个 entity 有 CRUD 端点）
   - 架构与 PRD 对应（每个核心功能有对应模块）
```

#### 质量标准

- OpenAPI 3.0 规范通过验证。
- 所有端点都有 request/response schema。
- 所有表都有主键和外键索引。
- 架构图清晰、可实施。

---

### SOP 4：Coder Agent（编码 Agent）

#### 目标

基于 APISpec、Schema、Wireframes 实现代码。

#### 输入

- APISpec.yaml。
- Schema.sql。
- Wireframes/。
- DesignSystem.md。

#### 输出

- `/backend/`（后端代码 + 测试）。
- `/frontend/`（前端代码 + 测试）。
- README.md（使用说明）。

#### 步骤

```yaml
1. 项目初始化
   - 创建项目结构
   - 配置依赖
   - 配置 lint、test

2. 后端实现
   - 按 APISpec 实现每个端点
   - 数据库连接
   - 错误处理
   - 中间件（认证、日志、限流）
   - 单元测试（覆盖率 > 80%）

3. 前端实现
   - 项目结构
   - 按 Wireframes 实现页面
   - API 调用
   - 状态管理
   - 组件复用

4. 集成
   - 前后端联调
   - E2E 测试
   - 性能测试

5. 文档
   - README.md
   - API 文档（自动生成）
```

#### 质量标准

- lint 通过（0 error）。
- type check 通过（TypeScript）。
- 单元测试覆盖率 > 80%。
- API 实现严格匹配 APISpec。
- README 完整。

---

### SOP 5：QA Agent（测试 Agent）

#### 目标

执行测试，发现 bug，输出测试报告。

#### 输入

- 代码（/backend/, /frontend/）。
- APISpec.yaml。
- AcceptanceCriteria.md。

#### 输出

- TestReport.md。
- BugReport.md。
- CodeReview.md。

#### 步骤

```yaml
1. 测试用例设计
   - 从 AcceptanceCriteria 提取测试用例
   - 边界 case
   - 异常 case
   - 输出：test_cases.json

2. 单元测试
   - 验证代码逻辑
   - 覆盖率检查

3. 集成测试
   - 验证模块间协作
   - API 端到端测试

4. UI 测试
   - 自动化 UI 测试（Playwright/Cypress）
   - 跨浏览器测试
   - 响应式测试

5. Bug 报告
   - 每个 bug：复现步骤、期望、实际、相关文件
   - 按严重程度排序
   - 输出：BugReport.md

6. 测试报告
   - 覆盖范围
   - 测试结果
   - 失败用例
   - 输出：TestReport.md

7. 代码审查
   - 可读性
   - 性能
   - 安全性
   - 输出：CodeReview.md
```

#### 质量标准

- 所有验收标准有对应测试。
- critical bug = 0。
- major bug ≤ 2。
- 测试覆盖率 > 80%。

---

### SOP 6：DevOps Agent（部署 Agent）

#### 目标

打包、部署、CI/CD 配置。

#### 输入

- /backend/、/frontend/。
- Architecture.md（部署方案）。

#### 输出

- Dockerfile。
- docker-compose.yml。
- .github/workflows/。
- DeploymentGuide.md。

#### 步骤

```yaml
1. Docker 化
   - 后端 Dockerfile
   - 前端 Dockerfile
   - 输出：Dockerfile

2. 编排
   - docker-compose.yml
   - 环境变量
   - 网络配置
   - 输出：docker-compose.yml

3. CI/CD
   - GitHub Actions / GitLab CI
   - 自动化测试
   - 自动化部署
   - 输出：.github/workflows/

4. 健康检查
   - /health 端点
   - readiness / liveness probes
   - 输出：在代码中

5. 部署文档
   - 部署步骤
   - 环境要求
   - 故障排查
   - 输出：DeploymentGuide.md
```

#### 质量标准

- Docker 镜像构建成功。
- 健康检查端点存在。
- 环境变量文档化。
- 部署幂等。

---

## 四、SOP 的维护

### 维护节奏

- **每次任务后**：检查 SOP 是否被遵循。
- **每月**：复盘 SOP，识别改进点。
- **每季度**：大版本更新 SOP。

### 改进流程

```
SOP 执行
    ↓
发现偏差（哪里不顺畅）
    ↓
分析原因
    ↓
更新 SOP
    ↓
重新发布
```

### SOP 改进的输入

- **失败案例**：哪里经常出错？
- **用户反馈**：哪里让 Agent 困惑？
- **新需求**：有什么新场景没覆盖？

---

## 五、SOP 编写最佳实践

### 实践 1：从"任务"出发

不要从"工具"出发。从"要完成什么任务"开始，再设计步骤。

### 实践 2：步骤要"原子化"

每个步骤 1-4 小时工作量。太粗不行，太细也不行。

### 实践 3：质量标准要可验证

"看起来不错"不是标准。"测试覆盖率 > 80%"是标准。

### 实践 4：异常处理要明确

每个 SOP 都要有"如果出错怎么办"。

### 实践 5：定期演练

SOP 是"纸面"的，必须**实际跑过**才能发现问题。

---

## 六、本章关键 takeaway

- **SOP** 是 Agent 团队稳定运行的关键。
- **6 类 Agent SOP**：Product、UI Designer、Architect、Coder、QA、DevOps。
- 每个 SOP 含**目标、输入、输出、步骤、质量标准、异常处理**。
- **SOP 维护**：每月复盘、季度大版本。
- **编写原则**：从任务出发、步骤原子化、质量可验证、异常明确、定期演练。

---

**返回**：[10-tech-multi-agent README](./README.md)
