# 03 - Agent 设计模式速查

> **TL;DR**：本篇整理 **15 个核心 Agent 设计模式**，每个模式给出**问题、解决方案、伪代码、适用场景**。这是 Agent 工程的"模式手册"。

---

## 模式 1：ReAct（Reasoning + Acting）

### 问题

LLM 直接给答案时容易"想当然"，需要让它**推理 + 行动交替**。

### 解决方案

```
Thought → Action → Observation → Thought → Action → ... → Answer
```

### 伪代码

```python
def react_agent(task, tools):
    context = f"Task: {task}\n\n"
    
    for step in range(max_steps):
        # LLM 推理下一步行动
        thought = llm.generate(f"{context}\nNext action:")
        action = parse_action(thought)
        
        # 执行行动
        if action.is_final_answer():
            return action.answer
        
        observation = tools[action.name](action.args)
        context += f"Thought: {thought}\n"
        context += f"Action: {action}\n"
        context += f"Observation: {observation}\n\n"
```

### 适用场景

- 需要调用多个工具的研究任务。
- 复杂多步推理。

---

## 模式 2：Plan-and-Execute

### 问题

ReAct 每步重新规划，效率低，且容易"卡住"。

### 解决方案

**先规划完整计划，再逐步执行**。

### 伪代码

```python
def plan_execute_agent(task, tools):
    # 1. 规划完整计划
    plan = llm.generate(f"请为以下任务制定详细计划：{task}")
    steps = parse_plan(plan)  # ["步骤1", "步骤2", ...]
    
    # 2. 逐步执行
    results = []
    for step in steps:
        result = execute_step(step, tools, results)
        results.append(result)
        
        # 检查是否需要重新规划
        if should_replan(result):
            plan = llm.generate(f"计划执行遇到问题：{result.error}，请重新规划")
            steps = parse_plan(plan)
            results = []
    
    return aggregate(results)
```

### 适用场景

- 长链路任务（10+ 步骤）。
- 可以预先规划的任务。

---

## 模式 3：Reflection（自我反思）

### 问题

LLM 输出可能不完美，需要**自我审视并改进**。

### 解决方案

```
生成初稿 → 自我批评 → 改进 → 输出
```

### 伪代码

```python
def reflective_agent(task):
    # 1. 生成初稿
    draft = llm.generate(f"任务：{task}\n请给出初稿。")
    
    # 2. 自我批评
    critique = llm.generate(f"""
    请批评以下初稿的问题：
    {draft}
    
    关注：
    - 准确性
    - 完整性
    - 风格
    """)
    
    # 3. 改进
    improved = llm.generate(f"""
    基于以下批评改进初稿：
    
    初稿：{draft}
    批评：{critique}
    
    请输出改进版。
    """)
    
    return improved
```

### 适用场景

- 内容生成（文章、代码）。
- 需要高质量输出的场景。

---

## 模式 4：Tool-use（工具调用）

### 问题

LLM 自身无法访问外部数据/系统。

### 解决方案

LLM 通过 **Function Calling** 调用外部工具。

### 伪代码

```python
def tool_use_agent(task, tools):
    response = llm.generate(
        prompt=task,
        tools=tools,  # [{"name": "search", "description": "...", "parameters": {...}}]
    )
    
    if response.tool_calls:
        for call in response.tool_calls:
            result = tools[call.name].execute(**call.args)
            # 把结果加到上下文，让 LLM 继续推理
            response.add_tool_result(call.id, result)
        
        # 再次调用 LLM，让它基于工具结果生成最终回答
        final_response = llm.generate(response.context)
        return final_response
    
    return response.text
```

### 适用场景

- 任何需要外部数据/系统的任务。

---

## 模式 5：Memory（长期记忆）

### 问题

LLM 是无状态的，每次对话是新开始。

### 解决方案

**Memory 系统**——短期（上下文）+ 长期（向量数据库）。

### 伪代码

```python
class AgentWithMemory:
    def __init__(self):
        self.short_term = []  # 当前会话上下文
        self.long_term = VectorDB()  # 长期记忆
    
    def respond(self, user_input):
        # 1. 检索相关长期记忆
        relevant_memories = self.long_term.search(user_input, top_k=5)
        
        # 2. 构造 prompt（短期 + 长期）
        context = "\n".join(self.short_term)
        memories = "\n".join(relevant_memories)
        prompt = f"""
        相关记忆：
        {memories}
        
        当前对话：
        {context}
        
        用户输入：{user_input}
        
        回应：
        """
        
        response = llm.generate(prompt)
        
        # 3. 更新短期记忆
        self.short_term.append(f"用户：{user_input}")
        self.short_term.append(f"AI：{response}")
        
        # 4. 重要信息存入长期记忆
        if is_important(response):
            self.long_term.add(response)
        
        return response
```

### 适用场景

- 个性化助手。
- 长期任务跟踪。
- 用户偏好学习。

---

## 模式 6：RAG（检索增强生成）

### 问题

LLM 知识有截止日期，且不知道私有数据。

### 解决方案

**检索外部知识 + LLM 生成**。

### 伪代码

```python
def rag_agent(query, knowledge_base):
    # 1. 检索相关文档
    relevant_docs = knowledge_base.search(query, top_k=5)
    
    # 2. 构造 prompt
    context = "\n\n".join([doc.content for doc in relevant_docs])
    prompt = f"""
    基于以下参考文档回答问题。如果参考文档不包含答案，请说明。
    
    参考文档：
    {context}
    
    问题：{query}
    
    回答：
    """
    
    # 3. LLM 生成
    response = llm.generate(prompt)
    
    # 4. 引用来源
    return {
        "answer": response,
        "sources": [doc.metadata for doc in relevant_docs]
    }
```

### 适用场景

- 文档问答。
- 企业知识库。
- 客服系统。

---

## 模式 7：Multi-Agent Orchestration（多 Agent 编排）

### 问题

单体 Agent 处理复杂任务时上下文爆炸。

### 解决方案

**Orchestrator 调度多个 Specialist**。

### 伪代码

```python
class MultiAgentSystem:
    def __init__(self):
        self.orchestrator = OrchestratorAgent()
        self.specialists = {
            "researcher": ResearchAgent(),
            "coder": CoderAgent(),
            "reviewer": ReviewerAgent()
        }
    
    def execute(self, task):
        # Orchestrator 分解任务
        subtasks = self.orchestrator.decompose(task)
        
        # 按依赖关系调度
        results = {}
        for subtask in subtasks:
            agent = self.specialists[subtask.agent]
            results[subtask.id] = agent.execute(subtask, results)
        
        # 汇总结果
        return self.orchestrator.aggregate(results)
```

### 适用场景

- 复杂多阶段任务。
- 需要专业分工的任务。

---

## 模式 8：HITL（Human-in-the-Loop）

### 问题

高风险操作不能完全自动化。

### 解决方案

在关键决策点插入**人工审批**。

### 伪代码

```python
def hitl_workflow(task):
    # AI 执行
    ai_output = ai_agent.execute(task)
    
    # 关键决策点：人工审批
    if ai_output.requires_approval():
        approval = human_review(ai_output)
        if approval.approved:
            return execute_action(ai_output)
        else:
            return revise_with_feedback(approval.feedback)
    
    return ai_output
```

### 适用场景

- 高风险操作（删除、转账、对外）。
- 关键决策点（PRD 确认、架构评审、上线审批）。

---

## 模式 9：Routing（路由分发）

### 问题

不同类型的请求需要不同的处理。

### 解决方案

**Router Agent 把请求分发到专门的处理 Agent**。

### 伪代码

```python
def routing_agent(request):
    # 1. 分类请求
    category = llm.generate(f"请分类以下请求：{request}", categories=["billing", "tech_support", "general_inquiry"])
    
    # 2. 路由到专门 Agent
    handlers = {
        "billing": BillingAgent(),
        "tech_support": TechSupportAgent(),
        "general_inquiry": GeneralAgent()
    }
    
    return handlers[category].handle(request)
```

### 适用场景

- 多领域客服。
- 多类型任务处理。

---

## 模式 10：Chain-of-Thought（思维链）

### 问题

复杂问题 LLM 直接给答案容易错。

### 解决方案

让 LLM **逐步推理**。

### 伪代码

```python
def cot_agent(question):
    prompt = f"""
    请逐步推理回答以下问题：
    
    问题：{question}
    
    推理步骤：
    """
    response = llm.generate(prompt)
    answer = extract_final_answer(response)
    return answer
```

### 适用场景

- 数学问题。
- 逻辑推理。
- 多步分析。

---

## 模式 11：Self-Consistency（自一致性）

### 问题

单次推理可能不稳定。

### 解决方案

**多次采样，取最一致答案**。

### 伪代码

```python
def self_consistency_agent(question, n=5):
    # 1. 多次采样
    answers = []
    for _ in range(n):
        response = llm.generate(question, temperature=0.7)
        answer = extract_answer(response)
        answers.append(answer)
    
    # 2. 投票取最一致
    most_common = Counter(answers).most_common(1)[0][0]
    return most_common
```

### 适用场景

- 需要高准确率的推理任务。
- 答案空间有限的任务。

---

## 模式 12：Constitutional AI（宪法式 AI）

### 问题

LLM 可能生成有害内容。

### 解决方案

**用一套原则（constitution）自我约束**。

### 伪代码

```python
CONSTITUTION = """
1. 不生成有害、歧视、违法内容。
2. 不泄露用户隐私。
3. 不假装是真人。
4. 对不确定的事情要说明。
"""

def constitutional_agent(task):
    response = llm.generate(task)
    
    # 自我审查
    critique = llm.generate(f"""
    请基于以下原则审查输出：
    {CONSTITUTION}
    
    输出：{response}
    
    是否违反原则？
    """)
    
    if "violation" in critique.lower():
        # 重新生成
        response = llm.generate(f"上次输出违反原则：{critique}。请重新生成符合原则的版本。")
    
    return response
```

### 适用场景

- 内容生成需要合规。
- 公共场景的 AI 部署。

---

## 模式 13：Streaming（流式响应）

### 问题

LLM 响应慢，用户体验差。

### 解决方案

**流式输出**，边生成边显示。

### 伪代码

```python
def streaming_agent(query):
    response = llm.stream_generate(query)
    
    accumulated = ""
    for chunk in response:
        accumulated += chunk.text
        yield chunk.text  # 实时推送给前端
```

### 适用场景

- 长文本生成。
- 实时对话场景。

---

## 模式 14：Fallback Chain（降级链）

### 问题

主模型不可用时系统崩溃。

### 解决方案

**主模型失败时降级到备用模型**。

### 伪代码

```python
def fallback_agent(task):
    chain = [
        ("gpt-4", {"max_tokens": 4000}),
        ("gpt-4-turbo", {"max_tokens": 4000}),
        ("claude-3-opus", {"max_tokens": 4000}),
        ("gpt-3.5-turbo", {"max_tokens": 2000})  # 最后兜底
    ]
    
    for model, kwargs in chain:
        try:
            return llm_client.generate(model, task, **kwargs)
        except (RateLimitError, TimeoutError, APIError):
            continue
    
    raise AllModelsFailed()
```

### 适用场景

- 生产环境的 LLM 调用。
- 高可用要求。

---

## 模式 15：Caching（缓存）

### 问题

相同/相似的请求重复调用，浪费成本。

### 解决方案

**缓存常见 query 的响应**。

### 伪代码

```python
class CachedAgent:
    def __init__(self):
        self.cache = Redis()
    
    def respond(self, query):
        cache_key = hash(query)
        
        # 查缓存
        cached = self.cache.get(cache_key)
        if cached:
            return cached
        
        # 未命中，调 LLM
        response = llm.generate(query)
        
        # 存缓存（设置 TTL）
        self.cache.setex(cache_key, ttl=3600, value=response)
        
        return response
```

### 适用场景

- 重复率高的 query（如 FAQ）。
- 节省成本的场景。

---

## 模式速查表

| # | 模式 | 核心思想 | 典型场景 |
|---|---|---|---|
| 1 | ReAct | 推理+行动交替 | 多步研究 |
| 2 | Plan-and-Execute | 先规划后执行 | 长链路任务 |
| 3 | Reflection | 自我批评改进 | 内容生成 |
| 4 | Tool-use | 调用外部工具 | 数据查询 |
| 5 | Memory | 长期记忆 | 个性化助手 |
| 6 | RAG | 检索增强 | 文档问答 |
| 7 | Multi-Agent | 多 Agent 协作 | 复杂任务 |
| 8 | HITL | 人工审批 | 高风险操作 |
| 9 | Routing | 路由分发 | 多领域客服 |
| 10 | CoT | 思维链 | 逻辑推理 |
| 11 | Self-Consistency | 多采样投票 | 高准确率 |
| 12 | Constitutional | 自我约束 | 合规场景 |
| 13 | Streaming | 流式输出 | 实时对话 |
| 14 | Fallback | 降级链 | 高可用 |
| 15 | Caching | 响应缓存 | 节省成本 |

---

## 本章关键 takeaway

- 15 个核心 Agent 设计模式，覆盖**推理、工具、记忆、多 Agent、容错、合规、流式、缓存**等维度。
- 每个模式有**问题、解决方案、伪代码、适用场景**。
- **没有"最好"的模式**，要根据场景组合使用（如 Multi-Agent + RAG + HITL）。

---

**返回**：[40-tech-best-practices README](./README.md)
**下一篇**：[04 - 可观测性实施清单](./04-可观测性实施清单.md)
