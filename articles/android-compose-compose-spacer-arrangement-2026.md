---
title: "Compose Arrangement完全ガイド — spacedBy/SpaceBetween/SpaceEvenly/カスタム配置"
emoji: "↔️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

**Compose Arrangement**（spacedBy、SpaceBetween、SpaceEvenly、SpaceAround、カスタムArrangement）を解説します。

---

## 基本Arrangement

```kotlin
@Composable
fun ArrangementDemo() {
    val items = listOf("A", "B", "C")

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        Text("spacedBy(16.dp)", fontWeight = FontWeight.Bold)
        Row(Modifier.fillMaxWidth().background(Color(0xFFE3F2FD)), horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            items.forEach { Box(Modifier.size(50.dp).background(Color.Blue), contentAlignment = Alignment.Center) { Text(it, color = Color.White) } }
        }

        Text("SpaceBetween", fontWeight = FontWeight.Bold)
        Row(Modifier.fillMaxWidth().background(Color(0xFFFCE4EC)), horizontalArrangement = Arrangement.SpaceBetween) {
            items.forEach { Box(Modifier.size(50.dp).background(Color.Red), contentAlignment = Alignment.Center) { Text(it, color = Color.White) } }
        }

        Text("SpaceEvenly", fontWeight = FontWeight.Bold)
        Row(Modifier.fillMaxWidth().background(Color(0xFFF3E5F5)), horizontalArrangement = Arrangement.SpaceEvenly) {
            items.forEach { Box(Modifier.size(50.dp).background(Color.Magenta), contentAlignment = Alignment.Center) { Text(it, color = Color.White) } }
        }

        Text("SpaceAround", fontWeight = FontWeight.Bold)
        Row(Modifier.fillMaxWidth().background(Color(0xFFE8F5E9)), horizontalArrangement = Arrangement.SpaceAround) {
            items.forEach { Box(Modifier.size(50.dp).background(Color.Green), contentAlignment = Alignment.Center) { Text(it, color = Color.White) } }
        }
    }
}
```

---

## Column Arrangement

```kotlin
@Composable
fun VerticalArrangementDemo() {
    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        verticalArrangement = Arrangement.SpaceBetween
    ) {
        Text("上部コンテンツ", style = MaterialTheme.typography.headlineSmall)

        Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
            Text("中央コンテンツ1")
            Text("中央コンテンツ2")
        }

        Button(onClick = {}, Modifier.fillMaxWidth()) {
            Text("下部ボタン")
        }
    }
}
```

---

## spacedBy + Alignment

```kotlin
@Composable
fun SpacedByWithAlignment() {
    Row(
        modifier = Modifier.fillMaxWidth().height(100.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp, Alignment.End),
        verticalAlignment = Alignment.CenterVertically
    ) {
        TextButton(onClick = {}) { Text("キャンセル") }
        Button(onClick = {}) { Text("保存") }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `spacedBy(dp)` | 固定間隔 |
| `SpaceBetween` | 両端揃え均等配置 |
| `SpaceEvenly` | 完全均等配置 |
| `SpaceAround` | 周囲均等配置 |

- `spacedBy`で固定間隔の配置
- `SpaceBetween`で両端揃え（ヘッダーUI等）
- `spacedBy`に`Alignment`引数で余白の配置方向を指定
- Row/Column両方で使用可能

---

8種類のAndroidアプリテンプレート（レイアウト最適化対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Weight](https://zenn.dev/myougatheaxo/articles/android-compose-compose-weight-2026)
- [Compose FlowColumn](https://zenn.dev/myougatheaxo/articles/android-compose-compose-flow-column-2026)
- [Compose IntrinsicSize](https://zenn.dev/myougatheaxo/articles/android-compose-compose-intrinsic-size-2026)
