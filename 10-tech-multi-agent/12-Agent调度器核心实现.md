# 12 - Agent 调度器核心实现

> **TL;DR**：AgentScheduler 是 Orchestrator 的执行引擎。它把任务图转换为可执行计划，按模式（串行/并行/波次/条件/HITL）调度，监控执行，触发质量门禁和异常恢复。

---

## 一、核心数据结构

```python
# orchestrator/core/scheduler.py
import asyncio
from typing import Dict, List, Optional, Callable
from dataclasses import dataclass, field
from enum import Enum
from datetime import datetime
import time
import json

class ExecutionMode(Enum):
    SEQUENTIAL = "sequential"
    PARALLEL = "parallel"
    WAVE = "wave"
    CONDITIONAL = "conditional"
    HITL = "hitl"

class TaskStatus(Enum):
    PENDING = "pending"
    READY = "ready"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    BLOCKED = "blocked"
    SKIPPED = "skipped"

@dataclass
class DispatchResult:
    task_id: str
    agent_id: str
    status: TaskStatus
    outputs: List[str]
    error: Optional[str] = None
    duration_seconds: float = 0.0
    tokens_used: int = 0
    cost_usd: float = 0.0
    started_at: Optional[str] = None
    completed_at: Optional[str] = None
    
    def to_dict(self) -> Dict:
        return {
            "task_id": self.task_id,
            "agent_id": self.agent_id,
            "status": self.status.value,
            "outputs": self.outputs,
            "error": self.error,
            "duration_seconds": self.duration_seconds,
            "tokens_used": self.tokens_used,
            "cost_usd": self.cost_usd,
            "started_at": self.started_at,
            "completed_at": self.completed_at
        }
```

---

## 二、AgentScheduler 主类

```python
class AgentScheduler:
    """
    OpenClaw 多 Agent 调度器
    负责任务分发、状态跟踪、异常恢复
    """
    
    def __init__(
        self,
        orchestrator_agent,
        project_state_path: str,
        decomposer,
        gate_checker,
        healer,
        monitor=None
    ):
        self.orchestrator = orchestrator_agent
        self.state_path = project_state_path
        self.decomposer = decomposer
        self.gate_checker = gate_checker
        self.healer = healer
        self.monitor = monitor or NullMonitor()
        
        self.state = self._load_state()
        self.active_tasks: Dict[str, asyncio.Task] = {}
        self.max_concurrent = 5  # 最大并发 Agent 数
        self.semaphore = asyncio.Semaphore(self.max_concurrent)
    
    async def execute_project(self, tasks: List[Task]) -> bool:
        """执行完整项目"""
        waves = self.decomposer.build_execution_waves(tasks)
        
        print(f"📋 Project has {len(tasks)} tasks in {len(waves)} waves")
        
        for wave_idx, wave in enumerate(waves):
            wave_num = wave_idx + 1
            print(f"\n🌊 Wave {wave_num}/{len(waves)}: {len(wave)} tasks")
            
            # 更新状态
            self.state["current_wave"] = wave_num
            self.state["status"] = "RUNNING"
            self._save_state()
            
            # 选择执行模式
            mode = self._select_mode(wave, waves, wave_idx)
            
            # 执行
            try:
                results = await self._execute_wave(wave, mode)
            except Exception as e:
                print(f"❌ Wave {wave_num} failed: {e}")
                await self._handle_wave_failure(wave, e)
                return False
            
            # 质量门禁检查
            gate_result = self.gate_checker.check_wave(wave, results)
            if not gate_result.passed:
                print(f"❌ Wave {wave_num} failed quality gate")
                
                # 尝试自愈
                healed = await self._attempt_heal(wave, results, gate_result)
                if not healed:
                    return False
                continue
            
            # 更新任务状态
            for result in results:
                self.state["tasks"][result.task_id] = result.to_dict()
                self.state["tasks"][result.task_id]["completed_at"] = datetime.utcnow().isoformat()
            self._save_state()
            
            self.monitor.record_wave_completed(wave_num, len(wave))
        
        self.state["status"] = "COMPLETED"
        self._save_state()
        print("\n✅ Project completed successfully")
        return True
    
    def _select_mode(self, wave: List[Task], all_waves: List[List[Task]], wave_idx: int) -> ExecutionMode:
        """根据任务特征选择执行模式"""
        # 单任务 → 串行
        if len(wave) == 1:
            return ExecutionMode.SEQUENTIAL
        
        # 包含 HITL 节点 → HITL 模式
        if any(t.metadata.get("hitl") for t in wave):
            return ExecutionMode.HITL
        
        # 包含条件分支 → 条件模式
        if any(t.metadata.get("conditional") for t in wave):
            return ExecutionMode.CONDITIONAL
        
        # 检查共享资源
        if not self._has_shared_resources(wave):
            return ExecutionMode.PARALLEL
        
        # 默认波次模式
        return ExecutionMode.WAVE
    
    def _has_shared_resources(self, wave: List[Task]) -> bool:
        """检查 wave 内任务是否有共享资源冲突"""
        # 简化判断：如果多个任务写同一文件 → 共享资源
        write_paths = {}
        for task in wave:
            for output in task.outputs:
                if output in write_paths:
                    return True
                write_paths[output] = task.id
        return False
```

---

## 三、执行模式实现

### 串行执行

```python
async def _execute_sequential(self, wave: List[Task]) -> List[DispatchResult]:
    """串行执行 wave 内所有任务"""
    results = []
    for task in wave:
        result = await self._dispatch(task)
        results.append(result)
        
        # 串行模式：任一任务失败则中止
        if result.status == TaskStatus.FAILED:
            raise WaveExecutionError(
                f"Task {task.id} failed: {result.error}",
                partial_results=results
            )
    return results
```

### 并行执行

```python
async def _execute_parallel(self, wave: List[Task]) -> List[DispatchResult]:
    """并行执行 wave 内所有任务"""
    async def dispatch_with_semaphore(task: Task) -> DispatchResult:
        async with self.semaphore:
            return await self._dispatch(task)
    
    coroutines = [dispatch_with_semaphore(task) for task in wave]
    raw_results = await asyncio.gather(*coroutines, return_exceptions=True)
    
    # 处理异常
    results = []
    for task, result in zip(wave, raw_results):
        if isinstance(result, Exception):
            results.append(DispatchResult(
                task_id=task.id,
                agent_id=task.agent,
                status=TaskStatus.FAILED,
                outputs=[],
                error=str(result)
            ))
        else:
            results.append(result)
    
    # 并行模式：收集所有结果，即使部分失败
    return results
```

### 条件分支执行

```python
async def _execute_conditional(self, wave: List[Task]) -> List[DispatchResult]:
    """条件分支执行"""
    results = []
    for task in wave:
        condition = task.metadata.get("condition")
        if not condition:
            # 没有条件的任务直接执行
            result = await self._dispatch(task)
            results.append(result)
            continue
        
        # 评估条件
        if self._evaluate_condition(condition):
            result = await self._dispatch(task)
            results.append(result)
        else:
            # 跳过
            results.append(DispatchResult(
                task_id=task.id,
                agent_id=task.agent,
                status=TaskStatus.SKIPPED,
                outputs=[]
            ))
    return results

def _evaluate_condition(self, condition: str) -> bool:
    """评估条件表达式（基于当前 state）"""
    # 简单实现：基于上一 wave 的结果
    context = {
        "tests_passed": self._last_tests_passed(),
        "critical_bugs": self._count_critical_bugs(),
        "previous_task_status": self._get_previous_status()
    }
    
    # 安全评估（避免 eval）
    try:
        return eval(condition, {"__builtins__": {}}, context)
    except Exception:
        return False
```

### HITL 模式

```python
async def _execute_hitl(self, wave: List[Task]) -> List[DispatchResult]:
    """人工审批模式"""
    results = []
    for task in wave:
        if task.metadata.get("hitl"):
            # 暂停等待审批
            approval_result = await self._wait_for_approval(task)
            results.append(approval_result)
        else:
            # 普通任务
            result = await self._dispatch(task)
            results.append(result)
    return results

async def _wait_for_approval(self, task: Task) -> DispatchResult:
    """等待人工审批"""
    artifact_path = task.metadata.get("approval_artifact")
    approvers = task.metadata.get("approvers", [])
    timeout_hours = task.metadata.get("timeout_hours", 24)
    
    print(f"⏸️  Pausing for approval on {task.id}")
    print(f"   Artifact: {artifact_path}")
    print(f"   Approvers: {approvers}")
    
    # 发送审批请求
    approval_id = await self.orchestrator.request_approval(
        artifact_path=artifact_path,
        approvers=approvers,
        criteria=task.metadata.get("approval_criteria", []),
        timeout_hours=timeout_hours
    )
    
    # 等待审批结果
    approval_result = await self.orchestrator.wait_for_approval(approval_id)
    
    if approval_result.approved:
        return DispatchResult(
            task_id=task.id,
            agent_id="human",
            status=TaskStatus.COMPLETED,
            outputs=[artifact_path],
            duration_seconds=approval_result.duration_seconds
        )
    else:
        return DispatchResult(
            task_id=task.id,
            agent_id="human",
            status=TaskStatus.FAILED,
            outputs=[],
            error=f"Rejected: {approval_result.reason}"
        )
```

---

## 四、任务分派核心

```python
async def _dispatch(self, task: Task) -> DispatchResult:
    """分派单个任务给 Agent"""
    start_time = time.time()
    
    with self.monitor.task_span(task) as span:
        try:
            # 1. 检查前置条件
            if not await self._check_preconditions(task):
                return DispatchResult(
                    task_id=task.id,
                    agent_id=task.agent,
                    status=TaskStatus.BLOCKED,
                    outputs=[],
                    error="前置条件不满足"
                )
            
            # 2. 解析 inputs（替换变量）
            inputs = [self._resolve(i) for i in task.inputs]
            
            # 3. 通过 OpenClaw sessions_send 调用 Agent
            response = await asyncio.wait_for(
                self.orchestrator.call_agent(
                    agent_id=task.agent,
                    message={
                        "type": "TASK_ASSIGN",
                        "task_id": task.id,
                        "task_description": task.description,
                        "inputs": inputs,
                        "expected_outputs": task.outputs,
                        "guardrails": task.guardrails
                    }
                ),
                timeout=task.estimated_duration_minutes * 60
            )
            
            # 4. 验证 Agent 产出符合契约
            if not self._validate_outputs(task, response.outputs):
                return DispatchResult(
                    task_id=task.id,
                    agent_id=task.agent,
                    status=TaskStatus.FAILED,
                    outputs=[],
                    error="Agent 产出不符合契约",
                    duration_seconds=time.time() - start_time
                )
            
            return DispatchResult(
                task_id=task.id,
                agent_id=task.agent,
                status=TaskStatus.COMPLETED if response.success else TaskStatus.FAILED,
                outputs=response.outputs,
                error=response.error if not response.success else None,
                duration_seconds=time.time() - start_time,
                tokens_used=response.tokens_used,
                cost_usd=response.cost_usd,
                started_at=datetime.fromtimestamp(start_time).isoformat(),
                completed_at=datetime.utcnow().isoformat()
            )
        
        except asyncio.TimeoutError:
            return DispatchResult(
                task_id=task.id,
                agent_id=task.agent,
                status=TaskStatus.FAILED,
                outputs=[],
                error=f"Task timeout ({task.estimated_duration_minutes}min)",
                duration_seconds=time.time() - start_time
            )
        except Exception as e:
            return DispatchResult(
                task_id=task.id,
                agent_id=task.agent,
                status=TaskStatus.FAILED,
                outputs=[],
                error=str(e),
                duration_seconds=time.time() - start_time
            )

async def _check_preconditions(self, task: Task) -> bool:
    """检查任务前置条件"""
    # 1. 依赖任务都已完成
    for dep_id in task.dependencies:
        dep_status = self.state["tasks"].get(dep_id, {}).get("status")
        if dep_status != TaskStatus.COMPLETED.value:
            return False
    
    # 2. 输入文件存在
    for input_path in task.inputs:
        resolved = self._resolve(input_path)
        if not Path(resolved).exists():
            return False
    
    # 3. Agent 可用
    if not self.orchestrator.is_agent_available(task.agent):
        return False
    
    return True

def _resolve(self, path: str) -> str:
    """解析路径中的变量"""
    # 替换 {{ variable_name }} 形式的变量
    import re
    pattern = r'\{\{\s*(\w+)\s*\}\}'
    def replacer(match):
        var_name = match.group(1)
        return self.state.get("variables", {}).get(var_name, match.group(0))
    return re.sub(pattern, replacer, path)

def _validate_outputs(self, task: Task, outputs: List[str]) -> bool:
    """验证 Agent 产出符合契约"""
    # 简化：检查所有 expected_outputs 都存在
    for expected in task.outputs:
        # 支持 glob 模式（如 "shared/02_UI/Wireframes/*.md"）
        if "*" in expected:
            import glob
            if not glob.glob(expected):
                return False
        else:
            if not Path(expected).exists():
                return False
    return True
```

---

## 五、质量门禁集成

```python
def _pass_quality_gate(self, wave: List[Task], results: List[DispatchResult]) -> bool:
    """检查 wave 是否通过质量门禁"""
    for task, result in zip(wave, results):
        if result.status != TaskStatus.COMPLETED:
            continue  # 失败的任务已经在 _handle_wave_failure 处理
        
        gate_result = self.gate_checker.check(task.id, result.outputs)
        if not gate_result.passed:
            print(f"❌ Quality gate failed for {task.id}:")
            for failed in gate_result.failed_gates:
                print(f"   - {failed.name}: {failed.error}")
            return False
    return True
```

---

## 六、自愈机制集成

```python
async def _attempt_heal(
    self,
    wave: List[Task],
    results: List[DispatchResult],
    gate_result
) -> bool:
    """尝试自愈失败的任务"""
    healed = True
    
    for task, result in zip(wave, results):
        if result.status == TaskStatus.FAILED:
            heal_result = await self.healer.heal(task, result)
            
            if heal_result.success:
                # 自愈成功，更新结果
                self.state["tasks"][task.id] = heal_result.task_result.to_dict()
                self.monitor.record_heal(task.id, success=True)
            else:
                # 自愈失败，标记整体失败
                healed = False
                if heal_result.escalated:
                    print(f"🚨 Task {task.id} escalated to human")
                    self.monitor.record_escalation(task.id, heal_result.reason)
    
    return healed

async def _handle_wave_failure(self, wave: List[Task], error: Exception):
    """处理 wave 失败"""
    self.state["status"] = "FAILED"
    self.state["last_error"] = str(error)
    self._save_state()
    
    # 通知相关人
    await self.orchestrator.notify_failure(
        wave=wave,
        error=error,
        state=self.state
    )
```

---

## 七、状态管理

```python
def _load_state(self) -> Dict:
    """加载项目状态"""
    if Path(self.state_path).exists():
        with open(self.state_path) as f:
            return json.load(f)
    return {
        "project_id": None,
        "status": "IDLE",
        "current_wave": 0,
        "total_waves": 0,
        "tasks": {},
        "started_at": None,
        "completed_at": None,
        "variables": {}
    }

def _save_state(self):
    """保存项目状态（每次变化都保存）"""
    self.state["last_updated"] = datetime.utcnow().isoformat()
    Path(self.state_path).parent.mkdir(parents=True, exist_ok=True)
    with open(self.state_path, "w") as f:
        json.dump(self.state, f, indent=2, ensure_ascii=False)

def save_checkpoint(self, wave_num: int):
    """保存断点（用于断点续跑）"""
    checkpoint = {
        "checkpoint_at_wave": wave_num,
        "state_snapshot": copy.deepcopy(self.state),
        "saved_at": datetime.utcnow().isoformat()
    }
        checkpoint_path = f"{self.state_path}.checkpoint.{wave_num}"
        with open(checkpoint_path, "w") as f:
            json.dump(checkpoint, f, indent=2, ensure_ascii=False)

async def resume_from_checkpoint(self, checkpoint_wave: int):
    """从断点恢复"""
    checkpoint_path = f"{self.state_path}.checkpoint.{checkpoint_wave}"
    if not Path(checkpoint_path).exists():
        raise FileNotFoundError(f"Checkpoint not found: {checkpoint_path}")
    
    with open(checkpoint_path) as f:
        checkpoint = json.load(f)
    
    self.state = checkpoint["state_snapshot"]
    self._save_state()
    print(f"↩️  Resumed from wave {checkpoint_wave}")
```

---

## 八、并发控制

```python
class ConcurrencyLimiter:
    """并发控制器"""
    
    def __init__(self, max_concurrent: int = 5):
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.active_count = 0
    
    async def __aenter__(self):
        await self.semaphore.acquire()
        self.active_count += 1
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        self.semaphore.release()
        self.active_count -= 1

# 在 _dispatch 中使用
async def _dispatch_with_limit(self, task: Task) -> DispatchResult:
    async with ConcurrencyLimiter(max_concurrent=self.max_concurrent):
        return await self._dispatch(task)
```

---

## 九、与 OpenClaw Gateway 集成

```python
class OpenClawGatewayClient:
    """OpenClaw Gateway 客户端"""
    
    def __init__(self, gateway_url: str, api_key: str):
        self.base_url = gateway_url
        self.api_key = api_key
    
    async def call_agent(self, agent_id: str, message: Dict) -> AgentResponse:
        """调用 Agent"""
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.base_url}/agents/{agent_id}/invoke",
                json=message,
                headers={"Authorization": f"Bearer {self.api_key}"},
                timeout=aiohttp.ClientTimeout(total=600)
            ) as resp:
                data = await resp.json()
                return AgentResponse(
                    success=data["success"],
                    outputs=data.get("outputs", []),
                    error=data.get("error"),
                    tokens_used=data.get("tokens_used", 0),
                    cost_usd=data.get("cost_usd", 0.0)
                )
    
    async def request_approval(self, artifact_path: str, approvers: List[str], criteria: List[str], timeout_hours: int) -> str:
        """请求人工审批"""
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.base_url}/approvals",
                json={
                    "artifact_path": artifact_path,
                    "approvers": approvers,
                    "criteria": criteria,
                    "timeout_hours": timeout_hours
                },
                headers={"Authorization": f"Bearer {self.api_key}"}
            ) as resp:
                data = await resp.json()
                return data["approval_id"]
    
    async def wait_for_approval(self, approval_id: str) -> ApprovalResult:
        """等待审批结果"""
        # 长轮询或 WebSocket
        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{self.base_url}/approvals/{approval_id}/wait",
                headers={"Authorization": f"Bearer {self.api_key}"},
                timeout=aiohttp.ClientTimeout(total=86400)  # 24h
            ) as resp:
                data = await resp.json()
                return ApprovalResult(
                    approved=data["approved"],
                    reason=data.get("reason"),
                    duration_seconds=data["duration_seconds"]
                )
```

---

## 十、完整使用示例

```python
# examples/run_project.py
import asyncio
from orchestrator.core.scheduler import AgentScheduler
from orchestrator.core.llm_client import OpenAIClient
from orchestrator.core.openclaw_client import OpenClawGatewayClient
from orchestrator.skills.task_decomposition import TaskDecomposer
from orchestrator.quality.gate_checker import GateChecker
from orchestrator.quality.self_healer import SelfHealer

async def run_ecommerce_project():
    # 初始化
    llm = OpenAIClient(api_key="sk-...", model="gpt-4")
    openclaw = OpenClawGatewayClient(
        gateway_url="http://openclaw-gateway:8080",
        api_key="oc_..."
    )
    decomposer = TaskDecomposer(llm)
    gate_checker = GateChecker("quality_gates.yaml")
    healer = SelfHealer("self_healing.yaml", openclaw)
    
    scheduler = AgentScheduler(
        orchestrator_agent=openclaw,
        project_state_path="project_state.yaml",
        decomposer=decomposer,
        gate_checker=gate_checker,
        healer=healer
    )
    
    # 分解需求
    tasks = decomposer.decompose(
        user_requirement="做一个电商小程序，包含商品展示、购物车、下单支付",
        project_type="mini_program"
    )
    
    # 执行
    success = await scheduler.execute_project(tasks)
    
    if success:
        print("🎉 项目完成！")
    else:
        print("❌ 项目失败，已升级人工处理")

asyncio.run(run_ecommerce_project())
```

---

## 十一、本章关键 takeaway

- AgentScheduler 是 Orchestrator 的执行引擎，按模式调度任务。
- 5 种执行模式：串行、并行、波次、条件、HITL。
- 核心流程：分派 → 验证契约 → 质量门禁 → 自愈 → 状态保存。
- 状态必须每次变化都持久化，支持断点续跑。
- 并发通过 Semaphore 限制，避免资源抢占。
- 与 OpenClaw Gateway 通过 sessions_send 集成。
- 完整的错误处理：超时、契约不符、门禁失败、Agent 卡死。

---

**上一篇**：[11 - 任务分解算法实现](./11-任务分解算法实现.md)
**下一篇**：[13 - OpenClaw 配置实战](./13-OpenClaw配置实战.md)
