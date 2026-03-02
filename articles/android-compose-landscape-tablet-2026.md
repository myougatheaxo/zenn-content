---
title: "横画面/タブレット対応完全ガイド — WindowSizeClass/適応レイアウト/2ペイン"
emoji: "📐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "responsive"]
published: true
---

## この記事で学べること

**横画面/タブレット対応**（WindowSizeClass、適応レイアウト、2ペイン表示、画面回転対応）を解説します。

---

## WindowSizeClass

```kotlin
@Composable
fun AdaptiveApp() {
    val windowSizeClass = calculateWindowSizeClass(LocalContext.current as Activity)

    when (windowSizeClass.widthSizeClass) {
        WindowWidthSizeClass.Compact -> PhoneLayout()      // スマホ縦
        WindowWidthSizeClass.Medium -> TabletPortrait()     // タブレット縦/スマホ横
        WindowWidthSizeClass.Expanded -> TabletLandscape()  // タブレット横
    }
}
```

---

## 2ペインレイアウト

```kotlin
@Composable
fun ListDetailLayout(
    items: List<Item>,
    selectedItem: Item?,
    onItemSelect: (Item) -> Unit
) {
    val windowSizeClass = calculateWindowSizeClass(LocalContext.current as Activity)
    val isExpanded = windowSizeClass.widthSizeClass == WindowWidthSizeClass.Expanded

    if (isExpanded) {
        // タブレット: 横並び2ペイン
        Row(Modifier.fillMaxSize()) {
            ItemList(
                items = items,
                selectedId = selectedItem?.id,
                onItemClick = onItemSelect,
                modifier = Modifier.weight(0.4f)
            )
            VerticalDivider()
            if (selectedItem != null) {
                ItemDetail(item = selectedItem, modifier = Modifier.weight(0.6f))
            } else {
                EmptyState(modifier = Modifier.weight(0.6f))
            }
        }
    } else {
        // スマホ: Navigation切替
        val navController = rememberNavController()
        NavHost(navController, startDestination = "list") {
            composable("list") {
                ItemList(items = items, onItemClick = { item ->
                    onItemSelect(item)
                    navController.navigate("detail/${item.id}")
                })
            }
            composable("detail/{id}") {
                selectedItem?.let { ItemDetail(item = it) }
            }
        }
    }
}
```

---

## 画面回転対応

```kotlin
@Composable
fun OrientationAwareLayout(content: @Composable () -> Unit) {
    val configuration = LocalConfiguration.current
    val isLandscape = configuration.orientation == Configuration.ORIENTATION_LANDSCAPE

    if (isLandscape) {
        Row(Modifier.fillMaxSize()) {
            // 横画面レイアウト
            content()
        }
    } else {
        Column(Modifier.fillMaxSize()) {
            // 縦画面レイアウト
            content()
        }
    }
}

// フォーム例: 横画面で2列
@Composable
fun AdaptiveForm() {
    val isLandscape = LocalConfiguration.current.orientation == Configuration.ORIENTATION_LANDSCAPE

    if (isLandscape) {
        Row(Modifier.fillMaxWidth().padding(16.dp), horizontalArrangement = Arrangement.spacedBy(16.dp)) {
            Column(Modifier.weight(1f)) {
                OutlinedTextField(value = "", onValueChange = {}, label = { Text("名前") }, modifier = Modifier.fillMaxWidth())
                OutlinedTextField(value = "", onValueChange = {}, label = { Text("メール") }, modifier = Modifier.fillMaxWidth())
            }
            Column(Modifier.weight(1f)) {
                OutlinedTextField(value = "", onValueChange = {}, label = { Text("電話") }, modifier = Modifier.fillMaxWidth())
                Button(onClick = {}, modifier = Modifier.fillMaxWidth()) { Text("送信") }
            }
        }
    } else {
        Column(Modifier.fillMaxWidth().padding(16.dp)) {
            OutlinedTextField(value = "", onValueChange = {}, label = { Text("名前") }, modifier = Modifier.fillMaxWidth())
            OutlinedTextField(value = "", onValueChange = {}, label = { Text("メール") }, modifier = Modifier.fillMaxWidth())
            OutlinedTextField(value = "", onValueChange = {}, label = { Text("電話") }, modifier = Modifier.fillMaxWidth())
            Button(onClick = {}, modifier = Modifier.fillMaxWidth()) { Text("送信") }
        }
    }
}
```

---

## まとめ

| サイズクラス | 端末 | レイアウト |
|-------------|------|-----------|
| Compact | スマホ縦 | 1カラム |
| Medium | タブレット縦 | 適応 |
| Expanded | タブレット横 | 2ペイン |

- `WindowSizeClass`で端末サイズに応じたレイアウト切替
- 2ペインでタブレットのワイド画面を活用
- `LocalConfiguration`で画面回転を検知
- フォームは横画面で2列配置がUX良好

---

8種類のAndroidアプリテンプレート（タブレット対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3 Adaptive](https://zenn.dev/myougatheaxo/articles/android-compose-material3-adaptive-2026)
- [WindowInsets](https://zenn.dev/myougatheaxo/articles/android-compose-window-insets-2026)
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-type-safe-navigation-2026)
