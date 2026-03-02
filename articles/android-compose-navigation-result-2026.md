---
title: "Navigation結果返却ガイド — savedStateHandle/previousBackStackEntry"
emoji: "↩️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

Navigation Composeでの**画面間データ返却**（savedStateHandle、previousBackStackEntry）を解説します。

---

## 結果の返却パターン

```kotlin
// 呼び出し元画面: 結果を受け取る
@Composable
fun HomeScreen(navController: NavController) {
    // 結果を監視
    val result = navController.currentBackStackEntry
        ?.savedStateHandle
        ?.getStateFlow<String?>("selectedItem", null)
        ?.collectAsStateWithLifecycle()

    LaunchedEffect(result?.value) {
        result?.value?.let { item ->
            // 結果を処理
            println("選択されたアイテム: $item")
            // 消費後にクリア
            navController.currentBackStackEntry?.savedStateHandle?.remove<String>("selectedItem")
        }
    }

    Column {
        Text("選択: ${result?.value ?: "なし"}")
        Button(onClick = { navController.navigate("picker") }) {
            Text("アイテムを選択")
        }
    }
}

// 遷移先画面: 結果を返す
@Composable
fun PickerScreen(navController: NavController) {
    val items = listOf("りんご", "みかん", "バナナ", "ぶどう")

    LazyColumn {
        items(items) { item ->
            ListItem(
                headlineContent = { Text(item) },
                modifier = Modifier.clickable {
                    navController.previousBackStackEntry
                        ?.savedStateHandle
                        ?.set("selectedItem", item)
                    navController.popBackStack()
                }
            )
        }
    }
}
```

---

## 複数の値を返却

```kotlin
@Serializable
data class FilterResult(
    val category: String,
    val minPrice: Int,
    val maxPrice: Int,
    val sortOrder: String
)

// フィルター画面
@Composable
fun FilterScreen(navController: NavController) {
    var category by remember { mutableStateOf("all") }
    var minPrice by remember { mutableIntStateOf(0) }
    var maxPrice by remember { mutableIntStateOf(10000) }

    Column(Modifier.padding(16.dp)) {
        // フィルターUI...

        Button(
            onClick = {
                val result = Json.encodeToString(
                    FilterResult(category, minPrice, maxPrice, "newest")
                )
                navController.previousBackStackEntry
                    ?.savedStateHandle
                    ?.set("filterResult", result)
                navController.popBackStack()
            },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("適用")
        }
    }
}

// 呼び出し元で受け取り
@Composable
fun ListScreen(navController: NavController) {
    val filterJson = navController.currentBackStackEntry
        ?.savedStateHandle
        ?.getStateFlow<String?>("filterResult", null)
        ?.collectAsStateWithLifecycle()

    val filter = remember(filterJson?.value) {
        filterJson?.value?.let { Json.decodeFromString<FilterResult>(it) }
    }

    // filterを使ってリストを表示
}
```

---

## ダイアログ画面からの返却

```kotlin
// ダイアログ画面
composable("color-picker") {
    ColorPickerDialog(
        onColorSelected = { color ->
            navController.previousBackStackEntry
                ?.savedStateHandle
                ?.set("selectedColor", color.toArgb())
            navController.popBackStack()
        },
        onDismiss = { navController.popBackStack() }
    )
}

// 呼び出し元
val colorArgb = navController.currentBackStackEntry
    ?.savedStateHandle
    ?.getStateFlow("selectedColor", Color.White.toArgb())
    ?.collectAsStateWithLifecycle()
```

---

## まとめ

- `previousBackStackEntry?.savedStateHandle?.set()`で結果設定
- `currentBackStackEntry?.savedStateHandle?.getStateFlow()`で結果受信
- `popBackStack()`で前画面に戻る
- 複雑なオブジェクトはJSON文字列で受け渡し
- 使用後は`remove()`で結果をクリア
- ダイアログ画面からも同パターンで返却可能

---

8種類のAndroidアプリテンプレート（Navigation設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-navigation-guide-2026)
- [Navigation引数ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-args-2026)
- [型安全Navigationガイド](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-typesafe-2026)
