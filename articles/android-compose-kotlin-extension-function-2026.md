---
title: "Kotlin Extension Function完全ガイド — 拡張関数/拡張プロパティ/Compose拡張"
emoji: "🔌"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "extension"]
published: true
---

## この記事で学べること

**Kotlin Extension Function**（拡張関数、拡張プロパティ、Modifier拡張、Compose向け拡張）を解説します。

---

## 基本

```kotlin
// String拡張
fun String.toTitleCase(): String =
    split(" ").joinToString(" ") { it.replaceFirstChar { c -> c.uppercase() } }

// List拡張
fun <T> List<T>.secondOrNull(): T? = if (size >= 2) this[1] else null

// Context拡張
fun Context.showToast(message: String, duration: Int = Toast.LENGTH_SHORT) {
    Toast.makeText(this, message, duration).show()
}

// 使用
"hello world".toTitleCase() // "Hello World"
listOf(1, 2, 3).secondOrNull() // 2
context.showToast("保存しました")
```

---

## Compose Modifier拡張

```kotlin
// 条件付きModifier
fun Modifier.conditional(condition: Boolean, modifier: Modifier.() -> Modifier): Modifier =
    if (condition) modifier() else this

// シマーエフェクト
fun Modifier.shimmer(): Modifier = composed {
    val transition = rememberInfiniteTransition(label = "shimmer")
    val alpha by transition.animateFloat(
        initialValue = 0.3f, targetValue = 1f,
        animationSpec = infiniteRepeatable(tween(800), RepeatMode.Reverse),
        label = "shimmer"
    )
    background(Color.LightGray.copy(alpha = alpha), RoundedCornerShape(4.dp))
}

// 使用
Text("テスト",
    Modifier.conditional(isLoading) { shimmer() }
)
```

---

## Android固有の拡張

```kotlin
// dpをpxに変換
val Int.dpToPx: Int
    @Composable get() = with(LocalDensity.current) { this@dpToPx.dp.roundToPx() }

// Flow → Compose State
@Composable
fun <T> Flow<T>.collectAsStateWithDefault(default: T): State<T> =
    collectAsStateWithLifecycle(initialValue = default)

// 日付フォーマット
fun Long.toFormattedDate(pattern: String = "yyyy/MM/dd"): String =
    SimpleDateFormat(pattern, Locale.getDefault()).format(Date(this))

// ファイルサイズ
fun Long.toFileSize(): String = when {
    this < 1024 -> "${this}B"
    this < 1024 * 1024 -> "${"%.1f".format(this / 1024.0)}KB"
    this < 1024 * 1024 * 1024 -> "${"%.1f".format(this / (1024.0 * 1024))}MB"
    else -> "${"%.1f".format(this / (1024.0 * 1024 * 1024))}GB"
}

// 使用
System.currentTimeMillis().toFormattedDate() // "2026/03/02"
1234567L.toFileSize() // "1.2MB"
```

---

## まとめ

| パターン | 用途 |
|----------|------|
| 拡張関数 | 既存クラスにメソッド追加 |
| 拡張プロパティ | 既存クラスにプロパティ追加 |
| `composed` | Compose Modifier拡張 |
| `conditional` | 条件付きModifier |

- 拡張関数は元のクラスを変更せずにメソッド追加
- `composed { }`でCompose固有のModifier拡張
- 拡張プロパティでバッキングフィールドは持てない
- 実体はstaticメソッド（第1引数がレシーバ）

---

8種類のAndroidアプリテンプレート（Kotlin対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Kotlin ScopeFunction](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-scope-function-2026)
- [Kotlin Delegation](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-delegation-2026)
- [Compose CustomModifier](https://zenn.dev/myougatheaxo/articles/android-compose-compose-custom-modifier-2026)
