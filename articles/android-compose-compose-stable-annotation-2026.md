---
title: "Stable/Immutable完全ガイド — @Stable/@Immutable/StabilityConfig/パフォーマンス"
emoji: "🔒"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "performance"]
published: true
---

## この記事で学べること

**Stable/Immutable**（@Stable、@Immutable、Stability Config、Skippable最適化）を解説します。

---

## @Immutableと@Stable

```kotlin
// @Immutable: 全プロパティが不変であることを保証
@Immutable
data class UserProfile(
    val id: Long,
    val name: String,
    val avatarUrl: String
)

// @Stable: 変更があれば通知されることを保証
@Stable
class CounterState {
    var count by mutableIntStateOf(0)
        private set

    fun increment() { count++ }
}

// Skippableなコンポーザブル（引数が全てStable）
@Composable
fun UserCard(profile: UserProfile, modifier: Modifier = Modifier) {
    Card(modifier) {
        Row(Modifier.padding(16.dp)) {
            AsyncImage(model = profile.avatarUrl, contentDescription = null)
            Spacer(Modifier.width(12.dp))
            Text(profile.name, style = MaterialTheme.typography.titleMedium)
        }
    }
}
```

---

## ImmutableListでSkippable化

```kotlin
// ❌ List<T>はUnstable → 毎回リコンポーズ
@Composable
fun TagList(tags: List<String>) {
    FlowRow { tags.forEach { Text(it) } }
}

// ✅ ImmutableListでSkippable化
@Composable
fun TagList(tags: ImmutableList<String>) {
    FlowRow {
        tags.forEach { tag ->
            AssistChip(onClick = {}, label = { Text(tag) })
        }
    }
}

// 変換
val tags = listOf("Kotlin", "Compose").toImmutableList()
```

---

## Stability Configuration

```kotlin
// stability_config.conf
// java.time.LocalDate
// java.time.LocalDateTime
// kotlinx.datetime.Instant

// build.gradle.kts
// composeCompiler {
//     stabilityConfigurationFile =
//         rootProject.layout.projectDirectory.file("stability_config.conf")
// }

// レポート出力でStability確認
// composeCompiler {
//     reportsDestination = layout.buildDirectory.dir("compose_reports")
// }
```

---

## まとめ

| アノテーション | 意味 |
|-------------|------|
| `@Immutable` | 全フィールド不変 |
| `@Stable` | 変更時にCompose通知 |
| `ImmutableList` | 不変コレクション |
| `StabilityConfig` | 外部型をStable扱い |

- `@Immutable`/`@Stable`でSkippableなリコンポーズ
- `ImmutableList`でコレクション引数もSkippable
- Compose Compilerレポートで安定性を確認
- Stability Configで外部ライブラリの型を安定化

---

8種類のAndroidアプリテンプレート（パフォーマンス最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Recomposition](https://zenn.dev/myougatheaxo/articles/android-compose-compose-recomposition-2026)
- [State Management](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-management-2026)
- [MacroBenchmark](https://zenn.dev/myougatheaxo/articles/android-compose-macro-benchmark-2026)
