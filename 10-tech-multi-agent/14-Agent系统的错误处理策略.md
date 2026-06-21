# 14 - Agent 系统的错误处理策略

> **TL;DR**：Agent 系统的错误比传统软件**更难诊断、更难恢复**。本篇给出 **5 类错误 + 5 大处理策略 + 实战模式**。

---

## 一、5 类错误

### 错误 1：LLM 调用错误

**表现**：
- API rate limit（429）。
- Timeout。
- API key 错误。
- 模型不可用。

**根因**：
- 服务端问题。
- 网络问题。
- 配额问题。

### 错误 2：LLM 输出错误

**表现**：
- 格式错误（不符合 schema）。
- 幻觉（编造信息）。
- 答非所问。
- 内容违规。

**根因**：
- 模型能力边界。
- Prompt 不清晰。
- 上下文超载。

### 错误 3：工具调用错误

**表现**：
- 工具不存在。
- 参数错误。
- 工具执行失败。
- 工具返回格式错误。

**根因**：
- 工具描述不清晰。
- 参数校验失败。
- 工具服务故障。

### 错误 4：Agent 编排错误

**表现**：
- 任务分解错误（任务粒度不对）。
- 依赖循环。
- 调度错误（任务分配错 Agent）。
- 状态丢失。

**根因**：
- Orchestrator 设计问题。
- 模板质量问题。
- 并发控制问题。

### 错误 5：系统级错误

**表现**：
- 内存溢出。
- 磁盘满。
- 网络断开。
- 服务崩溃。

**根因**：
- 资源管理问题。
- 部署问题。
- 第三方依赖。

---

## 二、5 大处理策略

### 策略 1：重试（Retry）

**适用**：瞬时错误（rate limit、timeout、网络问题）。

**实现**：

```python
def retry_with_backoff(fn, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fn()
        except (RateLimitError, TimeoutError, APIError) as e:
            if attempt == max_retries - 1:
                raise
            # 指数退避
            time.sleep(2 ** attempt)
```

**注意**：
- **只重试瞬时错误**，不要重试逻辑错误。
- **指数退避**（1s、2s、4s）。
- **有上限**（默认 3 次）。

### 策略 2：降级（Fallback）

**适用**：主模型/工具失败，但有备份。

**实现**：

```python
class FallbackChain:
    def __init__(self, models):
        self.models = models  # [primary, secondary, fallback]
    
    def generate(self, prompt):
        for model in self.models:
            try:
                return model.generate(prompt)
            except Exception as e:
                logger.warning(f"{model.name} failed: {e}")
                continue
        raise AllModelsFailed()
```

**注意**：
- **备份必须可靠**（不能"主模型挂了备份也挂"）。
- **监控降级率**（频繁降级 = 主模型有问题）。

### 策略 3：回滚（Rollback）

**适用**：Agent 做出错误决策，要撤销影响。

**实现**：

```python
class RollbackManager:
    def __init__(self):
        self.history = []  # 操作历史
    
    def record(self, operation):
        """记录操作"""
        self.history.append(operation)
    
    def rollback(self, to_step):
        """回滚到指定步骤"""
        for op in reversed(self.history[to_step:]):
            op.undo()  # 撤销操作
        self.history = self.history[:to_step]
```

**注意**：
- **每个写入操作都必须可回滚**。
- **定期保存 checkpoint**。

### 策略 4：升级（Escalate）

**适用**：Agent 无法处理，需要人类介入。

**实现**：

```python
class EscalationHandler:
    def escalate(self, task, reason, context):
        """升级到人类"""
        escalation = {
            "task_id": task.id,
            "reason": reason,
            "context": context,
            "timestamp": datetime.utcnow().isoformat(),
            "approver": task.assigned_human
        }
        
        # 发送通知（飞书 / 邮件 / Slack）
        send_notification(escalation)
        
        # 等待人类响应
        return wait_for_response(escalation, timeout=3600)
```

**注意**：
- **必须有 SLA**（如 24 小时内必须响应）。
- **必须有升级路径**（如主审批人忙 → 备份审批人）。

### 策略 5：熔断（Circuit Breaker）

**适用**：后端服务持续失败，需要暂时停止调用。

**实现**：

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure = None
        self.state = "closed"  # closed / open / half-open
    
    async def call(self, fn):
        if self.state == "open":
            if time.time() - self.last_failure > self.recovery_timeout:
                self.state = "half-open"
            else:
                raise CircuitOpenError("Circuit is open")
        
        try:
            result = await fn()
            self.failure_count = 0
            self.state = "closed"
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure = time.time()
            
            if self.failure_count >= self.failure_threshold:
                self.state = "open"
                logger.error(f"Circuit opened after {self.failure_count} failures")
            
            raise
```

**注意**：
- **熔断后必须能恢复**（half-open 状态）。
- **监控熔断频率**（频繁熔断 = 系统有问题）。

---

## 三、实战错误处理模式

### 模式 1：分层处理

```
LLM 调用错误
    ↓ 重试（3 次，指数退避）
    ↓ 降级（备用模型）
    ↓ 升级（人类）

LLM 输出错误
    ↓ Schema 校验失败 → 重试（更严格的 prompt）
    ↓ 幻觉 → RAG + 答案验证
    ↓ 内容违规 → 输出过滤

工具调用错误
    ↓ 参数错误 → 重试（修正参数）
    ↓ 工具失败 → 降级（其他工具）
    ↓ 工具不可用 → 跳过（标记 partial）

Agent 编排错误
    ↓ 任务分解错误 → 重新分解
    ↓ 依赖循环 → 报错（升级人工）
    ↓ 调度错误 → 重选 Agent
    ↓ 状态丢失 → 从 checkpoint 恢复
```

### 模式 2：错误分类

```python
class ErrorClassifier:
    """错误分类器"""
    
    @staticmethod
    def classify(error: Exception) -> str:
        if isinstance(error, RateLimitError):
            return "transient"  # 瞬时错误，可重试
        if isinstance(error, TimeoutError):
            return "transient"
        if isinstance(error, APIError):
            return "transient"
        if isinstance(error, ValidationError):
            return "logic"  # 逻辑错误，需修复
        if isinstance(error, HallucinationError):
            return "logic"
        if isinstance(error, ToolNotFoundError):
            return "config"  # 配置错误，需重新配置
        return "unknown"  # 未知错误
```

### 模式 3：死循环检测

```python
class LoopDetector:
    """检测 Agent 是否进入死循环"""
    
    def __init__(self, similarity_threshold=0.9, window=5):
        self.threshold = similarity_threshold
        self.window = window
        self.history = []
    
    def check(self, output: str) -> bool:
        """检查是否死循环"""
        self.history.append(output)
        if len(self.history) < self.window:
            return False
        
        # 与最近 N 次输出对比相似度
        recent = self.history[-self.window:]
        for i in range(len(recent) - 1):
            for j in range(i + 1, len(recent)):
                sim = self.similarity(recent[i], recent[j])
                if sim > self.threshold:
                    logger.error(f"Loop detected! similarity={sim}")
                    return True
        return False
    
    def similarity(self, a, b):
        """简单相似度（生产用 embedding 距离）"""
        from difflib import SequenceMatcher
        return SequenceMatcher(None, a, b).ratio()
```

### 模式 4：Checkpoint 与恢复

```python
class CheckpointManager:
    """检查点管理"""
    
    def __init__(self, storage):
        self.storage = storage
    
    def save(self, state: Dict, name: str):
        """保存检查点"""
        self.storage.save(
            f"checkpoints/{name}.json",
            json.dumps(state, ensure_ascii=False)
        )
    
    def load(self, name: str) -> Dict:
        """加载检查点"""
        data = self.storage.load(f"checkpoints/{name}.json")
        return json.loads(data)
    
    def list(self) -> List[str]:
        """列出所有检查点"""
        return self.storage.list("checkpoints/")
```

---

## 四、错误处理最佳实践

### 实践 1：失败安全优先

- 任何写入操作都可回滚。
- 关键操作必须 HITL。
- 高风险操作必须有二次确认。

### 实践 2：可观测性

- 所有错误都有日志 + trace。
- 错误率有告警。
- 失败模式有分析报告。

### 实践 3：错误预算

- 系统允许的失败率（如 0.1%）。
- 超过错误预算 = 暂停新功能上线。

### 实践 4：定期演练

- 每月做一次"故障演练"（Chaos Engineering）。
- 主动注入错误（如 kill 服务）。
- 验证恢复机制。

### 实践 5：错误日志标准化

```python
logger.error("agent_error", extra={
    "error_type": "RateLimitError",
    "agent_id": "orchestrator",
    "task_id": "T4.1",
    "retry_count": 2,
    "max_retries": 3,
    "next_action": "fallback_to_gpt_3.5",
    "trace_id": "uuid-xxx"
})
```

---

## 五、本章关键 takeaway

- **5 类错误**：LLM 调用、LLM 输出、工具调用、Agent 编排、系统级。
- **5 大处理策略**：重试（瞬时）、降级（备份）、回滚（撤销）、升级（人类）、熔断（暂停）。
- **实战模式**：分层处理、错误分类、死循环检测、checkpoint 恢复。
- **最佳实践**：失败安全优先、可观测性、错误预算、定期演练、日志标准化。
- **错误处理是 Agent 工程的核心能力**，从第一天起就要做。

---

**返回**：[10-tech-multi-agent README](./README.md)
**上一篇**：[13 - OpenClaw 配置实战](./13-OpenClaw配置实战.md)
