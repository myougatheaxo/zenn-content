---
title: "Kotlin Scope Function完全ガイド — let/run/with/apply/also使い分け"
emoji: "🎯"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "scopefunction"]
published: true
---

## この記事で学べること

**Kotlin Scope Function**（let、run、with、apply、also、使い分けチャート）を解説します。

---

## 5つのスコープ関数

```kotlin
// let: nullチェック+変換（it参照、戻り値=ラムダの結果）
val name: String? = "太郎"
val greeting = name?.let { "こんにちは、${it}さん" } ?: "ゲスト"

// run: オブジェクトの初期化+計算（this参照、戻り値=ラムダの結果）
val result = StringBuilder().run {
    append("Hello")
    append(" ")
    append("World")
    toString()
} // "Hello World"

// with: 非null確定オブジェクトの操作（this参照、戻り値=ラムダの結果）
val config = Configuration()
val summary = with(config) {
    "$locale / $fontScale / $uiMode"
}

// apply: オブジェクトの設定（this参照、戻り値=オブジェクト自身）
val paint = Paint().apply {
    color = Color.RED
    strokeWidth = 5f
    style = Paint.Style.STROKE
}

// also: 副作用（it参照、戻り値=オブジェクト自身）
val numbers = mutableListOf(1, 2, 3).also {
    println("初期リスト: $it")
}.also {
    it.add(4)
}
```

---

## Android/Compose実践

```kotlin
// let: nullチェック
@Composable
fun UserProfile(user: User?) {
    user?.let {
        Column {
            Text(it.name)
            Text(it.email)
        }
    } ?: Text("ユーザーが見つかりません")
}

// apply: ビルダーパターン
val notification = NotificationCompat.Builder(context, "channel_id").apply {
    setSmallIcon(R.drawable.ic_notification)
    setContentTitle("通知タイトル")
    setContentText("通知内容")
    setAutoCancel(true)
}.build()

// also: デバッグログ
viewModelScope.launch {
    repository.getItems()
        .also { Log.d("TAG", "Items loaded: ${it.size}") }
        .filter { it.isActive }
        .also { Log.d("TAG", "Active items: ${it.size}") }
}

// run: 条件付き処理
val errorMessage = runCatching {
    apiService.fetchData()
}.getOrElse { e ->
    e.message ?: "不明なエラー"
}
```

---

## 使い分けチャート

```
┌─────────┬───────────┬────────────┐
│         │ 戻り値:   │ 戻り値:    │
│         │ ラムダ結果 │ オブジェクト │
├─────────┼───────────┼────────────┤
│ this参照 │ run/with  │ apply      │
├─────────┼───────────┼────────────┤
│ it参照   │ let       │ also       │
└─────────┴───────────┴────────────┘
```

---

## まとめ

| 関数 | 参照 | 戻り値 | 主な用途 |
|------|------|--------|----------|
| `let` | it | ラムダ結果 | null安全変換 |
| `run` | this | ラムダ結果 | 初期化+計算 |
| `with` | this | ラムダ結果 | 非nullの操作 |
| `apply` | this | 自身 | オブジェクト設定 |
| `also` | it | 自身 | 副作用（ログ等） |

- `let`: null安全(`?.let`)と型変換に
- `apply`: ビルダーパターンに最適
- `also`: デバッグログ挿入に便利
- `run`/`with`: 計算結果を返す処理に

---

8種類のAndroidアプリテンプレート（Kotlin対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Kotlin ExtensionFunction](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-extension-function-2026)
- [Kotlin SealedClass](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-sealed-class-2026)
- [Kotlin Delegation](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-delegation-2026)
