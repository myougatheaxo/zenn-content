---
title: "Kotlin拡張関数ベストプラクティス — 便利なユーティリティ関数の書き方"
emoji: "🔧"
type: "tech"
topics: ["android", "kotlin", "tips", "extension"]
published: true
---

## この記事で学べること

Kotlinの**拡張関数**を使って、Android開発で便利なユーティリティを作る方法を解説します。実務で使えるパターン集付き。

---

## 拡張関数の基本

```kotlin
// Stringに機能を追加
fun String.toCapitalizedWords(): String {
    return split(" ").joinToString(" ") { word ->
        word.replaceFirstChar { it.uppercase() }
    }
}

// 使い方
val title = "hello world".toCapitalizedWords() // "Hello World"
```

---

## Context拡張

```kotlin
// Toast表示
fun Context.showToast(message: String, duration: Int = Toast.LENGTH_SHORT) {
    Toast.makeText(this, message, duration).show()
}

// dp → px変換
fun Context.dpToPx(dp: Int): Int {
    return (dp * resources.displayMetrics.density).roundToInt()
}

// ネットワーク接続確認
fun Context.isNetworkAvailable(): Boolean {
    val manager = getSystemService<ConnectivityManager>()
    val network = manager?.activeNetwork ?: return false
    val capabilities = manager.getNetworkCapabilities(network) ?: return false
    return capabilities.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
}
```

---

## View/Modifier拡張

```kotlin
// 条件付きModifier
fun Modifier.conditionalModifier(
    condition: Boolean,
    modifier: Modifier.() -> Modifier
): Modifier = if (condition) this.modifier() else this

// 使い方
Text(
    "テキスト",
    modifier = Modifier
        .conditionalModifier(isHighlighted) {
            background(Color.Yellow)
        }
)
```

---

## Flow拡張

```kotlin
// ネットワークエラーのリトライ
fun <T> Flow<T>.retryWithDelay(
    maxRetries: Int = 3,
    delayMillis: Long = 1000
): Flow<T> = retry(maxRetries.toLong()) { cause ->
    if (cause is IOException) {
        delay(delayMillis)
        true
    } else {
        false
    }
}

// 使い方
repository.observeData()
    .retryWithDelay(maxRetries = 3, delayMillis = 2000)
    .collect { data -> /* ... */ }
```

---

## 日付・時刻の拡張

```kotlin
// LocalDateTimeを表示用文字列に変換
fun LocalDateTime.toDisplayString(): String {
    val formatter = DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm")
    return format(formatter)
}

// 相対時間（「3時間前」「昨日」など）
fun LocalDateTime.toRelativeTimeString(): String {
    val now = LocalDateTime.now()
    val duration = Duration.between(this, now)

    return when {
        duration.toMinutes() < 1 -> "たった今"
        duration.toMinutes() < 60 -> "${duration.toMinutes()}分前"
        duration.toHours() < 24 -> "${duration.toHours()}時間前"
        duration.toDays() < 7 -> "${duration.toDays()}日前"
        else -> toDisplayString()
    }
}
```

---

## コレクション拡張

```kotlin
// nullと空を同時チェック
fun <T> List<T>?.isNullOrEmpty(): Boolean = this == null || isEmpty()

// 安全なインデックスアクセス
fun <T> List<T>.getOrDefault(index: Int, default: T): T {
    return if (index in indices) get(index) else default
}

// グループ化して集計
fun <T, K> List<T>.groupAndCount(keySelector: (T) -> K): Map<K, Int> {
    return groupBy(keySelector).mapValues { it.value.size }
}
```

---

## スコープ関数との組み合わせ

```kotlin
// SharedPreferencesの簡易ラッパー
fun SharedPreferences.edit(block: SharedPreferences.Editor.() -> Unit) {
    val editor = edit()
    editor.block()
    editor.apply()
}

// 使い方
prefs.edit {
    putString("name", "みょうが")
    putInt("age", 2)
}
```

---

## 拡張関数のルール

| ルール | 理由 |
|--------|------|
| 既存APIと衝突しない名前 | 混乱を防ぐ |
| 1つの責任だけ持つ | テストしやすさ |
| 副作用を最小限に | 予測可能性 |
| 汎用的すぎない | 過剰抽象化を避ける |
| ファイル名は`XxxExtensions.kt` | 発見しやすさ |

---

## まとめ

- Context拡張: Toast、dp変換、ネットワーク確認
- Modifier拡張: 条件付き装飾
- Flow拡張: リトライ、エラーハンドリング
- 日付拡張: フォーマット、相対時間
- コレクション拡張: 安全なアクセス

---

8種類のAndroidアプリテンプレート（便利な拡張関数付き）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Kotlin基礎10パターン](https://zenn.dev/myougatheaxo/articles/kotlin-basics-for-ai-code-2026)
- [Kotlin sealed class完全ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-class-2026)
- [Kotlin Coroutines & Flow入門](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-flow-2026)
