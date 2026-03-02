---
title: "Retrofit Multipart進捗完全ガイド — アップロード進捗/RequestBody/コールバック"
emoji: "📤"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "retrofit"]
published: true
---

## この記事で学べること

**Retrofit Multipartアップロード進捗**（カスタムRequestBody、進捗コールバック、UI表示）を解説します。

---

## ProgressRequestBody

```kotlin
class ProgressRequestBody(
    private val file: File,
    private val contentType: String,
    private val onProgress: (Float) -> Unit
) : RequestBody() {
    override fun contentType() = contentType.toMediaTypeOrNull()
    override fun contentLength() = file.length()

    override fun writeTo(sink: BufferedSink) {
        val totalBytes = file.length()
        var uploadedBytes = 0L
        val buffer = ByteArray(8192)

        file.inputStream().use { input ->
            var read: Int
            while (input.read(buffer).also { read = it } != -1) {
                sink.write(buffer, 0, read)
                uploadedBytes += read
                onProgress(uploadedBytes.toFloat() / totalBytes)
            }
        }
    }
}
```

---

## APIインターフェース

```kotlin
interface FileApi {
    @Multipart
    @POST("upload")
    suspend fun uploadFile(
        @Part file: MultipartBody.Part,
        @Part("description") description: RequestBody
    ): Response<UploadResponse>
}

// Repository
class FileRepository @Inject constructor(private val api: FileApi) {
    suspend fun uploadFile(
        file: File,
        description: String,
        onProgress: (Float) -> Unit
    ): Result<UploadResponse> {
        val requestBody = ProgressRequestBody(file, "image/*", onProgress)
        val part = MultipartBody.Part.createFormData("file", file.name, requestBody)
        val descBody = description.toRequestBody("text/plain".toMediaTypeOrNull())

        return try {
            val response = api.uploadFile(part, descBody)
            if (response.isSuccessful) Result.success(response.body()!!)
            else Result.failure(Exception("Upload failed: ${response.code()}"))
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

---

## Compose UI

```kotlin
@HiltViewModel
class UploadViewModel @Inject constructor(
    private val repository: FileRepository
) : ViewModel() {
    var uploadProgress by mutableFloatStateOf(0f)
        private set
    var isUploading by mutableStateOf(false)
        private set

    fun upload(file: File, description: String) {
        viewModelScope.launch {
            isUploading = true
            repository.uploadFile(file, description) { progress ->
                uploadProgress = progress
            }.onSuccess {
                isUploading = false
            }.onFailure {
                isUploading = false
            }
        }
    }
}

@Composable
fun UploadScreen(viewModel: UploadViewModel = hiltViewModel()) {
    Column(Modifier.padding(16.dp)) {
        if (viewModel.isUploading) {
            Text("アップロード中... ${(viewModel.uploadProgress * 100).toInt()}%")
            Spacer(Modifier.height(8.dp))
            LinearProgressIndicator(
                progress = { viewModel.uploadProgress },
                modifier = Modifier.fillMaxWidth()
            )
        } else {
            Button(onClick = { /* ファイル選択+アップロード */ }) {
                Icon(Icons.Default.Upload, null)
                Spacer(Modifier.width(8.dp))
                Text("ファイルをアップロード")
            }
        }
    }
}
```

---

## まとめ

| コンポーネント | 用途 |
|--------------|------|
| `ProgressRequestBody` | 進捗付きRequestBody |
| `MultipartBody.Part` | ファイルパート |
| `onProgress` | 進捗コールバック |
| `LinearProgressIndicator` | 進捗UI |

- `RequestBody`をカスタムして書き込み進捗を取得
- `writeTo`内でバイト数をカウントして進捗計算
- ViewModelで`mutableFloatStateOf`で進捗管理
- `LinearProgressIndicator`で視覚的フィードバック

---

8種類のAndroidアプリテンプレート（ファイル操作対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit Multipart](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-multipart-2026)
- [Retrofit Interceptor](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-interceptor-2026)
- [ProgressIndicator](https://zenn.dev/myougatheaxo/articles/android-compose-compose-progress-indicator-2026)
