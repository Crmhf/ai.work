# ai.work — 我的 AI Agent 日常思考与工程实践

> 一个长期沉淀的"AI Agent 思考 + 工程实战"知识库。
> 既记录**业务与认知**层面的思考（Agent 改变了什么、商业化路径、AGI 判断），也记录**技术工程**层面的实战方案（多 Agent 编排、飞书数据融合）。
> 持续更新，持续 push 到 GitHub。

---

## 这个仓库是什么

我在日常工作中大量使用 AI Agent —— 自己写 Agent、用 Agent、观察 Agent 产品迭代、做技术决策。
这个仓库是我把这些"碎片思考"系统化沉淀的地方：

- **业务向**：我对 Agent 行业、产品形态、人机协作模式的判断与随想。
- **技术向**：能直接落地的工程方案。两个重头戏：
  1. **多 Agent 工程编排** —— 基于 OpenClaw 的"指挥官 + 专业团队"模式。
  2. **飞书数据融合开发** —— 会议纪要 → 任务 → 多维表格的完整闭环。
- **趋势向**：从 AI Agent 到多 Agent 再到 AGI 的演化路线图。

所有内容都以**独立 markdown 文件**形式分门别类存放，方便单篇引用、单篇分享、单篇迭代。

---

## 仓库结构

```
ai.work/
├── README.md                            # 你正在看的总入口
├── 00-articles/                         # 📝 日常思考（业务向）
│   ├── README.md
│   └── 01~12-*.md                       # 共 12 篇
├── 10-tech-multi-agent/                 # 🤖 技术方向 1：多 Agent 工程编排
│   ├── README.md
│   └── 01~13-*.md                       # 共 13 篇（基于 OpenClaw）
├── 20-tech-feishu-data-fusion/          # 📨 技术方向 2：飞书数据融合
│   ├── README.md
│   └── 01~12-*.md                       # 共 12 篇
├── 30-roadmap-to-agi/                   # 🛤️ 趋势路线图：AI Agent → AGI
│   ├── README.md
│   └── 01~07-*.md                       # 共 7 篇
├── 99-changelog/                        # 📜 整理日志（每次 push 的快照）
│   └── 2026-06-21-initial-44-articles.md
└── assets/                              # 公共资源（图片、配置、模板）
```

**文件总数**：44 篇 markdown，覆盖 4 大模块。

---

## 阅读路径建议

> 如果你是第一次来，建议按以下顺序读：

### 🟢 入门（业务向，1 小时可读完）
1. `00-articles/01-AI-Agent到底改变了什么.md` —— 起点
2. `00-articles/02-我对Agent的认知演化.md`
3. `00-articles/05-AI-native应用的设计哲学.md`
4. `00-articles/08-从Copilot到Autopilot的思考.md`

### 🟡 进阶（技术向，2-3 小时）
1. `10-tech-multi-agent/README.md` —— 多 Agent 编排总览
2. `20-tech-feishu-data-fusion/README.md` —— 飞书数据融合总览
3. 选一个深入：`10-tech-multi-agent/02-Orchestrator设计模式详解.md`
4. 选一个深入：`20-tech-feishu-data-fusion/03-妙记AI产物提取实战.md`

### 🔴 趋势（路线图，30 分钟）
1. `30-roadmap-to-agi/01-AI-Agent发展阶段论.md`
2. `30-roadmap-to-agi/02-多Agent系统演化路径.md`
3. `30-roadmap-to-agi/05-AGI落地场景推演.md`

---

## 核心观点（TL;DR）

### 关于 AI Agent
- **Agent 不是新概念，是新范式**：LLM 让"自主决策"首次可大规模复用。
- **Copilot → Autopilot → Agent** 是三级跳，每级都需要产品形态重构，不是渐进。
- **多 Agent 协作的真正难点是"契约设计"**，不是模型能力。

### 关于多 Agent 工程
- **Orchestrator + Specialist Fleet** 是当前最成熟的工程范式。
- **任务依赖图（DAG）+ 产物驱动通信** 是工程骨架。
- **质量门禁 + 自愈机制** 决定系统能不能跑 7×24。

### 关于飞书数据融合
- **业务闭环 = 数据 → 会议 → 任务 → 数据**，四环缺一环就是信息孤岛。
- **API + Automation + 定时补偿** 三层混合架构是飞书场景的最优解。
- **妙记 AI 产物提取准确率 95%+**，是会议场景天然的结构化入口。

### 关于 AGI 路径
- **从工具到同事到自主体**，是 AGI 落地的三阶段。
- **未来 3 年的关键技术演进**在 Memory、Tool-use、Multi-Agent、Self-improvement 四个维度。
- **AGI 不会一蹴而就**，但每个垂直场景都会被"局部 AGI"逐步重写。

---

## 贡献与维护

- 这是个人思考沉淀仓库，**不接受外部 PR**。
- 文件命名规范：`NN-主题短语.md`（NN 为两位数字）。
- 每个文件独立成文，自包含，单文件可分享。
- 每次重大更新会在 `99-changelog/` 留一份快照。

---

## License

MIT — 欢迎个人学习参考，转载请保留出处。
