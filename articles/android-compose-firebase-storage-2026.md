---
title: "Firebase Storage完全ガイド — ファイルアップロード/進捗表示/画像リサイズ"
emoji: "☁️"
type: "tech"
topics: ["android", "firebase", "kotlin", "storage"]
published: true
---

## この記事で学べること

**Firebase Storage**（ファイルアップロード、ダウンロードURL取得、進捗表示、画像リサイズ）を解説します。

---

## アップロード

```kotlin
class StorageRepository @Inject constructor() {
    private val storage = Firebase.storage.reference

    fun uploadImage(uri: Uri, path: String): Flow<UploadState> = callbackFlow {
        val ref = storage.child(path)
        val uploadTask = ref.putFile(uri)

        uploadTask.addOnProgressListener { snapshot ->
            val progress = snapshot.bytesTransferred.toFloat() / snapshot.totalByteCount
            trySend(UploadState.Progress(progress))
        }

        uploadTask.addOnSuccessListener {
            ref.downloadUrl.addOnSuccessListener { downloadUri ->
                trySend(UploadState.Success(downloadUri.toString()))
                close()
            }
        }

        uploadTask.addOnFailureListener { e ->
            trySend(UploadState.Error(e.message ?: "Upload failed"))
            close()
        }

        awaitClose { uploadTask.cancel() }
    }

    suspend fun deleteFile(path: String) {
        storage.child(path).delete().await()
    }
}

sealed class UploadState {
    data class Progress(val percent: Float) : UploadState()
    data class Success(val downloadUrl: String) : UploadState()
    data class Error(val message: String) : UploadState()
}
```

---

## Compose UI

```kotlin
@Composable
fun ImageUploadButton(viewModel: UploadViewModel = hiltViewModel()) {
    val uploadState by viewModel.uploadState.collectAsStateWithLifecycle()

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.GetContent()
    ) { uri -> uri?.let { viewModel.uploadImage(it) } }

    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        when (val state = uploadState) {
            null -> {
                Button(onClick = { launcher.launch("image/*") }) {
                    Icon(Icons.Default.Upload, null)
                    Spacer(Modifier.width(8.dp))
                    Text("画像をアップロード")
                }
            }
            is UploadState.Progress -> {
                LinearProgressIndicator(
                    progress = { state.percent },
                    modifier = Modifier.fillMaxWidth()
                )
                Text("${(state.percent * 100).toInt()}%")
            }
            is UploadState.Success -> {
                Icon(Icons.Default.CheckCircle, null, tint = Color(0xFF4CAF50))
                Text("アップロード完了")
            }
            is UploadState.Error -> {
                Text(state.message, color = MaterialTheme.colorScheme.error)
                Button(onClick = { launcher.launch("image/*") }) { Text("再試行") }
            }
        }
    }
}
```

---

## 画像リサイズ

```kotlin
suspend fun compressImage(context: Context, uri: Uri, maxSize: Int = 1024): ByteArray {
    return withContext(Dispatchers.IO) {
        val bitmap = context.contentResolver.openInputStream(uri)?.use {
            BitmapFactory.decodeStream(it)
        } ?: throw IOException("Failed to decode")

        val ratio = minOf(maxSize.toFloat() / bitmap.width, maxSize.toFloat() / bitmap.height, 1f)
        val resized = Bitmap.createScaledBitmap(
            bitmap,
            (bitmap.width * ratio).toInt(),
            (bitmap.height * ratio).toInt(),
            true
        )

        ByteArrayOutputStream().use { stream ->
            resized.compress(Bitmap.CompressFormat.JPEG, 80, stream)
            stream.toByteArray()
        }
    }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| アップロード | `putFile`, `putBytes` |
| URL取得 | `downloadUrl` |
| 進捗 | `addOnProgressListener` |
| 削除 | `delete()` |

- `callbackFlow`でアップロード進捗をリアクティブ管理
- `GetContent`で画像選択→アップロード
- アップロード前に画像リサイズでデータ節約
- `downloadUrl`で共有可能URLを取得

---

8種類のAndroidアプリテンプレート（Storage対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Auth](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-auth-2026)
- [CameraX](https://zenn.dev/myougatheaxo/articles/android-compose-camera-compose-2026)
- [ファイル管理](https://zenn.dev/myougatheaxo/articles/android-compose-file-manager-2026)
