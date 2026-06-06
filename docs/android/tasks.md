# Android 任务清单（5 天版）

> 负责人: _（Android 组员）_  
> 配套: [README.md](README.md) · [api-contract.md](../api-contract.md) · [sprint-plan.md](../sprint-plan.md)

---

## D1 — 启动 + 骨架 + Mock 联调（6h）

- [ ] Android Studio 新建工程，包名 `com.aiphoto.counts`
- [ ] 配 `build.gradle.kts` 依赖（CameraX / Retrofit / OkHttp / Coil / Coroutines）
- [ ] 3 个空 Fragment：拍照 / 结果 / 历史
- [ ] 底部导航 Tab 切换
- [ ] CameraX 集成：预览 + 拍照
- [ ] **EXIF 校正**（必须做，否则拍出来的图方向错）
- [ ] Retrofit 接后端 `/api/v1/detect`（接 mock 数据）
- [ ] 结果页用假数据画框（红框 + "苹果 8 个"）

**D1 末交付**：拍照 → 后端 mock → 显示假结果 ✅

---

## D2 — 接真模型 + 框选（8h）

- [ ] 切到真后端（去掉 mock）
- [ ] `BoundingBoxView` 自定义 View：
  - 按 class 区分框颜色
  - 显示 `class_zh` 文字
- [ ] 类别 + emoji 汇总展示
- [ ] loading 状态
- [ ] 错误 Toast 处理
- [ ] 与后端联调 2h，看到真实识别结果

**D2 末交付**：拍照 → 真模型 → 显示真实框选结果 ✅

---

## D3 — 历史 + 手动修改（8h）

- [ ] 结果页 `+` `-` 按钮调整计数
- [ ] 长按框 → 弹确认 → 删除
- [ ] "保存修改"按钮 → 调 PUT `/records/{id}/detections`
- [ ] "重新拍照"按钮 → 回到拍照页
- [ ] 历史记录页：RecyclerView + Coil 加载缩略图
- [ ] 点击历史项 → 详情页
- [ ] 长按历史项 → 删除（DELETE）
- [ ] 实现 DTO（Moshi）+ Repository

**D3 末交付**：完整流程跑通 ✅

---

## D4 — 联调 + 录 demo（4h）

- [ ] 与后端联调 4h，修 bug
- [ ] 录 5 分钟 demo 视频：
  - 启动 App
  - 拍桌面（多种物体）
  - 显示结果 + 框选
  - 手动 +/- 调整
  - 保存
  - 看历史
- [ ] 截图 10 张（用于答辩 PPT）
- [ ] 打包 debug APK

**D4 末交付**：APK + demo 视频 ✅

---

## D5 — 交付 + 答辩（2h）

- [ ] 写用户视角 README
- [ ] 答辩 PPT 制作（你负责：UI 流程 5 页 + 核心代码 5 页）
- [ ] 答辩演练 3 遍

**D5 末交付物**：
- `aiphoto-counts.apk`
- demo.mp4
- 10 张截图

---

## 关键提醒

| # | 提醒 |
|---|---|
| 1 | **EXIF 校正是 D1 必修**，否则方向错乱（这是 D1 末最容易翻车的点） |
| 2 | **不要等后端**：用 `MockApiService` 推进 |
| 3 | **不要等 AI 模型**：先用 mock 显示假数据 |
| 4 | 真机测试，模拟器可能 EXIF 表现不一致 |
| 5 | 历史页建议用下拉刷新（`SwipeRefreshLayout`） |

---

## 自测 cheat sheet

```kotlin
// 在 BuildConfig 里配后端地址
// 模拟器：buildConfigField "String", "API_BASE_URL", "\"http://10.0.2.2:8000/\""
// 真机：buildConfigField "String", "API_BASE_URL", "\"http://192.168.x.x:8000/\""

// Logcat 关键 tag
adb logcat -s DetectRepo:V BoundingBoxView:V
```

---

## 变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 | 2026-06-06 | 5 天版（从 8 周压缩） |
