---
title: "ファイル管理完全ガイド — SAF/ContentResolver/ファイル選択/保存"
emoji: "📁"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "storage"]
published: true
---

## この記事で学べること

**ファイル管理**（Storage Access Framework、ContentResolver、ファイル選択/保存、ドキュメントツリー）を解説します。

---

## ファイル選択

```kotlin
@Composable
fun FilePickerScreen() {
    var selectedFile by remember { mutableStateOf<Uri?>(null) }
    val context = LocalContext.current

    val filePicker = rememberLauncherForActivityResult(
        ActivityResultContracts.OpenDocument()
    ) { uri -> selectedFile = uri }

    val multiFilePicker = rememberLauncherForActivityResult(
        ActivityResultContracts.OpenMultipleDocuments()
    ) { uris -> /* 複数ファイル処理 */ }

    Column(Modifier.padding(16.dp)) {
        Button(onClick = { filePicker.launch(arrayOf("*/*")) }) {
            Text("ファイルを選択")
        }

        Button(onClick = { filePicker.launch(arrayOf("application/pdf")) }) {
            Text("PDFを選択")
        }

        Button(onClick = { multiFilePicker.launch(arrayOf("image/*")) }) {
            Text("画像を複数選択")
        }

        selectedFile?.let { uri ->
            val fileName = getFileName(context, uri)
            Text("選択: $fileName")
        }
    }
}

fun getFileName(context: Context, uri: Uri): String? {
    return context.contentResolver.query(uri, null, null, null, null)?.use { cursor ->
        val nameIndex = cursor.getColumnIndex(OpenableColumns.DISPLAY_NAME)
        cursor.moveToFirst()
        cursor.getString(nameIndex)
    }
}
```

---

## ファイル保存

```kotlin
@Composable
fun FileSaveScreen() {
    val context = LocalContext.current

    val createDocument = rememberLauncherForActivityResult(
        ActivityResultContracts.CreateDocument("text/plain")
    ) { uri ->
        uri?.let {
            context.contentResolver.openOutputStream(it)?.use { stream ->
                stream.write("Hello, World!".toByteArray())
            }
        }
    }

    val createCsv = rememberLauncherForActivityResult(
        ActivityResultContracts.CreateDocument("text/csv")
    ) { uri ->
        uri?.let {
            context.contentResolver.openOutputStream(it)?.use { stream ->
                stream.write("Name,Age\nTaro,25\nHanako,30".toByteArray())
            }
        }
    }

    Column(Modifier.padding(16.dp)) {
        Button(onClick = { createDocument.launch("memo.txt") }) {
            Text("テキストファイルを保存")
        }
        Button(onClick = { createCsv.launch("data.csv") }) {
            Text("CSVを保存")
        }
    }
}
```

---

## ファイル読み込み

```kotlin
class FileRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    fun readTextFile(uri: Uri): String {
        return context.contentResolver.openInputStream(uri)?.use { stream ->
            stream.bufferedReader().readText()
        } ?: ""
    }

    fun readFileInfo(uri: Uri): FileInfo {
        val cursor = context.contentResolver.query(uri, null, null, null, null)
        return cursor?.use {
            it.moveToFirst()
            FileInfo(
                name = it.getString(it.getColumnIndexOrThrow(OpenableColumns.DISPLAY_NAME)),
                size = it.getLong(it.getColumnIndexOrThrow(OpenableColumns.SIZE)),
                mimeType = context.contentResolver.getType(uri) ?: "unknown"
            )
        } ?: FileInfo("unknown", 0, "unknown")
    }
}

data class FileInfo(val name: String, val size: Long, val mimeType: String)
```

---

## 内部ストレージ

```kotlin
class InternalStorageManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    fun saveToInternal(fileName: String, content: String) {
        context.openFileOutput(fileName, Context.MODE_PRIVATE).use {
            it.write(content.toByteArray())
        }
    }

    fun readFromInternal(fileName: String): String? {
        return try {
            context.openFileInput(fileName).bufferedReader().readText()
        } catch (e: FileNotFoundException) { null }
    }

    fun listFiles(): List<String> = context.fileList().toList()

    fun deleteFile(fileName: String): Boolean = context.deleteFile(fileName)
}
```

---

## まとめ

| 操作 | API |
|------|-----|
| ファイル選択 | `OpenDocument` |
| ファイル保存 | `CreateDocument` |
| 読み込み | `ContentResolver` |
| 内部保存 | `openFileOutput` |

- SAFでユーザーがファイルを安全に選択
- `ContentResolver`でMIMEタイプに応じたファイル操作
- 内部ストレージは`openFileOutput`でアプリ専用
- 永続アクセスは`takePersistableUriPermission`

---

8種類のAndroidアプリテンプレート（ファイル管理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Photo Picker](https://zenn.dev/myougatheaxo/articles/android-compose-photo-picker-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
- [パーミッション](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handling-2026)
