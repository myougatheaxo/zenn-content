---
title: "Retrofitマルチパート完全ガイド — ファイルアップロード/画像送信/進捗表示"
emoji: "📤"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "network"]
published: true
---

## この記事で学べること

**Retrofitマルチパート**（ファイルアップロード、画像送信、進捗表示、複数ファイル）を解説します。

---

## API定義

```kotlin
interface UploadApi {
    @Multipart
    @POST("upload")
    suspend fun uploadFile(
        @Part file: MultipartBody.Part,
        @Part("description") description: RequestBody
    ): Response<UploadResponse>

    @Multipart
    @POST("upload/multiple")
    suspend fun uploadMultipleFiles(
        @Part files: List<MultipartBody.Part>,
        @Part("title") title: RequestBody
    ): Response<UploadResponse>

    @Multipart
    @POST("profile/avatar")
    suspend fun uploadAvatar(
        @Part image: MultipartBody.Part
    ): Response<AvatarResponse>
}
```

---

## アップロード処理

```kotlin
class UploadRepository @Inject constructor(
    private val api: UploadApi,
    @ApplicationContext private val context: Context
) {
    suspend fun uploadImage(uri: Uri, description: String): Result<UploadResponse> {
        return try {
            val contentResolver = context.contentResolver
            val mimeType = contentResolver.getType(uri) ?: "image/jpeg"
            val fileName = getFileName(uri) ?: "image.jpg"

            val inputStream = contentResolver.openInputStream(uri) ?: throw IOException("Cannot open file")
            val bytes = inputStream.readBytes()

            val requestBody = bytes.toRequestBody(mimeType.toMediaType())
            val part = MultipartBody.Part.createFormData("file", fileName, requestBody)
            val descBody = description.toRequestBody("text/plain".toMediaType())

            val response = api.uploadFile(part, descBody)
            if (response.isSuccessful) Result.success(response.body()!!)
            else Result.failure(HttpException(response))
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    private fun getFileName(uri: Uri): String? {
        val cursor = context.contentResolver.query(uri, null, null, null, null)
        return cursor?.use {
            if (it.moveToFirst()) {
                it.getString(it.getColumnIndexOrThrow(OpenableColumns.DISPLAY_NAME))
            } else null
        }
    }
}
```

---

## Compose UI

```kotlin
@Composable
fun FileUploadScreen(viewModel: UploadViewModel = hiltViewModel()) {
    val uploadState by viewModel.uploadState.collectAsStateWithLifecycle()

    val launcher = rememberLauncherForActivityResult(ActivityResultContracts.GetContent()) { uri ->
        uri?.let { viewModel.upload(it, "説明テキスト") }
    }

    Column(Modifier.fillMaxSize().padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally) {
        when (val state = uploadState) {
            is UploadState.Idle -> {
                Button(onClick = { launcher.launch("image/*") }) {
                    Icon(Icons.Default.Upload, null)
                    Spacer(Modifier.width(8.dp))
                    Text("画像をアップロード")
                }
            }
            is UploadState.Uploading -> {
                CircularProgressIndicator()
                Text("アップロード中...")
            }
            is UploadState.Success -> {
                Icon(Icons.Default.CheckCircle, null, tint = Color(0xFF4CAF50), modifier = Modifier.size(48.dp))
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

## まとめ

| 機能 | API |
|------|-----|
| マルチパート | `@Multipart` + `@Part` |
| ファイル部品 | `MultipartBody.Part` |
| テキスト部品 | `RequestBody` |
| 複数ファイル | `List<MultipartBody.Part>` |

- `@Multipart`でファイルアップロード対応API定義
- `MultipartBody.Part.createFormData`でファイル部品作成
- `ContentResolver`でURIからファイルデータ取得
- `GetContent`で画像選択→アップロード

---

8種類のAndroidアプリテンプレート（API通信対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Retrofit/OkHttp](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-okhttp-2026)
- [Retrofit Interceptor](https://zenn.dev/myougatheaxo/articles/android-compose-retrofit-interceptor-2026)
- [Firebase Storage](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-storage-2026)
