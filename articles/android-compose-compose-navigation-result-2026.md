---
title: "Navigation Result完全ガイド — 画面間データ返却/SavedStateHandle/previousBackStackEntry"
emoji: "↩️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

**Navigation Result**（画面間のデータ返却、SavedStateHandle、previousBackStackEntry、型安全なResult）を解説します。

---

## SavedStateHandleでResult返却

```kotlin
// 選択画面: 結果をセット
@Composable
fun ColorPickerScreen(navController: NavController) {
    val colors = listOf(Color.Red, Color.Blue, Color.Green, Color.Yellow)

    LazyColumn {
        items(colors) { color ->
            Box(
                Modifier
                    .fillMaxWidth()
                    .height(60.dp)
                    .background(color)
                    .clickable {
                        navController.previousBackStackEntry
                            ?.savedStateHandle
                            ?.set("selected_color", color.toArgb())
                        navController.popBackStack()
                    }
            )
        }
    }
}

// 呼び出し側: 結果を受け取る
@Composable
fun SettingsScreen(navController: NavController) {
    val savedStateHandle = navController.currentBackStackEntry?.savedStateHandle
    val selectedColor = savedStateHandle
        ?.getStateFlow("selected_color", Color.White.toArgb())
        ?.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp)) {
        Box(
            Modifier
                .size(80.dp)
                .background(Color(selectedColor?.value ?: Color.White.toArgb()))
                .clickable { navController.navigate("color_picker") }
        )
        Text("タップしてカラーを選択")
    }
}
```

---

## 型安全なResultパターン

```kotlin
// Result用のデータクラス
@Serializable
data class PickerResult(
    val itemId: Int,
    val itemName: String
)

// 結果をセット
fun NavController.setResult(key: String, result: PickerResult) {
    previousBackStackEntry?.savedStateHandle?.set(key, Json.encodeToString(result))
}

// 結果を取得
@Composable
fun <T> NavController.getResult(key: String, deserializer: DeserializationStrategy<T>): T? {
    val json = currentBackStackEntry?.savedStateHandle
        ?.getStateFlow<String?>(key, null)
        ?.collectAsStateWithLifecycle()
    return json?.value?.let { Json.decodeFromString(deserializer, it) }
}
```

---

## ダイアログResult

```kotlin
@Composable
fun ConfirmDeleteDialog(navController: NavController) {
    AlertDialog(
        onDismissRequest = { navController.popBackStack() },
        title = { Text("削除確認") },
        text = { Text("本当に削除しますか？") },
        confirmButton = {
            TextButton(onClick = {
                navController.previousBackStackEntry
                    ?.savedStateHandle
                    ?.set("delete_confirmed", true)
                navController.popBackStack()
            }) { Text("削除") }
        },
        dismissButton = {
            TextButton(onClick = { navController.popBackStack() }) { Text("キャンセル") }
        }
    )
}
```

---

## まとめ

| 手法 | 用途 |
|------|------|
| `savedStateHandle.set` | 結果をセット |
| `previousBackStackEntry` | 前画面に返却 |
| `getStateFlow` | Flowで結果受信 |
| JSON Serialization | 型安全なResult |

- `previousBackStackEntry?.savedStateHandle`で前画面にデータ返却
- `getStateFlow`でリアクティブに結果を受信
- JSON Serializationで型安全なデータ受け渡し
- ダイアログの確認結果も同じパターンで返却

---

8種類のAndroidアプリテンプレート（Navigation設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Type-Safe Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-type-safe-navigation-2026)
- [BottomNavigation](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-navigation-2026)
- [BottomSheet/Dialog](https://zenn.dev/myougatheaxo/articles/android-compose-bottomsheet-dialog-2026)
