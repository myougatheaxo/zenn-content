---
title: "PDF表示/生成ガイド — PdfRenderer + Compose"
emoji: "📄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "pdf"]
published: true
---

## この記事で学べること

Androidでの**PDF表示**（PdfRenderer）と**PDF生成**（PdfDocument）をCompose連携で解説します。

---

## PDF表示（PdfRenderer）

```kotlin
@Composable
fun PdfViewer(pdfUri: Uri) {
    val context = LocalContext.current
    var currentPage by remember { mutableIntStateOf(0) }
    var pageCount by remember { mutableIntStateOf(0) }
    var bitmap by remember { mutableStateOf<Bitmap?>(null) }

    LaunchedEffect(pdfUri, currentPage) {
        withContext(Dispatchers.IO) {
            context.contentResolver.openFileDescriptor(pdfUri, "r")?.use { fd ->
                val renderer = PdfRenderer(fd)
                pageCount = renderer.pageCount
                renderer.openPage(currentPage).use { page ->
                    val bmp = Bitmap.createBitmap(
                        page.width * 2, page.height * 2, Bitmap.Config.ARGB_8888
                    )
                    page.render(bmp, null, null, PdfRenderer.Page.RENDER_MODE_FOR_DISPLAY)
                    bitmap = bmp
                }
                renderer.close()
            }
        }
    }

    Column(Modifier.fillMaxSize()) {
        // PDF表示
        bitmap?.let {
            Image(
                bitmap = it.asImageBitmap(),
                contentDescription = "PDF page ${currentPage + 1}",
                modifier = Modifier
                    .fillMaxWidth()
                    .weight(1f),
                contentScale = ContentScale.Fit
            )
        }

        // ページ制御
        Row(
            Modifier.fillMaxWidth().padding(8.dp),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically
        ) {
            IconButton(
                onClick = { if (currentPage > 0) currentPage-- },
                enabled = currentPage > 0
            ) { Icon(Icons.Default.ArrowBack, "前ページ") }

            Text("${currentPage + 1} / $pageCount")

            IconButton(
                onClick = { if (currentPage < pageCount - 1) currentPage++ },
                enabled = currentPage < pageCount - 1
            ) { Icon(Icons.Default.ArrowForward, "次ページ") }
        }
    }
}
```

---

## PDFファイル選択

```kotlin
@Composable
fun PdfPickerScreen() {
    var pdfUri by remember { mutableStateOf<Uri?>(null) }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.OpenDocument()
    ) { uri -> pdfUri = uri }

    if (pdfUri != null) {
        PdfViewer(pdfUri!!)
    } else {
        Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            Button(onClick = { launcher.launch(arrayOf("application/pdf")) }) {
                Text("PDFを開く")
            }
        }
    }
}
```

---

## PDF生成

```kotlin
suspend fun generatePdf(
    context: Context,
    title: String,
    items: List<String>
): Uri? = withContext(Dispatchers.IO) {
    val document = PdfDocument()
    val pageInfo = PdfDocument.PageInfo.Builder(595, 842, 1).create() // A4
    val page = document.startPage(pageInfo)
    val canvas = page.canvas
    val paint = Paint().apply {
        textSize = 24f
        color = android.graphics.Color.BLACK
    }

    // タイトル
    paint.textSize = 32f
    paint.isFakeBoldText = true
    canvas.drawText(title, 40f, 60f, paint)

    // アイテム一覧
    paint.textSize = 18f
    paint.isFakeBoldText = false
    items.forEachIndexed { index, item ->
        canvas.drawText("${index + 1}. $item", 40f, 120f + (index * 30f), paint)
    }

    document.finishPage(page)

    val values = ContentValues().apply {
        put(MediaStore.MediaColumns.DISPLAY_NAME, "report.pdf")
        put(MediaStore.MediaColumns.MIME_TYPE, "application/pdf")
        put(MediaStore.MediaColumns.RELATIVE_PATH, Environment.DIRECTORY_DOCUMENTS)
    }

    val uri = context.contentResolver.insert(MediaStore.Files.getContentUri("external"), values)
    uri?.let {
        context.contentResolver.openOutputStream(it)?.use { stream ->
            document.writeTo(stream)
        }
    }
    document.close()
    uri
}
```

---

## まとめ

- `PdfRenderer`でPDFページをBitmapに描画
- `openPage()`/`render()`でページ単位の表示
- SAF `OpenDocument`でPDFファイル選択
- `PdfDocument`でPDF生成（Canvas描画）
- MediaStoreでDocumentsフォルダに保存
- ページ送りUIでナビゲーション

---

8種類のAndroidアプリテンプレート（PDF対応可能）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ファイルストレージガイド](https://zenn.dev/myougatheaxo/articles/android-file-storage-2026)
- [Canvas描画ガイド](https://zenn.dev/myougatheaxo/articles/compose-canvas-drawing-2026)
- [画像ピッカーガイド](https://zenn.dev/myougatheaxo/articles/android-compose-image-picker-2026)
