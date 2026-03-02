---
title: "ダウンロード管理完全ガイド — DownloadManager/進捗表示/バックグラウンド"
emoji: "⬇️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "network"]
published: true
---

## この記事で学べること

**ダウンロード管理**（DownloadManager、進捗監視、通知連携、OkHttpダウンロード）を解説します。

---

## DownloadManager

```kotlin
class FileDownloader @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val downloadManager = context.getSystemService(Context.DOWNLOAD_SERVICE) as DownloadManager

    fun download(url: String, fileName: String): Long {
        val request = DownloadManager.Request(Uri.parse(url))
            .setTitle(fileName)
            .setDescription("ダウンロード中...")
            .setNotificationVisibility(DownloadManager.Request.VISIBILITY_VISIBLE_NOTIFY_COMPLETED)
            .setDestinationInExternalPublicDir(Environment.DIRECTORY_DOWNLOADS, fileName)
            .setAllowedOverMetered(true)

        return downloadManager.enqueue(request)
    }

    fun getProgress(downloadId: Long): Flow<DownloadProgress> = flow {
        while (true) {
            val query = DownloadManager.Query().setFilterById(downloadId)
            val cursor = downloadManager.query(query)

            cursor?.use {
                if (it.moveToFirst()) {
                    val bytesDownloaded = it.getLong(it.getColumnIndexOrThrow(DownloadManager.COLUMN_BYTES_DOWNLOADED_SO_FAR))
                    val bytesTotal = it.getLong(it.getColumnIndexOrThrow(DownloadManager.COLUMN_TOTAL_SIZE_BYTES))
                    val status = it.getInt(it.getColumnIndexOrThrow(DownloadManager.COLUMN_STATUS))

                    emit(DownloadProgress(
                        bytesDownloaded = bytesDownloaded,
                        bytesTotal = bytesTotal,
                        progress = if (bytesTotal > 0) bytesDownloaded.toFloat() / bytesTotal else 0f,
                        status = status
                    ))

                    if (status == DownloadManager.STATUS_SUCCESSFUL || status == DownloadManager.STATUS_FAILED) {
                        return@flow
                    }
                }
            }
            delay(500)
        }
    }
}

data class DownloadProgress(
    val bytesDownloaded: Long,
    val bytesTotal: Long,
    val progress: Float,
    val status: Int
)
```

---

## Compose進捗UI

```kotlin
@Composable
fun DownloadButton(
    url: String,
    fileName: String,
    viewModel: DownloadViewModel = hiltViewModel()
) {
    val progress by viewModel.progress.collectAsStateWithLifecycle()

    when {
        progress == null -> {
            Button(onClick = { viewModel.startDownload(url, fileName) }) {
                Icon(Icons.Default.Download, null)
                Spacer(Modifier.width(8.dp))
                Text("ダウンロード")
            }
        }
        progress!!.progress < 1f -> {
            Column(horizontalAlignment = Alignment.CenterHorizontally) {
                LinearProgressIndicator(
                    progress = { progress!!.progress },
                    modifier = Modifier.fillMaxWidth()
                )
                Text("${(progress!!.progress * 100).toInt()}%")
            }
        }
        else -> {
            Row {
                Icon(Icons.Default.CheckCircle, null, tint = Color(0xFF4CAF50))
                Spacer(Modifier.width(8.dp))
                Text("ダウンロード完了")
            }
        }
    }
}
```

---

## OkHttpダウンロード（大容量）

```kotlin
class LargeFileDownloader @Inject constructor(
    private val okHttpClient: OkHttpClient,
    @ApplicationContext private val context: Context
) {
    suspend fun download(url: String, fileName: String): Flow<Float> = flow {
        val request = Request.Builder().url(url).build()
        val response = okHttpClient.newCall(request).execute()
        val body = response.body ?: throw IOException("Empty body")
        val totalBytes = body.contentLength()

        val file = File(context.getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS), fileName)
        file.outputStream().use { output ->
            val buffer = ByteArray(8192)
            var bytesRead: Int
            var totalRead = 0L

            body.byteStream().use { input ->
                while (input.read(buffer).also { bytesRead = it } != -1) {
                    output.write(buffer, 0, bytesRead)
                    totalRead += bytesRead
                    emit(totalRead.toFloat() / totalBytes)
                }
            }
        }
    }.flowOn(Dispatchers.IO)
}
```

---

## まとめ

| 手法 | 用途 |
|------|------|
| `DownloadManager` | システム管理、通知付き |
| OkHttp | カスタム制御、大容量 |
| WorkManager | バックグラウンド保証 |

- `DownloadManager`でシステム通知付きダウンロード
- `Flow`で進捗をリアクティブに監視
- OkHttpで大容量ファイルのストリーミングダウンロード
- `LinearProgressIndicator`で進捗表示

---

8種類のAndroidアプリテンプレート（ダウンロード機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit/OkHttp](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-okhttp-2026)
- [WorkManager](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-2026)
- [ファイル管理](https://zenn.dev/myougatheaxo/articles/android-compose-file-manager-2026)
