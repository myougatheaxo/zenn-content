---
title: "PDF Viewer完全ガイド — PdfRenderer/ページ表示/ズーム操作"
emoji: "📄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "pdf"]
published: true
---

## この記事で学べること

**PDF Viewer**（PdfRenderer、ページ送り、ズーム、サムネイル表示）を解説します。

---

## PdfRenderer基本

```kotlin
@Composable
fun PdfViewer(uri: Uri) {
    val context = LocalContext.current
    var currentPage by remember { mutableIntStateOf(0) }
    var pageCount by remember { mutableIntStateOf(0) }
    var bitmap by remember { mutableStateOf<Bitmap?>(null) }

    LaunchedEffect(uri, currentPage) {
        withContext(Dispatchers.IO) {
            context.contentResolver.openFileDescriptor(uri, "r")?.use { fd ->
                PdfRenderer(fd).use { renderer ->
                    pageCount = renderer.pageCount
                    renderer.openPage(currentPage).use { page ->
                        val bmp = Bitmap.createBitmap(
                            page.width * 2, page.height * 2, Bitmap.Config.ARGB_8888
                        )
                        page.render(bmp, null, null, PdfRenderer.Page.RENDER_MODE_FOR_DISPLAY)
                        bitmap = bmp
                    }
                }
            }
        }
    }

    Column(Modifier.fillMaxSize()) {
        // ページ表示
        bitmap?.let { bmp ->
            Image(
                bitmap = bmp.asImageBitmap(),
                contentDescription = "PDF page ${currentPage + 1}",
                modifier = Modifier.weight(1f).fillMaxWidth(),
                contentScale = ContentScale.Fit
            )
        }

        // ページ操作
        Row(
            Modifier.fillMaxWidth().padding(8.dp),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically
        ) {
            IconButton(onClick = { if (currentPage > 0) currentPage-- }, enabled = currentPage > 0) {
                Icon(Icons.AutoMirrored.Filled.ArrowBack, "前のページ")
            }
            Text("${currentPage + 1} / $pageCount")
            IconButton(onClick = { if (currentPage < pageCount - 1) currentPage++ }, enabled = currentPage < pageCount - 1) {
                Icon(Icons.AutoMirrored.Filled.ArrowForward, "次のページ")
            }
        }
    }
}
```

---

## ズーム対応

```kotlin
@Composable
fun ZoomablePdfPage(bitmap: Bitmap?) {
    var scale by remember { mutableFloatStateOf(1f) }
    var offset by remember { mutableStateOf(Offset.Zero) }

    bitmap?.let { bmp ->
        Image(
            bitmap = bmp.asImageBitmap(),
            contentDescription = null,
            modifier = Modifier
                .fillMaxSize()
                .graphicsLayer {
                    scaleX = scale; scaleY = scale
                    translationX = offset.x; translationY = offset.y
                }
                .pointerInput(Unit) {
                    detectTransformGestures { _, pan, zoom, _ ->
                        scale = (scale * zoom).coerceIn(1f, 5f)
                        offset = if (scale > 1f) offset + pan else Offset.Zero
                    }
                },
            contentScale = ContentScale.Fit
        )
    }
}
```

---

## ファイル選択

```kotlin
@Composable
fun PdfViewerScreen() {
    var pdfUri by remember { mutableStateOf<Uri?>(null) }
    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.OpenDocument()
    ) { uri -> pdfUri = uri }

    if (pdfUri != null) {
        PdfViewer(uri = pdfUri!!)
    } else {
        Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            Button(onClick = { launcher.launch(arrayOf("application/pdf")) }) {
                Icon(Icons.Default.FileOpen, null)
                Spacer(Modifier.width(8.dp))
                Text("PDFを開く")
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `PdfRenderer` | PDF描画エンジン |
| `openPage` | ページ取得 |
| `render` | Bitmapに描画 |
| `detectTransformGestures` | ズーム操作 |

- `PdfRenderer`でPDFをBitmapに変換して表示
- ページ送りで前後ナビゲーション
- `detectTransformGestures`でピンチズーム
- `ActivityResultContracts`でファイル選択

---

8種類のAndroidアプリテンプレート（ファイル操作対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [画像読み込み](https://zenn.dev/myougatheaxo/articles/android-compose-compose-image-loading-2026)
- [ShareSheet](https://zenn.dev/myougatheaxo/articles/android-compose-compose-share-sheet-2026)
- [パーミッション管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permission-handler-2026)
