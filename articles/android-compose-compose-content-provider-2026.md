---
title: "Compose ContentProvider完全ガイド — コンテンツ共有/連絡先取得/メディア読み込み"
emoji: "📂"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "contentprovider"]
published: true
---

## この記事で学べること

**Compose ContentProvider**（ContentResolver、連絡先取得、メディアストア、コンテンツ共有）を解説します。

---

## 連絡先取得

```kotlin
@Composable
fun ContactListScreen() {
    val context = LocalContext.current
    var contacts by remember { mutableStateOf<List<Pair<String, String>>>(emptyList()) }
    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) {
            contacts = loadContacts(context)
        }
    }

    LaunchedEffect(Unit) {
        launcher.launch(Manifest.permission.READ_CONTACTS)
    }

    LazyColumn(Modifier.padding(16.dp)) {
        items(contacts) { (name, phone) ->
            ListItem(
                headlineContent = { Text(name) },
                supportingContent = { Text(phone) },
                leadingContent = { Icon(Icons.Default.Person, null) }
            )
        }
    }
}

fun loadContacts(context: Context): List<Pair<String, String>> {
    val contacts = mutableListOf<Pair<String, String>>()
    context.contentResolver.query(
        ContactsContract.CommonDataKinds.Phone.CONTENT_URI,
        arrayOf(ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME,
            ContactsContract.CommonDataKinds.Phone.NUMBER),
        null, null, ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME
    )?.use { cursor ->
        while (cursor.moveToNext()) {
            val name = cursor.getString(0)
            val phone = cursor.getString(1)
            contacts.add(name to phone)
        }
    }
    return contacts
}
```

---

## メディアストア画像一覧

```kotlin
@Composable
fun MediaGalleryScreen() {
    val context = LocalContext.current
    var images by remember { mutableStateOf<List<Uri>>(emptyList()) }

    LaunchedEffect(Unit) {
        images = loadImages(context)
    }

    LazyVerticalGrid(columns = GridCells.Fixed(3), Modifier.padding(4.dp),
        horizontalArrangement = Arrangement.spacedBy(4.dp),
        verticalArrangement = Arrangement.spacedBy(4.dp)) {
        items(images) { uri ->
            AsyncImage(model = uri, contentDescription = null,
                modifier = Modifier.aspectRatio(1f).clip(RoundedCornerShape(4.dp)),
                contentScale = ContentScale.Crop)
        }
    }
}

fun loadImages(context: Context): List<Uri> {
    val uris = mutableListOf<Uri>()
    context.contentResolver.query(
        MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
        arrayOf(MediaStore.Images.Media._ID),
        null, null, "${MediaStore.Images.Media.DATE_ADDED} DESC"
    )?.use { cursor ->
        val idColumn = cursor.getColumnIndexOrThrow(MediaStore.Images.Media._ID)
        while (cursor.moveToNext() && uris.size < 100) {
            val id = cursor.getLong(idColumn)
            uris.add(ContentUris.withAppendedId(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, id))
        }
    }
    return uris
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ContentResolver` | コンテンツ読み書き |
| `ContactsContract` | 連絡先アクセス |
| `MediaStore` | メディアファイル |
| `ContentUris` | URI生成 |

- `ContentResolver.query()`でデータ取得（Cursor使用）
- `use`ブロックでCursorを自動クローズ
- パーミッション取得後にデータアクセス
- `MediaStore`で画像/動画/音声にアクセス

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose FilePicker](https://zenn.dev/myougatheaxo/articles/android-compose-compose-file-picker-2026)
- [Compose ShareIntent](https://zenn.dev/myougatheaxo/articles/android-compose-compose-share-intent-2026)
- [Compose Permissions](https://zenn.dev/myougatheaxo/articles/android-compose-compose-permissions-2026)
