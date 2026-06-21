# 04 - OpenClaw 实战 FAQ

> OpenClaw 配置 / 部署 / 调试实战中最常被问到的问题。

---

## Q1：OpenClaw 怎么安装？

**答**：参考 [OpenClaw 官方文档](https://openclaw.dev)。一行命令：`curl -fsSL https://openclaw.dev/install.sh | bash`。

---

## Q2：第一个 Agent Workspace 怎么建？

**答**：

```bash
mkdir -p ai-team-project
cd ai-team-project
mkdir -p agents/{orchestrator,product,coder}
openclaw init
```

详见：[../10-tech-multi-agent/13-OpenClaw配置实战.md](../10-tech-multi-agent/13-OpenClaw配置实战.md)

---

## Q3：SOUL.md 是什么？

**答**：每个 Agent 的"心智"——身份、能力、规则、工具、约束、错误处理。

详见：[../10-tech-multi-agent/02-Orchestrator设计模式详解.md](../10-tech-multi-agent/02-Orchestrator设计模式详解.md)

---

## Q4：openclaw.json 怎么配？

**答**：定义 Gateway、Agents、Shared Storage、Monitoring 等。

详见：[../10-tech-multi-agent/13-OpenClaw配置实战.md](../10-tech-multi-agent/13-OpenClaw配置实战.md)

---

## Q5：Lobster DSL 是什么？

**答**：OpenClaw 的工作流编排语言，YAML 格式，描述多 Agent 协作流程。

详见：[../10-tech-multi-agent/07-Lobster-DSL工作流编排.md](../10-tech-multi-agent/07-Lobster-DSL工作流编排.md)

---

## Q6：Agent 之间怎么通信？

**答**：通过 `sessions_send` API + 共享存储（`shared/` 目录）。

详见：[../10-tech-multi-agent/06-Agent间通信协议与产物驱动.md](../10-tech-multi-agent/06-Agent间通信协议与产物驱动.md)

---

## Q7：质量门禁怎么配置？

**答**：在 `quality_gates.yaml` 中定义每个任务的可验证检查项。

详见：[../10-tech-multi-agent/08-质量门禁与自愈机制.md](../10-tech-multi-agent/08-质量门禁与自愈机制.md)

---

## Q8：OpenClaw 怎么调试？

**答**：
- **Trace**：每个 Agent 调用都有 trace_id，用 Jaeger 可视化。
- **Log**：结构化日志，按 trace_id 索引。
- **Metric**：Prometheus + Grafana 仪表盘。

详见：[../10-tech-multi-agent/09-可观测性与监控设计.md](../10-tech-multi-agent/09-可观测性与监控设计.md)

---

## Q9：Agent 卡在某个任务怎么办？

**答**：
- 检查任务日志，看哪里报错。
- 检查质量门禁，看哪个门禁失败。
- 检查 token 限制 / 超时设置。
- 必要时手动中断，重新调度。

详见：[../10-tech-multi-agent/12-Agent调度器核心实现.md](../10-tech-multi-agent/12-Agent调度器核心实现.md)

---

## Q10：怎么减少 Token 消耗？

**答**：
- 用便宜模型处理简单任务。
- 精简 Prompt。
- 启用缓存（重复 query）。
- 设置 max_iterations。

详见：[../40-tech-best-practices/05-成本控制实战.md](../40-tech-best-practices/05-成本控制实战.md)

---

## Q11：HITL 节点设计要注意什么？

**答**：
- 必须有 SLA。
- 必须有升级机制。
- 必须有超时处理。

详见：[../10-tech-multi-agent/07-Lobster-DSL工作流编排.md](../10-tech-multi-agent/07-Lobster-DSL工作流编排.md)

---

## Q12：怎么并行运行多个 Agent？

**答**：在 Lobster 工作流中用 `mode: parallel`。

详见：[../10-tech-multi-agent/05-调度决策引擎实现.md](../10-tech-multi-agent/05-调度决策引擎实现.md)

---

## Q13：怎么部署到生产？

**答**：
- 小规模：单机部署。
- 中等规模：K8s 集群。
- 大规模：多租户 + API Gateway。

详见：[../10-tech-multi-agent/10-部署与渐进式采用路径.md](../10-tech-multi-agent/10-部署与渐进式采用路径.md)

---

## Q14：怎么监控生产环境？

**答**：
- **Metrics**：成功率、延迟、Token 消耗。
- **Logs**：结构化日志 + 集中存储。
- **Traces**：调用链路追踪。
- **Alerts**：分级告警 + 自动响应。

详见：[../10-tech-multi-agent/09-可观测性与监控设计.md](../10-tech-multi-agent/09-可观测性与监控设计.md)

---

## Q15：怎么扩展 Agent 数量？

**答**：从最小（3 Agent）起步，逐步扩展。每个新 Agent 必须解决"明确问题"。

详见：[../10-tech-multi-agent/10-部署与渐进式采用路径.md](../10-tech-multi-agent/10-部署与渐进式采用路径.md)

---

## Q16：怎么让 Agent 持续学习？

**答**：
- 收集失败案例。
- 标注 → 训练数据。
- 迭代 SOUL.md。
- A/B 测试新旧版本。

详见：[../50-case-studies/03-客服Agent的5个月迭代.md](../50-case-studies/03-客服Agent的5个月迭代.md)

---

## Q17：怎么集成 Cursor / Claude Code？

**答**：OpenClaw 生成的契约（PRD、APISpec、Wireframes）就是 Cursor / Claude Code 的最佳输入。**OpenClaw 管流程，Cursor/Claude Code 管代码细节**。

详见：[../10-tech-multi-agent/13-OpenClaw配置实战.md](../10-tech-multi-agent/13-OpenClaw配置实战.md)

---

## Q18：OpenClaw 支持哪些 LLM？

**答**：OpenAI、Anthropic、Google、本地模型（Ollama 等）。在 `openclaw.json` 的 Agent 配置中指定 `model`。

---

## Q19：怎么备份项目状态？

**答**：所有 `shared/` 目录都进 Git。Orchestrator 状态（`project_state.yaml`）也进 Git。

---

## Q20：OpenClaw 怎么更新升级？

**答**：`openclaw update` 命令。升级前**备份 `openclaw.json` 和所有 `SOUL.md`**。

---

## 本章关键 takeaway

- 20 个 FAQ 覆盖**安装、配置、调试、部署、监控、扩展、集成**。
- **核心问题**：Agent 卡住、Token 消耗、并行执行、调试工具、持续学习。
- **核心实践**：SOUL.md + 产物驱动 + 质量门禁 + Trace + 渐进部署。
- **核心工具**：openclaw.json + Lobster DSL + sessions_send + 共享存储。

---

**返回**：[60-faq README](./README.md)
**下一篇**：[05 - 成本与限流 FAQ](./05-成本与限流FAQ.md)
