---
title: "カスタムComposable設計パターン — 再利用可能なUIコンポーネント"
emoji: "🧱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "component"]
published: true
---

## この記事で学べること

再利用可能な**カスタムComposable**の設計パターン（スロット、Modifier、状態管理）を解説します。

---

## スロットAPI

```kotlin
@Composable
fun AppCard(
    title: @Composable () -> Unit,
    subtitle: @Composable (() -> Unit)? = null,
    leading: @Composable (() -> Unit)? = null,
    trailing: @Composable (() -> Unit)? = null,
    modifier: Modifier = Modifier,
    onClick: (() -> Unit)? = null
) {
    Card(
        modifier = modifier
            .fillMaxWidth()
            .then(if (onClick != null) Modifier.clickable { onClick() } else Modifier)
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            leading?.invoke()
            leading?.let { Spacer(Modifier.width(12.dp)) }

            Column(Modifier.weight(1f)) {
                ProvideTextStyle(MaterialTheme.typography.titleMedium) {
                    title()
                }
                subtitle?.let {
                    Spacer(Modifier.height(4.dp))
                    ProvideTextStyle(MaterialTheme.typography.bodySmall) {
                        it()
                    }
                }
            }

            trailing?.let {
                Spacer(Modifier.width(12.dp))
                it()
            }
        }
    }
}

// 使用
AppCard(
    title = { Text("ユーザー名") },
    subtitle = { Text("user@example.com") },
    leading = { Icon(Icons.Default.Person, null) },
    trailing = { Icon(Icons.Default.ChevronRight, null) },
    onClick = { /* 遷移 */ }
)
```

---

## Modifier設計原則

```kotlin
// ✅ 良い設計: Modifierを引数で受け取る
@Composable
fun StatusBadge(
    text: String,
    color: Color,
    modifier: Modifier = Modifier // デフォルト値はModifier
) {
    Text(
        text = text,
        modifier = modifier
            .background(color.copy(alpha = 0.1f), RoundedCornerShape(4.dp))
            .padding(horizontal = 8.dp, vertical = 2.dp),
        color = color,
        style = MaterialTheme.typography.labelSmall
    )
}

// ❌ 悪い設計: 内部でサイズ固定
@Composable
fun BadBadge(text: String) {
    Text(
        text = text,
        modifier = Modifier
            .size(100.dp) // 呼び出し側でサイズ変更不可
            .padding(8.dp)
    )
}
```

---

## 状態ホルダーパターン

```kotlin
// 状態ホルダー
@Stable
class ExpandableState(initialExpanded: Boolean = false) {
    var expanded by mutableStateOf(initialExpanded)
        private set

    fun toggle() { expanded = !expanded }
    fun expand() { expanded = true }
    fun collapse() { expanded = false }
}

@Composable
fun rememberExpandableState(initialExpanded: Boolean = false): ExpandableState {
    return remember { ExpandableState(initialExpanded) }
}

// 使用
@Composable
fun ExpandableSection(
    title: String,
    state: ExpandableState = rememberExpandableState(),
    content: @Composable () -> Unit
) {
    Column {
        Row(
            Modifier
                .fillMaxWidth()
                .clickable { state.toggle() }
                .padding(16.dp)
        ) {
            Text(title, Modifier.weight(1f))
            Icon(
                if (state.expanded) Icons.Default.ExpandLess else Icons.Default.ExpandMore,
                null
            )
        }

        AnimatedVisibility(visible = state.expanded) {
            content()
        }
    }
}
```

---

## CompositionLocal

```kotlin
// アプリ全体で参照できる値
val LocalAppColors = staticCompositionLocalOf { AppColors() }

data class AppColors(
    val success: Color = Color(0xFF4CAF50),
    val warning: Color = Color(0xFFFF9800),
    val danger: Color = Color(0xFFF44336)
)

@Composable
fun AppTheme(content: @Composable () -> Unit) {
    CompositionLocalProvider(
        LocalAppColors provides AppColors()
    ) {
        MaterialTheme(content = content)
    }
}

// 使用
@Composable
fun StatusIcon(isSuccess: Boolean) {
    val colors = LocalAppColors.current
    Icon(
        Icons.Default.Circle,
        null,
        tint = if (isSuccess) colors.success else colors.danger
    )
}
```

---

## まとめ

- スロットAPIで柔軟なコンテンツ注入
- `modifier: Modifier = Modifier`を常に引数に
- `@Stable`状態ホルダーで複雑な状態をカプセル化
- `remember...State()`パターンで状態管理
- `CompositionLocalProvider`でツリー全体に値を提供
- `ProvideTextStyle`で子のテキストスタイルを設定

---

8種類のAndroidアプリテンプレート（コンポーネント設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [State Hoisting](https://zenn.dev/myougatheaxo/articles/android-compose-state-hoisting-2026)
- [Modifierチートシート](https://zenn.dev/myougatheaxo/articles/android-compose-modifier-cheatsheet-2026)
- [カスタムレイアウト](https://zenn.dev/myougatheaxo/articles/android-compose-custom-layout-2026)
