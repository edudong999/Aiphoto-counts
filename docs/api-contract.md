# AI 拍照快速计数小助手 - 接口契约文档

> **版本**: v1.0  
> **更新日期**: 2026-06-05  
> **适用范围**: 移动端前端 ↔ 后端 API 服务 ↔ AI 推理服务  
> **状态**: 启动会评审稿，等待三方签字确认

---

## 1. 总体架构

```
┌──────────────┐    HTTP/JSON     ┌──────────────┐    HTTP/JSON     ┌──────────────┐
│  Android App │ ────────────────>│  Backend API │ ────────────────>│  Model Svc   │
│  (no port)   │ <────────────────│  port 8000   │ <────────────────│  port 8001   │
└──────────────┘                  └──────┬───────┘                  └──────────────┘
                                        │
                                        │ 读写
                                        ▼
                                ┌──────────────┐
                                │   SQLite     │
                                │ records.db   │
                                └──────────────┘
                                        ▲
                                ┌──────┴───────┐
                                │  uploads/    │  ← 图片文件
                                │  目录         │
                                └──────────────┘
```

### 1.1 服务划分

| 服务 | 端口 | 对外/内网 | 职责 |
|---|---|---|---|
| Backend API | 8000 | 对外（前端） | 图片接收、模型调度、结果聚合、数据校验、历史记录 |
| Model Service | 8001 | 内网（仅后端） | 加载模型、跑推理、返回原始检测结果 |
| SQLite | 文件 | 内网 | 存历史记录 + 检测明细 |
| uploads/ | 目录 | 内网 | 存上传的原图和缩略图 |

### 1.2 通用约定

| 项目 | 规范 |
|---|---|
| API 版本前缀 | `/api/v1/` |
| 数据格式 | JSON（UTF-8） |
| 时间格式 | ISO 8601（如 `2026-06-05T14:23:45+08:00`） |
| 坐标单位 | 像素（整数，原图坐标系，**不归一化**） |
| 支持图片格式 | jpg / jpeg / png / webp |
| 单图大小上限 | 10 MB |
| 单图尺寸上限 | 长边 ≤ 4096 px（超出在后端做等比缩放） |
| 置信度 | float，范围 [0, 1] |
| 鉴权 | MVP 阶段无（内网部署）；预留 `X-API-Key` 请求头 |
| CORS | 后端默认放开 `*`（移动端原生调用，无需跨域） |

### 1.3 通用响应结构

**所有接口**（无论成功失败）均返回以下三段式 JSON：

**成功：**
```json
{
  "code": 0,
  "message": "success",
  "data": { ... }
}
```

**失败：**
```json
{
  "code": 4001,
  "message": "图片格式不支持",
  "data": null
}
```

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| code | int | 是 | 0 = 成功；非 0 见错误码表 |
| message | string | 是 | 人类可读消息，**前端不做逻辑判断**，仅用于弹 Toast |
| data | object \| null | 是 | 业务数据；失败时固定为 `null` |

### 1.4 错误码表（后端 API）

| code | HTTP | 含义 | 触发场景 |
|---|---|---|---|
| 0 | 200 | 成功 | - |
| 4000 | 400 | 请求参数错误 | 缺必填字段、参数类型错 |
| 4001 | 400 | 图片格式不支持 | 非 jpg/png/webp |
| 4002 | 400 | 图片过大 | > 10 MB 或长边 > 4096 |
| 4003 | 400 | 图片损坏 | 解码失败 |
| 4004 | 400 | 置信度非法 | 超出 [0.05, 0.95] |
| 4005 | 400 | 类别非法 | 编辑时 class 不在白名单 |
| 4040 | 404 | 资源不存在 | record_id / image_id 找不到 |
| 5000 | 500 | 服务器内部错误 | 兜底 |
| 5001 | 503 | AI 推理服务不可用 | 后端 ping 模型服务不通 |
| 5002 | 504 | AI 推理超时 | 推理耗时 > 30s |
| 5003 | 500 | AI 推理结果异常 | 返回空 / 字段缺失 |

> **前端处理建议**：code != 0 时直接弹 `message`；code == 5001/5002 时给"重试"按钮。

---

## 2. 后端 API（前端 ↔ 后端，对外）

### 2.1 拍照检测

**`POST /api/v1/detect`**

核心接口：上传图片，返回检测结果。**默认会自动创建一条历史记录**。

| 项目 | 值 |
|---|---|
| Method | POST |
| Content-Type | multipart/form-data |
| URL | `http://{host}:8000/api/v1/detect` |
| 建议超时 | 前端 30 秒 |

**Form 字段：**

| 字段 | 类型 | 必填 | 默认 | 说明 |
|---|---|---|---|---|
| `image` | file | 是 | - | 图片文件 |
| `confidence` | float | 否 | 0.25 | 置信度阈值，范围 [0.05, 0.95] |
| `save_record` | bool | 否 | true | 是否保存到历史记录 |
| `source` | string | 否 | `unknown` | 图片来源枚举：`camera` / `album` / `unknown`，用于统计与问题定位 |

**响应 200：**
```json
{
  "code": 0,
  "message": "success",
  "data": {
    "image_id": "img_20260605_a3f2b1",
    "image_url": "/api/v1/images/img_20260605_a3f2b1.jpg",
    "width": 1920,
    "height": 1080,
    "objects": [
      {
        "class": "apple",
        "class_zh": "苹果",
        "count": 8,
        "boxes": [
          {"x1": 120, "y1": 230, "x2": 310, "y2": 420, "confidence": 0.92},
          {"x1": 340, "y1": 250, "x2": 510, "y2": 430, "confidence": 0.88}
        ]
      },
      {
        "class": "bottle",
        "class_zh": "瓶子",
        "count": 3,
        "boxes": [
          {"x1": 50, "y1": 60, "x2": 180, "y2": 360, "confidence": 0.87}
        ]
      }
    ],
    "total_objects": 20,
    "inference_time_ms": 245,
    "record_id": "rec_20260605_x9k2m"
  }
}
```

**字段说明：**

| 字段 | 类型 | 说明 |
|---|---|---|
| `image_id` | string | 图片唯一 ID，前端可用于本地缓存 key |
| `image_url` | string | 相对路径，前端拼接 `http://{host}:8000` 访问 |
| `width`, `height` | int | **原图**尺寸，用于前端画框坐标映射（不取显示图尺寸） |
| `objects` | array | 按 class 聚合后的检测结果 |
| `objects[].class` | string | 英文类别标识（与 `/classes` 接口对齐） |
| `objects[].class_zh` | string | 中文显示名（前端**直接展示**，无需本地映射） |
| `objects[].count` | int | 该类别的数量（**保证 == boxes.length**） |
| `objects[].boxes` | array | 该类别下所有检测框 |
| `boxes[].x1,y1,x2,y2` | int | 像素坐标，**原图坐标系** |
| `boxes[].confidence` | float | 该框的检测置信度 [0,1] |
| `total_objects` | int | 所有类别总数 = sum(objects[].count) |
| `inference_time_ms` | int | 模型推理耗时（不含网络/IO），前端可用于显示"识别耗时" |
| `record_id` | string \| null | 历史记录 ID；`save_record=false` 时为 `null` |
| `source` | string | 图片来源（与请求参数回显一致）：`camera` / `album` / `unknown` |

**错误场景：**

| 触发条件 | code | message |
|---|---|---|
| 缺 `image` 字段 | 4000 | 缺少必填参数 image |
| 文件非 jpg/png/webp | 4001 | 图片格式不支持，仅支持 jpg/png/webp |
| 文件 > 10 MB | 4002 | 图片过大，不能超过 10MB |
| 长边 > 4096 px | 4002 | 图片尺寸过大 |
| 文件非合法图片 | 4003 | 图片损坏或无法解码 |
| `confidence` 越界 | 4004 | 置信度阈值需在 [0.05, 0.95] |
| `source` 取值非法 | 4000 | source 必须为 camera / album / unknown 之一 |
| 模型服务挂 | 5001 | AI 推理服务暂时不可用，请稍后重试 |
| 推理 > 30s | 5002 | AI 推理超时，请重试 |

**curl 示例：**
```bash
# 相机拍照
curl -X POST http://localhost:8000/api/v1/detect \
  -F "image=@/path/to/photo.jpg" \
  -F "confidence=0.3" \
  -F "source=camera"

# 相册选图
curl -X POST http://localhost:8000/api/v1/detect \
  -F "image=@/path/to/album.jpg" \
  -F "source=album"
```

---

### 2.2 获取支持的类别清单

**`GET /api/v1/classes`**

前端用于：① 结果页展示 class_zh 文案；② 引导页告诉用户"能识别什么"。

**响应 200：**
```json
{
  "code": 0,
  "message": "success",
  "data": {
    "total": 12,
    "classes": [
      {"class": "apple",     "class_zh": "苹果", "emoji": "🍎", "custom": false},
      {"class": "orange",    "class_zh": "橙子", "emoji": "🍊", "custom": false},
      {"class": "banana",    "class_zh": "香蕉", "emoji": "🍌", "custom": false},
      {"class": "bottle",    "class_zh": "瓶子", "emoji": "🍶", "custom": false},
      {"class": "cup",       "class_zh": "杯子", "emoji": "☕",  "custom": false},
      {"class": "bowl",      "class_zh": "碗",   "emoji": "🥣", "custom": false},
      {"class": "book",      "class_zh": "书",   "emoji": "📖", "custom": false},
      {"class": "scissors",  "class_zh": "剪刀", "emoji": "✂️", "custom": false},
      {"class": "cell phone","class_zh": "手机", "emoji": "📱", "custom": false},
      {"class": "keyboard",  "class_zh": "键盘", "emoji": "⌨️", "custom": false},
      {"class": "mouse",     "class_zh": "鼠标", "emoji": "🖱️", "custom": false},
      {"class": "clock",     "class_zh": "时钟", "emoji": "🕐", "custom": false}
    ]
  }
}
```

> `custom: true` 表示该类别**非 COCO 原生**，AI 同学用自定义数据集训练。  
> **前端建议**：可在引导页用 `custom` 字段打个"特殊支持"角标。

---

### 2.3 获取历史记录列表

**`GET /api/v1/records`**

**Query 参数：**

| 字段 | 类型 | 必填 | 默认 | 说明 |
|---|---|---|---|---|
| `page` | int | 否 | 1 | 页码，从 1 开始 |
| `page_size` | int | 否 | 20 | 每页条数，范围 [1, 100] |
| `keyword` | string | 否 | - | 模糊搜索（按 `class_zh` 或 `summary`） |

**响应 200：**
```json
{
  "code": 0,
  "message": "success",
  "data": {
    "total": 42,
    "page": 1,
    "page_size": 20,
    "items": [
      {
        "record_id": "rec_20260605_x9k2m",
        "image_id": "img_20260605_a3f2b1",
        "image_url": "/api/v1/images/img_20260605_a3f2b1.jpg",
        "thumbnail_url": "/api/v1/images/img_20260605_a3f2b1_thumb.jpg",
        "summary": "苹果 8 个、笔 12 支",
        "total_objects": 20,
        "is_edited": false,
        "created_at": "2026-06-05T14:23:45+08:00"
      }
    ]
  }
}
```

**字段说明：**

| 字段 | 说明 |
|---|---|
| `summary` | 摘要文本，前端**直接显示**（后端生成，无需前端再算） |
| `is_edited` | 用户是否手动修改过；前端可显示"已编辑"角标 |
| `thumbnail_url` | 缩略图（200x200 居中裁剪），列表用 |

---

### 2.4 获取单条历史详情

**`GET /api/v1/records/{record_id}`**

**响应 200：**
```json
{
  "code": 0,
  "message": "success",
  "data": {
    "record_id": "rec_20260605_x9k2m",
    "image_id": "img_20260605_a3f2b1",
    "image_url": "/api/v1/images/img_20260605_a3f2b1.jpg",
    "width": 1920,
    "height": 1080,
    "objects": [
      {
        "class": "apple",
        "class_zh": "苹果",
        "count": 8,
        "boxes": [
          {"x1": 120, "y1": 230, "x2": 310, "y2": 420, "confidence": 0.92}
        ]
      }
    ],
    "total_objects": 20,
    "is_edited": false,
    "summary": "苹果 8 个、笔 12 支",
    "created_at": "2026-06-05T14:23:45+08:00",
    "updated_at": "2026-06-05T14:23:45+08:00"
  }
}
```

**错误：**
- `record_id` 不存在 → code 4040

---

### 2.5 更新历史记录（手动修改同步）

**`PUT /api/v1/records/{record_id}/detections`**

用户在结果页做了 +/- 计数、删除某个框、长按修改类别后，前端调此接口把**完整最新状态**传给后端保存。

**请求 Body：**
```json
{
  "objects": [
    {
      "class": "apple",
      "count": 9,
      "boxes": [
        {"x1": 120, "y1": 230, "x2": 310, "y2": 420, "confidence": 0.92}
      ]
    }
  ],
  "summary": "苹果 9 个、笔 12 支（已手动调整）"
}
```

**约束（后端会校验）：**
- `objects[].class` 必须出现在 `/api/v1/classes` 返回的列表里 → 违反返回 4005
- `objects[].count` 必须 == `objects[].boxes.length` → 违反返回 4000
- 替换式更新：传什么就保存什么，**后端不做 diff**（前端是 single source of truth）

**响应 200：**
```json
{
  "code": 0,
  "message": "success",
  "data": {
    "record_id": "rec_20260605_x9k2m",
    "is_edited": true,
    "updated_at": "2026-06-05T14:35:12+08:00"
  }
}
```

---

### 2.6 删除历史记录

**`DELETE /api/v1/records/{record_id}`**

**响应 200：**
```json
{"code": 0, "message": "success", "data": null}
```

**副作用：** 同时删除 `uploads/` 下对应的 `image_id` 图片文件和缩略图。

---

### 2.7 获取图片文件

**`GET /api/v1/images/{filename}`**

返回 jpg/png/webp 二进制。`filename` 由 `image_url` / `thumbnail_url` 字段给出，前端直接拼 `http://{host}:8000` 访问。

**HTTP 头：**
- `Content-Type`：根据文件后缀设置
- `Cache-Control: public, max-age=2592000`（30 天）
- `ETag`：基于文件 mtime

**注意：**
- 路径必须做白名单校验，禁止 `..` 路径穿越 → 触发返回 4040
- thumbnail 文件命名规则：`{image_id}_thumb.jpg`

---

### 2.8 后端健康检查

**`GET /health`**

部署/监控用，不开放给前端业务逻辑。

**响应 200：**
```json
{
  "status": "ok",
  "version": "1.0.0",
  "model_service": "ok",
  "db": "ok",
  "timestamp": "2026-06-05T14:23:45+08:00"
}
```

`model_service` 字段会主动 ping 一次模型服务，`ok` / `unreachable`。

---

## 3. AI 推理服务 API（后端 ↔ 模型，**仅内网**）

> 该组接口**不开放给前端**，仅后端服务调用，端口 8001。

### 3.1 模型推理

**`POST /infer`**

| 项目 | 值 |
|---|---|
| Method | POST |
| Content-Type | multipart/form-data |
| URL | `http://localhost:8001/infer` |
| 内部超时 | 30 秒（后端会包一层超时） |

**Form 字段：**

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `image` | file | 是 | 图片文件 |
| `confidence` | float | 否 | 置信度阈值，默认 0.25 |

**响应 200：**
```json
{
  "code": 0,
  "message": "success",
  "data": {
    "width": 1920,
    "height": 1080,
    "detections": [
      {
        "class": "apple",
        "class_zh": "苹果",
        "confidence": 0.92,
        "bbox": {"x1": 120, "y1": 230, "x2": 310, "y2": 420}
      },
      {
        "class": "apple",
        "class_zh": "苹果",
        "confidence": 0.88,
        "bbox": {"x1": 340, "y1": 250, "x2": 510, "y2": 430}
      }
    ],
    "inference_time_ms": 245
  }
}
```

> **关键设计**：模型服务返回**扁平的 detections 数组**（每个 box 一条记录），**不做按 class 聚合**。  
> 聚合由后端在 `2.1` 接口里完成 —— 这样后端可灵活做：① 二次 NMS ② 类别白名单过滤 ③ 异常值剔除。

### 3.2 模型健康检查

**`GET /health`**

**响应 200：**
```json
{
  "status": "ok",
  "model_loaded": true,
  "model_name": "yolov8n",
  "model_version": "v1.0.0",
  "classes_count": 14,
  "device": "cuda:0"
}
```

### 3.3 模型信息

**`GET /model/info`**

**响应 200：**
```json
{
  "model_name": "yolov8n",
  "version": "v1.0.0",
  "input_size": 640,
  "classes": [
    {"class": "apple",  "class_zh": "苹果"},
    {"class": "bottle", "class_zh": "瓶子"}
  ]
}
```

---

## 4. 数据库表设计（后端负责落地）

### 4.1 `records` 表

| 字段 | 类型 | 约束 | 说明 |
|---|---|---|---|
| `record_id` | TEXT | PRIMARY KEY | 格式 `rec_{yyyymmdd}_{nanoid(6)}` |
| `image_id` | TEXT | NOT NULL, UNIQUE | 格式 `img_{yyyymmdd}_{nanoid(6)}` |
| `image_path` | TEXT | NOT NULL | 相对于 `uploads/` 的路径 |
| `thumbnail_path` | TEXT | NOT NULL | 缩略图路径 |
| `width` | INTEGER | NOT NULL | 原图宽 |
| `height` | INTEGER | NOT NULL | 原图高 |
| `total_objects` | INTEGER | NOT NULL | 总数 |
| `summary` | TEXT | | 摘要文本，如 "苹果 8 个、笔 12 支" |
| `is_edited` | INTEGER | NOT NULL DEFAULT 0 | 0 / 1 |
| `source` | TEXT | NOT NULL DEFAULT 'unknown' | 图片来源：`camera` / `album` / `unknown` |
| `created_at` | TEXT | NOT NULL | ISO 8601 |
| `updated_at` | TEXT | NOT NULL | ISO 8601 |

### 4.2 `detections` 表

| 字段 | 类型 | 约束 | 说明 |
|---|---|---|---|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | 自增主键 |
| `record_id` | TEXT | NOT NULL | 外键 → `records.record_id`（**应用层维护**，不强制外键） |
| `class` | TEXT | NOT NULL | 英文类别 |
| `class_zh` | TEXT | NOT NULL | 中文显示名 |
| `confidence` | REAL | NOT NULL | 0~1 |
| `x1` | INTEGER | NOT NULL | |
| `y1` | INTEGER | NOT NULL | |
| `x2` | INTEGER | NOT NULL | |
| `y2` | INTEGER | NOT NULL | |

**索引：**
- `idx_detections_record_id(record_id)`
- `idx_records_created_at(created_at DESC)`（列表按时间倒序）

### 4.3 文件存储

```
uploads/
├── img_20260605_a3f2b1.jpg           ← 原图
├── img_20260605_a3f2b1_thumb.jpg     ← 缩略图（200x200 居中裁剪）
├── img_20260605_b7c4d2.jpg
└── ...
```

**清理策略：**
- 每次 `POST /detect` 后，删除 30 天前的孤儿图片（数据库已删但文件残留）
- 或更简单：每次 `DELETE /records/{id}` 时同步删文件，**暂不写定时任务**

---

## 5. 类别清单（5 天版已锁）

**决策已锁**（v1.2 生效）：**仅支持 COCO 12 类**，不支持 pen/coin 自定义训练。

| # | class | class_zh | 数据来源 |
|---|---|---|---|
| 1 | apple | 苹果 | COCO |
| 2 | orange | 橙子 | COCO |
| 3 | banana | 香蕉 | COCO |
| 4 | bottle | 瓶子 | COCO |
| 5 | cup | 杯子 | COCO |
| 6 | bowl | 碗 | COCO |
| 7 | book | 书 | COCO |
| 8 | scissors | 剪刀 | COCO |
| 9 | cell phone | 手机 | COCO |
| 10 | keyboard | 键盘 | COCO |
| 11 | mouse | 鼠标 | COCO |
| 12 | clock | 时钟 | COCO |

**前端行为**：`/api/v1/classes` 返回上述 12 类；UI 上 pen/coin 入口不展示。

### 5.1 为什么不支持 pen/coin

- COCO 80 类中**没有** pen 和 coin
- 5 天冲刺无时间做自定义数据集（200+ 张标注 ≥ 1.5 人天）
- 故砍掉，后续可扩展

### 5.2 后续可扩展点（不在本期范围）

- 自定义数据集训练 pen/coin
- 用户上传少量图做 few-shot
- 类别动态注册（管理员后台）

---

## 6. 关键时序图

### 6.1 一次完整拍照识别

```
Android            Backend              Model Svc         SQLite   uploads/
  │                  │                      │                │         │
  │ POST /detect      │                      │                │         │
  │ multipart image   │                      │                │         │
  │ ────────────────> │                      │                │         │
  │                  │ 保存原图 + 生成缩略图 │                │         │
  │                  │ ─────────────────────────────────────────────────>│
  │                  │ 缩放至 1280px          │                │         │
  │                  │ POST /infer            │                │         │
  │                  │ multipart image        │                │         │
  │                  │ ─────────────────────> │                │         │
  │                  │                  模型推理               │         │
  │                  │ <───────────────────── │                │         │
  │                  │  detections[]          │                │         │
  │                  │ 按 class 聚合 + 校验    │                │         │
  │                  │ INSERT records + detections           │         │
  │                  │ ──────────────────────────────────────> │         │
  │ <──────────────── │                      │                │         │
  │  200 + JSON      │                      │                │         │
  │                  │                      │                │         │
  │ (前端画框 + 展示) │                      │                │         │
```

### 6.2 手动修改计数

```
Android            Backend              SQLite
  │                  │                     │
  │ 用户在结果页 +/-  │                     │
  │ 或删除某框        │                     │
  │ 本地即时更新 UI   │                     │
  │ (无需调后端)      │                     │
  │                  │                     │
  │ 用户点击"保存"    │                     │
  │ PUT /records/{id}/detections           │
  │ ────────────────>│                     │
  │                  │ 校验 class/box 关系  │
  │                  │ UPDATE records SET is_edited=1, ...
  │                  │ DELETE FROM detections WHERE record_id=?
  │                  │ INSERT INTO detections ...
  │                  │ ───────────────────>│
  │ <──────────────── │                     │
  │  200             │                     │
```

---

## 7. 性能与稳定性约束

| 指标 | 目标 |
|---|---|
| 单次识别端到端耗时 | ≤ 3 秒（不含网络） |
| 模型推理耗时 | ≤ 1 秒（YOLOv8n 1280px 输入） |
| 并发支持 | 至少 3 个前端同时拍照不卡顿 |
| 图片保存 | SQLite 中只存路径，文件存磁盘 |
| 缩略图生成 | 后端用 Pillow 在保存时同步生成 |
| 推理超时 | 后端 30s 硬超时，超时返回 5002 |

---

## 8. 启动会 checklist

> 三方都需签字（哪怕是微信 +1）才能动手开发。

- [ ] **架构**：后端 + 模型服务分两进程部署 ✓
- [ ] **类别清单**：**已锁 12 类**（v1.2，砍 pen/coin）✓
- [ ] **数据风险**：已消除（5 天版不训练自定义类别）✓
- [ ] **图片传输**：multipart/form-data ✓
- [ ] **坐标单位**：像素，原图坐标系 ✓
- [ ] **手动编辑**：替换式更新 ✓
- [ ] **错误码**：前端按 code 弹 message，不解析内容 ✓
- [ ] **健康检查**：`/health` 双端都实现 ✓
- [ ] **CORS**：后端放开 `*` ✓
- [ ] **认证**：MVP 不做，预留 `X-API-Key` 头 ✓

---

## 9. 变更记录

| 版本 | 日期 | 变更 | 作者 |
|---|---|---|---|
| v1.0 | 2026-06-05 | 初版（14 类） | Claude |
| v1.1 | 2026-06-05 | 新增 `source` 字段（请求 + 响应 + records 表），用于区分相机/相册来源 | Claude |
| v1.2 | 2026-06-06 | **类别清单缩到 12 类**：砍掉 pen / coin（5 天冲刺决策，不做自定义训练） | Claude |

---

## 10. 附录

### 10.1 curl 速查表

```bash
# 后端 base
HOST=http://localhost:8000

# 1. 检测
curl -X POST $HOST/api/v1/detect \
  -F "image=@test.jpg" \
  -F "confidence=0.3"

# 2. 类别清单
curl $HOST/api/v1/classes

# 3. 历史列表
curl "$HOST/api/v1/records?page=1&page_size=20"

# 4. 历史详情
curl $HOST/api/v1/records/rec_20260605_x9k2m

# 5. 更新检测
curl -X PUT $HOST/api/v1/records/rec_20260605_x9k2m/detections \
  -H "Content-Type: application/json" \
  -d '{"objects":[{"class":"apple","count":9,"boxes":[]}],"summary":"苹果 9 个"}'

# 6. 删除
curl -X DELETE $HOST/api/v1/records/rec_20260605_x9k2m

# 7. 健康
curl $HOST/health

# 模型服务
curl http://localhost:8001/infer -F "image=@test.jpg"
curl http://localhost:8001/health
curl http://localhost:8001/model/info
```

### 10.2 推荐 ID 生成规则

| ID 类型 | 格式 | 示例 |
|---|---|---|
| `image_id` | `img_{yyyymmdd}_{nanoid(6)}` | `img_20260605_a3f2b1` |
| `record_id` | `rec_{yyyymmdd}_{nanoid(6)}` | `rec_20260605_x9k2m` |

`nanoid(6)` 用 6 位大小写字母+数字，约 568 亿组合，碰撞概率极低。

### 10.3 后续可扩展点（不在本期范围）

- 用户系统（多用户隔离）
- 分享识别结果到微信
- 类别自定义（用户上传少量图微调）
- 模型端侧推理（TFLite / NCNN 直接跑在手机上）
- 视频流实时计数
