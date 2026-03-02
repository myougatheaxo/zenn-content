---
title: "Retrofit Download完全ガイド — ファイルダウンロード/進捗表示/レジューム"
emoji: "⬇️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "retrofit"]
published: true
---

## この記事で学べること

**Retrofit Download**（ファイルダウンロード、進捗通知、レジュームダウンロード、バックグラウンド実行）を解説します。

---

## ダウンロードAPI

```kotlin
interface DownloadApi {
    @Streaming // 大きなファイルをメモリに読み込まない
    @GET
    suspend fun downloadFile(@Url url: String): Response<ResponseBody>

    @Streaming
    @GET
    suspend fun downloadFileWithRange(
        @Url url: String,
        @Header("Range") range: String // レジューム用
    ): Response<ResponseBody>
}
```

---

## ダウンロード処理

```kotlin
class FileDownloader @Inject constructor(
    private val api: DownloadApi,
    @ApplicationContext private val context: Context
) {
    fun download(url: String, fileName: String): Flow<DownloadState> = flow {
        emit(DownloadState.Downloading(0))
        val response = api.downloadFile(url)
        val body = response.body() ?: throw IOException("Empty body")
        val totalBytes = body.contentLength()
        val file = File(context.getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS), fileName)

        body.byteStream().use { input ->
            file.outputStream().use { output ->
                val buffer = ByteArray(8192)
                var downloaded = 0L
                var read: Int
                while (input.read(buffer).also { read = it } != -1) {
                    output.write(buffer, 0, read)
                    downloaded += read
                    val progress = if (totalBytes > 0) (downloaded * 100 / totalBytes).toInt() else 0
                    emit(DownloadState.Downloading(progress))
                }
            }
        }
        emit(DownloadState.Success(file))
    }.catch { e ->
        emit(DownloadState.Error(e.message ?: "ダウンロード失敗"))
    }.flowOn(Dispatchers.IO)
}

sealed class DownloadState {
    data class Downloading(val progress: Int) : DownloadState()
    data class Success(val file: File) : DownloadState()
    data class Error(val message: String) : DownloadState()
}
```

---

## Compose UI

```kotlin
@Composable
fun DownloadScreen(viewModel: DownloadViewModel = hiltViewModel()) {
    val state by viewModel.downloadState.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        when (val s = state) {
            is DownloadState.Downloading -> {
                Text("ダウンロード中: ${s.progress}%")
                LinearProgressIndicator(
                    progress = { s.progress / 100f },
                    modifier = Modifier.fillMaxWidth()
                )
            }
            is DownloadState.Success -> {
                Icon(Icons.Default.CheckCircle, null, Modifier.size(48.dp), tint = Color(0xFF4CAF50))
                Text("完了: ${s.file.name}")
                Text("サイズ: ${s.file.length() / 1024}KB")
            }
            is DownloadState.Error -> {
                Icon(Icons.Default.Error, null, Modifier.size(48.dp), tint = Color.Red)
                Text(s.message)
                Button(onClick = { viewModel.retry() }) { Text("再試行") }
            }
            null -> {
                Button(onClick = { viewModel.startDownload() }) { Text("ダウンロード開始") }
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `@Streaming` | ストリーミングレスポンス |
| `ResponseBody` | バイナリレスポンス |
| `byteStream()` | 入力ストリーム |
| `Range` ヘッダー | レジューム対応 |

- `@Streaming`で大きなファイルをメモリに読み込まない
- `Flow`で進捗をリアルタイム通知
- `Dispatchers.IO`でバックグラウンド実行
- `Range`ヘッダーでレジュームダウンロード対応

---

8種類のAndroidアプリテンプレート（Retrofit対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose DownloadManager](https://zenn.dev/myougatheaxo/articles/android-compose-compose-download-manager-2026)
- [Retrofit Cache](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-cache-2026)
- [WorkManager Basic](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-basic-2026)
