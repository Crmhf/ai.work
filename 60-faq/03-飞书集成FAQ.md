# 03 - 飞书集成 FAQ

> 飞书 API / 数据融合实战中**最常被问到的问题**。

---

## Q1：飞书自建应用 vs 商店应用？

**答**：**选自建应用**。妙记 AI 产物 API 只对自建应用开放。

详见：[../20-tech-feishu-data-fusion/08-权限配置与发布上线.md](../20-tech-feishu-data-fusion/08-权限配置与发布上线.md)

---

## Q2：妙记 AI 产物 API 怎么调？

**答**：`GET /minutes/v1/minutes/:minute_token/artifacts`，权限 `minutes:minutes:readonly`。

详见：[../20-tech-feishu-data-fusion/03-妙记AI产物提取实战.md](../20-tech-feishu-data-fusion/03-妙记AI产物提取实战.md)

---

## Q3：妙记 AI 处理需要多久？

**答**：**5-30 分钟**。需要**轮询等待**，不能立即取。

详见：[../20-tech-feishu-data-fusion/03-妙记AI产物提取实战.md](../20-tech-feishu-data-fusion/03-妙记AI产物提取实战.md)

---

## Q4：会议结束后事件怎么获取？

**答**：订阅 `vc.meeting.meeting_ended_v1` 事件（仅 Open API 预约的会议）。

详见：[../20-tech-feishu-data-fusion/07-事件驱动与Automation桥接.md](../20-tech-feishu-data-fusion/07-事件驱动与Automation桥接.md)

---

## Q5：多维表格变更事件怎么订阅？

**答**：**无法直接订阅给外部 Bot**。必须用 Automation Center 配置 Webhook 桥接。

详见：[../20-tech-feishu-data-fusion/07-事件驱动与Automation桥接.md](../20-tech-feishu-data-fusion/07-事件驱动与Automation桥接.md)

---

## Q6：飞书任务 origin 字段有什么用？

**答**：**溯源**——记录任务来自哪个会议、哪个妙记、哪条多维表格记录。**没有 origin，闭环就是黑盒**。

详见：[../20-tech-feishu-data-fusion/04-任务管理与origin溯源.md](../20-tech-feishu-data-fusion/04-任务管理与origin溯源.md)

---

## Q7：tenant_access_token 怎么管理？

**答**：有效期 **2 小时**，必须**缓存并自动刷新**。多实例部署用 Redis 集中管理。

详见：[../20-tech-feishu-data-fusion/09-端到端Python实现.md](../20-tech-feishu-data-fusion/09-端到端Python实现.md)

---

## Q8：Webhook 签名校验怎么做？

**答**：用 HMAC-SHA256 + Base64 + 常数时间比较 + 时间戳校验。飞书有 v1 和 v2 两个版本，推荐 v2。

详见：[../20-tech-feishu-data-fusion/11-签名校验与安全实践.md](../20-tech-feishu-data-fusion/11-签名校验与安全实践.md)

---

## Q9：飞书 API 频率限制是多少？

**答**：
- 妙记 AI 产物：**5 QPS**。
- 任务 API：**10 QPS**。
- 多维表格 API：**10 QPS**。

详见：[../20-tech-feishu-data-fusion/10-部署运维与成本控制.md](../20-tech-feishu-data-fusion/10-部署运维与成本控制.md)

---

## Q10：高级权限审核要多久？

**答**：**3-5 个工作日**。提前 2 周申请，不要等到上线前。

详见：[../20-tech-feishu-data-fusion/08-权限配置与发布上线.md](../20-tech-feishu-data-fusion/08-权限配置与发布上线.md)

---

## Q11：同一多维表格能并发写吗？

**答**：**不能**。飞书官方说明：同一数据表不支持并发调用写接口。必须用队列串行化。

详见：[../20-tech-feishu-data-fusion/05-多维表格数据模型设计.md](../20-tech-feishu-data-fusion/05-多维表格数据模型设计.md)

---

## Q12：飞书多维表格批量操作的上限？

**答**：
- `batch_create`：500 条/次。
- `batch_update`：1000 条/次（建议 350 条/批）。
- `batch_delete`：500 条/次。

详见：[../20-tech-feishu-data-fusion/05-多维表格数据模型设计.md](../20-tech-feishu-data-fusion/05-多维表格数据模型设计.md)

---

## Q13：飞书有没有 MCP Server？

**答**：**官方 lark-cli 提供了 22+ Skills**（lark-minutes、lark-task、lark-base 等），可直接当作 MCP 工具用。

详见：[../20-tech-feishu-data-fusion/12-飞书生态中的Agent集成.md](../20-tech-feishu-data-fusion/12-飞书生态中的Agent集成.md)

---

## Q14：飞书任务系统能不能替代 Jira？

**答**：**看团队规模**。10-20 人小团队够用；50+ 人建议用专门的 PM 工具（飞书项目 / Jira）。

---

## Q15：飞书妙记 AI 提取准确率怎么样？

**答**：**95%+**。但**依赖会议质量**——多人抢话、议程不明确会降低准确率。

详见：[../50-case-studies/02-飞书会议Agent在创业公司的应用.md](../50-case-studies/02-飞书会议Agent在创业公司的应用.md)

---

## Q16：Automation 规则被误关怎么办？

**答**：
- Automation 启用管理员权限。
- 监控 Webhook 接收量。
- 异常时告警 + 人工介入。

详见：[../20-tech-feishu-data-fusion/07-事件驱动与Automation桥接.md](../20-tech-feishu-data-fusion/07-事件驱动与Automation桥接.md)

---

## Q17：飞书数据如何备份？

**答**：飞书多维表格不支持自动备份。**建议定期用 API 导出到自有数据库 / S3**。

---

## Q18：怎么监控飞书 API 调用成本？

**答**：监控调用次数 + 单价 + 异常率。**设置月度预算 + 告警**。

详见：[../20-tech-feishu-data-fusion/10-部署运维与成本控制.md](../20-tech-feishu-data-fusion/10-部署运维与成本控制.md)

---

## Q19：飞书生态中如何集成 OpenClaw？

**答**：把飞书数据融合封装为 MCP Server，OpenClaw 的 Specialist Agent 调用 MCP 工具。

详见：[../20-tech-feishu-data-fusion/12-飞书生态中的Agent集成.md](../20-tech-feishu-data-fusion/12-飞书生态中的Agent集成.md)

---

## Q20：飞书 Agent 项目最适合什么场景？

**答**：
- 会议密集型团队（销售、运营、PM）。
- 跨部门协作多的组织。
- 需要"会议→任务→执行→反馈"闭环的业务。

详见：[../50-case-studies/02-飞书会议Agent在创业公司的应用.md](../50-case-studies/02-飞书会议Agent在创业公司的应用.md)

---

## 本章关键 takeaway

- 20 个 FAQ 覆盖**权限、API、事件、签名、限流、MCP** 等核心问题。
- **关键限制**：妙记 AI 仅自建应用、多维表格事件不开放给外部 Bot、同表写串行化。
- **关键优势**：妙记 AI 准确率 95%+、API 完整、CLI 工具丰富。
- **关键实践**：权限提前申请、三层混合架构、origin 字段、签名校验。

---

**返回**：[60-faq README](./README.md)
**下一篇**：[04 - OpenClaw 实战 FAQ](./04-OpenClaw实战FAQ.md)
