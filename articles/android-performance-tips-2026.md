---
title: "Androidパフォーマンス最適化10選 — 遅いアプリを速くするテクニック"
emoji: "⚡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "performance"]
published: true
---

## この記事で学べること

アプリが重い・カクつく原因と、**Composeでの具体的な最適化テクニック**10個を解説します。

---

## 1. LazyColumnにkeyを付ける

```kotlin
// ❌ keyなし（全アイテム再コンポジション）
LazyColumn {
    items(users) { user -> UserItem(user) }
}

// ✅ key付き（変更アイテムだけ再コンポジション）
LazyColumn {
    items(users, key = { it.id }) { user -> UserItem(user) }
}
```

---

## 2. remember で再計算を防ぐ

```kotlin
// ❌ 毎回フィルタリング
@Composable
fun FilteredList(items: List<Item>, query: String) {
    val filtered = items.filter { it.name.contains(query) }
}

// ✅ 値が変わったときだけ
@Composable
fun FilteredList(items: List<Item>, query: String) {
    val filtered = remember(items, query) {
        items.filter { it.name.contains(query) }
    }
}
```

---

## 3. derivedStateOf で不要な再コンポジションを防ぐ

```kotlin
val listState = rememberLazyListState()

// ❌ スクロールのたびに再コンポジション
val showButton = listState.firstVisibleItemIndex > 0

// ✅ 値が変わったときだけ
val showButton by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 0 }
}
```

---

## 4. Stableアノテーション

```kotlin
// Composeが安定と判断できないクラス
@Stable
data class UserState(
    val name: String,
    val items: List<String>  // Listは不安定と判断される
)
```

`@Stable`を付けると、Composeが**スキップ可能**と判断し、不要な再コンポジションを防ぎます。

---

## 5. 画像の最適化

```kotlin
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(imageUrl)
        .size(Size(200, 200))    // 必要なサイズだけデコード
        .crossfade(true)
        .build(),
    contentDescription = null
)
```

フル解像度の画像をそのまま表示するとメモリを大量消費します。

---

## 6. collectAsStateWithLifecycle

```kotlin
// ❌ バックグラウンドでも購読
val data by viewModel.data.collectAsState()

// ✅ 画面表示時のみ購読
val data by viewModel.data.collectAsStateWithLifecycle()
```

---

## 7. Dispatchers.IOで重い処理

```kotlin
// ❌ メインスレッドでDB操作
fun loadData() {
    val data = database.query()
}

// ✅ IOスレッドで実行
suspend fun loadData() = withContext(Dispatchers.IO) {
    database.query()
}
```

---

## 8. Baseline Profiles

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.profileinstaller:profileinstaller:1.3.1")
}
```

Baseline Profilesを追加すると、**初回起動速度が最大40%向上**します。AOT（事前コンパイル）で頻繁に使われるコードパスを最適化。

---

## 9. R8でコードサイズ削減

```kotlin
buildTypes {
    release {
        isMinifyEnabled = true
        isShrinkResources = true
    }
}
```

APKサイズが小さいほど、ダウンロード速度もアプリ起動速度も向上します。

---

## 10. Compose Compiler Metrics

```kotlin
// gradle.properties
kotlin.compose.compiler.metrics=true
kotlin.compose.compiler.reports=true
```

```bash
./gradlew assembleRelease
# 出力: app/build/compose_metrics/
```

どのComposableが**スキップ可能/不可能**かを確認できます。

---

## チェックリスト

| 項目 | 効果 |
|------|------|
| LazyColumnにkey | リスト再コンポジション削減 |
| remember/derivedStateOf | 不要な再計算防止 |
| collectAsStateWithLifecycle | バッテリー節約 |
| Dispatchers.IO | ANR防止 |
| 画像サイズ指定 | メモリ削減 |
| R8有効化 | APKサイズ削減 |
| Baseline Profiles | 起動速度向上 |

---

## まとめ

1. `key`パラメータでリスト最適化
2. `remember`/`derivedStateOf`で再計算防止
3. `@Stable`で再コンポジションスキップ
4. 画像サイズを指定してメモリ節約
5. `collectAsStateWithLifecycle`でバッテリー最適化
6. `Dispatchers.IO`でメインスレッドを解放
7. Baseline Profilesで初回起動高速化

---

8種類のAndroidアプリテンプレート（パフォーマンス最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [APKサイズ最適化](https://zenn.dev/myougatheaxo/articles/android-apk-optimization-2026)
- [Composeの状態管理入門](https://zenn.dev/myougatheaxo/articles/compose-state-management-2026)
- [LazyColumn完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazycolumn-guide-2026)
