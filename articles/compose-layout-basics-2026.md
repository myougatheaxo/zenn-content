---
title: "Composeレイアウト入門 — Column, Row, Boxの使い分けを図解する"
emoji: "📐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

Jetpack Composeのレイアウトは3つの基本要素で構成されます。XMLの`LinearLayout`や`FrameLayout`を忘れて、**Column、Row、Box**だけ覚えれば大丈夫です。

---

## 3つの基本レイアウト

### Column — 縦に並べる

```kotlin
Column {
    Text("1行目")
    Text("2行目")
    Text("3行目")
}
```

```
┌──────────┐
│ 1行目     │
│ 2行目     │
│ 3行目     │
└──────────┘
```

### Row — 横に並べる

```kotlin
Row {
    Text("左")
    Text("中央")
    Text("右")
}
```

```
┌──────────────────┐
│ 左  中央  右      │
└──────────────────┘
```

### Box — 重ねる

```kotlin
Box {
    Image(painter = painterResource(R.drawable.bg), contentDescription = null)
    Text("画像の上にテキスト", modifier = Modifier.align(Alignment.Center))
}
```

```
┌──────────────────┐
│    ┌────────┐    │
│    │ テキスト │    │
│    └────────┘    │
│   （画像の上）     │
└──────────────────┘
```

---

## Modifier — レイアウトの調整

```kotlin
Text(
    "Hello",
    modifier = Modifier
        .fillMaxWidth()           // 横幅いっぱい
        .padding(16.dp)           // 内側に余白
        .background(Color.Gray)   // 背景色
        .clickable { /* ... */ }  // タップ可能
)
```

### よく使うModifier

| Modifier | 効果 |
|---------|------|
| `.fillMaxWidth()` | 横幅いっぱい |
| `.fillMaxSize()` | 縦横いっぱい |
| `.padding(16.dp)` | 内側余白 |
| `.size(48.dp)` | 固定サイズ |
| `.weight(1f)` | 残りスペースを分配 |
| `.background(color)` | 背景色 |
| `.clip(RoundedCornerShape(8.dp))` | 角丸 |

**重要**: Modifierの順序は結果に影響します。`padding → background`と`background → padding`で見た目が変わります。

---

## Arrangement — 子要素の配置

### Column の verticalArrangement

```kotlin
Column(
    verticalArrangement = Arrangement.SpaceBetween,
    modifier = Modifier.fillMaxHeight()
) {
    Text("上")
    Text("中")
    Text("下")
}
```

| Arrangement | 効果 |
|------------|------|
| `Top` | 上詰め（デフォルト） |
| `Center` | 中央寄せ |
| `Bottom` | 下詰め |
| `SpaceBetween` | 均等配置（両端ぴったり） |
| `SpaceEvenly` | 均等配置（両端にも余白） |
| `SpaceAround` | 均等配置（両端は半分の余白） |

### Row の horizontalArrangement

同じオプションが横方向に適用されます。`Start`、`Center`、`End`、`SpaceBetween`等。

---

## Alignment — 交差軸の配置

```kotlin
Row(
    verticalAlignment = Alignment.CenterVertically
) {
    Icon(Icons.Default.Check, contentDescription = null)
    Text("テキストとアイコンが縦中央揃え")
}
```

---

## weight — 残りスペースの分配

```kotlin
Row(Modifier.fillMaxWidth()) {
    Text("固定", modifier = Modifier.width(80.dp))
    Text("残り全部", modifier = Modifier.weight(1f))
    IconButton(onClick = {}) {
        Icon(Icons.Default.Delete, "削除")
    }
}
```

`weight(1f)`は「残りのスペースを全て使う」という意味。複数のweightがある場合は比率で分配。

---

## Spacer — 余白

```kotlin
Row {
    Text("左のテキスト")
    Spacer(Modifier.weight(1f))  // 間を最大に広げる
    Text("右のテキスト")
}
```

`Spacer(Modifier.weight(1f))`は「左右のテキストを端に寄せる」定番パターン。

---

## Scaffold — 画面全体の骨格

```kotlin
Scaffold(
    topBar = {
        TopAppBar(title = { Text("マイアプリ") })
    },
    floatingActionButton = {
        FloatingActionButton(onClick = { }) {
            Icon(Icons.Default.Add, "追加")
        }
    },
    bottomBar = {
        NavigationBar { /* ... */ }
    }
) { padding ->
    // メインコンテンツ
    LazyColumn(Modifier.padding(padding)) {
        // ...
    }
}
```

---

## 実践パターン：リストアイテム

```kotlin
@Composable
fun HabitItem(habit: Habit, onToggle: () -> Unit) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp, vertical = 4.dp)
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Checkbox(
                checked = habit.completed,
                onCheckedChange = { onToggle() }
            )
            Spacer(Modifier.width(8.dp))
            Column(Modifier.weight(1f)) {
                Text(habit.name, style = MaterialTheme.typography.bodyLarge)
                Text("作成日: ${habit.dateString}", style = MaterialTheme.typography.bodySmall)
            }
            IconButton(onClick = { /* 削除 */ }) {
                Icon(Icons.Default.Delete, "削除")
            }
        }
    }
}
```

---

## まとめ

- **Column** = 縦並び、**Row** = 横並び、**Box** = 重ねる
- **Modifier**でサイズ・余白・背景を調整
- **Arrangement**で配置、**Alignment**で揃え
- **weight**で残りスペース分配、**Spacer**で余白
- **Scaffold**で画面全体の骨格を定義

---

8種類のAndroidアプリテンプレート（全てCompose UI）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyColumn完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazycolumn-guide-2026)
- [Compose Animation入門](https://zenn.dev/myougatheaxo/articles/compose-animation-basics-2026)
- [Compose Navigation入門](https://zenn.dev/myougatheaxo/articles/compose-navigation-guide-2026)
- [Material3テーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/material3-theme-guide-2026)
