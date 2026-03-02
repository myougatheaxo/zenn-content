---
title: "Compose ImageCropper完全ガイド — 画像トリミング/アスペクト比/プロフィール画像"
emoji: "✂️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "image"]
published: true
---

## この記事で学べること

**Compose ImageCropper**（画像トリミング、アスペクト比固定、プロフィール画像設定、ActivityResult連携）を解説します。

---

## UCrop連携

```kotlin
// build.gradle
// implementation "com.github.yalantis:ucrop:2.2.8"

@Composable
fun ImagePickerWithCrop() {
    val context = LocalContext.current
    var croppedUri by remember { mutableStateOf<Uri?>(null) }

    val cropLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { result ->
        if (result.resultCode == Activity.RESULT_OK) {
            croppedUri = UCrop.getOutput(result.data!!)
        }
    }

    val pickerLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.GetContent()
    ) { uri ->
        uri?.let {
            val destUri = Uri.fromFile(File(context.cacheDir, "cropped_${System.currentTimeMillis()}.jpg"))
            val intent = UCrop.of(it, destUri)
                .withAspectRatio(1f, 1f)
                .withMaxResultSize(500, 500)
                .getIntent(context)
            cropLauncher.launch(intent)
        }
    }

    Column(Modifier.padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally) {
        croppedUri?.let {
            AsyncImage(model = it, contentDescription = null,
                modifier = Modifier.size(150.dp).clip(CircleShape), contentScale = ContentScale.Crop)
        } ?: Box(Modifier.size(150.dp).background(Color.Gray, CircleShape),
            contentAlignment = Alignment.Center) { Icon(Icons.Default.Person, null, Modifier.size(64.dp), tint = Color.White) }

        Spacer(Modifier.height(16.dp))
        Button(onClick = { pickerLauncher.launch("image/*") }) { Text("画像を選択") }
    }
}
```

---

## PhotoPicker（Android 13+）

```kotlin
@Composable
fun ModernImagePicker() {
    var selectedUri by remember { mutableStateOf<Uri?>(null) }

    val photoPickerLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.PickVisualMedia()
    ) { uri -> selectedUri = uri }

    Column(Modifier.padding(16.dp)) {
        selectedUri?.let {
            AsyncImage(model = it, contentDescription = null,
                modifier = Modifier.fillMaxWidth().height(200.dp).clip(RoundedCornerShape(12.dp)),
                contentScale = ContentScale.Crop)
        }

        Button(onClick = {
            photoPickerLauncher.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly))
        }, Modifier.fillMaxWidth().padding(top = 16.dp)) { Text("写真を選択") }
    }
}
```

---

## Bitmap操作

```kotlin
fun cropBitmapToCircle(source: Bitmap): Bitmap {
    val size = minOf(source.width, source.height)
    val output = Bitmap.createBitmap(size, size, Bitmap.Config.ARGB_8888)
    val canvas = android.graphics.Canvas(output)
    val paint = android.graphics.Paint().apply { isAntiAlias = true }
    canvas.drawCircle(size / 2f, size / 2f, size / 2f, paint)
    paint.xfermode = PorterDuffXfermode(PorterDuff.Mode.SRC_IN)
    canvas.drawBitmap(source, (size - source.width) / 2f, (size - source.height) / 2f, paint)
    return output
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `UCrop` | 高機能トリミング |
| `PickVisualMedia` | 写真ピッカー (API 33+) |
| `GetContent` | ファイルピッカー |
| `withAspectRatio` | アスペクト比固定 |

- `UCrop`で高機能な画像トリミングUI
- `PickVisualMedia`はAndroid 13+推奨のピッカー
- `withMaxResultSize`で出力サイズ制限
- プロフィール画像は`CircleShape`+`ContentScale.Crop`

---

8種類のAndroidアプリテンプレート（画像処理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Coil](https://zenn.dev/myougatheaxo/articles/android-compose-compose-coil-2026)
- [Compose CameraPreview](https://zenn.dev/myougatheaxo/articles/android-compose-compose-camera-preview-2026)
- [Compose FilePicker](https://zenn.dev/myougatheaxo/articles/android-compose-compose-file-picker-2026)
