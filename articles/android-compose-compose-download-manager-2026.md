---
title: "Compose DownloadManager完全ガイド — ファイルダウンロード/進捗表示/通知"
emoji: "⬇️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "download"]
published: true
---

## この記事で学べること

**Compose DownloadManager**（ファイルダウンロード、進捗表示、ダウンロード通知、Compose統合）を解説します。

---

## 基本DownloadManager

```kotlin
fun startDownload(context: Context, url: String, fileName: String): Long {
    val request = DownloadManager.Request(url.toUri())
        .setTitle(fileName)
        .setDescription("ダウンロード中...")
        .setNotificationVisibility(DownloadManager.Request.VISIBILITY_VISIBLE_NOTIFY_COMPLETED)
        .setDestinationInExternalPublicDir(Environment.DIRECTORY_DOWNLOADS, fileName)
        .setAllowedOverMetered(true)

    val manager = context.getSystemService(Context.DOWNLOAD_SERVICE) as DownloadManager
    return manager.enqueue(request)
}
```

---

## 進捗表示

```kotlin
@Composable
fun DownloadWithProgress(url: String, fileName: String) {
    val context = LocalContext.current
    var downloadId by remember { mutableLongStateOf(-1L) }
    var progress by remember { mutableFloatStateOf(0f) }
    var isDownloading by remember { mutableStateOf(false) }

    LaunchedEffect(downloadId) {
        if (downloadId < 0) return@LaunchedEffect
        isDownloading = true
        val manager = context.getSystemService(Context.DOWNLOAD_SERVICE) as DownloadManager

        while (isDownloading) {
            val query = DownloadManager.Query().setFilterById(downloadId)
            val cursor = manager.query(query)
            if (cursor.moveToFirst()) {
                val status = cursor.getInt(cursor.getColumnIndexOrThrow(DownloadManager.COLUMN_STATUS))
                val downloaded = cursor.getLong(cursor.getColumnIndexOrThrow(DownloadManager.COLUMN_BYTES_DOWNLOADED_SO_FAR))
                val total = cursor.getLong(cursor.getColumnIndexOrThrow(DownloadManager.COLUMN_TOTAL_SIZE_BYTES))

                if (total > 0) progress = downloaded.toFloat() / total
                if (status == DownloadManager.STATUS_SUCCESSFUL || status == DownloadManager.STATUS_FAILED) {
                    isDownloading = false
                }
            }
            cursor.close()
            delay(500)
        }
    }

    Column(Modifier.padding(16.dp)) {
        if (isDownloading) {
            LinearProgressIndicator(progress = { progress }, Modifier.fillMaxWidth())
            Text("${(progress * 100).toInt()}%")
        }
        Button(onClick = { downloadId = startDownload(context, url, fileName) },
            enabled = !isDownloading, modifier = Modifier.fillMaxWidth()) {
            Text(if (isDownloading) "ダウンロード中..." else "ダウンロード")
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `DownloadManager` | システムダウンロード |
| `Request` | ダウンロード設定 |
| `query()` | 進捗確認 |
| `VISIBILITY_VISIBLE` | 通知表示 |

- `DownloadManager`でバックグラウンドダウンロード
- 通知バーに自動的にダウンロード進捗表示
- `query()`でプログラム的に進捗を取得
- Wi-Fi制限/メータード制限等の条件設定可能

---

8種類のAndroidアプリテンプレート（ファイル操作対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose FilePicker](https://zenn.dev/myougatheaxo/articles/android-compose-compose-file-picker-2026)
- [Compose Notification](https://zenn.dev/myougatheaxo/articles/android-compose-compose-notification-2026)
- [WorkManager Basic](https://zenn.dev/myougatheaxo/articles/android-compose-workmanager-basic-2026)
