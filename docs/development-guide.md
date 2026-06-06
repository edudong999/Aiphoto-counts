# 开发规范

> 版本: v1.0 · 适用于 Android / Python / AI 三个子工程

---

## 1. Git 工作流

### 1.1 分支策略（GitHub Flow 简化版）

```
main (稳定)
  │
  ├── feat/xxx       新功能
  ├── fix/xxx        修 bug
  ├── docs/xxx       文档
  └── refactor/xxx   重构
```

**规则：**
- `main` 永远保持**可运行**，禁止直推
- 每个新功能/修复开新分支，命名 `feat/xxx` 或 `fix/xxx`（英文小写 + 短横线）
- 分支开发完提 PR，**至少 1 人 review** 才能 merge
- 合并方式：**Squash and merge**（保持 main 历史干净）

### 1.2 分支命名示例

```bash
feat/android-camera-capture      # Android 拍照功能
feat/server-detect-api           # 后端 /detect 接口
fix/model-pen-class-miss         # AI 模型笔类漏检
docs/update-api-contract         # 接口文档更新
```

### 1.3 Commit 信息规范（Conventional Commits）

```
<type>(<scope>): <subject>

<body 可选>

<footer 可选>
```

**type：**
- `feat` 新功能
- `fix` 修 bug
- `docs` 文档
- `style` 格式（不影响代码运行）
- `refactor` 重构
- `test` 测试
- `chore` 杂项（依赖、构建）

**scope 建议**：`android` / `server` / `model` / `docs`

**示例：**
```
feat(server): 实现 POST /api/v1/detect 接口
fix(android): 修复 EXIF 旋转后图片方向错误
docs(api): 新增 source 字段，区分相机/相册来源
```

---

## 2. PR 流程

### 2.1 PR Title

同 commit subject，例如：`feat(server): 实现 POST /api/v1/detect 接口`

### 2.2 PR 描述模板

```markdown
## 改了什么
- 实现了 X 接口
- 重构了 Y 模块

## 关联 Issue / 任务
- closes #12
- 对应 sprint-plan W3

## 测试
- [ ] 单元测试通过
- [ ] 本地手动跑过核心路径
- [ ] 截图（UI 改动）

## Checklist
- [ ] 代码风格符合规范
- [ ] 没有引入新的 lint warning
- [ ] 接口有变更则同步更新了 docs/api-contract.md
```

### 2.3 Review 要点

**Reviewer 看什么：**
- 是否符合接口契约（参考 `docs/api-contract.md`）
- 是否有边界处理（空值、超时、异常）
- 是否有日志/错误信息
- 是否有明显性能问题（如 N+1 查询）

**被 review 时：**
- 不要带情绪，技术讨论
- 不清楚就问，别瞎改

---

## 3. 代码风格

### 3.1 Python（后端 + 模型服务）

- **格式化**：`black` + `isort`
- **Lint**：`ruff`（或 flake8）
- **类型提示**：必须写（FastAPI 依赖）
- **docstring**：公开函数写一行 docstring；复杂函数写多行
- **命名**：`snake_case` 变量/函数，`PascalCase` 类
- **行宽**：100 字符
- **导入顺序**：标准库 → 第三方 → 本地（isort 自动）

**示例：**
```python
from typing import Optional
from fastapi import UploadFile

async def detect_objects(
    image: UploadFile,
    confidence: float = 0.25,
    source: str = "unknown",
) -> dict:
    """调用模型服务，返回检测结果。"""
    if not 0.05 <= confidence <= 0.95:
        raise ValueError("confidence 越界")
    return {"objects": [...]}
```

**关键依赖锁定：**
```txt
# requirements.txt 必须锁版本
fastapi==0.115.0
uvicorn[standard]==0.30.6
pillow==10.4.0
httpx==0.27.2
```

### 3.2 Kotlin（Android）

- **格式化**：Android Studio 默认即可
- **Lint**：Android Studio 自带
- **命名**：
  - 类 `PascalCase`
  - 变量/函数 `camelCase`
  - 常量 `SCREAMING_SNAKE_CASE`
- **空安全**：避免 `!!` 强制解包
- **协程**：网络/IO 用 `suspend` + `Dispatchers.IO`

**示例：**
```kotlin
class DetectRepository(
    private val api: DetectApi,
    private val db: RecordDao,
) {
    suspend fun detect(image: File, source: String): DetectResult {
        return withContext(Dispatchers.IO) {
            api.detect(image, source)
        }
    }
}
```

---

## 4. 错误处理规范

### 4.1 后端

- **绝不**返回 200 + 业务错误码再嵌一层（HTTP 状态码要用对）
- 业务错误抛 `HTTPException(status_code=..., detail=...)`
- 异常必须有 `logger.exception()` 记录
- 用户可见错误用通用消息（不暴露内部细节）

### 4.2 Android

- 网络错误 → 弹 Toast + 提供"重试"
- 业务错误（code != 0）→ 弹 `message`
- 未捕获异常 → Crash 上报到日志（开发期可打印 stacktrace）

### 4.3 模型服务

- 模型未加载 → 503 + "model_not_loaded"
- 推理超时 → 504 + "inference_timeout"
- 输入非法 → 400 + 详情

---

## 5. 日志规范

### 5.1 后端

```python
import logging
logger = logging.getLogger(__name__)

# 关键操作打 INFO
logger.info(f"detect image_id={image_id} source={source} objects={count}")

# 异常打 ERROR
logger.exception("model inference failed")

# 调试用 DEBUG（生产关闭）
logger.debug(f"raw detections: {detections}")
```

### 5.2 Android

```kotlin
// 开发期：Log.d / Log.e
// 发布期：替换为 Crashlytics 或自建上报
```

---

## 6. 接口变更流程

**任何一方修改 `docs/api-contract.md` 必须：**

1. 在群里发："⚠️ 接口变更：[简述]，请在 [日期] 前 review"
2. 改完合 main 后在群里说："✅ 接口已更新，新版字段：xxx"
3. 其他两方在 1 周内必须适配

**强制同步**：
- 后端加了字段 → 前端要兼容（旧字段不能删，先 deprecated）
- 后端删了字段 → 必须等前端发完版才能删

---

## 7. 测试要求

### 7.1 后端

- 核心接口必须有**集成测试**（用 `TestClient`）
- 测试覆盖率 ≥ 60%（不用追求 100%）
- 关键路径：`/detect` 成功/失败/超时 三种情况

```python
def test_detect_success(client, sample_image):
    response = client.post("/api/v1/detect", files={"image": sample_image})
    assert response.status_code == 200
    assert response.json()["code"] == 0
```

### 7.2 Android

- ViewModel / Repository 写**单元测试**（JUnit + MockK）
- UI 测试本期可不写（时间紧）

### 7.3 模型

- 推理服务：至少 1 个 smoke test（输入图 → 返回正确 schema）
- 模型精度：记录在 `model/eval/metrics.json`

---

## 8. 文件提交前 checklist

每次 `git commit` 前自检：

- [ ] 代码已格式化（black / Android Studio format）
- [ ] 没有遗留的 `print()` / `TODO` / 调试代码
- [ ] 没有把密钥、token 写进文件
- [ ] 没有提交大文件（模型权重 .pt / .onnx / dataset）
- [ ] commit message 符合规范
- [ ] 涉及接口改动的话，`docs/api-contract.md` 已更新

---

## 9. 每日站会模板（微信群）

每天上午 10 点前发一条：

```
【日期】姓名
✅ 昨天：xxx
🚧 今天：xxx
🚨 阻塞：xxx（如无就写"无"）
```

不超过 5 行，方便扫读。

---

## 10. 每周日晚周会

每周日 20:00，30 分钟，会议输出：

1. 本周 demo（5 分钟/人）
2. 下周计划（3 分钟/人）
3. 风险同步
4. 文档/Sprint 计划更新

会议纪要写到 `docs/meeting-notes/YYYY-MM-DD.md`。

---

## 11. 变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 | 2026-06-06 | 初版 |
