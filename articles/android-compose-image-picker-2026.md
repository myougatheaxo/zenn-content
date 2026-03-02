---
title: "Compose画像ピッカー実装ガイド — Photo Pickerでギャラリーから画像を選択"
emoji: "🖼️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "image"]
published: true
---

## この記事で学べること

Android 13+の**Photo Picker**を使って、パーミッション不要で安全に画像を選択する方法を解説します。

---

## Photo Picker（推奨）

```kotlin
@Composable
fun ImagePickerScreen() {
    var selectedUri by remember { mutableStateOf<Uri?>(null) }

    val pickMedia = rememberLauncherForActivityResult(
        ActivityResultContracts.PickVisualMedia()
    ) { uri ->
        selectedUri = uri
    }

    Column(Modifier.padding(16.dp)) {
        Button(onClick = {
            pickMedia.launch(
                PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
            )
        }) {
            Text("画像を選択")
        }

        selectedUri?.let { uri ->
            AsyncImage(
                model = uri,
                contentDescription = "選択した画像",
                modifier = Modifier
                    .fillMaxWidth()
                    .height(300.dp)
                    .clip(RoundedCornerShape(12.dp)),
                contentScale = ContentScale.Crop
            )
        }
    }
}
```

**パーミッション不要**。Photo Pickerはサンドボックス内で動作します。

---

## 複数画像の選択

```kotlin
val pickMultipleMedia = rememberLauncherForActivityResult(
    ActivityResultContracts.PickMultipleVisualMedia(maxItems = 5)
) { uris ->
    selectedUris = uris
}

Button(onClick = {
    pickMultipleMedia.launch(
        PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
    )
}) {
    Text("複数画像を選択（最大5枚）")
}
```

---

## 動画の選択

```kotlin
// 画像のみ
PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)

// 動画のみ
PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.VideoOnly)

// 画像と動画
PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageAndVideo)
```

---

## カメラで撮影

```kotlin
@Composable
fun CameraCapture() {
    val context = LocalContext.current
    var imageUri by remember { mutableStateOf<Uri?>(null) }

    val takePicture = rememberLauncherForActivityResult(
        ActivityResultContracts.TakePicture()
    ) { success ->
        if (success) {
            // imageUriに撮影画像が保存される
        }
    }

    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) {
            val uri = createImageUri(context)
            imageUri = uri
            takePicture.launch(uri)
        }
    }

    Button(onClick = {
        permissionLauncher.launch(Manifest.permission.CAMERA)
    }) {
        Text("カメラで撮影")
    }
}

fun createImageUri(context: Context): Uri {
    val file = File.createTempFile("photo_", ".jpg", context.cacheDir)
    return FileProvider.getUriForFile(context, "${context.packageName}.provider", file)
}
```

---

## Photo Pickerの対応確認

```kotlin
// Photo Pickerが利用可能かチェック
val isPhotoPickerAvailable = ActivityResultContracts
    .PickVisualMedia
    .isPhotoPickerAvailable(context)
```

Android 11以下ではGoogle Play Servicesを通じてバックポート。

---

## まとめ

- **Photo Picker**が推奨（パーミッション不要、Android 13+）
- `PickVisualMedia`で単一画像、`PickMultipleVisualMedia`で複数
- `ImageOnly`/`VideoOnly`/`ImageAndVideo`でメディアタイプ指定
- カメラ撮影は`TakePicture` + `FileProvider`
- Coil `AsyncImage`で選択画像を表示

---

8種類のAndroidアプリテンプレート（画像機能追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coilで画像を表示する](https://zenn.dev/myougatheaxo/articles/compose-coil-image-2026)
- [パーミッション完全ガイド](https://zenn.dev/myougatheaxo/articles/android-permission-runtime-2026)
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
