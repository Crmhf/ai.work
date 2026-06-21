# 17 - Agent 系统的版本管理与发布

> **TL;DR**：Agent 系统的版本比传统软件**复杂得多**——除了代码，还有 Prompt、模型、SOUL.md、契约。本篇给出 4 类资产的版本管理方法。

---

## 一、Agent 系统的 4 类资产

### 资产 1：代码（Code）

- Agent 工作流代码。
- Orchestrator / Scheduler 实现。
- 工具集成代码。

**版本管理**：Git（标准做法）。

### 资产 2：Prompt

- 系统提示词。
- 用户输入模板。
- Few-shot 示例。

**版本管理**：**Git + Prompt 版本号**。

### 资产 3：模型（Model）

- LLM 模型选择（GPT-4 / Claude / Llama）。
- 同一模型不同版本（GPT-4 → GPT-4 Turbo → GPT-4o）。

**版本管理**：**模型 registry + 版本号**。

### 资产 4：配置（Config）

- SOUL.md（每个 Agent 的心智）。
- 契约定义（schema）。
- 工作流定义（Lobster DSL）。

**版本管理**：**Git + 语义化版本**。

---

## 二、Prompt 版本管理

### 目录结构

```
prompts/
├── README.md
├── product/
│   ├── v1/
│   │   ├── system_prompt.txt
│   │   ├── user_template.txt
│   │   └── eval_results.json
│   ├── v2/
│   │   ├── system_prompt.txt
│   │   ├── user_template.txt
│   │   └── eval_results.json
│   └── CHANGELOG.md
├── ui_designer/
└── ...
```

### CHANGELOG 模板

```markdown
# Product Prompt 变更日志

## v2.1 (2026-06-21)
- 调整：明确"不要做技术选型"
- 评估：pass rate 从 88% 提升到 92%

## v2.0 (2026-06-15)
- 重构：拆分为 system + user template
- 评估：pass rate 从 80% 提升到 88%

## v1.0 (2026-05-01)
- 初始版本
- 评估：pass rate 80%
```

### Prompt 引用

```python
# config/prompts.yaml
prompts:
  product_agent:
    version: "v2.1"
    path: "prompts/product/v2.1"
  
  ui_designer_agent:
    version: "v1.5"
    path: "prompts/ui_designer/v1.5"
```

```python
# 加载 prompt
def load_prompt(agent_name):
    config = yaml.safe_load(open("config/prompts.yaml"))
    version = config["prompts"][agent_name]["version"]
    path = config["prompts"][agent_name]["path"]
    return {
        "system": open(f"{path}/system_prompt.txt").read(),
        "user": open(f"{path}/user_template.txt").read()
    }
```

---

## 三、模型版本管理

### 模型 Registry

```python
# config/models.yaml
models:
  gpt-4-turbo:
    provider: "openai"
    api_version: "2024-04-09"
    max_tokens: 4096
    temperature_default: 0.3
    cost_per_1k_input: 0.01
    cost_per_1k_output: 0.03
  
  claude-3-5-sonnet:
    provider: "anthropic"
    api_version: "2024-06-20"
    max_tokens: 4096
    temperature_default: 0.3
    cost_per_1k_input: 0.003
    cost_per_1k_output: 0.015
```

### 模型版本切换

```python
class ModelRegistry:
    def __init__(self, config_path):
        self.models = yaml.safe_load(open(config_path))
        self.active = self.models["gpt-4-turbo"]  # 当前激活
    
    def switch(self, model_name):
        """切换模型"""
        if model_name not in self.models:
            raise ValueError(f"Unknown model: {model_name}")
        
        old = self.active
        self.active = self.models[model_name]
        
        logger.info(f"Switched from {old} to {self.active}")
    
    def get_for_task(self, task_complexity):
        """根据任务复杂度选择模型"""
        if task_complexity == "high":
            return self.models["gpt-4-turbo"]
        elif task_complexity == "medium":
            return self.models["claude-3-5-sonnet"]
        else:
            return self.models["gpt-4o-mini"]
```

### A/B 测试新旧模型

```python
def ab_test_models(model_a, model_b, test_cases, traffic_split=0.5):
    """A/B 测试两个模型"""
    results = {"a": [], "b": []}
    
    for case in test_cases:
        if hash(case.id) % 100 < traffic_split * 100:
            output = call_model(model_a, case.input)
            results["a"].append({"case": case, "output": output, "score": evaluate(output, case.expected)})
        else:
            output = call_model(model_b, case.input)
            results["b"].append({"case": case, "output": output, "score": evaluate(output, case.expected)})
    
    score_a = sum(r["score"] for r in results["a"]) / len(results["a"])
    score_b = sum(r["score"] for r in results["b"]) / len(results["b"])
    
    return {
        "model_a": {"name": model_a, "score": score_a, "samples": len(results["a"])},
        "model_b": {"name": model_b, "score": score_b, "samples": len(results["b"])},
        "winner": model_a if score_a > score_b else model_b
    }
```

---

## 四、SOUL.md 版本管理

### 目录结构

```
agents/
├── orchestrator/
│   ├── SOUL.md            # 当前生效版本
│   ├── SOUL.v1.md         # 历史版本
│   ├── SOUL.v2.md
│   └── CHANGELOG.md
├── product/
│   └── ...
```

### SOUL.md 版本切换

```python
# config/agents.yaml
agents:
  orchestrator:
    soul_version: "v2.0"
    model: "gpt-4-turbo"
  
  product:
    soul_version: "v1.5"
    model: "gpt-4-turbo"
```

### 回滚

```python
def rollback_soul(agent_name, to_version):
    """回滚 SOUL.md"""
    src = f"agents/{agent_name}/SOUL.{to_version}.md"
    dst = f"agents/{agent_name}/SOUL.md"
    
    if not os.path.exists(src):
        raise FileNotFoundError(f"Version {to_version} not found")
    
    shutil.copy(src, dst)
    logger.info(f"Rolled back {agent_name} to {to_version}")
```

---

## 五、版本号策略

### 语义化版本（SemVer）

```
主版本.次版本.修订号
MAJOR.MINOR.PATCH
```

### 适用场景

- **代码**：标准 SemVer。
- **Prompt**：
  - MAJOR：不兼容变更（如换模型）。
  - MINOR：新增能力。
  - PATCH：bug 修复、措辞优化。
- **模型**：模型 vendor 版本号（如 `gpt-4-0613`）。
- **配置**：SemVer（与 Prompt 类似）。

### 版本变更记录

```yaml
# CHANGELOG.yaml (整体版本)
changelog:
  v2.1.0 (2026-06-21):
    - prompt: product v2.1 (pass rate 92%)
    - prompt: ui_designer v1.5
    - code: refactor scheduler
    - model: upgrade to gpt-4o-2024-08-06
  
  v2.0.0 (2026-06-15):
    BREAKING:
    - prompt: rewrite product (new schema)
    - code: migrate to LangGraph
    FEATURES:
    - add review agent
    - add HITL workflow
```

---

## 六、发布策略

### 阶段 1：开发环境

- 所有改动在 dev 环境跑。
- 自动测试 + 人工评估。

### 阶段 2：灰度（5% 流量）

- 5% 用户用新版本。
- 监控关键指标（成功率、错误率、成本）。

### 阶段 3：扩展（25% 流量）

- 扩展到 25% 用户。
- 继续监控。

### 阶段 4：全量（100%）

- 全量上线。
- 持续监控。

### 紧急回滚

```python
def emergency_rollback(current_version, last_stable_version):
    """紧急回滚"""
    # 1. 切换 prompt
    switch_prompt(last_stable_version)
    
    # 2. 切换模型（如果涉及）
    switch_model(last_stable_version)
    
    # 3. 切换代码（git revert）
    git_revert(current_version)
    
    # 4. 通知团队
    notify_emergency_rollback()
    
    logger.critical(f"Rolled back from {current_version} to {last_stable_version}")
```

---

## 七、CI/CD 集成

### CI Pipeline

```yaml
# .github/workflows/agent-ci.yml
name: Agent CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Run unit tests
        run: pytest tests/unit/
      
      - name: Run prompt evaluation
        run: python -m eval.run_eval --config config/eval.yaml
      
      - name: Check pass rate
        run: |
          if [ $(cat eval_results/pass_rate.txt) -lt 0.9 ]; then
            echo "Pass rate too low!"
            exit 1
          fi
      
      - name: Lint prompt files
        run: python -m tools.lint_prompts prompts/
      
      - name: Check SOUL.md validity
        run: python -m tools.validate_soul agents/
```

### CD Pipeline

```yaml
# .github/workflows/agent-cd.yml
name: Agent CD
on:
  push:
    tags: ['v*']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Deploy to staging
        run: ./scripts/deploy.sh staging
      
      - name: Run smoke tests
        run: python -m eval.smoke_test
      
      - name: Deploy to production (canary 5%)
        run: ./scripts/deploy.sh production --canary 5
      
      - name: Monitor for 1 hour
        run: sleep 3600
      
      - name: Check error rate
        run: python -m tools.check_error_rate --threshold 0.05
      
      - name: Deploy to production (100%)
        if: success()
        run: ./scripts/deploy.sh production --full
```

---

## 八、本章关键 takeaway

- Agent 系统的 **4 类资产**：代码、Prompt、模型、配置（SOUL.md / 契约）。
- 每类资产都需要**版本管理**，但方式不同。
- **Prompt 版本**：Git + 语义化版本 + 评估结果。
- **模型版本**：模型 registry + A/B 测试。
- **配置版本**：Git + 语义化版本。
- **发布策略**：开发 → 灰度（5%）→ 扩展（25%）→ 全量（100%）。
- **CI/CD 自动化**：评估通过才能合并，监控通过才能全量。

---

**返回**：[10-tech-multi-agent README](./README.md)
**上一篇**：[16 - Agent 团队的实战 SOP](./16-Agent团队的实战SOP.md)
