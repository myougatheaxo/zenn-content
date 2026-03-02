---
title: "Compose Profiler完全ガイド — Layout Inspector/リコンポジション追跡/パフォーマンス計測"
emoji: "📊"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "performance"]
published: true
---

## この記事で学べること

**Compose Profiler**（Layout Inspector、リコンポジション回数追跡、Composition Tracing、パフォーマンス最適化）を解説します。

---

## Layout Inspector設定

```groovy
// build.gradle（デバッグビルドでCompose情報を有効化）
android {
    buildTypes {
        debug {
            // Layout Inspectorでのリコンポジションハイライト
        }
    }
}

// gradle.properties
// Compose CompilerのソースマップをLayout Inspectorに渡す
android.enableComposeCompilerReport=true
```

```kotlin
// Android Studio:
// View > Tool Windows > Layout Inspector
// Compose Tree表示 → リコンポジション回数を確認
```

---

## リコンポジション最適化

```kotlin
// ❌ リスト全体がリコンポジション
@Composable
fun BadList(items: List<String>, selectedIndex: Int) {
    Column {
        items.forEachIndexed { index, item ->
            Text(item, color = if (index == selectedIndex) Color.Red else Color.Black)
        }
    }
}

// ✅ key指定 + derivedStateOfでスコープ制御
@Composable
fun GoodList(items: List<String>, selectedIndex: Int) {
    LazyColumn {
        itemsIndexed(items, key = { _, item -> item }) { index, item ->
            val isSelected by remember(selectedIndex) {
                derivedStateOf { index == selectedIndex }
            }
            Text(item, color = if (isSelected) Color.Red else Color.Black)
        }
    }
}

// ✅ ラムダでState読み取り遅延
@Composable
fun AnimatedBox() {
    val offset by animateFloatAsState(targetValue = 100f, label = "offset")

    // ❌ Compositionフェーズで読み取り
    // Box(Modifier.offset(x = offset.dp))

    // ✅ Layoutフェーズで読み取り（リコンポジションなし）
    Box(Modifier.offset { IntOffset(offset.roundToInt(), 0) })
}
```

---

## Composition Tracing

```groovy
// build.gradle
dependencies {
    implementation("androidx.compose.runtime:runtime-tracing:1.0.0-beta01")
}
```

```kotlin
// System Traceで確認
// 1. Android Studio > Profiler > CPU
// 2. System Trace を選択
// 3. Record
// 4. Compose関数名がトレースに表示される
```

---

## まとめ

| ツール | 用途 |
|--------|------|
| Layout Inspector | Composeツリー可視化 |
| Composition Tracing | Compose関数トレース |
| `derivedStateOf` | 不要リコンポジション回避 |
| ラムダ遅延読み取り | Layoutフェーズ最適化 |

- Layout Inspectorでリコンポジション回数を確認
- `key`指定でリスト要素の安定的な識別
- `derivedStateOf`で計算結果のキャッシュ
- `Modifier.offset { }`でLayoutフェーズに遅延

---

8種類のAndroidアプリテンプレート（パフォーマンス最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Lint](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lint-2026)
- [Compose BaselineProfile](https://zenn.dev/myougatheaxo/articles/android-compose-compose-baseline-profile-2026)
- [Compose Benchmark](https://zenn.dev/myougatheaxo/articles/android-compose-compose-benchmark-2026)
