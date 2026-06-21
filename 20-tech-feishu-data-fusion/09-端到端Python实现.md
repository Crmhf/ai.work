# 09 - 端到端 Python 实现

> **TL;DR**：本篇给出飞书数据融合系统的**端到端 Python 实现骨架**——从 Webhook 接收、会议检测、妙记提取、任务创建到多维表格回写的完整链路。复制即可跑（仅需替换凭证）。

---

## 一、项目结构

```
feishu_data_fusion/
├── main.py                    # FastAPI 入口
├── config.py                  # 配置加载
├── clients/
│   ├── __init__.py
│   ├── feishu_client.py       # 飞书 API 封装
│   ├── token_manager.py       # Token 管理
│   └── rate_limiter.py        # 频率限制
├── services/
│   ├── __init__.py
│   ├── minutes_extractor.py   # 妙记 AI 产物提取
│   ├── task_creator.py        # 任务创建
│   ├── bitable_syncer.py      # 多维表格同步
│   └── reconciler.py          # 定时对账
├── webhooks/
│   ├── __init__.py
│   └── bitable_webhook.py     # 多维表格 Webhook
├── models/
│   ├── __init__.py
│   ├── minutes.py             # 妙记数据模型
│   └── task.py                # 任务数据模型
├── utils/
│   ├── __init__.py
│   ├── logger.py              # 日志
│   └── audit.py               # 审计
├── .env                       # 环境变量
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

---

## 二、配置加载（`config.py`）

```python
import os
from dataclasses import dataclass
from dotenv import load_dotenv

load_dotenv()

@dataclass
class FeishuConfig:
    app_id: str
    app_secret: str
    encrypt_key: str = ""  # 事件签名加密 key
    verification_token: str = ""  # 事件验证 token

@dataclass
class BitableConfig:
    app_token: str          # 业务多维表格 app_token
    project_table_id: str   # 项目主表 table_id
    meeting_table_id: str   # 会议登记表 table_id
    task_table_id: str      # 任务跟踪表 table_id
    feedback_table_id: str  # 客户反馈表 table_id

@dataclass
class RedisConfig:
    url: str = "redis://localhost:6379/0"

@dataclass
class AppConfig:
    feishu: FeishuConfig
    bitable: BitableConfig
    redis: RedisConfig
    webhook_base_url: str
    log_level: str = "INFO"

    @classmethod
    def load(cls) -> "AppConfig":
        return cls(
            feishu=FeishuConfig(
                app_id=os.environ["FEISHU_APP_ID"],
                app_secret=os.environ["FEISHU_APP_SECRET"],
                encrypt_key=os.environ.get("FEISHU_ENCRYPT_KEY", ""),
                verification_token=os.environ.get("FEISHU_VERIFICATION_TOKEN", "")
            ),
            bitable=BitableConfig(
                app_token=os.environ["BITABLE_APP_TOKEN"],
                project_table_id=os.environ["BITABLE_PROJECT_TABLE_ID"],
                meeting_table_id=os.environ["BITABLE_MEETING_TABLE_ID"],
                task_table_id=os.environ["BITABLE_TASK_TABLE_ID"],
                feedback_table_id=os.environ.get("BITABLE_FEEDBACK_TABLE_ID", "")
            ),
            redis=RedisConfig(
                url=os.environ.get("REDIS_URL", "redis://localhost:6379/0")
            ),
            webhook_base_url=os.environ["WEBHOOK_BASE_URL"],
            log_level=os.environ.get("LOG_LEVEL", "INFO")
        )


CONFIG = AppConfig.load()
```

### `.env`

```bash
# 飞书应用凭证
FEISHU_APP_ID=cli_xxxxxxxxxxxxxxxx
FEISHU_APP_SECRET=xxxxxxxxxxxxxxxxxxxxxxxx
FEISHU_ENCRYPT_KEY=
FEISHU_VERIFICATION_TOKEN=

# 多维表格
BITABLE_APP_TOKEN=bitcnxxxxxxxxxxxxxxxx
BITABLE_PROJECT_TABLE_ID=tblxxxxxxxxxxxx
BITABLE_MEETING_TABLE_ID=tblxxxxxxxxxxxx
BITABLE_TASK_TABLE_ID=tblxxxxxxxxxxxx
BITABLE_FEEDBACK_TABLE_ID=tblxxxxxxxxxxxx

# Redis
REDIS_URL=redis://localhost:6379/0

# 服务
WEBHOOK_BASE_URL=https://your-domain.com
LOG_LEVEL=INFO
```

---

## 三、Token 管理（`clients/token_manager.py`）

```python
import asyncio
import time
import aiohttp
from typing import Optional

class TokenManager:
    """飞书 tenant_access_token 管理器"""
    
    def __init__(self, app_id: str, app_secret: str):
        self.app_id = app_id
        self.app_secret = app_secret
        self._token: Optional[str] = None
        self._expires_at: float = 0
        self._lock = asyncio.Lock()
    
    async def get_token(self) -> str:
        """获取 token（自动刷新）"""
        async with self._lock:
            if self._token and time.time() < self._expires_at - 300:
                return self._token
            
            await self._refresh_token()
            return self._token
    
    async def _refresh_token(self):
        """刷新 token"""
        url = "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal"
        payload = {
            "app_id": self.app_id,
            "app_secret": self.app_secret
        }
        
        async with aiohttp.ClientSession() as session:
            async with session.post(url, json=payload, timeout=10) as resp:
                data = await resp.json()
        
        if data.get("code") != 0:
            raise TokenError(f"获取 token 失败: {data.get('msg')}")
        
        self._token = data["tenant_access_token"]
        self._expires_at = time.time() + data["expire"]
        print(f"✓ token 刷新成功，过期时间: {data['expire']}s")


class TokenError(Exception):
    pass
```

---

## 四、飞书 API 客户端（`clients/feishu_client.py`）

```python
import asyncio
import time
import aiohttp
import json
from typing import Dict, List, Optional

class FeishuClient:
    """飞书 API 客户端"""
    
    BASE_URL = "https://open.feishu.cn/open-apis"
    
    def __init__(self, token_manager: TokenManager):
        self.token_manager = token_manager
        self.rate_limiter = RateLimiter()
    
    async def _request(
        self,
        method: str,
        path: str,
        params: Optional[Dict] = None,
        json_body: Optional[Dict] = None,
        api_name: str = "default"
    ) -> Dict:
        """通用 API 请求"""
        await self.rate_limiter.wait_if_needed(api_name)
        
        token = await self.token_manager.get_token()
        url = f"{self.BASE_URL}{path}"
        headers = {
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json; charset=utf-8"
        }
        
        async with aiohttp.ClientSession() as session:
            async with session.request(
                method, url,
                params=params,
                json=json_body,
                headers=headers,
                timeout=aiohttp.ClientTimeout(total=30)
            ) as resp:
                data = await resp.json()
        
        if data.get("code") != 0:
            raise FeishuAPIError(
                code=data["code"],
                msg=data["msg"],
                api=path
            )
        
        return data.get("data", {})
    
    # ===== 妙记 AI 产物 =====
    async def get_minutes_artifact(self, minute_token: str) -> Dict:
        """获取妙记 AI 产物"""
        return await self._request(
            "GET",
            f"/minutes/v1/minutes/{minute_token}/artifacts",
            api_name="minutes_artifact"
        )
    
    # ===== 任务管理 =====
    async def create_task(
        self,
        summary: str,
        description: str = "",
        assignee_open_ids: List[str] = None,
        due_timestamp_ms: int = None,
        priority: str = "medium",
        origin: Optional[Dict] = None
    ) -> str:
        """创建任务"""
        body = {
            "summary": summary[:200],
            "description": description[:5000],
            "priority": priority
        }
        
        if due_timestamp_ms:
            body["due"] = {
                "timestamp": str(due_timestamp_ms),
                "is_all_day": False
            }
        
        if assignee_open_ids:
            body["assignees"] = [
                {"id": open_id} for open_id in assignee_open_ids
            ]
        
        if origin:
            body["origin"] = json.dumps(origin, ensure_ascii=False)
        
        data = await self._request(
            "POST",
            "/task/v2/tasks",
            json_body=body,
            api_name="task"
        )
        return data["task"]["guid"]
    
    async def complete_task(self, task_guid: str) -> bool:
        """完成任务"""
        await self._request(
            "POST",
            f"/task/v2/tasks/{task_guid}/complete",
            api_name="task"
        )
        return True
    
    async def get_task(self, task_guid: str) -> Dict:
        """获取任务"""
        data = await self._request(
            "GET",
            f"/task/v2/tasks/{task_guid}",
            api_name="task"
        )
        return data["task"]
    
    # ===== 多维表格 =====
    async def create_bitable_record(
        self,
        app_token: str,
        table_id: str,
        fields: Dict
    ) -> str:
        """创建多维表格记录"""
        data = await self._request(
            "POST",
            f"/bitable/v1/apps/{app_token}/tables/{table_id}/records",
            json_body={"fields": fields},
            api_name="bitable"
        )
        return data["record"]["record_id"]
    
    async def update_bitable_record(
        self,
        app_token: str,
        table_id: str,
        record_id: str,
        fields: Dict
    ) -> bool:
        """更新多维表格记录"""
        await self._request(
            "PUT",
            f"/bitable/v1/apps/{app_token}/tables/{table_id}/records/{record_id}",
            json_body={"fields": fields},
            api_name="bitable"
        )
        return True
    
    async def batch_create_bitable_records(
        self,
        app_token: str,
        table_id: str,
        records: List[Dict],
        chunk_size: int = 300
    ) -> Dict:
        """批量创建记录"""
        success, failed = 0, 0
        
        for i in range(0, len(records), chunk_size):
            chunk = records[i:i + chunk_size]
            try:
                data = await self._request(
                    "POST",
                    f"/bitable/v1/apps/{app_token}/tables/{table_id}/records/batch_create",
                    json_body={"records": chunk},
                    api_name="bitable"
                )
                success += len(data.get("records", []))
            except FeishuAPIError as e:
                failed += len(chunk)
                print(f"批量创建失败: {e}")
            
            await asyncio.sleep(0.15)
        
        return {"success": success, "failed": failed}
    
    async def list_bitable_records(
        self,
        app_token: str,
        table_id: str,
        page_size: int = 500
    ) -> List[Dict]:
        """列出所有记录（自动分页）"""
        all_records = []
        page_token = None
        
        while True:
            params = {"page_size": page_size}
            if page_token:
                params["page_token"] = page_token
            
            data = await self._request(
                "GET",
                f"/bitable/v1/apps/{app_token}/tables/{table_id}/records",
                params=params,
                api_name="bitable"
            )
            
            all_records.extend(data.get("items", []))
            
            if not data.get("has_more"):
                break
            page_token = data.get("page_token")
        
        return all_records
    
    # ===== 会议管理 =====
    async def get_meeting_info(self, meeting_id: str) -> Dict:
        """获取会议信息"""
        data = await self._request(
            "GET",
            f"/vc/v1/meetings/{meeting_id}",
            api_name="vc"
        )
        return data["meeting"]


class FeishuAPIError(Exception):
    def __init__(self, code: int, msg: str, api: str):
        self.code = code
        self.msg = msg
        self.api = api
        super().__init__(f"[{code}] {msg} (api={api})")
```

---

## 五、频率限制（`clients/rate_limiter.py`）

```python
import asyncio
import time

class RateLimiter:
    """API 频率限制器"""
    
    LIMITS = {
        "minutes_artifact": 5,
        "task": 10,
        "bitable": 10,
        "vc": 10,
        "default": 10
    }
    
    def __init__(self):
        self._call_history = {}  # api → list of timestamps
    
    async def wait_if_needed(self, api_name: str):
        limit = self.LIMITS.get(api_name, self.LIMITS["default"])
        now = time.time()
        
        if api_name not in self._call_history:
            self._call_history[api_name] = []
        
        history = self._call_history[api_name]
        history[:] = [t for t in history if t > now - 1]
        
        if len(history) >= limit:
            sleep_time = max(0, 1 - (now - history[0]))
            if sleep_time > 0:
                await asyncio.sleep(sleep_time)
        
        history.append(time.time())
```

---

## 六、妙记提取服务（`services/minutes_extractor.py`）

```python
import asyncio
from datetime import datetime, timedelta
from clients.feishu_client import FeishuClient

class MinutesExtractor:
    """妙记 AI 产物提取器"""
    
    def __init__(self, feishu_client: FeishuClient):
        self.feishu = feishu_client
    
    async def extract_with_retry(
        self,
        minute_token: str,
        max_wait_minutes: int = 30,
        poll_interval_seconds: int = 120
    ) -> Dict:
        """轮询获取妙记 AI 产物"""
        deadline = datetime.now() + timedelta(minutes=max_wait_minutes)
        
        while datetime.now() < deadline:
            try:
                data = await self.feishu.get_minutes_artifact(minute_token)
                
                # 检查是否完整
                artifact = data.get("artifact", {})
                if artifact.get("summary") and artifact.get("version"):
                    return artifact
                
            except Exception as e:
                print(f"妙记 API 异常: {e}")
            
            await asyncio.sleep(poll_interval_seconds)
        
        raise TimeoutError(f"妙记 AI 产物在 {max_wait_minutes} 分钟内未生成")
```

---

## 七、任务创建服务（`services/task_creator.py`）

```python
import json
from datetime import datetime
from typing import Dict, List
from clients.feishu_client import FeishuClient

class TaskCreator:
    """从妙记产物创建任务"""
    
    def __init__(self, feishu_client: FeishuClient):
        self.feishu = feishu_client
    
    async def sync_minutes_to_tasks(
        self,
        minute_token: str,
        meeting_id: str,
        meeting_title: str,
        bitable_record_id: str,
        bitable_app_token: str,
        bitable_table_id: str
    ) -> List[str]:
        """把妙记产物转换为任务"""
        
        # 1. 提取妙记 AI 产物
        artifact = await self.minutes_extractor.extract_with_retry(minute_token)
        
        todo_list = artifact.get("todo_list", [])
        task_guids = []
        
        # 2. 对每个 todo 创建任务
        for idx, todo in enumerate(todo_list):
            origin = {
                "source": "minutes_artifact",
                "meeting_id": meeting_id,
                "meeting_title": meeting_title,
                "minute_token": minute_token,
                "artifact_version": artifact.get("version", 1),
                "todo_content": todo.get("content"),
                "todo_index": idx,
                "bitable_app_token": bitable_app_token,
                "bitable_table_id": bitable_table_id,
                "bitable_record_id": bitable_record_id,
                "raw_todo": todo,
                "extracted_by": "minutes_artifact_extractor",
                "extracted_at": datetime.utcnow().isoformat()
            }
            
            try:
                # 解析负责人（简化：先用名字作为 ID 占位）
                assignee_ids = await self._resolve_assignees(todo.get("owner"))
                
                # 解析截止时间
                due_ms = self._parse_due(todo.get("due"))
                
                # 创建任务
                task_guid = await self.feishu.create_task(
                    summary=todo.get("content", "未命名待办")[:200],
                    description=(
                        f"**原始提取内容**: {todo.get('content', '')}\n"
                        f"**来源会议**: {meeting_title} ({meeting_id})\n"
                        f"**妙记**: {minute_token}\n"
                        f"**关联多维表格记录**: {bitable_record_id}\n\n"
                        f"---\n\n"
                        f"**会议总结**:\n{artifact.get('summary', '')}"
                    ),
                    assignee_open_ids=assignee_ids,
                    due_timestamp_ms=due_ms,
                    priority=todo.get("priority", "medium"),
                    origin=origin
                )
                task_guids.append(task_guid)
                print(f"✓ 任务创建成功: {task_guid} - {todo.get('content')[:50]}")
                
            except Exception as e:
                print(f"任务创建失败: {e}")
                continue
        
        return task_guids
    
    async def _resolve_assignees(self, owner_name: str) -> List[str]:
        """解析负责人"""
        # 简化：用通讯录搜索
        if not owner_name:
            return []
        
        try:
            data = await self.feishu._request(
                "GET",
                "/contact/v3/users",
                params={"query": owner_name, "page_size": 5},
                api_name="default"
            )
            items = data.get("items", [])
            if items:
                return [items[0]["open_id"]]
        except Exception:
            pass
        
        return []
    
    def _parse_due(self, due_str: str) -> int:
        """解析截止时间"""
        if not due_str:
            return None
        
        try:
            dt = datetime.strptime(due_str, "%Y-%m-%d")
            return int(dt.timestamp() * 1000)
        except ValueError:
            return None
```

---

## 八、Webhook 处理（`webhooks/bitable_webhook.py`）

```python
import hashlib
import hmac
import base64
import json
import asyncio
from fastapi import Request, HTTPException
from clients.feishu_client import FeishuClient

class BitableWebhookHandler:
    """多维表格 Webhook 处理"""
    
    def __init__(self, feishu_client: FeishuClient, encrypt_key: str = ""):
        self.feishu = feishu_client
        self.encrypt_key = encrypt_key
    
    async def handle(self, request: Request):
        """FastAPI 端点"""
        # 1. 签名校验
        signature = request.headers.get("X-Lark-Signature", "")
        timestamp = request.headers.get("X-Lark-Request-Timestamp", "")
        nonce = request.headers.get("X-Lark-Request-Nonce", "")
        
        body = await request.body()
        
        if not self._verify_signature(timestamp, nonce, body, signature):
            raise HTTPException(status_code=401, detail="Invalid signature")
        
        # 2. 解析 payload
        try:
            payload = json.loads(body)
        except json.JSONDecodeError:
            raise HTTPException(status_code=400, detail="Invalid JSON")
        
        # 3. 异步处理（立即返回 200）
        asyncio.create_task(self._process_event(payload))
        
        return {"code": 0, "msg": "success"}
    
    def _verify_signature(self, timestamp, nonce, body, signature):
        """校验签名"""
        if not all([timestamp, nonce, signature, self.encrypt_key]):
            return False
        
        string_to_sign = f"{timestamp}{nonce}{self.encrypt_key}"
        digest = hmac.new(
            string_to_sign.encode(),
            body,
            hashlib.sha256
        ).digest()
        expected = base64.b64encode(digest).decode()
        
        return hmac.compare_digest(signature, expected)
    
    async def _process_event(self, payload: Dict):
        """处理事件"""
        try:
            event = payload.get("type") or payload.get("event", {}).get("type")
            
            if event == "record.updated" or "record_changed" in str(event):
                await self._handle_record_updated(payload)
        except Exception as e:
            print(f"事件处理失败: {e}")
    
    async def _handle_record_updated(self, payload: Dict):
        """处理记录更新"""
        # 提取数据
        record = payload.get("event", {}).get("record", {})
        record_id = record.get("record_id")
        fields = record.get("fields", {})
        
        # 检查是否是自动同步（避免循环）
        if fields.get("数据来源") == "auto_sync":
            return
        
        # 找到对应的飞书任务
        task_guid = fields.get("任务GUID")
        if not task_guid:
            return
        
        # 同步状态到飞书任务
        new_status = fields.get("任务状态")
        if new_status:
            await self._sync_status_to_feishu(task_guid, new_status)
    
    async def _sync_status_to_feishu(self, task_guid: str, bitable_status: str):
        """同步状态到飞书任务"""
        status_map = {
            "已完成": ("done", self.feishu.complete_task),
            "已取消": ("cancelled", self.feishu.cancel_task)
        }
        
        if bitable_status in status_map:
            feishu_status, fn = status_map[bitable_status]
            try:
                await fn(task_guid)
            except Exception as e:
                print(f"同步状态失败: {task_guid} - {e}")
```

---

## 九、FastAPI 主入口（`main.py`）

```python
import asyncio
import logging
from fastapi import FastAPI, Request
from contextlib import asynccontextmanager
from config import CONFIG
from clients.feishu_client import FeishuClient
from clients.token_manager import TokenManager
from services.minutes_extractor import MinutesExtractor
from services.task_creator import TaskCreator
from services.bitable_syncer import BitableSyncer
from webhooks.bitable_webhook import BitableWebhookHandler

# 配置日志
logging.basicConfig(
    level=CONFIG.log_level,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s"
)
logger = logging.getLogger(__name__)

# 全局客户端
feishu_client: FeishuClient = None
task_creator: TaskCreator = None
webhook_handler: BitableWebhookHandler = None


@asynccontextmanager
async def lifespan(app: FastAPI):
    """应用生命周期"""
    global feishu_client, task_creator, webhook_handler
    
    # 启动
    token_mgr = TokenManager(CONFIG.feishu.app_id, CONFIG.feishu.app_secret)
    feishu_client = FeishuClient(token_mgr)
    
    minutes_extractor = MinutesExtractor(feishu_client)
    task_creator = TaskCreator(feishu_client)
    webhook_handler = BitableWebhookHandler(
        feishu_client,
        encrypt_key=CONFIG.feishu.encrypt_key
    )
    
    # 启动定时任务
    asyncio.create_task(run_scheduled_jobs())
    
    logger.info("✓ 服务启动完成")
    yield
    
    # 关闭
    logger.info("服务关闭")


async def run_scheduled_jobs():
    """运行定时任务"""
    from services.reconciler import Reconciler
    
    reconciler = Reconciler(feishu_client, CONFIG.bitable)
    
    while True:
        try:
            # 每 5 分钟：增量对账
            await reconciler.reconcile_recent(minutes=10)
            logger.info("增量对账完成")
        except Exception as e:
            logger.error(f"对账失败: {e}")
        
        await asyncio.sleep(300)  # 5 分钟


app = FastAPI(lifespan=lifespan)


@app.post("/webhook/bitable")
async def bitable_webhook(request: Request):
    """多维表格 Webhook 端点"""
    return await webhook_handler.handle(request)


@app.post("/api/v1/sync-meeting")
async def sync_meeting(
    minute_token: str,
    meeting_id: str,
    meeting_title: str,
    bitable_record_id: str
):
    """手动触发：同步会议到任务"""
    task_guids = await task_creator.sync_minutes_to_tasks(
        minute_token=minute_token,
        meeting_id=meeting_id,
        meeting_title=meeting_title,
        bitable_record_id=bitable_record_id,
        bitable_app_token=CONFIG.bitable.app_token,
        bitable_table_id=CONFIG.bitable.task_table_id
    )
    return {"task_guids": task_guids, "count": len(task_guids)}


@app.get("/health")
async def health():
    return {"status": "ok"}


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## 十、部署（`docker-compose.yml`）

```yaml
version: "3.8"

services:
  app:
    build: .
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      - redis
    restart: unless-stopped
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  redis_data:
```

### `Dockerfile`

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装系统依赖
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# 安装 Python 依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制源码
COPY . .

# 启动
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### `requirements.txt`

```
fastapi==0.104.1
uvicorn[standard]==0.24.0
aiohttp==3.9.1
python-dotenv==1.0.0
redis==5.0.1
pydantic==2.5.0
APScheduler==3.10.4
```

---

## 十一、本章关键 takeaway

- 完整项目结构：config / clients / services / webhooks / models / utils。
- Token 自动刷新（带 Redis 集中管理）。
- 频率限制器避免 QPS 超限。
- 妙记产物轮询（处理时间 5-30 分钟）。
- 任务创建带 origin 字段（溯源关键）。
- Webhook 处理：签名校验 + 立即返回 200 + 异步处理。
- Docker + docker-compose 一键部署。
- 定时对账任务在 lifespan 中启动。

---

**上一篇**：[08 - 权限配置与发布上线](./08-权限配置与发布上线.md)
**下一篇**：[10 - 部署运维与成本控制](./10-部署运维与成本控制.md)
