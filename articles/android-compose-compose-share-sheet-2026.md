---
title: "ShareSheet完全ガイド — テキスト/画像/ファイル共有/ShareSheetCompose"
emoji: "📤"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "share"]
published: true
---

## この記事で学べること

**ShareSheet**（テキスト共有、画像共有、ファイル共有、カスタムShareSheet）を解説します。

---

## テキスト共有

```kotlin
@Composable
fun ShareTextExample() {
    val context = LocalContext.current

    fun shareText(text: String) {
        val intent = Intent(Intent.ACTION_SEND).apply {
            type = "text/plain"
            putExtra(Intent.EXTRA_TEXT, text)
            putExtra(Intent.EXTRA_SUBJECT, "共有タイトル")
        }
        context.startActivity(Intent.createChooser(intent, "共有先を選択"))
    }

    Button(onClick = { shareText("Jetpack Composeは素晴らしい！") }) {
        Icon(Icons.Default.Share, null)
        Spacer(Modifier.width(8.dp))
        Text("テキストを共有")
    }
}
```

---

## 画像共有

```kotlin
@Composable
fun ShareImageExample() {
    val context = LocalContext.current

    fun shareImage(uri: Uri) {
        val intent = Intent(Intent.ACTION_SEND).apply {
            type = "image/*"
            putExtra(Intent.EXTRA_STREAM, uri)
            addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        }
        context.startActivity(Intent.createChooser(intent, "画像を共有"))
    }

    // 複数ファイル共有
    fun shareMultipleImages(uris: List<Uri>) {
        val intent = Intent(Intent.ACTION_SEND_MULTIPLE).apply {
            type = "image/*"
            putParcelableArrayListExtra(Intent.EXTRA_STREAM, ArrayList(uris))
            addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        }
        context.startActivity(Intent.createChooser(intent, "画像を共有"))
    }
}
```

---

## カスタムShareSheet

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CustomShareSheet(text: String, onDismiss: () -> Unit) {
    val context = LocalContext.current

    ModalBottomSheet(onDismissRequest = onDismiss) {
        Column(Modifier.padding(16.dp)) {
            Text("共有先", style = MaterialTheme.typography.titleLarge)
            Spacer(Modifier.height(16.dp))

            LazyVerticalGrid(columns = GridCells.Fixed(4), horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                items(listOf(
                    Triple("X", Icons.Default.Share, "com.twitter.android"),
                    Triple("LINE", Icons.Default.Chat, "jp.naver.line.android"),
                    Triple("コピー", Icons.Default.ContentCopy, "copy"),
                    Triple("その他", Icons.Default.MoreHoriz, "other")
                )) { (label, icon, action) ->
                    Column(
                        Modifier.clickable {
                            when (action) {
                                "copy" -> {
                                    val clipboard = context.getSystemService(Context.CLIPBOARD_SERVICE) as ClipboardManager
                                    clipboard.setPrimaryClip(ClipData.newPlainText("text", text))
                                }
                                else -> {
                                    val intent = Intent(Intent.ACTION_SEND).apply {
                                        type = "text/plain"
                                        putExtra(Intent.EXTRA_TEXT, text)
                                    }
                                    context.startActivity(Intent.createChooser(intent, null))
                                }
                            }
                            onDismiss()
                        }.padding(8.dp),
                        horizontalAlignment = Alignment.CenterHorizontally
                    ) {
                        Icon(icon, label)
                        Text(label, fontSize = 10.sp)
                    }
                }
            }
        }
    }
}
```

---

## まとめ

| 共有タイプ | Intent設定 |
|----------|-----------|
| テキスト | `text/plain` + `EXTRA_TEXT` |
| 画像 | `image/*` + `EXTRA_STREAM` |
| ファイル | `*/*` + `EXTRA_STREAM` |
| 複数 | `ACTION_SEND_MULTIPLE` |

- `Intent.createChooser`でシステムShareSheet表示
- `FileProvider`で安全なURI共有
- `FLAG_GRANT_READ_URI_PERMISSION`で読み取り権限
- カスタムBottomSheetで独自の共有UI

---

8種類のAndroidアプリテンプレート（共有機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Intent Filter](https://zenn.dev/myougatheaxo/articles/android-compose-intent-filter-2026)
- [BottomSheet](https://zenn.dev/myougatheaxo/articles/android-compose-compose-bottom-sheet-2026)
- [Clipboard](https://zenn.dev/myougatheaxo/articles/android-compose-compose-clipboard-2026)
