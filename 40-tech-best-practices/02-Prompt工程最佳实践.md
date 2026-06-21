# 02 - Prompt 工程最佳实践

> **TL;DR**：Prompt 是 AI 项目的"代码"。本篇给出 **12 条 Prompt 设计铁律** + 6 类常见模板 + 5 个反模式。

---

## 一、12 条铁律

### 铁律 1：明确角色（Role）

给 LLM 一个**具体角色**，让它的输出符合角色视角。

```python
# ✅ 好
prompt = """
你是资深软件架构师，专注于微服务系统设计。
请基于以下需求设计系统架构：
{requirements}
"""

# ❌ 差
prompt = "请帮我设计一个系统架构。"
```

### 铁律 2：明确任务（Task）

任务描述要**具体、可操作**。

```python
# ✅ 好
prompt = """
请为电商小程序设计 3 个核心页面：
1. 首页（商品轮播、分类入口）
2. 商品详情页（图、参数、加购）
3. 购物车页（商品列表、数量、总价）

每个页面输出：
- 布局结构
- 关键组件
- 交互状态（加载/空/有数据/错误）
"""

# ❌ 差
prompt = "请设计几个页面。"
```

### 铁律 3：明确约束（Constraints）

明确告诉 LLM 什么**不能做**。

```python
prompt = """
任务：设计商品详情页。

约束：
- 不超过 200 字
- 不使用专业术语（用户是普通消费者）
- 必须包含价格区域
- 不要涉及支付流程
"""
```

### 铁律 4：明确格式（Format）

明确输出格式，**便于解析**。

```python
prompt = """
请分析以下评论的情感。

输出格式（JSON）：
{
    "sentiment": "positive" | "negative" | "neutral",
    "score": 0.0-1.0,
    "key_phrases": ["关键短语1", "关键短语2"],
    "reasoning": "判断理由（一句话）"
}

评论：{comment}
"""
```

### 铁律 5：Few-shot（示例驱动）

给 LLM **2-3 个示例**，比纯文字描述更有效。

```python
prompt = """
请把用户问题分类到以下类别：bug / 功能请求 / 咨询 / 投诉 / 其他。

示例：
输入："登录按钮点了没反应"
输出：{"category": "bug", "confidence": 0.95}

输入："能不能加一个深色模式？"
输出：{"category": "功能请求", "confidence": 0.9}

输入："退款什么时候到账？"
输出：{"category": "咨询", "confidence": 0.85}

现在请处理：
输入："{user_input}"
输出：
"""
```

### 铁律 6：Chain-of-Thought（思维链）

让 LLM **逐步推理**，而非直接给答案。

```python
prompt = """
请判断以下用户反馈是否紧急。

逐步思考：
1. 反馈描述了什么问题？
2. 影响范围有多大？（个人 / 部分用户 / 所有用户）
3. 是否有替代方案？
4. 紧急程度评估？

紧急程度：低 / 中 / 高 / 紧急

反馈内容：{feedback}
"""
```

### 铁律 7：避免歧义

**歧义是 prompt 失败的最大原因**。

```python
# ❌ 差
prompt = "这个 bug 严重吗？"  # "严重" 是相对的

# ✅ 好
prompt = """
按以下标准评估 bug 严重性：
- 致命：影响所有用户、无法使用核心功能
- 严重：影响部分用户、有 workaround
- 中等：影响个别用户
- 轻微：UI 问题或文字错误

bug 描述：{description}
"""
```

### 铁律 8：变量与模板分离

**Prompt 模板和变量分离**，便于复用和版本化。

```python
# prompts/customer_classifier.txt
你是客服系统的 AI 助手。请分类客户反馈。

客户反馈：{{ user_input }}

输出 JSON：
{
    "category": "bug | feature_request | inquiry | complaint | other",
    "confidence": 0.0-1.0,
    "reasoning": "..."
}
```

```python
# 代码里
template = load_template("customer_classifier.txt")
prompt = render(template, user_input="登录按钮没反应")
```

### 铁律 9：分而治之

**复杂任务拆为多个简单 prompt**，而非一个复杂 prompt。

```python
# ✅ 好
step1_prompt = "从用户反馈中提取关键事实。"
step2_prompt = "基于事实判断严重程度。"
step3_prompt = "基于严重程度给出处理建议。"

# ❌ 差（一个超长 prompt）
all_in_one_prompt = """
请完成以下任务：
1. 提取用户反馈的关键事实
2. 判断严重程度
3. 给出处理建议
4. 翻译成英文
5. 用 JSON 输出
...
"""
```

### 铁律 10：明确输出长度

```python
prompt = """
请总结以下文章。

要求：
- 不超过 200 字
- 保留关键数据
- 不要添加文章中没有的信息

文章：{article}
"""
```

### 铁律 11：保护敏感信息

```python
# 避免把敏感信息直接放入 prompt
sanitized_input = sanitize_pii(user_input)  # 清洗 PII

prompt = f"分析以下用户反馈：{sanitized_input}"
```

### 铁律 12：明确失败模式

告诉 LLM 在不确定时**应该怎么做**。

```python
prompt = """
请提取客户合同的关键信息。

如果合同中没有相关信息，输出 null，不要猜测。

字段：
- 合同金额（数字，CNY）
- 合同期限（如 "1年"，未知则 null）
- 付款方式（如 "月付"，未知则 null）
"""
```

---

## 二、6 类常见 Prompt 模板

### 模板 1：分类（Classification）

```python
CLASSIFY_PROMPT = """
请将以下文本分类到指定类别之一。

类别：{categories}
文本：{text}

输出 JSON：
{
    "category": "...",
    "confidence": 0.0-1.0
}
"""
```

### 模板 2：提取（Extraction）

```python
EXTRACT_PROMPT = """
从以下文本中提取指定字段。

字段：{fields}
说明：{field_descriptions}

文本：{text}

输出 JSON：
{output_schema}
"""
```

### 模板 3：总结（Summarization）

```python
SUMMARIZE_PROMPT = """
请总结以下文本。

要求：
- 长度：{length}
- 风格：{style}
- 重点：{focus}

文本：{text}

输出：
"""
```

### 模板 4：转换（Transformation）

```python
TRANSFORM_PROMPT = """
请将以下文本从 {from_format} 转换为 {to_format}。

文本：{text}

输出（仅 {to_format}，不要其他内容）：
"""
```

### 模板 5：分析（Analysis）

```python
ANALYZE_PROMPT = """
请分析以下数据。

数据：{data}

分析维度：
- {dimension_1}
- {dimension_2}
- {dimension_3}

输出：
- 每个维度的分析
- 综合结论
- 行动建议
"""
```

### 模板 6：生成（Generation）

```python
GENERATE_PROMPT = """
请生成 {type} 内容。

主题：{topic}
风格：{style}
长度：{length}
受众：{audience}

要求：
- {requirement_1}
- {requirement_2}

输出：
"""
```

---

## 三、5 个常见反模式

### 反模式 1：超长 prompt

```python
# ❌ 把整个项目的需求都塞进一个 prompt
huge_prompt = f"""
我们要做一个电商系统，包含以下功能：
- 用户管理（注册、登录、找回密码、个人中心）
- 商品管理（商品列表、商品详情、商品搜索、商品分类）
- 购物车（加入购物车、删除、修改数量）
- 订单（下单、支付、取消、退款）
- 库存管理（库存预警、自动补货）
- 营销活动（优惠券、满减、秒杀）
...
请设计完整的数据库 schema。
"""
```

**问题**：LLM 注意力分散，每个点都做不好。

**解决**：拆分任务。

### 反模式 2：模糊描述

```python
# ❌
prompt = "写得专业一点"  # "专业"是什么意思？

# ✅
prompt = "使用行业术语，避免口语化，参考 AWS 官方文档的写作风格"
```

### 反模式 3：缺少约束

```python
# ❌
prompt = "请写一个 Python 函数处理订单。"
# 输出可能是 1000 行的复杂实现

# ✅
prompt = """
请写一个 Python 函数处理订单折扣。

约束：
- 不超过 50 行
- 只处理百分比折扣（不支持满减）
- 必须包含单元测试
- 异常要抛出自定义异常
"""
```

### 反模式 4：忽视上下文窗口

```python
# ❌ 把 100K token 都塞进 prompt
prompt = "以下是公司过去 5 年的所有会议纪要，{long_text}。请总结。"
# 超出上下文窗口，LLM 截断，结果不可控

# ✅ 先用 RAG 检索相关段落
relevant_chunks = retrieve_relevant(text, query)
prompt = "基于以下相关内容：{relevant_chunks}，请回答：{query}"
```

### 反模式 5：缺少评估

```python
# ❌ 改完 prompt 后不评估
new_prompt = "..."
# 直接上线

# ✅ 改完 prompt 后跑评估
old_score = eval(old_prompt, test_cases)
new_score = eval(new_prompt, test_cases)
if new_score > old_score:
    deploy(new_prompt)
```

---

## 四、Prompt 调试技巧

### 技巧 1：从简单开始

```python
# 第一版：最简单的 prompt
prompt_v1 = "请把 {input} 翻译成英文。"

# 第二版：加角色
prompt_v2 = "你是专业翻译。请把 {input} 翻译成英文。"

# 第三版：加示例
prompt_v3 = "你是专业翻译。请把 {input} 翻译成英文。\n示例：\n输入：'你好'，输出：'Hello'"

# 第四版：加约束
prompt_v4 = """你是专业翻译。要求：
- 信达雅
- 保留专业术语
- 输出仅英文，不要其他

请翻译：{input}"""
```

每次小步迭代，**对比效果**。

### 技巧 2：分析失败 case

```python
# 找出 prompt 失败的 5 个 case
failed_cases = analyze_failures(prompt, test_set)

# 找共同点
patterns = find_patterns(failed_cases)
# 例如：失败的都是"长输入"

# 针对性修改
prompt_v2 = """
你是专业翻译。如果输入超过 100 字，先总结再翻译。

请翻译：{input}
"""
```

### 技巧 3：温度调节

```python
# 创意任务：高温度（0.7-1.0）
response = llm.generate(prompt, temperature=0.9)

# 准确任务：低温度（0.0-0.3）
response = llm.generate(prompt, temperature=0.0)

# 平衡：中等温度（0.3-0.5）
response = llm.generate(prompt, temperature=0.4)
```

### 技巧 4：自检

```python
prompt = """
请提取以下合同的客户名称和金额。

提取后请自检：
- 客户名称是否完整（公司全称）？
- 金额是否包含币种？

合同：{contract}

输出 JSON：{schema}
"""
```

让 LLM **自己检查输出**，比单纯要求输出更可靠。

---

## 五、Prompt 版本管理

### 目录结构

```
prompts/
├── v1/
│   ├── system.txt
│   ├── user_template.txt
│   └── eval_results.json
├── v2/
│   ├── system.txt
│   ├── user_template.txt
│   └── eval_results.json
└── CHANGELOG.md
```

### CHANGELOG 示例

```markdown
# Prompt 变更日志

## v2.0 (2026-06-21)
- 调整：增加 Few-shot 示例（2 个）
- 评估：pass rate 从 75% 提升到 88%

## v1.1 (2026-06-15)
- 调整：明确输出 JSON 格式
- 评估：pass rate 从 60% 提升到 75%

## v1.0 (2026-06-01)
- 初始版本
- 评估：pass rate 60%
```

---

## 六、本章关键 takeaway

- 12 条铁律：角色、任务、约束、格式、Few-shot、CoT、避免歧义、模板分离、分而治之、长度、保护信息、失败模式。
- 6 类常见模板：分类、提取、总结、转换、分析、生成。
- 5 个反模式：超长、模糊、缺约束、超窗口、无评估。
- 4 个调试技巧：迭代、失败分析、温度、自检。
- Prompt 跟代码一样，必须版本化。

---

**返回**：[40-tech-best-practices README](./README.md)
**下一篇**：[03 - Agent 设计模式速查](./03-Agent设计模式速查.md)
