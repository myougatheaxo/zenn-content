---
title: "ファイルストレージガイド — 内部/外部/SAFの使い分け"
emoji: "📂"
type: "tech"
topics: ["android", "kotlin", "storage", "file"]
published: true
---

## この記事で学べること

Androidの**ファイルストレージ**（内部ストレージ、外部ストレージ、SAF）の使い分けを解説します。

---

## ストレージの種類

```
内部ストレージ（Internal）
  - アプリ専用、他アプリからアクセス不可
  - context.filesDir / context.cacheDir
  - アンインストール時に削除

外部ストレージ（External - Scoped）
  - アプリ専用外部領域
  - context.getExternalFilesDir()
  - パーミッション不要

共有ストレージ（MediaStore / SAF）
  - 写真・動画・ドキュメント共有
  - MediaStore API or Storage Access Framework
  - パーミッション or ユーザー選択必要
```

---

## 内部ストレージ

```kotlin
class FileRepository(private val context: Context) {

    // テキストファイルの書き込み
    fun saveText(fileName: String, content: String) {
        context.openFileOutput(fileName, Context.MODE_PRIVATE).use { stream ->
            stream.write(content.toByteArray())
        }
    }

    // テキストファイルの読み込み
    fun readText(fileName: String): String? {
        return try {
            context.openFileInput(fileName).bufferedReader().use { it.readText() }
        } catch (e: FileNotFoundException) {
            null
        }
    }

    // JSONの保存/読み込み
    fun saveJson(fileName: String, data: Any) {
        val json = Json.encodeToString(data)
        saveText(fileName, json)
    }

    inline fun <reified T> readJson(fileName: String): T? {
        val text = readText(fileName) ?: return null
        return Json.decodeFromString<T>(text)
    }

    // キャッシュファイル
    fun saveTempFile(data: ByteArray): File {
        val file = File.createTempFile("temp_", ".tmp", context.cacheDir)
        file.writeBytes(data)
        return file
    }

    // キャッシュクリア
    fun clearCache() {
        context.cacheDir.listFiles()?.forEach { it.delete() }
    }
}
```

---

## 外部ストレージ（アプリ専用）

```kotlin
class ExternalFileRepository(private val context: Context) {

    // アプリ専用外部ディレクトリ（パーミッション不要）
    fun saveToExternal(fileName: String, content: String) {
        val dir = context.getExternalFilesDir(null) ?: return
        File(dir, fileName).writeText(content)
    }

    // 画像保存（アプリ専用）
    fun saveImage(fileName: String, bitmap: Bitmap) {
        val dir = context.getExternalFilesDir(Environment.DIRECTORY_PICTURES) ?: return
        File(dir, fileName).outputStream().use { stream ->
            bitmap.compress(Bitmap.CompressFormat.PNG, 100, stream)
        }
    }

    // ダウンロードディレクトリ
    fun saveDownload(fileName: String, data: ByteArray) {
        val dir = context.getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS) ?: return
        File(dir, fileName).writeBytes(data)
    }
}
```

---

## Storage Access Framework（SAF）

```kotlin
@Composable
fun DocumentPickerExample() {
    var content by remember { mutableStateOf("") }

    // ファイル選択
    val openLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.OpenDocument()
    ) { uri ->
        uri?.let { readTextFromUri(context, it) }?.let { content = it }
    }

    // ファイル保存
    val saveLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.CreateDocument("text/plain")
    ) { uri ->
        uri?.let { writeTextToUri(context, it, content) }
    }

    Column(Modifier.padding(16.dp)) {
        Button(onClick = { openLauncher.launch(arrayOf("text/*")) }) {
            Text("ファイルを開く")
        }
        Button(onClick = { saveLauncher.launch("document.txt") }) {
            Text("ファイルを保存")
        }
        Text(content)
    }
}

fun readTextFromUri(context: Context, uri: Uri): String? {
    return context.contentResolver.openInputStream(uri)?.bufferedReader()?.use {
        it.readText()
    }
}

fun writeTextToUri(context: Context, uri: Uri, content: String) {
    context.contentResolver.openOutputStream(uri)?.use { stream ->
        stream.write(content.toByteArray())
    }
}
```

---

## MediaStore（写真の保存）

```kotlin
suspend fun saveImageToGallery(
    context: Context,
    bitmap: Bitmap,
    displayName: String
): Uri? {
    return withContext(Dispatchers.IO) {
        val values = ContentValues().apply {
            put(MediaStore.Images.Media.DISPLAY_NAME, "$displayName.png")
            put(MediaStore.Images.Media.MIME_TYPE, "image/png")
            put(MediaStore.Images.Media.RELATIVE_PATH, Environment.DIRECTORY_PICTURES)
            put(MediaStore.Images.Media.IS_PENDING, 1)
        }

        val resolver = context.contentResolver
        val uri = resolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values)

        uri?.let {
            resolver.openOutputStream(it)?.use { stream ->
                bitmap.compress(Bitmap.CompressFormat.PNG, 100, stream)
            }
            values.clear()
            values.put(MediaStore.Images.Media.IS_PENDING, 0)
            resolver.update(it, values, null, null)
        }

        uri
    }
}
```

---

## まとめ

| 用途 | API | パーミッション |
|------|-----|---------------|
| アプリ設定・キャッシュ | `context.filesDir` | 不要 |
| アプリ専用画像 | `getExternalFilesDir()` | 不要 |
| ユーザーファイル選択 | SAF `OpenDocument` | 不要 |
| ギャラリー保存 | MediaStore | 不要（Android 10+） |

- 内部ストレージ: アプリ専用データ
- 外部ストレージ: アプリ専用メディア
- SAF: ユーザー主導のファイル操作
- MediaStore: 共有メディアへの保存

---

8種類のAndroidアプリテンプレート（ファイル操作設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DataStore完全ガイド](https://zenn.dev/myougatheaxo/articles/android-datastore-2026)
- [Room Databaseガイド](https://zenn.dev/myougatheaxo/articles/android-room-database-2026)
- [画像ピッカーガイド](https://zenn.dev/myougatheaxo/articles/android-compose-image-picker-2026)
