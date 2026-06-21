# 07 - 事件驱动与 Automation 桥接

> **TL;DR**：飞书多维表格变更事件**只开放给 Automation Center**，外部 Bot 不能直接订阅。Automation 桥接是弥补这一缺口的唯一可靠方案。配置 + Webhook + 重试，缺一不可。

---

## 一、为什么必须用 Automation 桥接

### 平台限制

飞书多维表格的事件机制：

| 事件类型 | 谁可以订阅 | 说明 |
|---|---|---|
| 记录创建 | Automation Center | ✅ 内部使用 |
| 记录更新 | Automation Center | ✅ 内部使用 |
| 记录删除 | Automation Center | ✅ 内部使用 |
| 字段值变更 | Automation Center | ✅ 内部使用 |
| 表结构变更 | Automation Center | ✅ 内部使用 |

**关键限制**：外部 Bot（自建应用）**无法直接订阅**多维表格事件。

### 没有 Automation 桥接会怎样

如果你只靠"轮询"检测变更：

- **5 分钟轮询** → 最多延迟 5 分钟。
- **1 分钟轮询** → API 调用量翻 5 倍，QPS 压力大。
- **秒级轮询** → 不可能，QPS 限制 10。

**结论**：要"准实时"变更通知，必须用 Automation 桥接。

---

## 二、Automation 工作原理

```
用户在飞书多维表格的 Automation Center 配置规则：
"当 任务状态 字段变更时，调用 Webhook"

        ↓

多维表格内部事件触发
        ↓

Automation Center 执行 Webhook 动作
        ↓

HTTP POST 到 自建服务的 /webhook/bitable
        ↓

自建服务处理（更新任务状态等）
```

### 关键概念

| 概念 | 说明 |
|---|---|
| **Trigger（触发器）** | 什么事件触发（如"字段变更"） |
| **Action（动作）** | 触发后做什么（如"发送 Webhook"） |
| **Filter（过滤器）** | 只在特定条件下触发（如"状态变更为已完成"） |
| **Webhook** | HTTP POST 请求，包含变更数据 |

---

## 三、Automation 配置步骤

### 步骤 1：进入 Automation Center

1. 打开多维表格。
2. 点击右上角「···」→「更多」→「自动化」。

### 步骤 2：创建触发器

选择触发器类型：

- **"记录变更时"** → 任意字段变更
- **"满足特定条件时"** → 字段值变更且满足过滤条件

**推荐**：用"满足特定条件时"，避免无意义的 Webhook 调用。

#### 示例配置

```
触发器类型：满足特定条件时
触发表：任务跟踪表
触发条件：
  - 字段：任务状态
  - 条件：从 [todo, doing, blocked] 变为 [done]
```

### 步骤 3：添加 Webhook 动作

#### URL

```
https://your-domain.com/webhook/bitable
```

（你的服务地址，需要公网可访问）

#### 请求方法

```
POST
```

#### Headers

```
Content-Type: application/json
X-Lark-Signature: {{signature}}
```

#### Body

```json
{
  "event": "record.updated",
  "app_token": "{{app_token}}",
  "table_id": "{{table_id}}",
  "record_id": "{{record_id}}",
  "before": {{before_fields_json}},
  "after": {{after_fields_json}},
  "trigger_time": "{{trigger_time}}",
  "automation_id": "{{automation_id}}"
}
```

### 步骤 4：测试并启用

1. 在 Automation Center 测试按钮。
2. 手动触发一次，看 Webhook 是否收到。
3. 确认无误后启用规则。

---

## 四、Webhook 服务端实现

### 接收端点

```python
from fastapi import FastAPI, Request, HTTPException
import json

app = FastAPI()

@app.post("/webhook/bitable")
async def bitable_webhook(request: Request):
    """接收多维表格 Automation Webhook"""
    
    # 1. 签名校验
    signature = request.headers.get("X-Lark-Signature")
    body = await request.body()
    
    if not verify_signature(body, signature):
        raise HTTPException(status_code=401, detail="Invalid signature")
    
    # 2. 解析 payload
    payload = json.loads(body)
    
    # 3. 异步处理（立即返回 200，避免 Automation 重试）
    asyncio.create_task(handle_webhook(payload))
    
    return {"code": 0, "msg": "success"}


async def handle_webhook(payload: Dict):
    """处理 Webhook（异步）"""
    try:
        event = payload.get("event")
        
        if event == "record.updated":
            await handle_record_updated(payload)
        elif event == "record.created":
            await handle_record_created(payload)
        elif event == "record.deleted":
            await handle_record_deleted(payload)
        else:
            print(f"未知事件: {event}")
    
    except Exception as e:
        print(f"Webhook 处理失败: {e}")
        # 记录到重试队列
        await retry_queue.enqueue(payload)


async def handle_record_updated(payload: Dict):
    """处理记录更新事件"""
    record_id = payload["record_id"]
    after = payload["after"]
    
    # 1. 提取变更字段
    task_guid = after.get("任务GUID")
    if not task_guid:
        return
    
    # 2. 检查是否是自动同步触发的（避免循环）
    data_source = after.get("数据来源")
    if data_source == "auto_sync":
        print(f"跳过自动同步: {record_id}")
        return
    
    # 3. 同步到飞书任务
    new_status = after.get("任务状态")
    if new_status:
        await update_feishu_task(task_guid, new_status)
```

---

## 五、签名校验（安全关键）

飞书 Automation Webhook 携带 `X-Lark-Signature` header 用于校验。

### 实现

```python
import hmac
import hashlib
import base64

def verify_signature(body: bytes, signature: str, secret: str = None) -> bool:
    """校验 Webhook 签名"""
    if not signature:
        return False
    
    # 飞书使用 HMAC-SHA256
    # secret 是 Automation 配置时设置的（可选）
    
    if secret:
        digest = hmac.new(
            secret.encode(),
            body,
            hashlib.sha256
        ).digest()
        expected = base64.b64encode(digest).decode()
        return hmac.compare_digest(signature, expected)
    
    # 如果没有 secret，至少校验签名存在
    return bool(signature)


# 重要：实际飞书签名的格式可能不同，请参考最新官方文档
def verify_lark_signature_v2(timestamp: str, nonce: str, body: bytes, signature: str, encrypt_key: str) -> bool:
    """飞书事件订阅签名校验（v2）"""
    # 1. 把 timestamp + nonce + encrypt_key 拼接
    string_to_sign = f"{timestamp}{nonce}{encrypt_key}"
    
    # 2. HMAC-SHA256 + Base64
    digest = hmac.new(
        string_to_sign.encode(),
        body,
        hashlib.sha256
    ).digest()
    expected = base64.b64encode(digest).decode()
    
    return hmac.compare_digest(signature, expected)
```

**实战踩坑**：飞书签名机制有 v1 和 v2 两个版本，文档不清晰，**实测时务必用官方 SDK 验证**。

---

## 六、重试机制

Automation Webhook **不保证一定送达**：

- 网络抖动。
- 服务暂时不可用。
- 服务响应时间过长。

所以服务端必须实现**重试机制**。

### 重试策略

```python
import asyncio
from typing import Callable

class RetryQueue:
    """重试队列"""
    
    def __init__(self, redis_client):
        self.redis = redis_client
        self.max_retries = 5
        self.backoff_base = 2  # 指数退避基数
    
    async def enqueue(self, payload: Dict):
        """加入重试队列"""
        await self.redis.lpush(
            "webhook_retry_queue",
            json.dumps(payload, ensure_ascii=False)
        )
    
    async def process_queue(self):
        """处理重试队列（每 30 秒跑一次）"""
        while True:
            item = await self.redis.rpop("webhook_retry_queue")
            if not item:
                await asyncio.sleep(30)
                continue
            
            payload = json.loads(item)
            retry_count = payload.get("_retry_count", 0)
            
            try:
                await handle_webhook(payload)
            except Exception as e:
                if retry_count < self.max_retries:
                    # 指数退避
                    delay = self.backoff_base ** retry_count
                    print(f"重试 {retry_count+1}/{self.max_retries} 延迟 {delay}s")
                    
                    payload["_retry_count"] = retry_count + 1
                    await asyncio.sleep(delay)
                    await self.enqueue(payload)
                else:
                    print(f"重试耗尽，告警: {payload}")
                    await self.alert_failure(payload)


async def alert_failure(payload: Dict):
    """告警"""
    # 发送钉钉/飞书/邮件告警
    ...
```

### 幂等性保障

Webhook 可能**重复送达**（如重试场景），服务端必须**幂等**。

```python
class WebhookIdempotency:
    """Webhook 幂等性"""
    
    def __init__(self, redis_client):
        self.redis = redis_client
        self.ttl = 3600  # 1 小时去重窗口
    
    async def process_once(self, event_id: str, handler: Callable):
        """保证同一 event 只处理一次"""
        key = f"webhook_event:{event_id}"
        
        # 检查是否已处理
        if await self.redis.exists(key):
            print(f"事件已处理: {event_id}")
            return
        
        # 执行 handler
        await handler()
        
        # 标记已处理
        await self.redis.setex(key, self.ttl, "1")


# 使用
idemp = WebhookIdempotency(redis_client)

async def handle_record_updated(payload: Dict):
    event_id = payload.get("event_id") or f"{payload['record_id']}:{payload.get('trigger_time')}"
    await idemp.process_once(event_id, lambda: _do_handle(payload))
```

---

## 七、Automation 配置最佳实践

### 1. 用过滤器减少无效 Webhook

```
❌ 不好：所有字段变更都触发
✅ 好：只"任务状态"字段变更时触发

❌ 不好：状态变更为任何值都触发
✅ 好：只"doing → done"或"todo → doing"等关键转换触发
```

### 2. 配置多个 Automation 规则

不要试图用一个 Automation 规则覆盖所有场景。**为每个关键事件单独配置**：

- 规则 A：任务状态变更 → 同步到飞书任务
- 规则 B：进展变更 → 通知 PM
- 规则 C：负责人变更 → 通知新老负责人

### 3. 用 Automation 同时做"备份通知"

除了 Webhook，还可以让 Automation 同时**发送飞书消息**：

```yaml
actions:
  - type: webhook
    url: "https://your-domain.com/webhook/bitable"
  - type: send_message
    target: "user_open_id"
    content: "任务 {{任务名称}} 状态变更为 {{任务状态}}"
```

这样即使 Webhook 失败，用户也能收到消息通知。

### 4. 给 Webhook URL 加健康检查

```python
@app.get("/health")
async def health():
    return {"status": "ok", "timestamp": int(time.time())}
```

Automation Center 通常有"重试机制"，健康检查端点 200 时才认为成功。

---

## 八、实战踩坑

### 坑 1：Automation 触发不稳定

**症状**：有时候 Webhook 不来。

**原因**：飞书 Automation 本身有故障率，官方不保证 100% 送达。

**应对**：**必须有定时补偿**（参考 [`06-数据同步策略与冲突解决`](./06-数据同步策略与冲突解决.md)）。

### 坑 2：Webhook body 解析失败

**症状**：收到 Webhook 但解析失败。

**原因**：Automation body 模板用了不存在的字段引用。

**应对**：服务端做容错（解析失败不影响 200 返回，进入重试队列）。

### 坑 3：触发循环

**症状**：A 字段变更触发 Webhook → 服务更新 B 字段 → B 字段变更触发 Webhook → ...

**应对**：
- 用"数据来源"字段标记自动同步，Webhook 处理时跳过。
- 用节流（同一记录 60 秒内只处理一次）。

### 坑 4：Automation 规则被误关

**症状**：用户误操作关闭了 Automation 规则，Webhook 完全停止。

**应对**：
- 定期用 API 测试 Automation 状态。
- Webhook 长时间无调用 → 自动告警。

---

## 九、本章关键 takeaway

- 多维表格变更事件**只对 Automation Center 开放**，Automation 桥接是唯一可靠方案。
- Automation 触发条件要"窄而准"，避免无意义 Webhook。
- Webhook 服务端必须：签名校验 + 立即返回 200 + 异步处理 + 重试 + 幂等。
- **Automation 不保证 100% 送达**，必须有定时补偿兜底。
- 触发循环：用"数据来源"标记 + 节流避免。
- Automation 配置可加消息通知作为"备份通知"。

---

**上一篇**：[06 - 数据同步策略与冲突解决](./06-数据同步策略与冲突解决.md)
**下一篇**：[08 - 权限配置与发布上线](./08-权限配置与发布上线.md)
