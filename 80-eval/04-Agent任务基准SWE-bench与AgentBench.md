# 04 - Agent 任务基准 SWE-bench 与 AgentBench

> **TL;DR**：SWE-bench 和 AgentBench 是 Agent 能力的两个**最权威基准**。本篇详解它们的设计、用法、局限性，以及如何借鉴做自己的评测。

---

## 一、SWE-bench 详解

### 背景

SWE-bench（Software Engineering Benchmark）由 Princeton 和 Stanford 在 2023 年提出，是**评估 LLM / Agent 软件工程能力**的事实标准。

### 设计

#### 任务来源

- **1,000+** 真实 GitHub issue。
- 来自 12 个 Python 仓库（Django、Scikit-learn、Sphinx 等）。
- 每个 issue：
  - 有完整的 issue 描述。
  - 有对应的代码修改（PR）。
  - 有"通过标准"（test cases）。

#### 任务流程

```
输入：
  - GitHub issue 文本
  - 仓库当前状态（baseline 代码）

输出：
  - 修改后的代码（patch）

评估：
  - 应用 patch 到仓库。
  - 跑 issue 对应的 test cases。
  - 全部通过 → issue 被解决。
```

#### 评估指标

- **Pass rate**：issue 被解决的比例。
- **Resolve rate**：类似。

### 代表结果

| Agent / Model | SWE-bench Pass Rate | SWE-bench Verified |
|---|---|---|
| GPT-4（基础） | 1.7% | - |
| Claude 3 Opus | 4.2% | - |
| AutoCodeRover | 7.5% | - |
| SWE-Agent（GPT-4） | 12.5% | - |
| **Devin**（2024.3） | **13.86%** | - |
| SWE-Agent + Claude 3.5 Sonnet | - | 33.6% |
| Factory Code Droid | - | 37.0% |
| **Claude 3.5 Sonnet + Agent** | - | **49.0%** |
| Aider + GPT-4o | - | 38.0% |

### SWE-bench Verified

- **500 个 issue** 的"人类验证"子集。
- 由人类验证每个 issue 的可解性。
- **去除了有问题的 issue**（依赖问题、test 不完整等）。

### 局限

1. **只覆盖 Python**——不能反映多语言能力。
2. **依赖特定仓库**——不能反映新项目。
3. **issue 偏简单**——很多是"小修小补"。
4. **无法评估代码质量**——只评估 test pass。
5. **无法评估工程实践**——架构、性能、可维护性。

### 链接

- 论文：https://arxiv.org/abs/2310.06770
- 排行榜：https://www.swebench.com
- GitHub：https://github.com/princeton-nlp/SWE-bench

---

## 二、AgentBench 详解

### 背景

AgentBench 由清华大学 THUDM 团队 2023 年提出，是**评估 LLM 作为 Agent 综合能力**的基准。

### 设计

#### 8 类任务

| 任务类别 | 描述 |
|---|---|
| **OS** | 操作系统任务（文件、进程管理） |
| **DB** | 数据库操作（SQL、CRUD） |
| **Knowledge Graph** | 知识图谱查询 |
| **Digital Card Game** | 数字卡牌游戏（博弈） |
| **LTP** | 长时间跨度任务 |
| **Householding (ALFWorld)** | 家庭场景（模拟家务） |
| **Web Shopping** | 网络购物 |
| **Web Browsing (Mind2Web)** | 网页浏览 |

#### 评估维度

- **能力**：完成率。
- **效率**：完成任务的时间 / 步数。
- **鲁棒性**：失败时的恢复能力。
- **泛化性**：在新任务上的表现。

### 代表结果

| Model | 平均完成率 |
|---|---|
| GPT-4 | 56.1% |
| Claude 3 Opus | 52.8% |
| GPT-3.5 | 35.8% |
| Llama 2 70B | 25.6% |
| Vicuna 13B | 18.5% |

### 优势

- **任务多样**：8 类任务覆盖 Agent 能力全谱。
- **环境真实**：真实操作系统、真实数据库、真实 Web。
- **可扩展**：可加入自定义任务。

### 局限

1. **任务偏研究**——很多任务不直接对应业务。
2. **环境搭建复杂**——OS、Web 模拟器配置麻烦。
3. **评估成本高**——每个任务要完整跑一次。
4. **更新慢**——发布后难以更新。

### 链接

- 论文：https://arxiv.org/abs/2308.03688
- GitHub：https://github.com/THUDM/AgentBench

---

## 三、其他 Agent 基准

### WebArena

- **目标**：真实 Web 任务。
- **任务**：在真实网站上完成用户指令（购物、订餐等）。
- **特点**：使用真实网站（Amazon、Reddit 等）。
- **挑战**：环境搭建复杂。

### ToolBench

- **目标**：评估 LLM 使用工具的能力。
- **任务**：真实 API 调用（数千个 API）。
- **代表论文**：ToolLLM。

### HumanEval / MBPP

- **目标**：基础 Python 编程能力。
- **不足**：题目偏简单，不能反映真实项目。

### MATH / GSM8K

- **目标**：数学推理能力。
- **特点**：评估推理质量，不评估 Agent 能力。

### MMLU / C-Eval

- **目标**：多领域知识。
- **不足**：评估"知识"不评估"做"。

---

## 四、借鉴 SWE-bench / AgentBench 设计自己的评测

### 借鉴 1：真实任务

- **SWE-bench**：从真实 GitHub issue 提取。
- **借鉴**：从真实业务场景提取任务（脱敏后）。

### 借鉴 2：可执行的评估

- **SWE-bench**：跑真实 test cases。
- **借鉴**：每个任务都有可执行的"通过标准"。

### 借鉴 3：端到端

- **AgentBench**：完整环境 + 完整任务。
- **借鉴**：端到端测试，不只测单步。

### 借鉴 4：公开排行榜

- **SWE-bench / AgentBench**：都有公开 leaderboard。
- **借鉴**：内部排行榜，推动团队改进。

---

## 五、我的实战建议

### 评测策略

```
MVP 阶段：
  - 用通用基准（SWE-bench Verified）+ 自己的小测试集（50 条）
  - 重点：快速迭代

生产阶段：
  - 自己搭完整测试集（500+ 条）
  - 通用基准作为辅助参考
  - 重点：业务场景覆盖

持续改进：
  - 每周跑一次完整评测
  - 每月 review 评测集（更新、新增、删除）
  - 重点：长期趋势
```

### 评测集设计原则

1. **真实场景**：用真实用户问题。
2. **覆盖典型 + 边缘**：80/20 原则。
3. **可执行**：每个任务都有明确的"通过标准"。
4. **可维护**：定期更新、清理过期 case。
5. **不可泄漏**：评测集不能进入训练数据。

### 评测流程

```
改 prompt / 模型 / 配置
    ↓
跑 holdout 测试集
    ↓
跑完整测试集
    ↓
对比历史结果
    ↓
回归测试
    ↓
灰度上线
    ↓
全量上线
```

---

## 六、注意事项

### 1. 不要迷信 benchmark

- SWE-bench 49% 不代表你的 Agent 能解决 49% 的真实 bug。
- **benchmark 是参考，不是答案**。

### 2. 业务测试集 > 通用 benchmark

- 你的 Agent 是为了**业务**，不是**benchmark**。
- **业务测试集永远是核心**。

### 3. 评测集需要持续更新

- 模型升级、用户场景变化 → 评测集要更新。
- 否则评测结果会"过时"。

### 4. 不要"刷分"

- 为 benchmark 优化 prompt，实际效果可能下降。
- **benchmark 优化 ≠ 业务优化**。

### 5. 多维度评估

- 不只 pass rate，还要看成本、延迟、用户体验。
- **综合指标**才是真实指标。

---

## 七、本章关键 takeaway

- **SWE-bench** 是软件工程 Agent 能力的事实标准（Claude 3.5 Sonnet 49% Verified）。
- **AgentBench** 是 Agent 综合能力的权威基准（GPT-4 平均 56%）。
- 两个基准都有局限，**不能直接反映业务能力**。
- **借鉴设计原则**：真实任务、可执行评估、端到端、公开排行榜。
- **业务测试集 > 通用 benchmark**，自己搭评测体系是关键。

---

**返回**：[80-eval README](./README.md)
