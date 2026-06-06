# Android 开发指南

> 目标设备: Android 8.0+ (API 26)  
> 技术: Kotlin · CameraX · OkHttp/Retrofit · Room · Coroutines  
> 负责人: _（Android 组员）_

---

## 1. 你要交付什么

一个 Android App，实现：
1. 相机拍照 / 相册选图
2. 上传到后端 `/api/v1/detect`
3. 在原图上画框 + 显示类别 + 计数
4. 手动 +/- / 删除框 / 长按修改
5. 历史记录列表 + 详情
6. 首次启动引导页
7. 本地缓存历史缩略图

完整接口定义见 [`docs/api-contract.md`](../api-contract.md)。

---

## 2. 目录结构

```
android/
├── app/
│   ├── src/main/
│   │   ├── java/com/aiphoto/counts/
│   │   │   ├── AiphotoApp.kt           # Application 类
│   │   │   ├── MainActivity.kt         # 单 Activity 容器
│   │   │   ├── ui/
│   │   │   │   ├── camera/             # 拍照页
│   │   │   │   │   ├── CameraFragment.kt
│   │   │   │   │   └── CameraViewModel.kt
│   │   │   │   ├── result/             # 结果展示页
│   │   │   │   │   ├── ResultFragment.kt
│   │   │   │   │   ├── ResultViewModel.kt
│   │   │   │   │   └── BoundingBoxView.kt   # 自定义 View 画框
│   │   │   │   ├── history/            # 历史记录页
│   │   │   │   │   ├── HistoryFragment.kt
│   │   │   │   │   ├── HistoryViewModel.kt
│   │   │   │   │   └── HistoryAdapter.kt
│   │   │   │   ├── onboarding/         # 引导页
│   │   │   │   │   ├── OnboardingFragment.kt
│   │   │   │   │   └── OnboardingAdapter.kt
│   │   │   │   └── common/
│   │   │   ├── camera/                 # CameraX 封装
│   │   │   │   ├── CameraManager.kt
│   │   │   │   └── ExifCorrector.kt
│   │   │   ├── network/
│   │   │   │   ├── ApiService.kt       # Retrofit interface
│   │   │   │   ├── NetworkModule.kt    # OkHttp + Retrofit 配置
│   │   │   │   └── dto/                # 数据传输对象
│   │   │   │       ├── DetectResponse.kt
│   │   │   │       ├── DetectRequest.kt
│   │   │   │       └── RecordDto.kt
│   │   │   ├── data/
│   │   │   │   ├── local/              # Room
│   │   │   │   │   ├── AppDatabase.kt
│   │   │   │   │   ├── RecordEntity.kt
│   │   │   │   │   ├── DetectionEntity.kt
│   │   │   │   │   └── RecordDao.kt
│   │   │   │   ├── repository/
│   │   │   │   │   ├── DetectRepository.kt
│   │   │   │   │   └── RecordRepository.kt
│   │   │   │   └── prefs/              # SharedPreferences
│   │   │   │       └── UserPrefs.kt
│   │   │   ├── model/                  # 业务模型
│   │   │   │   ├── Detection.kt
│   │   │   │   ├── DetectedObject.kt
│   │   │   │   └── Record.kt
│   │   │   └── util/
│   │   │       ├── ImageUtil.kt        # 压缩、EXIF
│   │   │       ├── Logger.kt
│   │   │       └── PermissionUtil.kt
│   │   ├── res/
│   │   │   ├── layout/                 # XML 布局
│   │   │   ├── drawable/
│   │   │   ├── values/strings.xml
│   │   │   ├── values/colors.xml
│   │   │   └── values/themes.xml
│   │   └── AndroidManifest.xml
│   ├── src/test/                       # 单元测试
│   └── build.gradle.kts
├── build.gradle.kts                    # 顶层
├── settings.gradle.kts
├── gradle.properties
└── README.md
```

---

## 3. 快速开始

### 3.1 环境准备

```bash
# Android Studio Hedgehog | 2023.1.1 或更新
# Android SDK 34
# JDK 17
# 真机（推荐）或 模拟器
```

### 3.2 配置后端地址

在 `app/build.gradle.kts` 加：
```kotlin
android {
    defaultConfig {
        // ...
        buildConfigField("String", "API_BASE_URL", "\"http://10.0.2.2:8000/\"")
        // 模拟器看 host 用 10.0.2.2
        // 真机改成局域网 IP，如 "http://192.168.1.100:8000/"
    }
}
```

### 3.3 运行

```bash
# 用 Android Studio 打开 android/ 目录
# 等待 Gradle 同步
# 连接真机或启动模拟器
# 点击 Run
```

---

## 4. 关键依赖（app/build.gradle.kts）

```kotlin
dependencies {
    // AndroidX 核心
    implementation("androidx.core:core-ktx:1.13.1")
    implementation("androidx.appcompat:appcompat:1.7.0")
    implementation("com.google.android.material:material:1.12.0")
    implementation("androidx.constraintlayout:constraintlayout:2.1.4")
    
    // Fragment + Navigation
    implementation("androidx.fragment:fragment-ktx:1.8.2")
    implementation("androidx.navigation:navigation-fragment-ktx:2.7.7")
    implementation("androidx.navigation:navigation-ui-ktx:2.7.7")
    
    // Lifecycle + ViewModel
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.8.4")
    implementation("androidx.lifecycle:lifecycle-livedata-ktx:2.8.4")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.4")
    
    // CameraX
    implementation("androidx.camera:camera-core:1.3.4")
    implementation("androidx.camera:camera-camera2:1.3.4")
    implementation("androidx.camera:camera-lifecycle:1.3.4")
    implementation("androidx.camera:camera-view:1.3.4")
    
    // 网络
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.squareup.retrofit2:converter-moshi:2.11.0")
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
    implementation("com.squareup.moshi:moshi-kotlin:1.15.1")
    
    // 图片加载
    implementation("io.coil-kt:coil:2.7.0")
    
    // 本地数据库
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    annotationProcessor("androidx.room:room-compiler:2.6.1")
    
    // 协程
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")
    
    // 测试
    testImplementation("junit:junit:4.13.2")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")
    testImplementation("io.mockk:mockk:1.13.12")
}
```

---

## 5. 关键模块设计

### 5.1 网络层（network/ApiService.kt）

```kotlin
interface ApiService {
    @Multipart
    @POST("api/v1/detect")
    suspend fun detect(
        @Part image: MultipartBody.Part,
        @Part("confidence") confidence: RequestBody,
        @Part("save_record") saveRecord: RequestBody,
        @Part("source") source: RequestBody,
    ): Response<ApiResponse<DetectResult>>
    
    @GET("api/v1/classes")
    suspend fun getClasses(): Response<ApiResponse<ClassesData>>
    
    @GET("api/v1/records")
    suspend fun getRecords(
        @Query("page") page: Int = 1,
        @Query("page_size") pageSize: Int = 20,
        @Query("keyword") keyword: String? = null,
    ): Response<ApiResponse<RecordsData>>
    
    @GET("api/v1/records/{id}")
    suspend fun getRecord(@Path("id") id: String): Response<ApiResponse<RecordData>>
    
    @PUT("api/v1/records/{id}/detections")
    suspend fun updateRecord(
        @Path("id") id: String,
        @Body body: UpdateRequest,
    ): Response<ApiResponse<UpdateData>>
    
    @DELETE("api/v1/records/{id}")
    suspend fun deleteRecord(@Path("id") id: String): Response<ApiResponse<Unit>>
}
```

### 5.2 拍照（camera/CameraManager.kt）

```kotlin
class CameraManager(
    private val context: Context,
    private val lifecycleOwner: LifecycleOwner,
) {
    private var imageCapture: ImageCapture? = null
    
    fun startCamera(previewView: PreviewView) {
        val cameraProvider = ProcessCameraProvider.getInstance(context).get()
        val preview = Preview.Builder().build().also {
            it.setSurfaceProvider(previewView.surfaceProvider)
        }
        imageCapture = ImageCapture.Builder()
            .setCaptureMode(ImageCapture.CAPTURE_MODE_MINIMIZE_LATENCY)
            .build()
        
        cameraProvider.bindToLifecycle(
            lifecycleOwner,
            CameraSelector.DEFAULT_BACK_CAMERA,
            preview, imageCapture,
        )
    }
    
    suspend fun takePicture(): File = suspendCoroutine { cont ->
        val file = File(context.cacheDir, "capture_${System.currentTimeMillis()}.jpg")
        imageCapture?.takePicture(
            ImageCapture.OutputFileOptions.Builder(file).build(),
            ContextCompat.getMainExecutor(context),
            object : ImageCapture.OnImageSavedCallback {
                override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                    cont.resume(file)
                }
                override fun onError(exc: ImageCaptureException) {
                    cont.resumeWithException(exc)
                }
            }
        )
    }
}
```

### 5.3 EXIF 校正（util/ImageUtil.kt）

```kotlin
fun correctOrientation(file: File): File {
    val exif = ExifInterface(file.absolutePath)
    val rotation = exif.getAttributeInt(
        ExifInterface.TAG_ORIENTATION,
        ExifInterface.ORIENTATION_NORMAL,
    )
    if (rotation == ExifInterface.ORIENTATION_NORMAL) return file
    
    val bitmap = BitmapFactory.decodeFile(file.absolutePath)
    val matrix = Matrix()
    when (rotation) {
        ExifInterface.ORIENTATION_ROTATE_90 -> matrix.postRotate(90f)
        ExifInterface.ORIENTATION_ROTATE_180 -> matrix.postRotate(180f)
        ExifInterface.ORIENTATION_ROTATE_270 -> matrix.postRotate(270f)
    }
    val rotated = Bitmap.createBitmap(bitmap, 0, 0, bitmap.width, bitmap.height, matrix, true)
    FileOutputStream(file).use { rotated.compress(Bitmap.CompressFormat.JPEG, 90, it) }
    return file
}

fun compress(file: File, maxSide: Int = 1024): File {
    val opts = BitmapFactory.Options().apply { inJustDecodeBounds = true }
    BitmapFactory.decodeFile(file.absolutePath, opts)
    val sample = max(opts.outWidth, opts.outHeight) / maxSide
    val opts2 = BitmapFactory.Options().apply {
        inSampleSize = maxOf(1, sample)
    }
    val bmp = BitmapFactory.decodeFile(file.absolutePath, opts2)
    FileOutputStream(file).use { bmp.compress(Bitmap.CompressFormat.JPEG, 85, it) }
    return file
}
```

### 5.4 画框（ui/result/BoundingBoxView.kt）

```kotlin
class BoundingBoxView @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null,
) : View(context, attrs) {
    
    private var imageWidth = 1
    private var imageHeight = 1
    private var objects: List<DetectedObject> = emptyList()
    private var onBoxClick: ((Int, Int) -> Unit)? = null  // (objIndex, boxIndex)
    
    private val boxPaint = Paint().apply {
        color = Color.RED; style = Paint.Style.STROKE; strokeWidth = 6f
    }
    private val textPaint = Paint().apply {
        color = Color.WHITE; textSize = 36f; isAntiAlias = true
    }
    private val bgPaint = Paint().apply { color = Color.RED }
    
    fun setData(width: Int, height: Int, objects: List<DetectedObject>) {
        this.imageWidth = width
        this.imageHeight = height
        this.objects = objects
        invalidate()
    }
    
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        val scaleX = width.toFloat() / imageWidth
        val scaleY = height.toFloat() / imageHeight
        
        objects.forEachIndexed { oi, obj ->
            obj.boxes.forEachIndexed { bi, box ->
                val l = box.x1 * scaleX
                val t = box.y1 * scaleY
                val r = box.x2 * scaleX
                val b = box.y2 * scaleY
                canvas.drawRect(l, t, r, b, boxPaint)
                
                val label = "${obj.classZh} ${box.confidence.toString().take(4)}"
                val textW = textPaint.measureText(label)
                canvas.drawRect(l, t - 50f, l + textW + 20f, t, bgPaint)
                canvas.drawText(label, l + 10f, t - 10f, textPaint)
            }
        }
    }
}
```

---

## 6. 权限（AndroidManifest.xml）

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.CAMERA" />

<!-- Android 13+ -->
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<!-- Android 12 及以下 -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32" />

<uses-feature android:name="android.hardware.camera" android:required="true" />
```

**运行时申请**：拍照前用 `ActivityCompat.requestPermissions()` 申请 CAMERA；选图前申请 READ_MEDIA_IMAGES。

---

## 7. 调试技巧

### 7.1 用 mock 数据开发

如果后端还没好，临时改 `ApiService`：
```kotlin
class MockApiService : ApiService {
    override suspend fun detect(...): Response<ApiResponse<DetectResult>> {
        delay(500)  // 模拟网络
        return Response.success(ApiResponse(
            code = 0, message = "success",
            data = DetectResult(
                imageId = "mock", imageUrl = "", width = 1920, height = 1080,
                objects = listOf(
                    DetectedObject("apple", "苹果", 8, listOf(
                        Box(100, 200, 300, 400, 0.9f)
                    ))
                ),
                totalObjects = 8, inferenceTimeMs = 100,
                recordId = "mock_record", source = "camera",
            )
        ))
    }
}
```

### 7.2 用 Logcat 调试

```kotlin
class Logger private constructor() {
    companion object {
        fun d(tag: String, msg: String) {
            if (BuildConfig.DEBUG) Log.d(tag, msg)
        }
    }
}
```

---

## 8. 任务清单索引

逐周任务见 [`tasks.md`](tasks.md)。

---

## 9. 变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 | 2026-06-06 | 初版 |
