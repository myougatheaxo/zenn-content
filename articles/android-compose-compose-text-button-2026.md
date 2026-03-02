---
title: "Compose TextButton完全ガイド — TextButton/OutlinedButton/ElevatedButton/使い分け"
emoji: "🔤"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose TextButton**（TextButton、OutlinedButton、ElevatedButton、M3ボタン使い分け）を解説します。

---

## M3ボタン5種類

```kotlin
@Composable
fun ButtonVariantsDemo() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // 1. Filled（最も強調）
        Button(onClick = {}) { Text("保存する") }

        // 2. FilledTonal（中程度の強調）
        FilledTonalButton(onClick = {}) { Text("下書き保存") }

        // 3. Elevated（影付き）
        ElevatedButton(onClick = {}) { Text("エクスポート") }

        // 4. Outlined（枠線のみ）
        OutlinedButton(onClick = {}) { Text("キャンセル") }

        // 5. Text（最も控えめ）
        TextButton(onClick = {}) { Text("詳細を見る") }
    }
}
```

---

## アイコン付きボタン

```kotlin
@Composable
fun IconWithButtonDemo() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        Button(onClick = {}, contentPadding = ButtonDefaults.ButtonWithIconContentPadding) {
            Icon(Icons.Default.Send, null, Modifier.size(ButtonDefaults.IconSize))
            Spacer(Modifier.size(ButtonDefaults.IconSpacing))
            Text("送信")
        }

        OutlinedButton(onClick = {}) {
            Icon(Icons.Default.Download, null, Modifier.size(ButtonDefaults.IconSize))
            Spacer(Modifier.size(ButtonDefaults.IconSpacing))
            Text("ダウンロード")
        }

        TextButton(onClick = {}) {
            Text("もっと見る")
            Icon(Icons.Default.ArrowForward, null, Modifier.size(ButtonDefaults.IconSize))
        }
    }
}
```

---

## ダイアログでの使い分け

```kotlin
@Composable
fun DialogButtonPattern() {
    var showDialog by remember { mutableStateOf(false) }

    Button(onClick = { showDialog = true }) { Text("削除") }

    if (showDialog) {
        AlertDialog(
            onDismissRequest = { showDialog = false },
            title = { Text("確認") },
            text = { Text("このアイテムを削除しますか？") },
            confirmButton = {
                // 破壊的アクション → FilledButton
                Button(
                    onClick = { showDialog = false },
                    colors = ButtonDefaults.buttonColors(
                        containerColor = MaterialTheme.colorScheme.error
                    )
                ) { Text("削除") }
            },
            dismissButton = {
                // キャンセル → TextButton
                TextButton(onClick = { showDialog = false }) { Text("キャンセル") }
            }
        )
    }
}
```

---

## まとめ

| API | 強調度 | 用途 |
|-----|--------|------|
| `Button` | 最高 | メインアクション |
| `FilledTonalButton` | 高 | サブアクション |
| `ElevatedButton` | 中 | 補助的アクション |
| `OutlinedButton` | 低 | キャンセル等 |
| `TextButton` | 最低 | リンク的アクション |

- M3では5種類のボタンを強調度で使い分け
- ダイアログでは確認=Button、キャンセル=TextButton
- `ButtonDefaults.IconSize`/`IconSpacing`でアイコン付き
- 破壊的アクションは`colorScheme.error`カラーを使用

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose IconButton](https://zenn.dev/myougatheaxo/articles/android-compose-compose-icon-button-2026)
- [Compose FAB](https://zenn.dev/myougatheaxo/articles/android-compose-compose-fab-2026)
- [Compose Dialog](https://zenn.dev/myougatheaxo/articles/android-compose-compose-dialog-2026)
