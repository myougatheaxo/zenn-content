---
title: "Compose FilePicker完全ガイド — SAF/ファイル選択/保存/複数ファイル"
emoji: "📂"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "file"]
published: true
---

## この記事で学べること

**Compose FilePicker**（SAF、ファイル選択/保存、MIME型フィルタ、複数ファイル選択）を解説します。

---

## 基本ファイル選択

```kotlin
@Composable
fun FilePickerDemo() {
    val context = LocalContext.current
    var selectedFile by remember { mutableStateOf<String?>(null) }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.OpenDocument()
    ) { uri ->
        uri?.let {
            val cursor = context.contentResolver.query(it, null, null, null, null)
            cursor?.use { c ->
                if (c.moveToFirst()) {
                    val nameIndex = c.getColumnIndex(OpenableColumns.DISPLAY_NAME)
                    selectedFile = c.getString(nameIndex)
                }
            }
        }
    }

    Column(Modifier.padding(16.dp)) {
        Button(onClick = { launcher.launch(arrayOf("*/*")) }) {
            Text("ファイルを選択")
        }
        selectedFile?.let { Text("選択: $it", Modifier.padding(top = 8.dp)) }
    }
}
```

---

## MIME型フィルタ

```kotlin
@Composable
fun TypedFilePicker() {
    // PDF選択
    val pdfLauncher = rememberLauncherForActivityResult(ActivityResultContracts.OpenDocument()) { /* 処理 */ }

    // 画像選択
    val imageLauncher = rememberLauncherForActivityResult(ActivityResultContracts.GetContent()) { /* 処理 */ }

    // 複数ファイル
    val multiLauncher = rememberLauncherForActivityResult(ActivityResultContracts.OpenMultipleDocuments()) { uris ->
        // uris: List<Uri>
    }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        Button(onClick = { pdfLauncher.launch(arrayOf("application/pdf")) }) { Text("PDF選択") }
        Button(onClick = { imageLauncher.launch("image/*") }) { Text("画像選択") }
        Button(onClick = { multiLauncher.launch(arrayOf("*/*")) }) { Text("複数ファイル") }
    }
}
```

---

## ファイル保存

```kotlin
@Composable
fun FileSaveDemo() {
    val context = LocalContext.current

    val saveLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.CreateDocument("text/csv")
    ) { uri ->
        uri?.let {
            context.contentResolver.openOutputStream(it)?.use { out ->
                val csv = "名前,年齢,メール\nみょうが,3,test@example.com"
                out.write(csv.toByteArray())
            }
        }
    }

    Button(onClick = { saveLauncher.launch("export.csv") }) {
        Text("CSVエクスポート")
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `OpenDocument` | ファイル選択 |
| `GetContent` | コンテンツ選択 |
| `OpenMultipleDocuments` | 複数選択 |
| `CreateDocument` | ファイル保存 |

- SAF（Storage Access Framework）でセキュアなファイルアクセス
- MIME型配列で選択可能なファイル種別をフィルタ
- `contentResolver`でUri経由のI/O
- Android 10+はScoped Storageのためパス直接アクセス不可

---

8種類のAndroidアプリテンプレート（ファイル操作対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose ImageCropper](https://zenn.dev/myougatheaxo/articles/android-compose-compose-image-cropper-2026)
- [Room Backup](https://zenn.dev/myougatheaxo/articles/android-compose-room-backup-2026)
- [Compose ShareIntent](https://zenn.dev/myougatheaxo/articles/android-compose-compose-share-intent-2026)
