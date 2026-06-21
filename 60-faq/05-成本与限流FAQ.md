# 05 - 成本与限流 FAQ

> LLM 成本控制和 API 限流应对的实战问题。

---

## Q1：LLM 成本为什么失控？

**答**：常见原因：
- 没有 max_iterations → Agent 进入死循环。
- 没有缓存 → 重复 query 反复调用。
- 用贵模型 → 所有任务都用 GPT-4。
- Prompt 过长 → 输入 token 多。

详见：[../40-tech-best-practices/05-成本控制实战.md](../40-tech-best-practices/05-成本控制实战.md)

---

## Q2：怎么估算 LLM 成本？

**答**：

```
monthly_cost = daily_tasks * cost_per_task * 30
cost_per_task = sum(llm_call_cost + tool_call_cost)
llm_call_cost = (input_tokens / 1000) * input_price + (output_tokens / 1000) * output_price
```

详见：[../40-tech-best-practices/05-成本控制实战.md](../40-tech-best-practices/05-成本控制实战.md)

---

## Q3：缓存命中率怎么提升？

**答**：
- 标准化 query（FAQ 类）。
- 用语义相似度检索（不要求完全相同）。
- 设置合理 TTL（不能太长也不能太短）。
- 监控命中率，调整策略。

详见：[../40-tech-best-practices/05-成本控制实战.md](../40-tech-best-practices/05-成本控制实战.md)

---

## Q4：怎么减少输入 token？

**答**：
- Prompt 精简（删除冗余说明）。
- 用 Few-shot 而非长篇说明。
- 用 RAG 检索关键信息而非塞全文档。
- 压缩对话历史（只保留关键信息）。

---

## Q5：怎么减少输出 token？

**答**：
- 设置 max_tokens 上限。
- 限制输出格式（JSON / 列表 / 简短回答）。
- 用流式响应 + 用户提前终止。
- Few-shot 给"简短"的示例。

---

## Q6：Agent 死循环怎么防止？

**答**：
- 设置 max_iterations（默认 10）。
- 设置 max_cost 单任务上限。
- 设置 timeout（默认 30 分钟）。
- 检测相似输出（如果输出相似度 > 90% 多次，中断）。

---

## Q7：API 限流怎么办？

**答**：
- 用 Semaphore 控制并发。
- 实现指数退避重试。
- 实现降级链（主模型 → 备用模型）。
- 监控 QPS，避免触发限流。

详见：[../20-tech-feishu-data-fusion/10-部署运维与成本控制.md](../20-tech-feishu-data-fusion/10-部署运维与成本控制.md)

---

## Q8：飞书 API 限流具体数字？

**答**：
- 妙记 AI 产物：**5 QPS**。
- 任务 API：**10 QPS**。
- 多维表格 API：**10 QPS**。

详见：[../20-tech-feishu-data-fusion/10-部署运维与成本控制.md](../20-tech-feishu-data-fusion/10-部署运维与成本控制.md)

---

## Q9：怎么选 LLM 模型？

**答**：
- **简单任务**（分类、提取）：GPT-3.5 / Claude Haiku。
- **通用任务**（总结、改写）：GPT-4 Turbo / Claude Sonnet。
- **复杂推理**（架构设计）：GPT-4 / Claude Opus。
- **代码生成**：GPT-4 / Claude Sonnet。

---

## Q10：本地模型 vs API？

**答**：
- **本地模型**：量大且稳定 + 数据敏感 + 延迟要求高。
- **API**：起步阶段 + 量小 + 灵活性优先。

详见：[../30-roadmap-to-agi/03-未来3年关键技术演进.md](../30-roadmap-to-agi/03-未来3年关键技术演进.md)

---

## Q11：怎么监控成本？

**答**：
- Prometheus + Grafana 仪表盘。
- 按 Agent / 模型 / 用户 维度统计。
- 设置月度预算 + 告警。
- 每日审查成本趋势。

---

## Q12：怎么设置成本告警？

**答**：

```yaml
alerts:
  - name: "DailyCostHigh"
    condition: "increase(cost_usd_total[1d]) > 50"
    severity: "warning"
  
  - name: "DailyCostCritical"
    condition: "increase(cost_usd_total[1d]) > 100"
    severity: "critical"
```

---

## Q13：怎么优化多 Agent 的成本？

**答**：
- 简单任务用便宜模型。
- 复杂任务用贵模型（但任务量小）。
- 缓存重复 query。
- 严格的质量门禁（避免重试浪费）。
- HITL 优先（避免 AI 反复尝试）。

详见：[../10-tech-multi-agent/10-部署与渐进式采用路径.md](../10-tech-multi-agent/10-部署与渐进式采用路径.md)

---

## Q14：怎么判断任务"值得"用 LLM？

**答**：**成本 / 价值比 < 1**。如果 LLM 处理任务花 $0.01，节省人工 5 分钟（$0.50），成本价值比 1:50，**值得**。

如果成本 $1，节省 5 分钟（$0.50），成本价值比 2:1，**不值得**。

---

## Q15：怎么用 Fine-tuning 降本？

**答**：Fine-tuning 适合**大量重复任务**。比如每天 10,000 次同样的分类，Fine-tune 一个便宜模型（如 Llama 3 8B）替代 GPT-4，可能**节省 80% 成本**。

但 Fine-tuning 本身有成本（标注数据、训练、部署），要**先验证 ROI**。

---

## Q16：怎么用 RAG 降本？

**答**：RAG **减少幻觉**（避免错误的人工修复），**不一定降低 token 成本**（反而可能增加）。

但 RAG 可以**让便宜模型达到贵模型的效果**（节省模型成本）。

---

## Q17：多 Agent 的总成本比单体高多少？

**答**：通常高 30-50%，因为：
- 通信开销（任务分解、调度、消息传递）。
- 多次 LLM 调用（每个 Agent 一次）。
- 协调成本。

但多 Agent 通常**质量更高、错误率更低**，长期可能**总体成本更低**（少返工）。

---

## Q18：怎么给客户定价？

**答**：
- **按调用量**：简单但便宜了 AI。
- **按任务**：适合标准化任务。
- **按业务结果**：最适合 Agent 价值，但度量难。
- **混合**：基础费 + 用量费。

详见：[../../00-articles/03-Agent产品的商业化路径.md](../../00-articles/03-Agent产品的商业化路径.md)

---

## Q19：怎么免费试用？

**答**：
- 限制每日调用次数。
- 用便宜模型（GPT-3.5）。
- 限制 prompt 长度。
- 限制上下文窗口。

---

## Q20：开源 LLM 能省钱吗？

**答**：**看场景**。
- **省钱**：量大 + 私有化部署 + 性能要求不极致。
- **不省钱**：量小 + 性能要求高（GPT-4 仍是 SOTA）+ 维护成本高。

Llama 3 70B 在某些任务上接近 GPT-4，但**维护成本高**（GPU、运维、监控）。

---

## 本章关键 takeaway

- 20 个 FAQ 覆盖**成本估算、缓存、降本、限流应对、模型选择**。
- **核心降本三招**：分级模型、缓存（命中率 60%+）、精简 prompt。
- **核心限流应对**：Semaphore、指数退避、降级链、监控 QPS。
- **核心决策**：本地 vs API、Fine-tuning、RAG、定价模型。
- **核心心法**：成本 / 价值比 < 1 才值得用 LLM。

---

**返回**：[60-faq README](./README.md)
