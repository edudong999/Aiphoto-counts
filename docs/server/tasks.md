# 后端任务清单（5 天版）

> 负责人: _（你）_  
> 配套: [README.md](README.md) · [api-contract.md](../api-contract.md) · [sprint-plan.md](../sprint-plan.md)

---

## D1 — 启动 + 骨架 + Mock 联调（6h）

- [ ] FastAPI 骨架：`app/main.py` + `app/core/{config,logging}.py`
- [ ] `/health` 接口（30min）
- [ ] `model_client.py` **写 mock 模式**（返回假 detections）
- [ ] `/api/v1/detect`（接 mock，写库，返回假数据）
- [ ] `/api/v1/classes` 返回 12 类清单
- [ ] `image_service.py`：保存原图
- [ ] SQLite + `records`/`detections` 表
- [ ] 联调自测：curl /health / /classes / /detect 都通
- [ ] 把 `http://<你的IP>:8000` 发给 Android 同学

**D1 末交付**：Android 拿到你的 mock 数据 ✅

---

## D2 — 接真模型 + 端到端 ⭐（8h）

> D2 是最关键的一天。

- [ ] `model_client.py` 去掉 mock，调 `http://localhost:8001/infer`
- [ ] `aggregate_by_class()`：扁平 detections → 按 class 聚合
- [ ] 数据校验：图片格式 / size / `count == len(boxes)` / class 在白名单
- [ ] 错误码映射：
  - 模型挂 → 5001
  - 推理超时 → 5002
  - 响应异常 → 5003
- [ ] `summary` 字段生成（"苹果 8 个、笔 12 支"——虽然 5 天版没笔）
- [ ] 联调：与 AI 同学配合，看到真实检测结果
- [ ] 日志：每张图记录 `inference_time_ms`

**D2 末交付**：拍照 → 真模型 → 显示带框结果 ✅

**风险**：如果 AI 同学 /infer 没准备好 → 你用 mock 推进，**不等他**。

---

## D3 — CRUD + 集成（8h）

- [ ] `GET /api/v1/records` 列表（分页 + 搜索）
- [ ] `GET /api/v1/records/{id}` 详情
- [ ] `PUT /api/v1/records/{id}/detections` 手动修改（替换式）
- [ ] `DELETE /api/v1/records/{id}` 删记录（同步删文件）
- [ ] `GET /api/v1/images/{filename}` 静态图片服务
- [ ] SQLite WAL 模式（防并发锁）
- [ ] 集成测试 1 个 smoke test（detect → records → delete）

**D3 末交付**：完整流程跑通 ✅
- 拍照 → 识别 → 手动改 → 保存 → 历史 → 删除

---

## D4 — 联调 + 优化 + 录 demo（4h）

- [ ] 与 Android 联调 4h
- [ ] 修联调 bug
- [ ] 错误码自测：格式错 / 超大 / 模型挂 / 记录不存在
- [ ] 3 并发手动压测（用 curl `&` 并发调 /detect）
- [ ] 写 `server/README.md`：启动方式 + 接口示例
- [ ] 写 `docs/known-issues.md`

**D4 末交付**：无 P0 bug，README 完整 ✅

---

## D5 — 交付 + 答辩（2h）

- [ ] `start.sh` 一键启动（venv + 启动两个服务）
- [ ] Swagger 截图（3 张关键接口）
- [ ] 部署 README
- [ ] 答辩 PPT 制作（你负责：架构 5 页 + 后端 5 页）
- [ ] 答辩演练 3 遍

**D5 末交付物**：
- `server/` 完整工程
- `start.sh`
- Swagger 截图

---

## 关键提醒

| # | 提醒 |
|---|---|
| 1 | **D2 端到端跑通**是整个项目最关键的卡点，必须完成 |
| 2 | **不要等 AI 同学**：他的模型没好，你就用 mock 推 |
| 3 | **不要等 Android 同学**：用 curl 自测自己的接口 |
| 4 | D3 开始每天提交 PR，让另两人 review |
| 5 | 任何接口变更先在群里说，再改代码 |

---

## 自测 cheat sheet

```bash
# 启动
uvicorn app.main:app --host 0.0.0.0 --port 8000

# 测 mock 模式（如果 AI 没好）
USE_MOCK_MODEL=1 uvicorn app.main:app --port 8000

# 健康
curl http://localhost:8000/health

# 检测（用真模型）
curl -X POST http://localhost:8000/api/v1/detect \
  -F "image=@testdata/sample.jpg" -F "source=camera"

# 列表
curl "http://localhost:8000/api/v1/records?page=1"

# 测错误：模型挂了
# 先杀模型服务，再调 /detect → 应返回 5001
```

---

## 变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 | 2026-06-06 | 5 天版（从 8 周压缩） |
