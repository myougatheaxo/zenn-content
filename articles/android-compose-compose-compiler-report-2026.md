---
title: "Compose Compiler Report完全ガイド — 安定性分析/リコンポジション最適化"
emoji: "📊"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "performance"]
published: true
---

## この記事で学べること

**Compose Compiler Report**（安定性分析、Skippable/Restartable、不安定な型の修正、パフォーマンス最適化）を解説します。

---

## レポート生成

```kotlin
// build.gradle.kts
kotlin {
    compilerOptions {
        freeCompilerArgs.addAll(
            "-P", "plugin:androidx.compose.compiler.plugins.kotlin:reportsDestination=${layout.buildDirectory.get()}/compose_reports",
            "-P", "plugin:androidx.compose.compiler.plugins.kotlin:metricsDestination=${layout.buildDirectory.get()}/compose_metrics"
        )
    }
}
```

```bash
# レポート生成
./gradlew assembleRelease

# 生成されるファイル:
# build/compose_reports/app_release-composables.txt
# build/compose_reports/app_release-classes.txt
# build/compose_metrics/app_release-module.json
```

---

## レポートの読み方

```
// composables.txt の例
restartable skippable scheme("[androidx.compose.ui.UiComposable]") fun HomeScreen(
  stable viewModel: HomeViewModel
)

// ✅ restartable skippable = 最適（スキップ可能）
// ⚠️ restartable = スキップ不可（毎回リコンポジション）
// ❌ unstable パラメータがある = 常にリコンポジション

// classes.txt の例
unstable class UserState {
  unstable val users: List<User>  // ❌ List は unstable
  stable val isLoading: Boolean
}
```

---

## 不安定な型の修正

```kotlin
// ❌ unstable: 通常のList
data class HomeState(
    val items: List<Item>,  // unstable
    val isLoading: Boolean
)

// ✅ stable: ImmutableList
@Immutable
data class HomeState(
    val items: ImmutableList<Item>,  // stable
    val isLoading: Boolean
)

// ❌ unstable: 外部ライブラリの型
data class MapState(
    val location: LatLng  // 外部ライブラリの型 = unstable
)

// ✅ @Stable アノテーション
@Stable
data class MapState(
    val latitude: Double,
    val longitude: Double
)

// ❌ unstable: MutableState を公開
class CounterViewModel : ViewModel() {
    var count by mutableStateOf(0)  // OK（Compose runtime型）
    val items = mutableListOf<String>()  // ❌ unstable
}
```

---

## Stability Configuration

```
// compose_compiler_config.conf（プロジェクトルート）
// 外部ライブラリの型をstableとして扱う
com.google.android.gms.maps.model.LatLng
java.time.LocalDate
java.time.LocalDateTime
kotlinx.datetime.Instant
```

```kotlin
// build.gradle.kts
kotlin {
    compilerOptions {
        freeCompilerArgs.addAll(
            "-P", "plugin:androidx.compose.compiler.plugins.kotlin:stabilityConfigurationPath=${rootDir}/compose_compiler_config.conf"
        )
    }
}
```

---

## リコンポジション可視化

```kotlin
// デバッグ用: リコンポジション回数を表示
@Composable
fun RecompositionCounter(label: String) {
    val count = remember { mutableIntStateOf(0) }
    count.intValue++

    if (BuildConfig.DEBUG) {
        Text(
            "$label: ${count.intValue}",
            fontSize = 10.sp,
            color = Color.Red,
            modifier = Modifier.padding(2.dp)
        )
    }
}
```

---

## まとめ

| 状態 | 意味 | 対策 |
|------|------|------|
| `skippable` | スキップ可能 | ✅ 最適 |
| `restartable` only | スキップ不可 | パラメータを安定化 |
| `unstable` | 不安定な型 | `@Immutable`/`@Stable` |

- Compiler Reportで安定性を定量分析
- `ImmutableList`で`List`を安定化
- `@Stable`で外部型をマーク
- Stability Configで外部ライブラリの型を設定

---

8種類のAndroidアプリテンプレート（パフォーマンス最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Stability/Performance](https://zenn.dev/myougatheaxo/articles/android-compose-stability-performance-2026)
- [Macrobenchmark](https://zenn.dev/myougatheaxo/articles/android-compose-benchmark-macro-2026)
- [デバッグツール](https://zenn.dev/myougatheaxo/articles/android-compose-debug-tools-2026)
