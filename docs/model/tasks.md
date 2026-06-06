# AI 模型任务清单（5 天版）

> 负责人: _（AI 组员）_  
> 配套: [README.md](README.md) · [api-contract.md](../api-contract.md) · [sprint-plan.md](../sprint-plan.md)

---

## D1 — 启动 + 骨架 + Baseline（6h）

- [ ] 装 PyTorch (GPU 版) + ultralytics
- [ ] 下载 `yolov8n.pt` 到 `weights/`
- [ ] `app/server.py` FastAPI 骨架
- [ ] `/health` + `/model/info` 接口
- [ ] `/infer` 接口（先接 YOLOv8n 不做过滤）
- [ ] `class_map.py` 定义 **12 类**（不含 pen/coin）
- [ ] 测试集：准备 10 张桌面图
- [ ] COCO 12 类 baseline 跑通（看哪些类能识别）

**D1 末交付**：`/infer` 返回检测结果（不过滤），后端能调通 ✅

---

## D2 — 接真模型 + 性能 + MODEL_CARD（8h）

> D2 是最关键的一天。

- [ ] 类别过滤：只输出 12 类（COCO 原生：apple/orange/banana/bottle/cup/bowl/book/scissors/cell phone/keyboard/mouse/clock）
- [ ] 性能：单图推理 < 1s（GPU）
- [ ] 边界测试：遮挡 / 密集堆叠 / 反光（2h）
- [ ] 配合后端联调 2h，看真实结果
- [ ] 写 `MODEL_CARD.md` 草稿

**D2 末交付**：
- 类别准确，12 类都能识别
- 单图 < 1s
- MODEL_CARD 草稿 ✅

**风险**：如果某些类识别差（比如 cup/bowl 容易混），调 `confidence` 阈值。

---

## D3 — 性能 + 错误处理 + Demo 素材（8h）

- [ ] 性能再优化（imgsz=640 / batch=1 / FP16）
- [ ] 错误处理：
  - 输入非 jpg/png → 400
  - 模型未加载 → 503
  - 推理超时 → 504
- [ ] 准备 10 张 demo 测试图（不同物体、不同难度）
- [ ] 输出 `eval/metrics.json`（mAP / 各类 P/R）
- [ ] 配合联调 2h

**D3 末交付**：服务稳定 + 评估报告 + demo 图 ✅

---

## D4 — 联调 + 录 demo + 归档（4h）

- [ ] 与后端联调 4h
- [ ] 录 demo 视频的"识别效果"片段（重点展示 12 类都能识别）
- [ ] 模型权重备份到 `weights/yolov8n_v1.pt`
- [ ] 写完整的 `MODEL_CARD.md`

**D4 末交付**：MODEL_CARD + 权重备份 + demo 视频素材 ✅

---

## D5 — 交付 + 答辩（2h）

- [ ] 模型权重 + 训练代码 + 标注数据打包
- [ ] 答辩 PPT 制作（你负责：模型选型 3 页 + 训练流程 3 页 + 效果展示 4 页）
- [ ] 答辩演练 3 遍

**D5 末交付物**：
- `model/` 完整工程
- `weights/yolov8n_v1.pt`
- `MODEL_CARD.md`
- `eval/metrics.json`

---

## ⚠️ 关键决策（已锁）

**5 天版不训练 pen/coin**，只支持 COCO 12 类：

```python
CLASSES = [
    "apple",        # 苹果
    "orange",       # 橙子
    "banana",       # 香蕉
    "bottle",       # 瓶子
    "cup",          # 杯子
    "bowl",         # 碗
    "book",         # 书
    "scissors",     # 剪刀
    "cell phone",   # 手机
    "keyboard",     # 键盘
    "mouse",        # 鼠标
    "clock",        # 时钟
]
```

API 契约里的 14 类 → **改成 12 类**（已确认）。

---

## 关键提醒

| # | 提醒 |
|---|---|
| 1 | **D2 类别过滤是必修**，否则后端会被乱七八糟的类淹没 |
| 2 | **D2 性能 < 1s 是硬指标**，否则端到端太慢 |
| 3 | **不要等后端**：用 curl 测自己的 `/infer` |
| 4 | **不要等 Android**：单独跑通就行 |
| 5 | MODEL_CARD 不要拖到 D5 写，至少 D2 末出草稿 |

---

## 自测 cheat sheet

```bash
# 启动模型服务
cd model
python -m app.server

# 测健康
curl http://localhost:8001/health
# 期望：{"status":"ok","model_loaded":true,...}

# 测推理
curl -X POST http://localhost:8001/infer \
  -F "image=@testdata/sample.jpg" -F "confidence=0.25"
# 期望：data.detections[] 至少 1 个

# 看模型信息
curl http://localhost:8001/model/info
```

---

## 变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 | 2026-06-06 | 5 天版（从 8 周压缩，砍 pen/coin） |
