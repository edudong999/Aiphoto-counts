# AI 拍照快速计数小助手 (Aiphoto-counts)

> 移动端拍摄桌面物品，AI 自动识别并统计同类物体数量，支持分类计数、手动修正、历史记录。  
> 三人小组项目 · **5 天交付** · 计算机视觉方向。

---

## 团队

| 角色 | 负责人 | 职责 |
|---|---|---|
| **项目统筹 + 后端** | _(填)_ | 项目框架、API 服务、部署 |
| **Android 前端** | _(填)_ | Android App |
| **AI 模型** | _(填)_ | YOLO 训练 + 推理服务 |

> 沟通群：_(填)_ · 周会时间：每周日 _(填)_ · 代码托管：GitHub

---

## 项目简介

手机拍摄桌面物品（零件、水果、文具、硬币等），AI 自动识别并按类别统计数量，用户可手动增删，最终保存到历史记录。

详细需求见 [项目2.pdf](项目2.pdf)。

---

## 文档索引

### 共享文档（全员必读）

- [docs/architecture.md](docs/architecture.md) — 系统架构、目录结构、技术选型
- [docs/api-contract.md](docs/api-contract.md) — **三方接口契约（核心文档）**
- [docs/development-guide.md](docs/development-guide.md) — Git、PR、代码风格
- [docs/sprint-plan.md](docs/sprint-plan.md) — 8 周 Sprint 计划 + 里程碑

### 角色专属

- **后端（你）**：[docs/server/README.md](docs/server/README.md) + [docs/server/tasks.md](docs/server/tasks.md)
- **Android 组员**：[docs/android/README.md](docs/android/README.md) + [docs/android/tasks.md](docs/android/tasks.md)
- **AI 组员**：[docs/model/README.md](docs/model/README.md) + [docs/model/tasks.md](docs/model/tasks.md)

---

## 技术栈一览

| 层 | 技术 | 文档 |
|---|---|---|
| Android | Kotlin · CameraX · OkHttp · Room | [android/README.md](docs/android/README.md) |
| Backend | Python 3.10 · FastAPI · SQLite · Pillow | [server/README.md](docs/server/README.md) |
| Model | YOLOv8n (ultralytics) · PyTorch · FastAPI | [model/README.md](docs/model/README.md) |

---

## 目录结构

```
Aiphoto-counts/
├── android/              Android App 工程（Kotlin）
├── server/               后端 FastAPI 工程
├── model/                AI 推理服务（YOLOv8n + FastAPI）
├── docs/                 项目文档
│   ├── architecture.md
│   ├── api-contract.md
│   ├── development-guide.md
│   ├── sprint-plan.md
│   ├── server/           后端专属
│   ├── android/          Android 专属
│   └── model/            AI 模型专属
├── 项目2.pdf             原始需求
└── README.md             你正在看的
```

---

## 快速开始

各子工程的运行方式见对应 `README.md`。

```bash
# 克隆仓库
git clone <repo-url>
cd Aiphoto-counts

# 跑后端（需先安装 Python 3.10+）
cd server
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000

# 跑模型服务（需 GPU + PyTorch）
cd ../model
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m app.server

# 跑 Android
cd ../android
# 用 Android Studio 打开，等待 Gradle 同步
```

---

## 变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 | 2026-06-06 | 项目初始化 |
