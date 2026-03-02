---
title: "キーボードショートカット完全ガイド — 物理キーボード/ショートカットヘルパー/IME制御"
emoji: "⌨️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**キーボードショートカット**（物理キーボード対応、Modifier+Key、ショートカットヘルパー、IME制御）を解説します。

---

## キーボードイベント検知

```kotlin
@Composable
fun KeyboardShortcutScreen() {
    var lastAction by remember { mutableStateOf("なし") }

    Box(
        Modifier
            .fillMaxSize()
            .focusable()
            .onPreviewKeyEvent { event ->
                if (event.type == KeyEventType.KeyDown) {
                    when {
                        event.isCtrlPressed && event.key == Key.S -> {
                            lastAction = "保存 (Ctrl+S)"
                            true
                        }
                        event.isCtrlPressed && event.key == Key.Z -> {
                            lastAction = "元に戻す (Ctrl+Z)"
                            true
                        }
                        event.isCtrlPressed && event.isShiftPressed && event.key == Key.Z -> {
                            lastAction = "やり直し (Ctrl+Shift+Z)"
                            true
                        }
                        event.key == Key.Escape -> {
                            lastAction = "キャンセル (Esc)"
                            true
                        }
                        else -> false
                    }
                } else false
            }
    ) {
        Text("最後の操作: $lastAction", Modifier.align(Alignment.Center))
    }
}
```

---

## ショートカットヘルパー

```kotlin
data class ShortcutInfo(
    val keys: String,
    val description: String
)

@Composable
fun ShortcutHelperDialog(
    shortcuts: List<ShortcutInfo>,
    onDismiss: () -> Unit
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("キーボードショートカット") },
        text = {
            LazyColumn {
                items(shortcuts) { shortcut ->
                    Row(
                        Modifier.fillMaxWidth().padding(vertical = 4.dp),
                        horizontalArrangement = Arrangement.SpaceBetween
                    ) {
                        Text(shortcut.description)
                        Surface(
                            color = MaterialTheme.colorScheme.secondaryContainer,
                            shape = RoundedCornerShape(4.dp)
                        ) {
                            Text(
                                shortcut.keys,
                                Modifier.padding(horizontal = 8.dp, vertical = 2.dp),
                                style = MaterialTheme.typography.labelMedium
                            )
                        }
                    }
                }
            }
        },
        confirmButton = { TextButton(onClick = onDismiss) { Text("閉じる") } }
    )
}
```

---

## IME制御

```kotlin
@Composable
fun ImeControlExample() {
    val focusManager = LocalFocusManager.current
    val keyboardController = LocalSoftwareKeyboardController.current
    var text by remember { mutableStateOf("") }

    OutlinedTextField(
        value = text,
        onValueChange = { text = it },
        label = { Text("検索") },
        keyboardOptions = KeyboardOptions(imeAction = ImeAction.Search),
        keyboardActions = KeyboardActions(
            onSearch = {
                keyboardController?.hide()
                focusManager.clearFocus()
                // 検索実行
            }
        ),
        modifier = Modifier.fillMaxWidth()
    )
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| キーイベント | `onPreviewKeyEvent` |
| Modifier検知 | `isCtrlPressed`, `isShiftPressed` |
| IME制御 | `LocalSoftwareKeyboardController` |
| フォーカス | `LocalFocusManager` |

- `onPreviewKeyEvent`で物理キーボードショートカットを処理
- ショートカットヘルパーダイアログでユーザー案内
- `KeyboardActions`でIMEアクションをカスタマイズ
- タブレット/ChromeOS対応に必須

---

8種類のAndroidアプリテンプレート（キーボード対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [横画面/タブレット](https://zenn.dev/myougatheaxo/articles/android-compose-landscape-tablet-2026)
- [テキスト入力](https://zenn.dev/myougatheaxo/articles/android-compose-text-selection-2026)
- [フォーカス管理](https://zenn.dev/myougatheaxo/articles/android-compose-focus-management-2026)
