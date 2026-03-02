---
title: "Compose Lint完全ガイド — カスタムLintルール/Compose固有チェック/CI統合"
emoji: "🔍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "lint"]
published: true
---

## この記事で学べること

**Compose Lint**（Compose固有Lintルール、パフォーマンスチェック、カスタムルール、CI統合）を解説します。

---

## Compose Lint設定

```groovy
// build.gradle
android {
    lint {
        warningsAsErrors = true
        abortOnError = true
        checkDependencies = true
    }
}

// Compose Compiler設定（安定性レポート）
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_reports")
    metricsDestination = layout.buildDirectory.dir("compose_metrics")
}
```

---

## よくあるLint警告と対策

```kotlin
// ❌ 不安定なパラメータ（リコンポジション発生）
@Composable
fun UserList(users: List<User>) { // List = 不安定
    LazyColumn {
        items(users) { Text(it.name) }
    }
}

// ✅ 安定化: ImmutableListを使用
@Immutable
data class User(val id: String, val name: String)

@Composable
fun UserList(users: ImmutableList<User>) { // ImmutableList = 安定
    LazyColumn {
        items(users) { Text(it.name) }
    }
}

// ❌ Composable内でのオブジェクト生成
@Composable
fun BadExample() {
    val formatter = SimpleDateFormat("yyyy/MM/dd") // 毎回生成
    Text(formatter.format(Date()))
}

// ✅ rememberで保持
@Composable
fun GoodExample() {
    val formatter = remember { SimpleDateFormat("yyyy/MM/dd", Locale.getDefault()) }
    Text(formatter.format(Date()))
}
```

---

## lint.xml設定

```xml
<!-- lint.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<lint>
    <!-- Compose固有 -->
    <issue id="UnrememberedMutableState" severity="error" />
    <issue id="UnrememberedGetBackStackEntry" severity="error" />
    <issue id="CoroutineCreationDuringComposition" severity="error" />

    <!-- パフォーマンス -->
    <issue id="FrequentlyChangedStateReadInComposition" severity="warning" />

    <!-- プロジェクト固有で無視 -->
    <issue id="MissingTranslation" severity="ignore" />
</lint>
```

---

## まとめ

| 設定 | 用途 |
|------|------|
| `composeCompiler.reportsDestination` | 安定性レポート |
| `@Immutable` | 安定クラスマーク |
| `@Stable` | 安定性保証 |
| `lint.xml` | ルール設定 |

- Compose Compilerレポートで不安定な引数を特定
- `@Immutable`/`@Stable`で安定性を明示
- `UnrememberedMutableState`は必ずerrorに設定
- `./gradlew lint`でCI/CDに統合

---

8種類のAndroidアプリテンプレート（Lint設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Profiler](https://zenn.dev/myougatheaxo/articles/android-compose-compose-profiler-2026)
- [Compose BaselineProfile](https://zenn.dev/myougatheaxo/articles/android-compose-compose-baseline-profile-2026)
- [Compose Benchmark](https://zenn.dev/myougatheaxo/articles/android-compose-compose-benchmark-2026)
