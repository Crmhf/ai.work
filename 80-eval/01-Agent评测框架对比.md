# 01 - Agent 评测框架对比

> **TL;DR**：本篇对比 10 个主流 Agent 评测框架，从 **评测目标、任务类型、评估指标、适用场景** 4 个维度分析。

---

## 一、为什么需要评测框架

Agent 系统的输出**不确定、难量化**。需要一个**标准化、可重复**的评估方式：

- **模型选型**：哪个 LLM 适合做 Agent？
- **框架选型**：OpenClaw、LangGraph、AutoGen 哪个更好？
- **版本迭代**：升级 prompt 后效果是否变好？
- **产品对比**：自研 Agent vs 商业 Agent。

---

## 二、评测框架的 4 个维度

| 维度 | 说明 |
|---|---|
| **评测目标** | 模型能力 / Agent 能力 / 产品体验 |
| **任务类型** | 软件工程 / 研究 / 客服 / 通用 |
| **评估指标** | Pass rate / User satisfaction / Cost |
| **适用场景** | 学术研究 / 工程选型 / 产品迭代 |

---

## 三、10 个主流评测框架对比

### 1. SWE-bench

- **全称**：Software Engineering Benchmark。
- **目标**：评估 LLM 解决真实 GitHub issue 的能力。
- **任务**：1,000+ 真实 Python issue。
- **指标**：Pass rate（issue 被解决的比例）。
- **代表 Agent**：Devin（解决率 13.86%）。
- **链接**：https://www.swebench.com

**优势**：任务真实、贴近工程。
**劣势**：只覆盖 Python、固定仓库。

### 2. SWE-bench Verified

- **目标**：SWE-bench 的"人类验证"子集（500 个 issue）。
- **改进**：由人类验证每个 issue 的可解性。
- **代表 Agent**：Claude 3.5 Sonnet + Agent（解决率 49%）。

### 3. AgentBench

- **全称**：Agent Benchmark。
- **目标**：评估 LLM 作为 Agent 的综合能力。
- **任务类型**：8 类（OS、DB、Knowledge Graph、Digital Card Game 等）。
- **指标**：每个任务的 pass rate。
- **代表论文**：AgentBench: Evaluating LLMs as Agents。
- **链接**：https://github.com/THUDM/AgentBench

**优势**：覆盖广（多领域）。
**劣势**：任务偏研究、不直接对应业务场景。

### 4. WebArena

- **目标**：评估 Agent 在真实 Web 环境中的能力。
- **任务**：在真实网站上完成用户指令（购物、订餐等）。
- **指标**：任务完成率。
- **链接**：https://webarena.dev

**优势**：任务真实（真实网站）。
**劣势**：环境搭建复杂。

### 5. HumanEval

- **目标**：评估 LLM 生成代码的能力（基础）。
- **任务**：164 个 Python 编程问题。
- **指标**：Pass@k。
- **链接**：https://github.com/openai/human-eval

**优势**：简单、广泛使用。
**劣势**：题目偏简单、不反映真实项目。

### 6. MBPP (Mostly Basic Python Problems)

- **目标**：基础 Python 编程问题。
- **任务**：1,000 个简单编程题。
- **指标**：Pass rate。
- **链接**：https://github.com/google-research/mbpp

### 7. MMLU (Massive Multitask Language Understanding)

- **目标**：评估 LLM 的多领域知识。
- **任务**：57 个学科的多选题。
- **指标**：准确率。
- **代表分数**：GPT-4 86.4%、Claude 3 Opus 86.8%。

**优势**：评估通用知识。
**劣势**：不能直接反映 Agent 能力。

### 8. GSM8K

- **目标**：小学数学应用题。
- **任务**：8,500 个问题。
- **指标**：准确率。
- **代表分数**：GPT-4 92%、Claude 3 Opus 95%。

### 9. HotpotQA

- **目标**：多跳推理（需要多个文档联合推理）。
- **任务**：问答。
- **指标**：Exact Match、F1。

### 10. ToolBench

- **目标**：评估 LLM 使用工具的能力。
- **任务**：真实 API 调用。
- **指标**：成功率。
- **代表论文**：ToolLLM。

---

## 四、按场景选择评测框架

### 场景 1：选 LLM 做 Agent

- **推荐**：SWE-bench Verified + AgentBench。
- **理由**：直接评估 Agent 在真实任务中的能力。

### 场景 2：选 Agent 框架

- **推荐**：自己搭评测集（业务场景）+ AgentBench（通用能力）。
- **理由**：框架的"通用能力"差异小，"业务适配"差异大。

### 场景 3：迭代 prompt / Agent 配置

- **推荐**：业务场景测试集 + A/B 测试。
- **理由**：必须用真实数据迭代。

### 场景 4：评估产品体验

- **推荐**：用户调研 + A/B 测试 + NPS。
- **理由**：技术指标不能反映产品价值。

---

## 五、自己搭评测体系的 9 步法

### 步骤 1：定义评测目标

- 我要评估什么？（模型？框架？产品？）
- 用什么指标？（准确率？用户满意度？成本？）

### 步骤 2：收集评测数据

- 真实用户问题（脱敏）。
- 人工标注的"理想回答"。
- 覆盖典型场景和边界场景。

### 步骤 3：设计评测任务

- 把每个评测问题变成 Agent 可执行的任务。
- 明确输入、期望输出、评估标准。

### 步骤 4：实现评测脚本

```python
def evaluate(agent, test_cases):
    results = []
    for case in test_cases:
        output = agent.run(case.input)
        score = score_output(output, case.expected)
        results.append({
            "case_id": case.id,
            "input": case.input,
            "output": output,
            "expected": case.expected,
            "score": score
        })
    return aggregate(results)
```

### 步骤 5：跑评测

- 在稳定的硬件 / 模型配置下跑。
- 控制变量（每次只改一个）。

### 步骤 6：分析结果

- 总体 pass rate。
- 失败的 case 分类。
- 失败模式分析。

### 步骤 7：迭代

- 基于失败 case 改 prompt / 模型 / 配置。
- 重新跑评测，对比。

### 步骤 8：回归测试

- 改进后必须保证**没有回归**。
- 跑完整测试集。

### 步骤 9：定期评测

- 每周 / 每月跑一次完整评测。
- 监控长期趋势。

---

## 六、评测常见陷阱

### 陷阱 1：评测集太简单

只测"理想场景"，真实场景多样性远超评测集。

**解决**：用真实用户数据（脱敏）+ 边界 case。

### 陷阱 2：评测集太学术

用 MMLU、GSM8K 等学术基准，但**真实业务场景和学术基准差距巨大**。

**解决**：业务场景测试集为主，学术基准为辅。

### 陷阱 3：单一指标

只看 pass rate，不看用户体验。

**解决**：自动指标 + 人工评估 + 用户反馈。

### 陷阱 4：忽略成本

只看效果，不看成本。Agent 用 GPT-4 跑通了，成本可能是 GPT-3.5 的 30 倍。

**解决**：**成本 / 效果**综合评估。

### 陷阱 5：评测集污染

评测集被模型见过（训练数据泄漏），分数虚高。

**解决**：定期更新评测集，用模型没见过的数据。

---

## 七、本章关键 takeaway

- 10 个评测框架覆盖**代码、数学、推理、Agent、工具使用**5 类任务。
- **SWE-bench + AgentBench** 是 Agent 能力评估的标配。
- **自己搭评测体系**是工程化迭代的关键（9 步法）。
- **避免 5 个常见陷阱**：评测集太简单 / 太学术 / 单一指标 / 忽略成本 / 评测集污染。
- **自动 + 人工 + 业务**三维评估才是完整的评估。

---

**返回**：[80-eval README](./README.md)
**下一篇**：[02 - LLM 模型对比](./02-LLM模型对比.md)
