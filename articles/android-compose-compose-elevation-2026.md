---
title: "Compose Elevation完全ガイド — Shadow/TonalElevation/Material3/カスタムシャドウ"
emoji: "🏔️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose Elevation**（Shadow、TonalElevation、Material3のエレベーションシステム、カスタムシャドウ）を解説します。

---

## Material3エレベーション

```kotlin
@Composable
fun ElevationShowcase() {
    val levels = listOf(0.dp, 1.dp, 3.dp, 6.dp, 8.dp, 12.dp)

    LazyVerticalGrid(columns = GridCells.Fixed(3), contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp),
        horizontalArrangement = Arrangement.spacedBy(16.dp)) {
        items(levels) { elevation ->
            Surface(
                tonalElevation = elevation,
                shadowElevation = elevation,
                shape = RoundedCornerShape(12.dp),
                modifier = Modifier.aspectRatio(1f)
            ) {
                Box(contentAlignment = Alignment.Center) {
                    Text("${elevation.value.toInt()}dp")
                }
            }
        }
    }
}
```

---

## Tonal Elevation

```kotlin
@Composable
fun TonalElevationDemo() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        // Tonal Elevation: 背景色が微妙に変化
        Surface(tonalElevation = 0.dp, modifier = Modifier.fillMaxWidth().height(60.dp)) {
            Box(contentAlignment = Alignment.Center) { Text("Level 0 (Surface)") }
        }
        Surface(tonalElevation = 1.dp, modifier = Modifier.fillMaxWidth().height(60.dp)) {
            Box(contentAlignment = Alignment.Center) { Text("Level 1") }
        }
        Surface(tonalElevation = 3.dp, modifier = Modifier.fillMaxWidth().height(60.dp)) {
            Box(contentAlignment = Alignment.Center) { Text("Level 3") }
        }
        Surface(tonalElevation = 6.dp, modifier = Modifier.fillMaxWidth().height(60.dp)) {
            Box(contentAlignment = Alignment.Center) { Text("Level 6") }
        }
    }
}
```

---

## カスタムシャドウ

```kotlin
@Composable
fun CustomShadowCard() {
    Box(
        Modifier.fillMaxWidth().padding(24.dp)
            .shadow(
                elevation = 8.dp,
                shape = RoundedCornerShape(16.dp),
                ambientColor = Color(0xFF6200EE).copy(alpha = 0.3f),
                spotColor = Color(0xFF6200EE).copy(alpha = 0.3f)
            )
            .background(Color.White, RoundedCornerShape(16.dp))
            .padding(24.dp)
    ) {
        Column {
            Text("カスタムシャドウ", style = MaterialTheme.typography.titleMedium)
            Text("紫色のシャドウ", color = MaterialTheme.colorScheme.onSurfaceVariant)
        }
    }
}

// graphicsLayerでのシャドウ
@Composable
fun GraphicsLayerShadow() {
    Box(
        Modifier.size(120.dp)
            .graphicsLayer {
                shadowElevation = 12f
                shape = RoundedCornerShape(16.dp)
                clip = true
            }
            .background(MaterialTheme.colorScheme.surface),
        contentAlignment = Alignment.Center
    ) { Text("12dp") }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `shadowElevation` | シャドウ |
| `tonalElevation` | 色調変化 |
| `Modifier.shadow` | カスタムシャドウ |
| `graphicsLayer` | 低レベル制御 |

- Material3はTonal Elevationで色調変化を表現
- `shadowElevation`で従来のドロップシャドウ
- `Modifier.shadow`でカスタム色のシャドウ
- ダークモードではTonal Elevationがより効果的

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Shape](https://zenn.dev/myougatheaxo/articles/android-compose-compose-shape-2026)
- [Compose GraphicsLayer](https://zenn.dev/myougatheaxo/articles/android-compose-compose-graphicsLayer-2026)
- [Compose DynamicColor](https://zenn.dev/myougatheaxo/articles/android-compose-compose-dynamic-color-2026)
