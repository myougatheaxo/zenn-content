---
title: "Compose ImageVector完全ガイド — カスタムアイコン/SVGパス/ベクタ描画"
emoji: "🖼️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Compose ImageVector**（カスタムアイコン作成、SVGパスからベクタ生成、プログラム的アイコン描画）を解説します。

---

## 基本ImageVector

```kotlin
val CustomIcon: ImageVector
    get() = ImageVector.Builder(
        name = "Custom",
        defaultWidth = 24.dp,
        defaultHeight = 24.dp,
        viewportWidth = 24f,
        viewportHeight = 24f
    ).apply {
        path(fill = SolidColor(Color.Black)) {
            moveTo(12f, 2f)
            lineTo(20f, 20f)
            lineTo(4f, 20f)
            close()
        }
    }.build()

@Composable
fun IconDemo() {
    Icon(
        imageVector = CustomIcon,
        contentDescription = "カスタムアイコン",
        modifier = Modifier.size(48.dp),
        tint = MaterialTheme.colorScheme.primary
    )
}
```

---

## 複数パスアイコン

```kotlin
val HeartIcon: ImageVector
    get() = ImageVector.Builder(
        name = "Heart",
        defaultWidth = 24.dp,
        defaultHeight = 24.dp,
        viewportWidth = 24f,
        viewportHeight = 24f
    ).apply {
        path(
            fill = SolidColor(Color(0xFFE91E63)),
            stroke = SolidColor(Color(0xFFC2185B)),
            strokeLineWidth = 1f
        ) {
            moveTo(12f, 21.35f)
            curveToRelative(-0.55f, -0.5f, -9f, -7.5f, -9f, -12.35f)
            arcToRelative(5f, 5f, 0f, false, true, 9f, -1f)
            arcToRelative(5f, 5f, 0f, false, true, 9f, 1f)
            curveToRelative(0f, 4.85f, -8.45f, 11.85f, -9f, 12.35f)
            close()
        }
    }.build()
```

---

## rememberVectorPainter

```kotlin
@Composable
fun VectorPainterDemo() {
    val painter = rememberVectorPainter(
        defaultWidth = 100.dp,
        defaultHeight = 100.dp,
        viewportWidth = 100f,
        viewportHeight = 100f
    ) { _, _ ->
        Path(
            pathData = PathData {
                moveTo(50f, 10f)
                lineTo(90f, 90f)
                lineTo(10f, 90f)
                close()
            },
            fill = SolidColor(Color.Blue)
        )
    }

    Image(
        painter = painter,
        contentDescription = null,
        modifier = Modifier.size(100.dp)
    )
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ImageVector.Builder` | ベクタアイコン構築 |
| `path {}` | SVGパス描画 |
| `rememberVectorPainter` | Composable内パス |
| `Icon()` | アイコン表示 |

- `ImageVector.Builder`でSVGライクなアイコンをコードで定義
- `path`内で`moveTo`/`lineTo`/`curveTo`で形状描画
- `SolidColor`でfill/stroke設定
- Material Iconsと同じ方法で使用可能

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Canvas](https://zenn.dev/myougatheaxo/articles/android-compose-compose-canvas-2026)
- [Compose CustomShape](https://zenn.dev/myougatheaxo/articles/android-compose-compose-custom-shape-2026)
- [Compose Path](https://zenn.dev/myougatheaxo/articles/android-compose-compose-path-2026)
