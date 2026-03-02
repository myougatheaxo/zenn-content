---
title: "ファイルダウンロード/アップロード — Retrofit+Progress表示"
emoji: "📥"
type: "tech"
topics: ["android", "kotlin", "retrofit", "file"]
published: true
---

## この記事で学べること

Androidの**ファイルダウンロード/アップロード**（Retrofit、進捗表示、バックグラウンド処理）を解説します。

---

## ファイルダウンロード（Retrofit）

```kotlin
interface FileApi {
    @Streaming
    @GET
    suspend fun downloadFile(@Url url: String): Response<ResponseBody>
}

class FileRepository @Inject constructor(
    private val api: FileApi,
    @ApplicationContext private val context: Context
) {
    suspend fun downloadFile(
        url: String,
        fileName: String,
        onProgress: (Float) -> Unit
    ): File = withContext(Dispatchers.IO) {
        val response = api.downloadFile(url)
        val body = response.body() ?: throw IOException("Empty response")
        val totalBytes = body.contentLength()

        val file = File(context.cacheDir, fileName)
        body.byteStream().use { input ->
            file.outputStream().use { output ->
                val buffer = ByteArray(8192)
                var downloadedBytes = 0L
                var bytesRead: Int

                while (input.read(buffer).also { bytesRead = it } != -1) {
                    output.write(buffer, 0, bytesRead)
                    downloadedBytes += bytesRead
                    if (totalBytes > 0) {
                        onProgress(downloadedBytes.toFloat() / totalBytes)
                    }
                }
            }
        }
        file
    }
}
```

---

## ダウンロードViewModel

```kotlin
@HiltViewModel
class DownloadViewModel @Inject constructor(
    private val repository: FileRepository
) : ViewModel() {

    private val _progress = MutableStateFlow(0f)
    val progress: StateFlow<Float> = _progress.asStateFlow()

    private val _state = MutableStateFlow<DownloadState>(DownloadState.Idle)
    val state: StateFlow<DownloadState> = _state.asStateFlow()

    fun download(url: String, fileName: String) {
        viewModelScope.launch {
            _state.value = DownloadState.Downloading
            try {
                val file = repository.downloadFile(url, fileName) { progress ->
                    _progress.value = progress
                }
                _state.value = DownloadState.Success(file)
            } catch (e: Exception) {
                _state.value = DownloadState.Error(e.message ?: "Download failed")
            }
        }
    }
}

sealed interface DownloadState {
    data object Idle : DownloadState
    data object Downloading : DownloadState
    data class Success(val file: File) : DownloadState
    data class Error(val message: String) : DownloadState
}
```

---

## ダウンロードUI

```kotlin
@Composable
fun DownloadScreen(viewModel: DownloadViewModel = hiltViewModel()) {
    val progress by viewModel.progress.collectAsStateWithLifecycle()
    val state by viewModel.state.collectAsStateWithLifecycle()

    Column(
        Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        when (val s = state) {
            DownloadState.Idle -> {
                Button(onClick = {
                    viewModel.download(
                        "https://example.com/file.pdf",
                        "document.pdf"
                    )
                }) {
                    Icon(Icons.Default.Download, null)
                    Spacer(Modifier.width(8.dp))
                    Text("ダウンロード")
                }
            }
            DownloadState.Downloading -> {
                CircularProgressIndicator(progress = { progress })
                Spacer(Modifier.height(8.dp))
                Text("${(progress * 100).toInt()}%")
                Spacer(Modifier.height(4.dp))
                LinearProgressIndicator(
                    progress = { progress },
                    modifier = Modifier.fillMaxWidth()
                )
            }
            is DownloadState.Success -> {
                Icon(Icons.Default.CheckCircle, null, tint = Color(0xFF4CAF50))
                Text("ダウンロード完了: ${s.file.name}")
            }
            is DownloadState.Error -> {
                Icon(Icons.Default.Error, null, tint = MaterialTheme.colorScheme.error)
                Text(s.message)
            }
        }
    }
}
```

---

## ファイルアップロード（Multipart）

```kotlin
interface UploadApi {
    @Multipart
    @POST("upload")
    suspend fun uploadFile(
        @Part file: MultipartBody.Part,
        @Part("description") description: RequestBody
    ): Response<UploadResponse>
}

suspend fun uploadFile(context: Context, uri: Uri, description: String): UploadResponse {
    val contentResolver = context.contentResolver
    val fileName = getFileName(context, uri) ?: "file"
    val mimeType = contentResolver.getType(uri) ?: "application/octet-stream"

    val inputStream = contentResolver.openInputStream(uri)
        ?: throw IOException("Cannot open file")

    val requestBody = inputStream.readBytes().toRequestBody(mimeType.toMediaType())
    val filePart = MultipartBody.Part.createFormData("file", fileName, requestBody)
    val descriptionPart = description.toRequestBody("text/plain".toMediaType())

    val response = api.uploadFile(filePart, descriptionPart)
    if (!response.isSuccessful) throw HttpException(response)
    return response.body()!!
}

fun getFileName(context: Context, uri: Uri): String? {
    context.contentResolver.query(uri, null, null, null, null)?.use { cursor ->
        val nameIndex = cursor.getColumnIndex(OpenableColumns.DISPLAY_NAME)
        cursor.moveToFirst()
        return cursor.getString(nameIndex)
    }
    return null
}
```

---

## WorkManagerでバックグラウンドダウンロード

```kotlin
class DownloadWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val url = inputData.getString("url") ?: return Result.failure()
        val fileName = inputData.getString("fileName") ?: return Result.failure()

        return try {
            setForeground(createForegroundInfo())

            val file = downloadFile(url, fileName) { progress ->
                setProgress(workDataOf("progress" to progress))
            }

            Result.success(workDataOf("filePath" to file.absolutePath))
        } catch (e: Exception) {
            Result.retry()
        }
    }

    private fun createForegroundInfo(): ForegroundInfo {
        val notification = NotificationCompat.Builder(applicationContext, "download")
            .setSmallIcon(R.drawable.ic_download)
            .setContentTitle("ダウンロード中...")
            .setProgress(100, 0, true)
            .build()

        return ForegroundInfo(1, notification)
    }
}
```

---

## まとめ

- `@Streaming`でメモリ効率の良いダウンロード
- `byteStream()`で進捗計算しながら書き込み
- `MultipartBody.Part`でファイルアップロード
- `WorkManager` + `setForeground`でバックグラウンドダウンロード
- `CircularProgressIndicator(progress)`で進捗表示
- `contentResolver.openInputStream(uri)`でUriからストリーム取得

---

8種類のAndroidアプリテンプレート（ファイル処理実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit設定](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-setup-2026)
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-work-manager-2026)
- [画像ピッカー](https://zenn.dev/myougatheaxo/articles/android-compose-image-picker-camera-2026)
