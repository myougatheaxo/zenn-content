---
title: "Compose Firebase Storage完全ガイド — アップロード/ダウンロード/進捗表示/画像管理"
emoji: "📁"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**Compose Firebase Storage**（ファイルアップロード、ダウンロード、進捗表示、画像管理）を解説します。

---

## セットアップ

```groovy
dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.1.0"))
    implementation("com.google.firebase:firebase-storage-ktx")
}
```

---

## アップロード

```kotlin
class StorageRepository @Inject constructor() {
    private val storage = Firebase.storage.reference

    fun uploadImage(uri: Uri, path: String): Flow<UploadState> = callbackFlow {
        val ref = storage.child(path)
        val uploadTask = ref.putFile(uri)

        uploadTask.addOnProgressListener { snapshot ->
            val progress = (snapshot.bytesTransferred * 100 / snapshot.totalByteCount).toInt()
            trySend(UploadState.Progress(progress))
        }.addOnSuccessListener {
            ref.downloadUrl.addOnSuccessListener { downloadUri ->
                trySend(UploadState.Success(downloadUri.toString()))
                close()
            }
        }.addOnFailureListener { e ->
            trySend(UploadState.Error(e.message ?: "アップロード失敗"))
            close()
        }

        awaitClose { uploadTask.cancel() }
    }

    suspend fun deleteFile(path: String) {
        storage.child(path).delete().await()
    }
}

sealed class UploadState {
    data class Progress(val percent: Int) : UploadState()
    data class Success(val url: String) : UploadState()
    data class Error(val message: String) : UploadState()
}
```

---

## Compose UI

```kotlin
@Composable
fun ImageUploadScreen(viewModel: UploadViewModel = hiltViewModel()) {
    val state by viewModel.uploadState.collectAsStateWithLifecycle()
    var selectedUri by remember { mutableStateOf<Uri?>(null) }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.GetContent()
    ) { uri -> selectedUri = uri }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Button(onClick = { launcher.launch("image/*") }, modifier = Modifier.fillMaxWidth()) {
            Icon(Icons.Default.Image, null)
            Spacer(Modifier.width(8.dp))
            Text("画像を選択")
        }

        selectedUri?.let { uri ->
            Spacer(Modifier.height(16.dp))
            AsyncImage(model = uri, contentDescription = null,
                modifier = Modifier.fillMaxWidth().height(200.dp).clip(RoundedCornerShape(12.dp)),
                contentScale = ContentScale.Crop)
            Spacer(Modifier.height(8.dp))
            Button(
                onClick = { viewModel.upload(uri) },
                modifier = Modifier.fillMaxWidth(),
                enabled = state !is UploadState.Progress
            ) { Text("アップロード") }
        }

        when (val s = state) {
            is UploadState.Progress -> {
                Spacer(Modifier.height(16.dp))
                LinearProgressIndicator(progress = { s.percent / 100f },
                    modifier = Modifier.fillMaxWidth())
                Text("${s.percent}%")
            }
            is UploadState.Success -> {
                Spacer(Modifier.height(16.dp))
                Text("アップロード完了!", color = Color(0xFF4CAF50))
            }
            is UploadState.Error -> {
                Spacer(Modifier.height(16.dp))
                Text(s.message, color = MaterialTheme.colorScheme.error)
            }
            else -> {}
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `putFile` | ファイルアップロード |
| `downloadUrl` | ダウンロードURL取得 |
| `addOnProgressListener` | 進捗監視 |
| `delete` | ファイル削除 |

- Firebase Storageでクラウドにファイルをアップロード
- `addOnProgressListener`で進捗をリアルタイム表示
- `downloadUrl`でダウンロードURLを取得
- セキュリティルールでアクセス制御

---

8種類のAndroidアプリテンプレート（Firebase対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose FirebaseFirestore](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-firestore-2026)
- [Compose FirebaseAuth](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-auth-2026)
- [Compose Coil3](https://zenn.dev/myougatheaxo/articles/android-compose-compose-coil3-2026)
