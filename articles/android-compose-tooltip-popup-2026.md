---
title: "Tooltip/Popupガイド — ComposeでのフローティングUI"
emoji: "💡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Composeの**Tooltip/Popup/DropdownMenu**（ツールチップ、ポップアップ、ドロップダウン）を解説します。

---

## PlainTooltip

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TooltipExample() {
    val tooltipState = rememberTooltipState()

    TooltipBox(
        positionProvider = TooltipDefaults.rememberPlainTooltipPositionProvider(),
        tooltip = {
            PlainTooltip { Text("お気に入りに追加") }
        },
        state = tooltipState
    ) {
        IconButton(onClick = { /* お気に入り処理 */ }) {
            Icon(Icons.Default.Favorite, contentDescription = "お気に入り")
        }
    }
}
```

---

## RichTooltip

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun RichTooltipExample() {
    val tooltipState = rememberTooltipState(isPersistent = true)
    val scope = rememberCoroutineScope()

    TooltipBox(
        positionProvider = TooltipDefaults.rememberRichTooltipPositionProvider(),
        tooltip = {
            RichTooltip(
                title = { Text("プレミアム機能") },
                action = {
                    TextButton(onClick = { scope.launch { tooltipState.dismiss() } }) {
                        Text("詳しく見る")
                    }
                }
            ) {
                Text("この機能はプレミアムプランで利用できます。")
            }
        },
        state = tooltipState
    ) {
        IconButton(onClick = { scope.launch { tooltipState.show() } }) {
            Icon(Icons.Default.Info, contentDescription = "情報")
        }
    }
}
```

---

## DropdownMenu

```kotlin
@Composable
fun DropdownMenuExample() {
    var expanded by remember { mutableStateOf(false) }

    Box {
        IconButton(onClick = { expanded = true }) {
            Icon(Icons.Default.MoreVert, "メニュー")
        }

        DropdownMenu(
            expanded = expanded,
            onDismissRequest = { expanded = false }
        ) {
            DropdownMenuItem(
                text = { Text("編集") },
                leadingIcon = { Icon(Icons.Default.Edit, null) },
                onClick = { expanded = false }
            )
            DropdownMenuItem(
                text = { Text("共有") },
                leadingIcon = { Icon(Icons.Default.Share, null) },
                onClick = { expanded = false }
            )
            HorizontalDivider()
            DropdownMenuItem(
                text = { Text("削除", color = MaterialTheme.colorScheme.error) },
                leadingIcon = { Icon(Icons.Default.Delete, null, tint = MaterialTheme.colorScheme.error) },
                onClick = { expanded = false }
            )
        }
    }
}
```

---

## Popup

```kotlin
@Composable
fun PopupExample() {
    var showPopup by remember { mutableStateOf(false) }

    Box {
        Button(onClick = { showPopup = true }) {
            Text("ポップアップ表示")
        }

        if (showPopup) {
            Popup(
                alignment = Alignment.BottomCenter,
                offset = IntOffset(0, 20),
                onDismissRequest = { showPopup = false }
            ) {
                Card(
                    elevation = CardDefaults.cardElevation(8.dp),
                    modifier = Modifier.padding(16.dp)
                ) {
                    Column(Modifier.padding(16.dp)) {
                        Text("ポップアップ内容", style = MaterialTheme.typography.titleSmall)
                        Spacer(Modifier.height(8.dp))
                        Text("ここに追加情報を表示")
                        TextButton(onClick = { showPopup = false }) {
                            Text("閉じる")
                        }
                    }
                }
            }
        }
    }
}
```

---

## まとめ

- `PlainTooltip`でシンプルなツールチップ
- `RichTooltip`でタイトル+アクション付きツールチップ
- `DropdownMenu`でコンテキストメニュー
- `Popup`でカスタムフローティングUI
- `rememberTooltipState(isPersistent = true)`で永続表示
- `onDismissRequest`で外部タップ時の閉じ処理

---

8種類のAndroidアプリテンプレート（UI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [BottomSheetガイド](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-sheet-modal-2026)
- [Snackbar/Toast](https://zenn.dev/myougatheaxo/articles/android-compose-snackbar-toast-2026)
- [Material3アイコン](https://zenn.dev/myougatheaxo/articles/android-compose-material3-icons-2026)
