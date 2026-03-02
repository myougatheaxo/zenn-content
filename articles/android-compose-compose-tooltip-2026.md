---
title: "Tooltip完全ガイド — PlainTooltip/RichTooltip/TooltipBox"
emoji: "💬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Tooltip**（PlainTooltip、RichTooltip、TooltipBox、カスタムツールチップ）を解説します。

---

## PlainTooltip

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PlainTooltipExample() {
    val tooltipState = rememberTooltipState()

    TooltipBox(
        positionProvider = TooltipDefaults.rememberPlainTooltipPositionProvider(),
        tooltip = { PlainTooltip { Text("お気に入りに追加") } },
        state = tooltipState
    ) {
        IconButton(onClick = { /* action */ }) {
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
                title = { Text("データ同期") },
                action = {
                    TextButton(onClick = { scope.launch { tooltipState.dismiss() } }) {
                        Text("了解")
                    }
                }
            ) {
                Text("クラウドとデータを同期します。Wi-Fi接続時のみ実行されます。")
            }
        },
        state = tooltipState
    ) {
        IconButton(onClick = { /* action */ }) {
            Icon(Icons.Default.Sync, contentDescription = "同期")
        }
    }
}
```

---

## ツールバーTooltip

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ToolbarWithTooltips() {
    TopAppBar(
        title = { Text("エディタ") },
        actions = {
            TooltipBox(
                positionProvider = TooltipDefaults.rememberPlainTooltipPositionProvider(),
                tooltip = { PlainTooltip { Text("元に戻す") } },
                state = rememberTooltipState()
            ) {
                IconButton(onClick = {}) { Icon(Icons.Default.Undo, "元に戻す") }
            }

            TooltipBox(
                positionProvider = TooltipDefaults.rememberPlainTooltipPositionProvider(),
                tooltip = { PlainTooltip { Text("やり直す") } },
                state = rememberTooltipState()
            ) {
                IconButton(onClick = {}) { Icon(Icons.Default.Redo, "やり直す") }
            }

            TooltipBox(
                positionProvider = TooltipDefaults.rememberPlainTooltipPositionProvider(),
                tooltip = { PlainTooltip { Text("共有") } },
                state = rememberTooltipState()
            ) {
                IconButton(onClick = {}) { Icon(Icons.Default.Share, "共有") }
            }
        }
    )
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `PlainTooltip` | シンプルなテキストヒント |
| `RichTooltip` | タイトル+説明+アクション |
| `TooltipBox` | ツールチップ表示コンテナ |
| `rememberTooltipState` | 表示状態管理 |

- `PlainTooltip`でアイコンボタンの説明表示
- `RichTooltip`で詳細説明+アクションボタン
- `isPersistent = true`で手動閉じのみ
- 長押しで自動表示、アクセシビリティ対応

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Snackbar](https://zenn.dev/myougatheaxo/articles/android-compose-compose-snackbar-2026)
- [AlertDialog](https://zenn.dev/myougatheaxo/articles/android-compose-compose-alert-dialog-2026)
- [BottomSheet](https://zenn.dev/myougatheaxo/articles/android-compose-compose-bottom-sheet-2026)
