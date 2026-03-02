---
title: "Compose FileProvider完全ガイド — ファイル共有/カメラ撮影/PDF表示/Intent連携"
emoji: "📂"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "fileprovider"]
published: true
---

## この記事で学べること

**Compose FileProvider**（ファイル共有、カメラ撮影URI、安全なファイルアクセス、Intent連携）を解説します。

---

## FileProvider設定

```xml
<!-- AndroidManifest.xml -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

```xml
<!-- res/xml/file_paths.xml -->
<paths>
    <files-path name="images" path="images/" />
    <cache-path name="cache" path="/" />
    <external-files-path name="external" path="/" />
</paths>
```

---

## カメラ撮影

```kotlin
@Composable
fun CameraCaptureScreen() {
    val context = LocalContext.current
    var imageUri by remember { mutableStateOf<Uri?>(null) }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.TakePicture()
    ) { success ->
        if (!success) imageUri = null
    }

    fun createImageUri(): Uri {
        val file = File(context.filesDir, "images").also { it.mkdirs() }
            .let { File(it, "photo_${System.currentTimeMillis()}.jpg") }
        return FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file)
    }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Button(onClick = {
            val uri = createImageUri()
            imageUri = uri
            launcher.launch(uri)
        }) { Text("写真を撮影") }

        Spacer(Modifier.height(16.dp))

        imageUri?.let { uri ->
            AsyncImage(
                model = uri,
                contentDescription = "撮影した写真",
                modifier = Modifier.fillMaxWidth().height(300.dp).clip(RoundedCornerShape(16.dp)),
                contentScale = ContentScale.Crop
            )
        }
    }
}
```

---

## ファイル共有

```kotlin
@Composable
fun ShareFileScreen() {
    val context = LocalContext.current

    fun shareFile(file: File) {
        val uri = FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file)
        val intent = Intent(Intent.ACTION_SEND).apply {
            type = "application/pdf"
            putExtra(Intent.EXTRA_STREAM, uri)
            addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        }
        context.startActivity(Intent.createChooser(intent, "共有"))
    }

    fun shareMultipleFiles(files: List<File>) {
        val uris = files.map { file ->
            FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file)
        }
        val intent = Intent(Intent.ACTION_SEND_MULTIPLE).apply {
            type = "*/*"
            putParcelableArrayListExtra(Intent.EXTRA_STREAM, ArrayList(uris))
            addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        }
        context.startActivity(Intent.createChooser(intent, "共有"))
    }

    Button(onClick = {
        val file = File(context.filesDir, "report.pdf")
        if (file.exists()) shareFile(file)
    }) { Text("PDFを共有") }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `FileProvider` | 安全なURI生成 |
| `getUriForFile` | URI取得 |
| `grantUriPermissions` | 権限付与 |
| `file_paths.xml` | パス設定 |

- `FileProvider`でfile:// URIの代わりにcontent:// URIを生成
- `file_paths.xml`で公開するディレクトリを制限
- `FLAG_GRANT_READ_URI_PERMISSION`で一時的な読み取り権限
- カメラ撮影、ファイル共有、PDF表示等に必須

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose CameraX](https://zenn.dev/myougatheaxo/articles/android-compose-compose-camerax-2026)
- [Compose Permission](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permission-2026)
- [Compose DeepLink](https://zenn.dev/myougatheaxo/articles/android-compose-compose-deep-link-2026)
