# AI 模型开发指南

> 服务端口: **8001**（仅内网）  
> 技术: Python 3.10+ · YOLOv8n (ultralytics) · PyTorch · FastAPI  
> 负责人: _（AI 组员）_

---

## 1. 你要交付什么

一个 AI 推理微服务，负责：
1. 加载 YOLOv8n 目标检测模型
2. 提供 `/infer` `/health` `/model/info` 三个 HTTP 接口
3. 输入图片 → 输出该图片中所有物体的位置、类别、置信度

完整接口定义见 [`docs/api-contract.md`](../api-contract.md) 第 3 节。

---

## 2. 目录结构

```
model/
├── app/
│   ├── server.py            # FastAPI 入口（:8001）
│   ├── inference.py         # 核心推理逻辑
│   ├── config.py            # 配置（模型路径、阈值、设备）
│   ├── schemas.py           # Pydantic
│   └── class_map.py         # 14 类清单
├── weights/                 # 模型权重（gitignore）
│   └── yolov8n_v1.pt
├── data/                    # 训练/验证集（gitignore）
│   ├── images/
│   ├── labels/
│   └── dataset.yaml
├── train.py                 # 训练入口
├── eval.py                  # 评估脚本
├── export.py                # 导出 ONNX
├── tests/
│   └── test_infer.py
├── requirements.txt
├── .env.example
├── MODEL_CARD.md            # 模型卡
└── README.md
```

---

## 3. 快速开始

### 3.1 环境准备

```bash
# 推荐 GPU（CPU 也能跑，慢 10 倍）
# NVIDIA GPU + CUDA 11.8/12.1

# Python 3.10+
cd model
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 3.2 准备模型权重

**阶段一（先用预训练）：**
```bash
# 自动下载 yolov8n.pt
wget https://github.com/ultralytics/assets/releases/download/v8.2.0/yolov8n.pt -O weights/yolov8n.pt
```

**阶段二（自定义训练后）：**
```bash
# 训练完会得到 weights/best.pt
# 复制为 yolov8n_v1.pt
cp runs/detect/train/weights/best.pt weights/yolov8n_v1.pt
```

### 3.3 启动

```bash
# 默认用预训练权重
python -m app.server

# 或指定权重
MODEL_WEIGHTS=weights/yolov8n_v1.pt python -m app.server
```

### 3.4 测试

```bash
# 健康检查
curl http://localhost:8001/health

# 推理
curl -X POST http://localhost:8001/infer -F "image=@test.jpg"

# 模型信息
curl http://localhost:8001/model/info
```

---

## 4. 关键依赖

```txt
# 推理
ultralytics==8.3.40          # YOLOv8 官方库
torch==2.4.1                 # PyTorch
torchvision==0.19.1

# 服务
fastapi==0.115.0
uvicorn[standard]==0.30.6
python-multipart==0.0.12

# 工具
pillow==10.4.0
numpy==1.26.4
pydantic==2.9.2
python-dotenv==1.0.1

# 训练（可选，CPU 训练很慢）
# torch 和 torchvision 已经包含训练所需

# 测试
pytest==8.3.3
```

**安装提示：**
- 有 GPU：`pip install torch==2.4.1+cu118 --index-url https://download.pytorch.org/whl/cu118`
- 纯 CPU：`pip install torch==2.4.1`（约 200MB）

---

## 5. 核心代码

### 5.1 配置（app/config.py）

```python
from pydantic_settings import BaseSettings
from pathlib import Path

class Settings(BaseSettings):
    # 服务
    host: str = "0.0.0.0"
    port: int = 8001
    
    # 模型
    model_weights: str = "weights/yolov8n.pt"
    confidence_threshold: float = 0.25
    iou_threshold: float = 0.45
    device: str = "auto"           # auto / cpu / cuda:0
    img_size: int = 640
    
    # 后端
    allowed_origins: list[str] = ["*"]
    
    class Config:
        env_file = ".env"

settings = Settings()
```

### 5.2 类别映射（app/class_map.py）

**这是必须三方对齐的清单：**

```python
# 14 类清单
CLASS_MAP = {
    # class: class_zh
    "apple":      "苹果",
    "orange":     "橙子",
    "banana":     "香蕉",
    "bottle":     "瓶子",
    "cup":        "杯子",
    "bowl":       "碗",
    "book":       "书",
    "scissors":   "剪刀",
    "cell phone": "手机",
    "keyboard":   "键盘",
    "mouse":      "鼠标",
    "clock":      "时钟",
    "pen":        "笔",     # ⚠️ 自定义
    "coin":       "硬币",   # ⚠️ 自定义
}

# COCO 80 类中我们需要的 12 个
COCO_CLASSES_USED = [
    "apple", "orange", "banana", "bottle", "cup", "bowl",
    "book", "scissors", "cell phone", "keyboard", "mouse", "clock",
]

CUSTOM_CLASSES = ["pen", "coin"]
```

### 5.3 推理（app/inference.py）

```python
from ultralytics import YOLO
from app.config import settings
from app.class_map import CLASS_MAP, COCO_CLASSES_USED, CUSTOM_CLASSES
import time

class Detector:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._load()
        return cls._instance
    
    def _load(self):
        self.model = YOLO(settings.model_weights)
        self.model.to(settings.device)
        # 预热（第一次推理很慢）
        self.model.predict(source=None, imgsz=settings.img_size, verbose=False)
        print(f"Model loaded: {settings.model_weights}")
    
    def infer(self, image_bytes: bytes, confidence: float = None) -> dict:
        from PIL import Image
        import io
        import numpy as np
        
        conf = confidence or settings.confidence_threshold
        image = Image.open(io.BytesIO(image_bytes)).convert("RGB")
        width, height = image.size
        img_array = np.array(image)
        
        t0 = time.time()
        results = self.model.predict(
            source=img_array,
            conf=conf,
            iou=settings.iou_threshold,
            imgsz=settings.img_size,
            verbose=False,
        )
        elapsed_ms = int((time.time() - t0) * 1000)
        
        detections = []
        for r in results:
            for box in r.boxes:
                cls_id = int(box.cls[0])
                cls_name = self.model.names[cls_id]
                if cls_name not in CLASS_MAP:
                    continue  # 过滤不在 14 类清单中的
                xyxy = box.xyxy[0].tolist()
                detections.append({
                    "class": cls_name,
                    "class_zh": CLASS_MAP[cls_name],
                    "confidence": float(box.conf[0]),
                    "bbox": {
                        "x1": int(xyxy[0]),
                        "y1": int(xyxy[1]),
                        "x2": int(xyxy[2]),
                        "y2": int(xyxy[3]),
                    },
                })
        
        return {
            "width": width,
            "height": height,
            "detections": detections,
            "inference_time_ms": elapsed_ms,
        }

detector = Detector()
```

### 5.4 服务（app/server.py）

```python
from fastapi import FastAPI, File, Form, UploadFile, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from app.inference import detector
from app.config import settings
from app.schemas import InferResponse, HealthResponse, ModelInfo
import io

app = FastAPI(title="AI Counting Service", version="1.0.0")
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,
    allow_methods=["*"], allow_headers=["*"],
)

@app.get("/health", response_model=HealthResponse)
async def health():
    return {
        "status": "ok",
        "model_loaded": True,
        "model_name": "yolov8n",
        "model_version": "v1.0.0",
        "classes_count": len(CLASS_MAP),
        "device": str(detector.model.device),
    }

@app.get("/model/info", response_model=ModelInfo)
async def model_info():
    return {
        "model_name": "yolov8n",
        "version": "v1.0.0",
        "input_size": settings.img_size,
        "classes": [{"class": k, "class_zh": v} for k, v in CLASS_MAP.items()],
    }

@app.post("/infer")
async def infer(
    image: UploadFile = File(...),
    confidence: float = Form(0.25),
):
    if image.content_type not in ("image/jpeg", "image/png", "image/webp"):
        raise HTTPException(400, detail="仅支持 jpg/png/webp")
    
    image_bytes = await image.read()
    result = detector.infer(image_bytes, confidence)
    return {"code": 0, "message": "success", "data": result}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run("app.server:app", host=settings.host, port=settings.port, reload=False)
```

---

## 6. 训练自定义数据集（仅 pen / coin 需要）

### 6.1 决策点（W3 评审）

- W2 末评估 COCO 预训练效果，决定是否需要自定义训练
- 数据不够 → **砍掉 pen/coin**（更稳妥）
- 数据够 → 走训练流程

### 6.2 准备数据

**数据来源：**
- 自拍（手机拍桌面上的笔和硬币）
- 公开数据集：Roboflow Universe 搜 "pen detection" / "coin detection"
- 爬虫：Google 图片（注意版权）

**数据量：**
- 训练集 ≥ 200 张（每类 100 张）
- 验证集 ≥ 50 张

**标注工具：**
- [Roboflow](https://roboflow.com)（推荐，在线）
- [labelImg](https://github.com/heartexlabs/labelImg)（本地）
- 格式：YOLO 格式（txt，每行 `class_id x_center y_center w h`）

### 6.3 数据集结构

```
data/
├── images/
│   ├── train/         ← 训练集图片
│   └── val/           ← 验证集图片
├── labels/
│   ├── train/         ← 训练集标注（同名 .txt）
│   └── val/           ← 验证集标注
└── dataset.yaml
```

`dataset.yaml`:
```yaml
path: ./data
train: images/train
val: images/val

names:
  0: pen
  1: coin
```

### 6.4 训练（train.py）

```python
from ultralytics import YOLO

def train():
    model = YOLO("yolov8n.pt")  # 从预训练开始
    model.train(
        data="data/dataset.yaml",
        epochs=50,
        imgsz=640,
        batch=16,
        device="0",                # GPU 0；CPU 改成 "cpu"
        project="runs/detect",
        name="train",
        patience=10,               # 早停
    )

if __name__ == "__main__":
    train()
```

```bash
python train.py
# 训练完：runs/detect/train/weights/best.pt
```

### 6.5 评估（eval.py）

```python
from ultralytics import YOLO
import json

def eval():
    model = YOLO("weights/yolov8n_v1.pt")
    metrics = model.val(data="data/dataset.yaml")
    result = {
        "mAP50": float(metrics.box.map50),
        "mAP50-95": float(metrics.box.map),
        "precision": float(metrics.box.mp),
        "recall": float(metrics.box.mr),
    }
    with open("eval/metrics.json", "w") as f:
        json.dump(result, f, indent=2)
    print(result)

if __name__ == "__main__":
    eval()
```

### 6.6 导出 ONNX（export.py）

```python
from ultralytics import YOLO

model = YOLO("weights/yolov8n_v1.pt")
model.export(format="onnx", imgsz=640, simplify=True)
# 输出：weights/yolov8n_v1.onnx
```

---

## 7. 性能优化

| 优化项 | 做法 | 预期效果 |
|---|---|---|
| 减小输入 | `imgsz=640` → `imgsz=416` | 速度 +50% |
| 量化 | `model.export(format="onnx", half=True)` | 速度 +30% |
| TensorRT | `model.export(format="engine")` | 速度 +200% |
| CPU 推理 | `device="cpu"` | 慢 10 倍但可跑 |

**本期目标**：YOLOv8n 1280px 输入 < 1 秒（GPU）。

---

## 8. 测试

```python
# tests/test_infer.py
import pytest
from app.server import app
from fastapi.testclient import TestClient

client = TestClient(app)

def test_health():
    resp = client.get("/health")
    assert resp.status_code == 200
    assert resp.json()["status"] == "ok"

def test_infer_with_image(sample_image):
    resp = client.post(
        "/infer",
        files={"image": ("test.jpg", sample_image, "image/jpeg")},
        data={"confidence": 0.25},
    )
    assert resp.status_code == 200
    data = resp.json()["data"]
    assert "detections" in data
    assert "inference_time_ms" in data
```

---

## 9. 任务清单索引

逐周任务见 [`tasks.md`](tasks.md)。

---

## 10. 变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 | 2026-06-06 | 初版 |
