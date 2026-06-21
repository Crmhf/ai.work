# 02 - 多 Agent 编排 FAQ

> 多 Agent 编排实战中**最常被问到的问题**。

---

## Q1：什么时候该用多 Agent？

**答**：任务能被清晰拆分为多个**有依赖关系的子任务**、每个子任务需要**领域专业化**、可以**并行执行**。

**不应该**：所有任务都能用单体 Agent 完成 → 不要上多 Agent。

详见：[../../00-articles/04-多Agent协作的边界与陷阱.md](../../00-articles/04-多Agent协作的边界与陷阱.md)

---

## Q2：OpenClaw 适合什么场景？

**答**：标准化、有 SOP 的端到端任务。最典型：**软件工程多 Agent 团队**（产品 → UI → 架构 → 编码 → 测试 → 部署）。

**不适合**：强实时性、强合规、强审计、高创造性任务。

详见：[../10-tech-multi-agent/01-多Agent编排总览与定位.md](../10-tech-multi-agent/01-多Agent编排总览与定位.md)

---

## Q3：Orchestrator 怎么调度 Agent？

**答**：5 种模式：串行 / 并行 / 波次 / 条件 / 人工审批。

详见：[../10-tech-multi-agent/05-调度决策引擎实现.md](../10-tech-multi-agent/05-调度决策引擎实现.md)

---

## Q4：Agent 之间怎么通信？

**答**：**产物驱动**（共享存储） > 直接对话。每个 Agent 完成工作把产物写到 `shared/` 目录，下游 Agent 读产物。

详见：[../10-tech-multi-agent/06-Agent间通信协议与产物驱动.md](../10-tech-multi-agent/06-Agent间通信协议与产物驱动.md)

---

## Q5：如何避免 Agent 循环依赖？

**答**：构建 DAG 时**强制做循环检测**（DFS 检环）。OpenClaw 的 `TaskDecomposer` 内置了 `_has_cycle` 方法。

详见：[../10-tech-multi-agent/11-任务分解算法实现.md](../10-tech-multi-agent/11-任务分解算法实现.md)

---

## Q6：质量门禁怎么设计？

**答**：每个阶段都有**可机器验证**的检查项，**不通过则重试**（最多 3 次），**3 次不过升级人工**。

详见：[../10-tech-multi-agent/08-质量门禁与自愈机制.md](../10-tech-multi-agent/08-质量门禁与自愈机制.md)

---

## Q7：HITL 节点设计要注意什么？

**答**：**必须有 SLA + 升级机制**。否则人工忙不过来，系统会卡住。

详见：[../10-tech-multi-agent/07-Lobster-DSL工作流编排.md](../10-tech-multi-agent/07-Lobster-DSL工作流编排.md)

---

## Q8：多 Agent 怎么调试？

**答**：完整的 **Trace + Log + Metric**。每个 Agent 调用都有 trace_id，可以拉出整个调用链路。

详见：[../10-tech-multi-agent/09-可观测性与监控设计.md](../10-tech-multi-agent/09-可观测性与监控设计.md)

---

## Q9：多 Agent 项目的成本怎么控制？

**答**：
- 设置单任务最大 token 预算。
- 设置 max_iterations（避免死循环）。
- 失败时升级人工，而不是无限重试。

详见：[../40-tech-best-practices/05-成本控制实战.md](../40-tech-best-practices/05-成本控制实战.md)

---

## Q10：第一个多 Agent 项目应该多大？

**答**：**最小可用**——3 个 Agent（Orchestrator + 1 个 Specialist + 1 个 Reviewer）。跑通后再扩展。

详见：[../50-case-studies/01-多Agent团队交付电商小程序复盘.md](../50-case-studies/01-多Agent团队交付电商小程序复盘.md)

---

## Q11：OpenClaw 与 LangGraph、AutoGen 的区别？

**答**：
- **OpenClaw**：CLI 工具 + Lobster DSL，**声明式工作流**。
- **LangGraph**：图结构编排，**代码优先**。
- **AutoGen**：对话驱动多 Agent，**灵活但难工程化**。

详见：[../70-resources/02-必用工具清单.md](../70-resources/02-必用工具清单.md)

---

## Q12：如何选择 Agent 的 LLM 模型？

**答**：
- **简单任务**（分类、提取）：GPT-3.5 / Claude Haiku。
- **通用任务**（总结、改写）：GPT-4 Turbo / Claude Sonnet。
- **复杂推理**（架构设计）：GPT-4 / Claude Opus。

详见：[../40-tech-best-practices/05-成本控制实战.md](../40-tech-best-practices/05-成本控制实战.md)

---

## Q13：Agent 失败如何恢复？

**答**：**自愈机制**——错误分类（瞬时 / 逻辑 / 需求 / 系统）→ 对应策略（重试 / 回溯 / 升级）。

详见：[../10-tech-multi-agent/08-质量门禁与自愈机制.md](../10-tech-multi-agent/08-质量门禁与自愈机制.md)

---

## Q14：如何保证 Agent 输出质量？

**答**：
- 明确的契约（JSON Schema）。
- Few-shot 示例。
- 自检（让 LLM 自己检查输出）。
- 质量门禁（自动校验）。

详见：[../40-tech-best-practices/02-Prompt工程最佳实践.md](../40-tech-best-practices/02-Prompt工程最佳实践.md)

---

## Q15：多 Agent 项目需要多长时间跑通 MVP？

**答**：**2-4 周**可以做 demo，**2-3 个月**可以做到生产可用（7×24 稳定）。

详见：[../50-case-studies/03-客服Agent的5个月迭代.md](../50-case-studies/03-客服Agent的5个月迭代.md)

---

## Q16：多 Agent 在生产环境的稳定性如何保证？

**答**：
- 完整的 Trace + Log + Metric。
- 告警 + 自动响应。
- 熔断器 + 降级链。
- 定期复盘 + 持续优化。

详见：[../10-tech-multi-agent/09-可观测性与监控设计.md](../10-tech-multi-agent/09-可观测性与监控设计.md)

---

## Q17：OpenClaw 的部署成本高吗？

**答**：**低**。单机部署即可跑小项目。规模化用 K8s 集群。

详见：[../10-tech-multi-agent/10-部署与渐进式采用路径.md](../10-tech-multi-agent/10-部署与渐进式采用路径.md)

---

## Q18：如何让 Agent 团队持续学习？

**答**：
- 收集失败案例。
- 人工标注。
- 迭代 prompt。
- A/B 测试新旧 prompt。
- 定期更新 SOUL.md。

详见：[../50-case-studies/03-客服Agent的5个月迭代.md](../50-case-studies/03-客服Agent的5个月迭代.md)

---

## Q19：什么时候 Agent 应该"升级"到更复杂的架构？

**答**：
- 单 Agent 上下文超载 → 拆多 Agent。
- 错误率上升 → 加门禁 + 自愈。
- 任务变复杂 → 加 HITL。
- 量变大 → 加可观测性 + 监控。

详见：[../10-tech-multi-agent/10-部署与渐进式采用路径.md](../10-tech-multi-agent/10-部署与渐进式采用路径.md)

---

## Q20：多 Agent 项目的 ROI 怎么算？

**答**：
- **节省时间**：原人工耗时 - 多 Agent 耗时。
- **质量提升**：错误率、文档完整度等。
- **业务收益**：项目延期减少、客户满意度等。
- **总产出 / 总投入** = ROI。

详见：[../50-case-studies/02-飞书会议Agent在创业公司的应用.md](../50-case-studies/02-飞书会议Agent在创业公司的应用.md)

---

## 本章关键 takeaway

- 20 个 FAQ 覆盖**架构、调度、通信、调试、成本、ROI**等核心问题。
- **核心原则**：能用单体就别多 Agent；产物驱动 > 直接对话；HITL 必须有 SLA；持续迭代。
- **实战数据**：MVP 2-4 周、生产可用 2-3 个月。
- 每个问题都链接到深度文档。

---

**返回**：[60-faq README](./README.md)
**下一篇**：[03 - 飞书集成 FAQ](./03-飞书集成FAQ.md)
