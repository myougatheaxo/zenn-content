---
title: "画像ピッカー/カメラ撮影ガイド — Compose版"
emoji: "📸"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "camera"]
published: true
---

## この記事で学べること

Composeでの**画像選択**（PhotoPicker、カメラ撮影、ActivityResult API）を解説します。

---

## Photo Picker (Android 13+)

```kotlin
@Composable
fun PhotoPickerExample() {
    var imageUri by remember { mutableStateOf<Uri?>(null) }

    val pickMedia = rememberLauncherForActivityResult(
        ActivityResultContracts.PickVisualMedia()
    ) { uri ->
        imageUri = uri
    }

    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        imageUri?.let { uri ->
            AsyncImage(
                model = uri,
                contentDescription = null,
                modifier = Modifier
                    .size(200.dp)
                    .clip(RoundedCornerShape(12.dp)),
                contentScale = ContentScale.Crop
            )
        }

        Spacer(Modifier.height(16.dp))

        Button(onClick = {
            pickMedia.launch(
                PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
            )
        }) {
            Text("画像を選択")
        }
    }
}
```

---

## 複数画像選択

```kotlin
@Composable
fun MultiplePhotoPickerExample() {
    var imageUris by remember { mutableStateOf<List<Uri>>(emptyList()) }

    val pickMultipleMedia = rememberLauncherForActivityResult(
        ActivityResultContracts.PickMultipleVisualMedia(maxItems = 5)
    ) { uris ->
        imageUris = uris
    }

    Column {
        LazyRow(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            items(imageUris) { uri ->
                AsyncImage(
                    model = uri,
                    contentDescription = null,
                    modifier = Modifier
                        .size(100.dp)
                        .clip(RoundedCornerShape(8.dp)),
                    contentScale = ContentScale.Crop
                )
            }
        }

        Button(onClick = {
            pickMultipleMedia.launch(
                PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
            )
        }) {
            Text("複数画像を選択 (最大5枚)")
        }
    }
}
```

---

## カメラ撮影

```kotlin
@Composable
fun CameraCaptureExample() {
    val context = LocalContext.current
    var imageUri by remember { mutableStateOf<Uri?>(null) }

    val takePicture = rememberLauncherForActivityResult(
        ActivityResultContracts.TakePicture()
    ) { success ->
        if (!success) imageUri = null
    }

    val cameraPermission = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) {
            val uri = createImageUri(context)
            imageUri = uri
            takePicture.launch(uri)
        }
    }

    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        imageUri?.let { uri ->
            AsyncImage(
                model = uri,
                contentDescription = null,
                modifier = Modifier.size(200.dp).clip(RoundedCornerShape(12.dp)),
                contentScale = ContentScale.Crop
            )
        }

        Button(onClick = {
            cameraPermission.launch(Manifest.permission.CAMERA)
        }) {
            Icon(Icons.Default.CameraAlt, null)
            Spacer(Modifier.width(8.dp))
            Text("カメラで撮影")
        }
    }
}

fun createImageUri(context: Context): Uri {
    val file = File(context.cacheDir, "photo_${System.currentTimeMillis()}.jpg")
    return FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file)
}
```

---

## まとめ

- `PickVisualMedia`でAndroid 13+のPhoto Picker
- `PickMultipleVisualMedia`で複数選択（maxItems指定）
- `TakePicture`でカメラ撮影（FileProviderでURI作成）
- `rememberLauncherForActivityResult`で結果受け取り
- カメラ利用は`CAMERA`パーミッション必須
- `AsyncImage`で選択/撮影画像を即座に表示

---

8種類のAndroidアプリテンプレート（画像機能実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coil画像読み込み](https://zenn.dev/myougatheaxo/articles/android-compose-coil-image-2026)
- [QRコードスキャナー](https://zenn.dev/myougatheaxo/articles/android-compose-qrcode-scanner-2026)
- [パーミッション処理](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handling-2026)
