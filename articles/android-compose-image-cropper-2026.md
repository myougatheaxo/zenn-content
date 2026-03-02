---
title: "画像クロッパー完全ガイド — UCrop/カスタム切り抜き/アスペクト比/Compose"
emoji: "✂️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "image"]
published: true
---

## この記事で学べること

**画像クロッパー**（UCrop連携、カスタムクロップUI、アスペクト比指定、円形切り抜き）を解説します。

---

## UCrop連携

```kotlin
@Composable
fun ImageCropScreen() {
    val context = LocalContext.current
    var sourceUri by remember { mutableStateOf<Uri?>(null) }
    var croppedUri by remember { mutableStateOf<Uri?>(null) }

    val cropLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { result ->
        if (result.resultCode == Activity.RESULT_OK) {
            croppedUri = UCrop.getOutput(result.data!!)
        }
    }

    val pickImage = rememberLauncherForActivityResult(
        ActivityResultContracts.PickVisualMedia()
    ) { uri ->
        uri?.let {
            sourceUri = it
            val destUri = Uri.fromFile(File(context.cacheDir, "cropped_${System.currentTimeMillis()}.jpg"))

            val uCropIntent = UCrop.of(it, destUri)
                .withAspectRatio(1f, 1f)  // 正方形
                .withMaxResultSize(1080, 1080)
                .withOptions(UCrop.Options().apply {
                    setCompressionQuality(90)
                    setToolbarTitle("画像を切り抜き")
                    setFreeStyleCropEnabled(true)
                })
                .getIntent(context)

            cropLauncher.launch(uCropIntent)
        }
    }

    Column(Modifier.fillMaxSize().padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally) {
        croppedUri?.let { uri ->
            AsyncImage(
                model = uri,
                contentDescription = "切り抜き画像",
                modifier = Modifier.size(200.dp).clip(CircleShape)
            )
        }

        Spacer(Modifier.height(16.dp))

        Button(onClick = { pickImage.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)) }) {
            Text("画像を選択して切り抜き")
        }
    }
}
```

---

## カスタムクロップUI

```kotlin
@Composable
fun CustomCropView(
    bitmap: ImageBitmap,
    aspectRatio: Float = 1f,
    onCrop: (Rect) -> Unit
) {
    var cropRect by remember { mutableStateOf(Rect.Zero) }
    var imageSize by remember { mutableStateOf(IntSize.Zero) }

    Box(Modifier.fillMaxWidth().aspectRatio(1f)) {
        Image(
            bitmap = bitmap,
            contentDescription = null,
            modifier = Modifier
                .fillMaxSize()
                .onSizeChanged { imageSize = it }
        )

        // クロップ領域
        Canvas(
            Modifier
                .fillMaxSize()
                .pointerInput(Unit) {
                    detectDragGestures { change, dragAmount ->
                        cropRect = Rect(
                            left = (cropRect.left + dragAmount.x).coerceIn(0f, size.width.toFloat()),
                            top = (cropRect.top + dragAmount.y).coerceIn(0f, size.height.toFloat()),
                            right = (cropRect.right + dragAmount.x).coerceIn(0f, size.width.toFloat()),
                            bottom = (cropRect.bottom + dragAmount.y).coerceIn(0f, size.height.toFloat())
                        )
                    }
                }
        ) {
            // 暗いオーバーレイ
            drawRect(Color.Black.copy(alpha = 0.5f))
            // クロップ領域を透明に
            drawRect(Color.Transparent, topLeft = cropRect.topLeft, size = cropRect.size, blendMode = BlendMode.Clear)
            // 枠線
            drawRect(Color.White, topLeft = cropRect.topLeft, size = cropRect.size, style = Stroke(2.dp.toPx()))
        }
    }
}
```

---

## プロフィール画像設定

```kotlin
@Composable
fun ProfileImagePicker(
    currentImage: Uri?,
    onImageSelected: (Uri) -> Unit
) {
    val context = LocalContext.current

    val cropLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { result ->
        if (result.resultCode == Activity.RESULT_OK) {
            UCrop.getOutput(result.data!!)?.let { onImageSelected(it) }
        }
    }

    val pickImage = rememberLauncherForActivityResult(
        ActivityResultContracts.PickVisualMedia()
    ) { uri ->
        uri?.let {
            val dest = Uri.fromFile(File(context.cacheDir, "profile.jpg"))
            val intent = UCrop.of(it, dest)
                .withAspectRatio(1f, 1f)
                .withMaxResultSize(512, 512)
                .getIntent(context)
            cropLauncher.launch(intent)
        }
    }

    Box(
        Modifier.size(120.dp).clip(CircleShape)
            .clickable { pickImage.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)) },
        contentAlignment = Alignment.Center
    ) {
        if (currentImage != null) {
            AsyncImage(currentImage, "プロフィール", Modifier.fillMaxSize(), contentScale = ContentScale.Crop)
        } else {
            Box(Modifier.fillMaxSize().background(MaterialTheme.colorScheme.surfaceVariant)) {
                Icon(Icons.Default.CameraAlt, null, Modifier.align(Alignment.Center))
            }
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| 画像選択 | `PickVisualMedia` |
| クロップ | UCrop |
| 円形 | `CircleShape` + `clip` |
| カスタム | `Canvas` + `detectDragGestures` |

- UCropでアスペクト比指定の切り抜き
- `PickVisualMedia`で画像選択
- 円形クロップでプロフィール画像設定
- カスタムUIで自由なクロップ体験

---

8種類のAndroidアプリテンプレート（画像処理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Photo Picker](https://zenn.dev/myougatheaxo/articles/android-compose-photo-picker-2026)
- [Coil画像](https://zenn.dev/myougatheaxo/articles/android-compose-coil-image-2026)
- [CameraX](https://zenn.dev/myougatheaxo/articles/android-compose-camera-compose-2026)
