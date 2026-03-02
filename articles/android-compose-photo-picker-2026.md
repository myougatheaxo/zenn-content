---
title: "Photo Picker完全ガイド — Android 13+写真選択API/権限不要/Compose連携"
emoji: "🖼️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "photopicker"]
published: true
---

## この記事で学べること

**Photo Picker**（Android 13+標準API、権限不要、単一/複数選択、動画対応、Compose UI連携）を解説します。

---

## 基本の写真選択

```kotlin
@Composable
fun SinglePhotoPickerScreen() {
    var selectedUri by remember { mutableStateOf<Uri?>(null) }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.PickVisualMedia()
    ) { uri -> selectedUri = uri }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Button(onClick = {
            launcher.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly))
        }) { Text("写真を選択") }

        selectedUri?.let { uri ->
            AsyncImage(
                model = uri,
                contentDescription = "選択した写真",
                modifier = Modifier.fillMaxWidth().height(300.dp).padding(top = 16.dp),
                contentScale = ContentScale.Crop
            )
        }
    }
}
```

---

## 複数選択

```kotlin
@Composable
fun MultiPhotoPickerScreen() {
    var selectedUris by remember { mutableStateOf<List<Uri>>(emptyList()) }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.PickMultipleVisualMedia(maxItems = 5)
    ) { uris -> selectedUris = uris }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Button(onClick = {
            launcher.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageAndVideo))
        }) { Text("写真/動画を選択（最大5件）") }

        Text("${selectedUris.size}件選択中", Modifier.padding(vertical = 8.dp))

        LazyVerticalGrid(
            columns = GridCells.Fixed(3),
            horizontalArrangement = Arrangement.spacedBy(4.dp),
            verticalArrangement = Arrangement.spacedBy(4.dp)
        ) {
            items(selectedUris) { uri ->
                AsyncImage(
                    model = uri,
                    contentDescription = null,
                    modifier = Modifier.aspectRatio(1f),
                    contentScale = ContentScale.Crop
                )
            }
        }
    }
}
```

---

## 動画のみ選択

```kotlin
val launcher = rememberLauncherForActivityResult(
    ActivityResultContracts.PickVisualMedia()
) { uri -> /* 動画URI */ }

launcher.launch(
    PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.VideoOnly)
)
```

---

## MIMEタイプ指定

```kotlin
// GIFのみ
launcher.launch(
    PickVisualMediaRequest(
        ActivityResultContracts.PickVisualMedia.SingleMimeType("image/gif")
    )
)
```

---

## Photo Picker利用可否チェック

```kotlin
@Composable
fun PhotoPickerWithFallback() {
    val isPhotoPickerAvailable = ActivityResultContracts.PickVisualMedia.isPhotoPickerAvailable(LocalContext.current)

    if (isPhotoPickerAvailable) {
        // Photo Picker (権限不要)
        PhotoPickerButton()
    } else {
        // フォールバック: 従来のIntent
        LegacyImagePickerButton()
    }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| 単一選択 | `PickVisualMedia()` |
| 複数選択 | `PickMultipleVisualMedia(max)` |
| 画像のみ | `ImageOnly` |
| 動画のみ | `VideoOnly` |
| 両方 | `ImageAndVideo` |

- Photo Pickerは権限リクエスト不要
- Android 13+で標準搭載、11-12はGMS経由
- `PickMultipleVisualMedia`で複数選択
- `isPhotoPickerAvailable`で利用可否チェック

---

8種類のAndroidアプリテンプレート（メディア選択対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カメラ](https://zenn.dev/myougatheaxo/articles/android-compose-camera-capture-2026)
- [画像読み込みCoil](https://zenn.dev/myougatheaxo/articles/android-compose-image-loading-coil-2026)
- [パーミッション](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handling-2026)
