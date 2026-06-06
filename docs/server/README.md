# 后端开发指南

> 服务端口: **8000**  
> 技术: Python 3.10+ / FastAPI / SQLite / Pillow / httpx  
> 负责人: _（你）_

---

## 1. 你要交付什么

一个对外提供 RESTful API 的 FastAPI 服务，负责：
1. 接收 Android 上传的图
2. 调用模型服务（8001）做推理
3. 聚合检测结果 + 校验
4. 存 SQLite（历史记录 + 检测明细）
5. 存 uploads/ 目录（原图 + 缩略图）
6. 提供历史记录 CRUD API

完整接口定义见 [`docs/api-contract.md`](../api-contract.md)。

---

## 2. 目录结构

```
server/
├── app/
│   ├── main.py                # FastAPI 入口，挂载路由
│   ├── core/
│   │   ├── config.py          # 读 .env，提供 settings
│   │   ├── logging.py         # 日志配置
│   │   └── lifespan.py        # 启动/关闭钩子
│   ├── api/
│   │   └── v1/
│   │       ├── __init__.py    # APIRouter
│   │       ├── detect.py      # POST /api/v1/detect
│   │       ├── classes.py     # GET  /api/v1/classes
│   │       ├── records.py     # GET/POST/PUT/DELETE /api/v1/records
│   │       ├── images.py      # GET  /api/v1/images/{filename}
│   │       └── health.py      # GET  /health
│   ├── models/                # Pydantic schema
│   │   ├── detect.py
│   │   ├── record.py
│   │   └── common.py          # 通用 Response[T]
│   ├── services/              # 业务逻辑
│   │   ├── model_client.py    # 调模型服务
│   │   ├── image_service.py   # 保存/缩略图
│   │   └── record_service.py  # 历史记录 CRUD
│   ├── db/
│   │   ├── __init__.py
│   │   ├── database.py        # SQLite 连接
│   │   ├── schema.sql         # 建表 SQL
│   │   └── dao.py             # DAO
│   └── utils/
│       ├── id_generator.py    # nanoid
│       └── errors.py          # 统一错误处理
├── data/                      # 运行时生成（gitignore）
│   └── records.db
├── uploads/                   # 运行时生成（gitignore）
├── tests/
│   ├── test_detect.py
│   ├── test_records.py
│   └── conftest.py
├── requirements.txt
├── .env.example
├── pyproject.toml             # black/ruff/isort 配置
└── README.md
```

---

## 3. 快速开始

### 3.1 环境准备

```bash
# Python 3.10+
python --version

# 虚拟环境
cd server
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 3.2 配置文件

```bash
cp .env.example .env
# 编辑 .env，至少改：
# MODEL_SERVICE_URL=http://localhost:8001
```

### 3.3 启动

```bash
# 开发模式（热重载）
uvicorn app.main:app --reload --port 8000

# 打开浏览器
# http://localhost:8000/docs         ← Swagger UI
# http://localhost:8000/health       ← 健康检查
```

### 3.4 测试

```bash
pytest                     # 全部
pytest tests/test_detect.py # 单文件
pytest -v --cov=app        # 带覆盖率
```

---

## 4. 关键依赖（requirements.txt 模板）

```txt
# Web 框架
fastapi==0.115.0
uvicorn[standard]==0.30.6
python-multipart==0.0.12     # multipart 上传

# 工具
pydantic==2.9.2
pydantic-settings==2.5.2
python-dotenv==1.0.1

# 异步 HTTP 客户端
httpx==0.27.2

# 图片处理
pillow==10.4.0

# 数据库
aiosqlite==0.20.0           # 异步 SQLite

# 工具
nanoid==2.0.0               # 短 ID 生成

# 测试
pytest==8.3.3
pytest-asyncio==0.24.0
pytest-cov==5.0.0

# 代码质量
black==24.10.0
ruff==0.6.8
isort==5.13.2
mypy==1.11.2
```

---

## 5. 关键模块设计

### 5.1 配置（app/core/config.py）

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # 服务
    host: str = "0.0.0.0"
    port: int = 8000
    debug: bool = True
    
    # 模型服务
    model_service_url: str = "http://localhost:8001"
    model_timeout_sec: float = 30.0
    
    # 文件
    upload_dir: str = "uploads"
    max_image_size_mb: int = 10
    max_image_dimension: int = 4096
    thumbnail_size: int = 200
    
    # DB
    db_path: str = "data/records.db"
    
    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"

settings = Settings()
```

### 5.2 通用响应（app/models/common.py）

```python
from typing import Generic, Optional, TypeVar
from pydantic import BaseModel

T = TypeVar("T")

class Response(BaseModel, Generic[T]):
    code: int = 0
    message: str = "success"
    data: Optional[T] = None
```

### 5.3 错误码（app/utils/errors.py）

```python
class BizError(Exception):
    def __init__(self, code: int, message: str, http_status: int = 400):
        self.code = code
        self.message = message
        self.http_status = http_status

# 错误码常量
class ErrCode:
    PARAM_INVALID   = 4000
    IMAGE_FORMAT    = 4001
    IMAGE_TOO_LARGE = 4002
    IMAGE_CORRUPTED = 4003
    CONF_INVALID    = 4004
    CLASS_INVALID   = 4005
    NOT_FOUND       = 4040
    INTERNAL        = 5000
    MODEL_DOWN      = 5001
    MODEL_TIMEOUT   = 5002
    MODEL_BAD_RESP  = 5003
```

### 5.4 模型客户端（app/services/model_client.py）

```python
import httpx
from app.core.config import settings
from app.utils.errors import BizError, ErrCode

async def call_infer(image_bytes: bytes, confidence: float) -> dict:
    """调模型服务 /infer，返回原始检测结果。"""
    async with httpx.AsyncClient(timeout=settings.model_timeout_sec) as client:
        try:
            resp = await client.post(
                f"{settings.model_service_url}/infer",
                files={"image": ("image.jpg", image_bytes, "image/jpeg")},
                data={"confidence": confidence},
            )
        except httpx.RequestError:
            raise BizError(ErrCode.MODEL_DOWN, "AI 推理服务暂时不可用", 503)
        except httpx.TimeoutException:
            raise BizError(ErrCode.MODEL_TIMEOUT, "AI 推理超时", 504)
    
    if resp.status_code != 200:
        raise BizError(ErrCode.MODEL_BAD_RESP, "AI 推理结果异常", 500)
    
    return resp.json()["data"]
```

### 5.5 检测路由（app/api/v1/detect.py 核心逻辑）

```python
from fastapi import APIRouter, File, Form, UploadFile

@router.post("/detect")
async def detect(
    image: UploadFile = File(...),
    confidence: float = Form(0.25),
    save_record: bool = Form(True),
    source: str = Form("unknown"),
):
    # 1. 校验
    if source not in ("camera", "album", "unknown"):
        raise BizError(ErrCode.PARAM_INVALID, "source 必须为 camera/album/unknown")
    validate_image(image)  # 格式 + 大小
    
    # 2. 保存图片
    image_id = gen_image_id()
    image_path = await save_image(image, image_id)
    thumb_path = await make_thumbnail(image_path, image_id)
    
    # 3. 调模型
    image_bytes = Path(image_path).read_bytes()
    result = await call_infer(image_bytes, confidence)
    detections = result["detections"]
    
    # 4. 按 class 聚合
    objects = aggregate_by_class(detections)
    
    # 5. 写库
    record_id = None
    if save_record:
        record_id = await save_record_to_db(image_id, image_path, objects, source)
    
    return Response(data={
        "image_id": image_id,
        "image_url": f"/api/v1/images/{Path(image_path).name}",
        "width": result["width"],
        "height": result["height"],
        "objects": objects,
        "total_objects": sum(o["count"] for o in objects),
        "inference_time_ms": result["inference_time_ms"],
        "record_id": record_id,
        "source": source,
    })
```

---

## 6. 调试技巧

### 6.1 用 mock 模型开发（W2 阶段）

如果模型服务还没好，可以在 `model_client.py` 加个 mock 开关：

```python
import os

async def call_infer(image_bytes: bytes, confidence: float) -> dict:
    if os.getenv("USE_MOCK_MODEL") == "1":
        return {
            "width": 1920, "height": 1080,
            "detections": [
                {"class": "apple", "class_zh": "苹果", "confidence": 0.9,
                 "bbox": {"x1": 100, "y1": 200, "x2": 300, "y2": 400}},
            ] * 8,  # 8 个苹果
            "inference_time_ms": 100,
        }
    # ... 真实调用
```

### 6.2 用 curl 测接口

参考 `docs/api-contract.md` 第 10.1 节的 curl 速查表。

---

## 7. 常见问题

**Q: SQLite 并发写会锁？**  
A: 用 WAL 模式 + busy_timeout。`PRAGMA journal_mode=WAL; PRAGMA busy_timeout=5000;`

**Q: 上传大图 OOM？**  
A: 校验 size + 限制 ≤ 10MB；推理前先 resize 到 1280px 再发给模型。

**Q: 日志输出到哪里？**  
A: 开发期 stdout；生产重定向到 `logs/server.log`。

**Q: 如何跨域？**  
A: FastAPI 加 CORSMiddleware，放开 `*`（内网部署无需限制）。

---

## 8. 部署（开发期）

```bash
# 跑在后台
nohup uvicorn app.main:app --host 0.0.0.0 --port 8000 > logs/server.log 2>&1 &

# 查日志
tail -f logs/server.log

# 重启
pkill -f "uvicorn app.main" && nohup uvicorn app.main:app ... &
```

---

## 9. 任务清单索引

逐周任务见 [`tasks.md`](tasks.md)。

---

## 10. 变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 | 2026-06-06 | 初版 |
