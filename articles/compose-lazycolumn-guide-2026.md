---
title: "LazyColumn完全ガイド — Composeのリスト表示を最適化する"
emoji: "📜"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

Composeでリストを表示する`LazyColumn`。RecyclerViewの後継で、はるかにシンプルに書けます。基本から最適化まで解説します。

---

## 基本のLazyColumn

```kotlin
@Composable
fun HabitList(habits: List<Habit>) {
    LazyColumn {
        items(habits) { habit ->
            HabitItem(habit)
        }
    }
}
```

これだけでスクロール可能なリストが完成。画面外のアイテムは自動で破棄・再利用されるため、メモリ効率が良い。

---

## items vs itemsIndexed

```kotlin
LazyColumn {
    // インデックス不要
    items(habits) { habit ->
        HabitItem(habit)
    }

    // インデックスが必要な場合
    itemsIndexed(habits) { index, habit ->
        Text("${index + 1}. ${habit.name}")
    }
}
```

---

## key パラメータ（重要）

```kotlin
LazyColumn {
    items(
        items = habits,
        key = { habit -> habit.id }  // ← 必ず指定
    ) { habit ->
        HabitItem(habit)
    }
}
```

**`key`を指定しないと**：アイテムの追加・削除時にリスト全体が再描画される。

**`key`を指定すると**：変更があったアイテムだけが再描画される。**パフォーマンスが大幅に向上**。

---

## ヘッダー付きリスト

```kotlin
LazyColumn {
    stickyHeader {
        Surface(
            color = MaterialTheme.colorScheme.primaryContainer,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text(
                "今日の習慣",
                modifier = Modifier.padding(16.dp),
                style = MaterialTheme.typography.titleMedium
            )
        }
    }

    items(todayHabits, key = { it.id }) { habit ->
        HabitItem(habit)
    }

    stickyHeader {
        Surface(
            color = MaterialTheme.colorScheme.secondaryContainer,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text(
                "完了済み",
                modifier = Modifier.padding(16.dp),
                style = MaterialTheme.typography.titleMedium
            )
        }
    }

    items(completedHabits, key = { it.id }) { habit ->
        HabitItem(habit)
    }
}
```

`stickyHeader`はスクロール時に画面上部に固定されます。

---

## LazyRow（横スクロール）

```kotlin
LazyRow(
    horizontalArrangement = Arrangement.spacedBy(8.dp),
    contentPadding = PaddingValues(horizontal = 16.dp)
) {
    items(categories, key = { it.id }) { category ->
        FilterChip(
            selected = category.isSelected,
            onClick = { onCategoryClick(category) },
            label = { Text(category.name) }
        )
    }
}
```

カテゴリフィルターやタグ表示に最適。

---

## スワイプ削除

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SwipeableHabitItem(habit: Habit, onDelete: () -> Unit) {
    val dismissState = rememberSwipeToDismissBoxState(
        confirmValueChange = { value ->
            if (value == SwipeToDismissBoxValue.EndToStart) {
                onDelete()
                true
            } else false
        }
    )

    SwipeToDismissBox(
        state = dismissState,
        backgroundContent = {
            Box(
                Modifier
                    .fillMaxSize()
                    .background(MaterialTheme.colorScheme.error)
                    .padding(16.dp),
                contentAlignment = Alignment.CenterEnd
            ) {
                Icon(Icons.Default.Delete, "削除", tint = Color.White)
            }
        }
    ) {
        HabitItem(habit)
    }
}
```

---

## パフォーマンスTips

| やること | 理由 |
|---------|------|
| `key`パラメータを指定 | 差分更新を有効化 |
| `Modifier.fillMaxWidth()`を最初に | レイアウト計算の最適化 |
| 重いComposableは`remember`で囲む | 不要な再計算を防止 |
| 画像は`coil`の`AsyncImage`で | 非同期ロード+キャッシュ |
| `contentPadding`を使う | パディングもスクロール範囲に含まれる |

---

## まとめ

- `LazyColumn`はRecyclerViewの後継（コード量1/3）
- `key`パラメータは必ず指定する
- `stickyHeader`でセクション分け
- `LazyRow`で横スクロール
- スワイプ削除は`SwipeToDismissBox`

---

8種類のAndroidアプリテンプレート（全てLazyColumn使用）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Navigation入門](https://zenn.dev/myougatheaxo/articles/compose-navigation-guide-2026)
- [Compose Animation入門](https://zenn.dev/myougatheaxo/articles/compose-animation-basics-2026)
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [Material3テーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/material3-theme-guide-2026)
