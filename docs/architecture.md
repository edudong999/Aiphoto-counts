# 系统架构

> 版本: v1.0 · 更新日期: 2026-06-06

---

## 1. 整体架构

```
┌─────────────────┐     HTTP/JSON      ┌──────────────────┐    HTTP/JSON     ┌──────────────────┐
│   Android App   │ ─────────────────> │   Backend API    │ ───────────────> │   Model Service  │
│  (Kotlin, 原生) │ <───────────────── │  FastAPI :8000   │ <─────────────── │  FastAPI :8001   │
└─────────────────┘     拍照/识别       └────────┬─────────┘    推理请求       └──────────────────┘
                                                 │
                                          ┌──────┴──────┐
                                          │             │
                                          ▼             ▼
                                  ┌──────────┐  ┌────────────┐
                                  │ SQLite   │  │  uploads/  │
                                  │ 元数据    │  │  原图+缩略图│
                                  └──────────┘  └────────────┘
```

三个**独立部署**的服务，职责清晰、易于并行开发。

---

## 2. 服务清单

| 服务 | 端口 | 进程 | 数据存储 | 启动方式 |
|---|---|---|---|---|
| Android App | - | - | 本地 Room | Android Studio |
| Backend API | 8000 | uvicorn | SQLite + uploads/ | `uvicorn app.main:app` |
| Model Service | 8001 | uvicorn | 模型权重文件 | `python -m app.server` |

**为什么拆分后端和模型服务？**
- 模型可独立迭代（换模型不影响后端）
- 模型可单独压测
- 后端无需装 torch（节省 ~2GB 依赖）

---

## 3. 技术选型与理由

| 类别 | 选择 | 理由 |
|---|---|---|
| **Android 语言** | Kotlin | 官方推荐，语法现代；Java 备选 |
| **Android 相机** | CameraX | Google 官方，兼容性好，比 Camera2 简单 |
| **Android 网络** | OkHttp + Retrofit | 业界标准，multipart 上传稳定 |
| **Android 本地存储** | Room | Google 官方 ORM，基于 SQLite |
| **后端框架** | FastAPI | 异步、自动 Swagger、类型提示；Flask 也可 |
| **后端 DB** | SQLite | 零运维，单文件够用；后续可平滑迁 Postgres |
| **后端图片处理** | Pillow | 缩放、缩略图、格式转换 |
| **后端 HTTP 客户端** | httpx | 异步、支持超时，替代 requests |
| **模型** | YOLOv8n | 轻量（~6MB）、速度快、ultralytics 库好用 |
| **推理服务** | FastAPI | 与后端同栈，部署简单 |
| **包管理** | pip + venv | 标配；写 `requirements.txt` 锁版本 |

**否决的方案：**
- ❌ YOLOv5：v8 接口更现代，文档更好
- ❌ RT-DETR / Grounding DINO：过大或过慢，桌面端用不上
- ❌ MySQL / Redis：MVP 阶段过度设计
- ❌ Flutter / 小程序：要求 Android 原生，跨端框架会拖慢进度

---

## 4. 目录结构（项目根）

```
Aiphoto-counts/
├── android/              Android 工程（Gradle + Kotlin）
│   ├── app/
│   │   ├── src/main/
│   │   │   ├── java/com/aiphoto/counts/
│   │   │   │   ├── ui/           页面（Activity/Fragment/Compose）
│   │   │   │   ├── camera/       CameraX 封装
│   │   │   │   ├── network/      Retrofit/OkHttp 接口
│   │   │   │   ├── data/         Room 实体 + DAO
│   │   │   │   └── util/
│   │   │   ├── res/              布局/图片/字符串
│   │   │   └── AndroidManifest.xml
│   │   └── build.gradle.kts
│   └── build.gradle.kts
│
├── server/               后端 FastAPI 工程
│   ├── app/
│   │   ├── main.py               FastAPI 入口
│   │   ├── api/                  路由
│   │   │   ├── v1/
│   │   │   │   ├── detect.py
│   │   │   │   ├── classes.py
│   │   │   │   ├── records.py
│   │   │   │   └── images.py
│   │   ├── core/                 配置、日志、依赖
│   │   ├── models/               Pydantic schema
│   │   ├── services/             业务逻辑
│   │   │   ├── model_client.py
│   │   │   ├── image_service.py
│   │   │   └── record_service.py
│   │   ├── db/                   SQLite + DAO
│   │   └── utils/
│   ├── data/                     SQLite 文件、测试数据
│   ├── uploads/                  运行时生成的图片（gitignore）
│   ├── tests/                    pytest
│   ├── requirements.txt
│   └── .env.example
│
├── model/                AI 推理服务
│   ├── app/
│   │   ├── server.py             FastAPI 入口（:8001）
│   │   ├── inference.py          推理逻辑
│   │   ├── config.py
│   │   └── schemas.py
│   ├── weights/                  模型权重（gitignore，大文件）
│   ├── data/                     训练/验证集
│   ├── train.py                  训练入口
│   ├── eval.py                   评估脚本
│   ├── export.py                 导出 ONNX
│   ├── requirements.txt
│   └── README.md
│
├── docs/                 文档
├── 项目2.pdf             原始需求
├── README.md
└── .gitignore
```

---

## 5. 关键数据流

### 5.1 拍照识别（核心流程）

```
用户
  │  点击"拍照"
  ▼
Android 拍照（CameraX） ───> 临时文件 + EXIF
  │  自动 EXIF 校正
  │  缩放至长边 ≤ 1024 px
  │  multipart 上传
  ▼
Backend POST /api/v1/detect
  │  ① 校验图片（格式/大小）
  │  ② 保存到 uploads/ + 生成 200x200 缩略图
  │  ③ 转发到 Model Service
  ▼
Model Service POST /infer
  │  YOLOv8n 推理
  │  返回扁平 detections[]
  ▼
Backend 聚合 + 校验
  │  按 class 聚合 → objects[]
  │  写入 SQLite
  ▼
Android
  │  解析 JSON
  │  原图上画框 + 类别标签
  │  显示"苹果 8 个"
```

### 5.2 手动修改计数

```
用户在 Android 上 +/- 或删除框
  │  本地即时更新 UI（无需调后端）
  │  用户点"保存"
  ▼
Backend PUT /api/v1/records/{id}/detections
  │  校验 class 在白名单
  │  校验 count == boxes.length
  │  替换式更新 SQLite
  ▼
200 OK
```

### 5.3 加载历史记录

```
Android GET /api/v1/records?page=1
  ▼
Backend 查询 SQLite（按 created_at 倒序）
  ▼
返回列表（含 thumbnail_url）
  │
  ▼
Android 列表页渲染（缩略图懒加载）
  │
  └──> 点击 → GET /api/v1/records/{id} → 详情页
```

---

## 6. 关键架构决策记录（ADR）

### ADR-001: 后端与模型服务分进程部署

**决策**：后端(8000) 和模型服务(8001) 拆成两个独立 FastAPI 应用。

**理由**：
- AI 同学可以独立迭代模型，不影响后端
- 后端无需安装 torch/onnxruntime（~2GB 依赖）
- 模型服务可单独压测、调优
- 生产环境可分别扩缩容

**代价**：多一次 HTTP 调用，增加 ~10ms 网络开销。

---

### ADR-002: 坐标使用原图像素（不归一化）

**决策**：API 返回的 `x1,y1,x2,y2` 是**原图坐标系**的像素值（整数）。

**理由**：
- 前端画框时要按显示尺寸缩放，知道原图尺寸才能算 ratio
- 归一化坐标反而要前端多一步 ×width/×height
- YOLO 输出本来就是像素

---

### ADR-003: 手动修改采用替换式更新

**决策**：`PUT /records/{id}/detections` 接收**完整新状态**，后端替换存储。

**理由**：
- 前端是 single source of truth，避免双端 diff 逻辑
- 简化后端代码（无 apply_diff 逻辑）
- 网络开销可接受（一次最多 ~50 个 box）

**代价**：网络包稍大（~5KB），但无感。

---

### ADR-004: 模型服务返回扁平 detections，不做聚合

**决策**：模型服务只输出 `[{class, bbox, confidence}, ...]`，按 class 聚合由后端做。

**理由**：
- 后端可灵活做 NMS、类别白名单、异常剔除
- 模型服务保持纯无状态

---

### ADR-005: SQLite 而非 MySQL/Postgres

**决策**：MVP 用 SQLite 单文件。

**理由**：
- 零运维（无需起数据库容器）
- 单文件备份/迁移方便
- 写并发 1 个 reader，多 writer 用 WAL 模式够用

**升级路径**：记录数 > 10 万时迁移到 Postgres，SQLAlchemy 改连接串即可。

---

## 7. 部署拓扑（开发期）

```
开发机（你的笔记本）
├── :8000  Backend (uvicorn --reload)
├── :8001  Model Service (python -m app.server)
└── Android Studio 模拟器 / 真机
    └── 访问 http://10.0.2.2:8000  (模拟器看 host)
        访问 http://<局域网IP>:8000  (真机)
```

**生产期**（暂不实施，但留接口）：
- Backend: `uvicorn` + `nginx` 反代
- Model: 同机部署或独立 GPU 机器
- DB: 考虑迁 Postgres

---

## 8. 性能与稳定性目标

| 指标 | 目标 |
|---|---|
| 端到端识别耗时（不含网络） | ≤ 3 秒 |
| 模型推理耗时（YOLOv8n 1280px） | ≤ 1 秒 |
| 并发支持 | 至少 3 个前端同时拍照 |
| 后端可用性 | ≥ 99%（开发期） |
| 图片存储可用 | 100%（文件删了数据库也清） |

---

## 9. 待评审项

- [ ] 是否需要用户系统（多用户隔离） — **当前不做**
- [ ] 是否需要分享到微信 — **当前不做**
- [ ] 是否需要模型端侧推理（手机本地跑） — **当前不做**
- [ ] 是否需要视频流实时计数 — **当前不做**

---

## 10. 变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 | 2026-06-06 | 初版 |
